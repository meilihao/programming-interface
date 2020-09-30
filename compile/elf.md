# elf
参考:
- [ELF Format](http://www.skyfree.org/linux/references/ELF_Format.pdf)
- [Linux内核之ELF格式解析](https://mudongliang.github.io/2015/10/31/linuxelf.html)
- [计算机那些事(4)——ELF文件结构](http://chuquan.me/2018/05/21/elf-introduce/)
- [ELF文件格式](https://www.cntofu.com/book/114/Theory/ELF.md)

ELF (Executeable and Linkable Format,可执行与可链接格式)是linux 下二进制可执行可链接文件的格式, 目前常见的Linux、 Android可执行文件、共享库（.so）、目标文件（ .o）以及Core 文件（吐核）均为此格式, 可通过`readelf -a xxx`查看.

[GNU Binutils(GNU Binary Utilities)](https://www.gnu.org/software/binutils/binutils.html)软件包里包含了一系列生成、解析和处理ELF文件的命令行工具.

![程序编译过程](/misc/img/compile/1671100-20190512202937314-1323961004.jpg)

![可执行程序的ELF](/misc/img/process/v2-85a5b44f20d53e6e992269dccc20ac6b_1200x500.jpg)

![一个简单的程序被编译成目标文件后的结构](/misc/img/compile/cpp_segment_example.jpg)

![目标文件各个段在文件中的布局](/misc/img/compile/cpp_segment_layout.png)

![一个ELF格式的简单示意图](/misc/img/compile/3-1.png)
左边是链接器的视角，ELF文件由若干个节(section)组成。右边是加载器的视角，这些节被习惯性称之为段(segment).

> 编译时生成的 .o（目标文件）以及链接后的 .so （共享库）均可通过链接视图解析
> ELF 规格也允许定义一个解释器(ELF 程序头部的 PT_INTERP 元素)来运行程序. 如果定义了解释器,内核则基于指定解释器可执行文件的各段来构建进程映像,转而由解释器负责加载和执行程序.

ELF 文件的头(elf header)是用于描述整个文件的, 包含了描述整个文件的基本属性, 可通过`readelf -h xxx`查看. 这个文件格式在内核中有定义,分别为 `[struct elf32_hdr](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L244) 和 [struct elf64_hdr](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L221)`:
```c
typedef struct elf64_hdr {
  unsigned char	e_ident[EI_NIDENT];	/* ELF "magic number" */
  Elf64_Half e_type; // elf文件的类型: DYN, 共享目标文件,比如.so; EXEC, 可执行文件; REL, 可重定位文件, 比如.o
  Elf64_Half e_machine; // Machine
  Elf64_Word e_version; // Version
  Elf64_Addr e_entry;		/* Entry point virtual address */ 虚拟地址,是这个程序运行的入口即_start符号的地址
  Elf64_Off e_phoff;		/* Program header table file offset */
  Elf64_Off e_shoff;		/* Section header table file offset */
  Elf64_Word e_flags;
  Elf64_Half e_ehsize;    // Size of this header
  Elf64_Half e_phentsize; // Size of program headers
  Elf64_Half e_phnum;     // Number of program headers
  Elf64_Half e_shentsize; // Size of section headers
  Elf64_Half e_shnum;     // Number of section headers
  Elf64_Half e_shstrndx;  // Section header string table index // .shstrtab在段表中的下标
} Elf64_Ehdr;
```

elf Magic(`7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00`)解析:
- 0x7f : elf标志
- 0x454c46 : "elf"的ascii码
- 0x01/0x02 : elf的class, 即32/64位
- 0x01/0x02 : 字节序, 0x01表示小端
- 0x01 : elf文件的主版本号
- 后续的9个字节elf标准未定义, 一般用0填充, 或作为自定义的扩展标志

> [elf标准, v1.2后未更新](https://refspecs.linuxfoundation.org/elf/elf.pdf)

### 可重定位文件 (Relocatable File),
即编译时生成的`.o`文件, ELF 的一种类型

节头部表(Section Header Table)是sections的元数据, 在代码里面的定义是`struct elf32_shdr 和 [struct elf64_shdr](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L316)`

```c
typedef struct elf64_shdr {
  Elf64_Word sh_name;   /* Section name, index in string tbl */ sh_name是段名在.shstrtab中的偏移量
  Elf64_Word sh_type;   /* Type of section */
  Elf64_Xword sh_flags;   /* Miscellaneous section attributes */
  Elf64_Addr sh_addr;   /* Section virtual addr at execution */ 如果该段可以被加载, 则sh_addr为该段被加载后在进程地址空间中的虚拟地址
  Elf64_Off sh_offset;    /* Section file offset */ 如果该段存在于文件中, 则表示在文件中的偏移; 否则无意义, 比如对bss段
  Elf64_Xword sh_size;    /* Size of section in bytes */
  Elf64_Word sh_link;   /* Index of another section */
  Elf64_Word sh_info;   /* Additional section information */ sh_link和sh_info仅在段类型是与链接相关的, 比如重定位表, 符号表等情况下有意义.
  Elf64_Xword sh_addralign; /* Section alignment */ 某些段对段地址对齐有要求, 即sh_addr % (2^sh_addralign) =0
  Elf64_Xword sh_entsize; /* Entry size if section holds table */ 某些段包含了一些固定大小的项, 比如符号表, 此时表示每个项的大小. 如果为0, 表示该段不包含固定大小的项
} Elf64_Shdr;

```

段名只在链接和编译中有意义, 但它不能真正地表示段的类型, 真正决定段属性的是[sh_type](https://refspecs.linuxfoundation.org/LSB_5.0.0/LSB-Core-generic/LSB-Core-generic/sections.html)和sh_flags.

sh_type:
- SHT_DYNAMIC, 0x6 : 动态链接信息. 一个目标文件应只有一个该节, 但是将来可以放宽此限制
- SHT_DYNSYM,  0xb : 动态链接符号表. 当前目标文件可以具有SHT_SYMTAB类型的部分或SHT_DYNSYM类型的部分，但不能同时具有两者, 将来可能会放宽此限制
- SHT_FINI_ARRAY,  0xf : 包含指向终止函数的指针数组, 数组中的每个指针都被视为无参数的过程，并且返回void
- SHT_HASH,  0x5 : 一个符号哈希表. 当前一个目标文件应仅具有一个哈希表，但是将来可以放宽此限制
- SHT_INIT_ARRAY,  0xe :  指向初始化函数的指针的数组，数组中的每个指针都被视为无参数的过程，并且返回void
- SHT_NOBITS,  0x8 : 该段在文件中没有内容(比如.bss)，但与SHT_PROGBITS相似, 尽管本节不包含任何字节，但sh_offset成员包含概念性文件偏移量
- SHT_NOTE,  0x7 : 包含以某种方式标记文件的信息
- SHT_NULL,  0x0 : 无效段
- SHT_PREINIT_ARRAY, 0x10 : 包含指向在所有其他初始化函数之前调用的函数的指针数组，数组中的每个指针都被视为无参数的过程，且返回值是空的
- SHT_PROGBITS,  0x1 : 程序段, 即保存程序定义的信息，其格式和含义仅由程序确定. 数据段, 代码段都是这种类型
- SHT_REL, 0x9 : 重定位表, 包含重定位信息, 没有显式加载项的重定位条目. 一个目标文件可能具有多个重定位部分
- SHT_RELA,  0x4 :  重定位表, 包含重定位信息, 有显式加载项的重定位条目. 一个目标文件可能具有多个重定位部分
- SHT_STRTAB,  0x3 : 字符串表, 一个目标文件可能具有多个字符串表节
- SHT_SYMTAB,  0x2 : 符号表, 当前目标文件可以具有SHT_SYMTAB类型的部分或SHT_DYNSYM类型的部分，但不能同时具有两者, 将来可能会放宽此限制. 通常，SHT_SYMTAB提供用于链接编辑的符号，尽管它也可以用于动态链接. 作为完整的符号表，它可能包含许多动态链接不需要的符号.

sh_flags会在`readelf -S xxx`的尾部显示:
- alloc : 该段在进程空间需要分配空间
- execute : 在进程空间可执行, 通常指代码段
- write : 在进程空间可写

文件的各section:
- .init : 程序初始化代码段，在main()之前运行
- .fini : 程序终结代码段
- .text(代码段):放编译好的部分二进制可执行代码,比如各种函数, 进程代码段不仅包括`.text`, 还有`.init`, `.fini`等.

    这部分区域的大小在程序运行前就已经确定，并且内存区域通常属于只读，某些架构也允许代码段为可写，即允许修改程序.
- .data(数据段): 存储了已初始化的全局变量，但初始化为0的全局变量出于编译优化的策略还是被保存在`.bss`中, 属于静态内存分配.
- .bss(Block Started by Symbol): 存储了**未初始化的全局变量或者是默认初始化为0的全局变量**, 且不占空间, 属于静态内存分配, 因此比`.data`节省空间
    `.bss`的size仅表示程序加载器在加载程序时候, 需要为该段分配的空间大小

    可通过`objdump -h *.o`查看
- .rodata: 该段也叫常量区(只读)，用于存放常量数据, 比如字符串常量, 全局const变量和#define定义的常量.

    特殊:
    1. 部分立即数会直接存放在`.text`中
    1. 对于字符串常量，编译器会去掉重复的常量，让程序的每个字符串常量只有一份
    1. `.rodata`是在多个进程间是共享的，这可以提高空间利用率
- .symtab:符号表,保存了符号信息, 基本信息可通过`readelf -s xxx`查看, 具体信息可通过`nm`或`readelf -s xxx`查看

  在链接中, 将函数和变量统称为符号(symbol), 函数名和变量名是符号名(symbol name).
  符号表中定义的每个符号有一个对应的值叫符号值. 对于变量和函数, 符号值就是它们的地址.

  ```c
  // https://elixir.bootlin.com/linux/v5.9-rc7/source/include/uapi/linux/elf.h#L193
  typedef struct elf64_sym {
    Elf64_Word st_name;   /* Symbol name, index in string tbl */ 符号名
    unsigned char st_info;  /* Type and binding attributes */  符号类型和绑定信息(低4bit是类型, 高28bit是符号绑定信息)
    unsigned char st_other; /* No defined meaning, 0 */ 未使用
    Elf64_Half st_shndx;    /* Associated section index */ 符号所在的段
    Elf64_Addr st_value;    /* Value of the symbol */ 符号值
    Elf64_Xword st_size;    /* Associated symbol size */ 符号大小. 对于明确数据类型的符号, 该值为其大小. 0表示该符号大小为0或未知
  } Elf64_Sym;
  ```

  符号绑定见[ELF64_ST_BIND](https://docs.oracle.com/cd/E19683-01/816-1386/chapter6-79797/index.html)
  符号类型见[ELF64_ST_TYPE](https://docs.oracle.com/cd/E19683-01/816-1386/chapter6-79797/index.html)

  符号修饰是为了防止符号名冲突. c++使用namespace解决多模块的符号冲突, `c++filt`可用于解析被修饰过的名称. c++中的`extern "C"`用于声明一个C的符号, 即其中的内容会当做c语言处理, 使用c语言的修改规则. `__cpluspulus`宏可以让声明同时兼容c和c++.
- .strtab:用于存储elf文件中用到的各种字符串, 比如存储变量名，函数名, 是被`.symtab`引用的符号名称

    例如, `char* szPath="/root"`, `void func()`的变量名szPath和函数名func就存储在`.strtab`段.
- .shstrtab: 用于保存section header中的段名. bss,text,data等段名存储在这里
- .comment :注释信息段
- .node.GUN-stack :堆栈提示段
- .debug: 一个调试符号表
- .eh_frame: 记录调试和异常处理时用到的信息
- 以`.rec`开头的 sections 里面装载了需要重定位的符号

    - .rel.text : 针对`.text`段的重定位表，还有rel.data(针对data段的重定位表).
- .rel.* : 比如.rel.text是针对".text"段的重定位表. 用于链接过程，做完链接后会被删除
- .plt/.got : 动态链接的跳转表和全局入口.
- .note.* : 编译器信息
- .line : 调试时的行号表, 即源代码行号与编译后指令的对应表
- .dynamic : 动态链接信息
- `.debug*/.rel.debug*` : 调试信息

  gcc使用`-g`参数生成的.o中就有不少debug相关的section. elfs文件使用DWARF(Debugging With Attributed Record Formats, 最新是v5, 通用是v4)组织调试信息. 在linux下可使用"strip"命令去除elf的调试信息.

> .data与.bss没有本质区别, 都是用于存放静态变量, 只是.data是已初始化过的静态数据, 而.bss程序是运行时会分配空间并置零的静态数据.

> static 声明的变量，无论它是全局变量还是在函数之中，只要是没有赋初值都存放在.bss段，如果赋了初值，则把它放在.data段.

> elf文件中可能存在多个同名的段.

> gcc允许通过`__attribute__((section("name")))`属性把相应的变量或函数放到以"name"作为段名的段中.

![.o文件的ELF](/misc/img/compile/1671100-20190512203047832-334199166.jpg)

#### 强符号和弱符号
在链接时经常会碰到重复定义, 多个目标文件中含有名字相同全局符号的定义，那么这些目标文件在链接的时候将会出现符号重复定义的错误, 此类符号被称之为强符号, 除此之外就是弱符号.

> `extern xxx`即非强符号也非弱符号，因为而是一个外部变量的引用

GCC通过`__attribute__((weak))`指令可将强符号的函数或变量定义为弱符号（Weak Symbol），实际上这个指令大部分时候都是用来定义函数，很少用于定义变量.

对于c/c++, 编译器默认函数和初始化了的全局变量为强符号（Strong Symbol），未初始化的全局变量为弱符号（Weak Symbol）. 强弱符号是针对定义来说而不是针对符号的引用.

使用弱符号的目的是，当不确定这个符号是否被定义的情况下，链接器也可以成功链接出ELF文件，适用于某些模块还未实现的情况下，其他模块的先行调试. 弱符号在C语言和C++语言的规范里面没有被提及，所以使用弱符号的代码，移植性不是非常好.

针对强弱符号的概念，链接器就会按照如下规则处理与选择被多次定义的全局符号：
- 不允许强符号被多次定义 (即不同的目标文件中不能有同名的强符号)，如果有多个强符号定义，则链接器报符号重复定义错误
- 如果一个符号在某个目标文件中是强符号，在其他文件中都是弱符号，那么选择强符号
- 如果一个符号在所有目标文件中都是弱符号，那么选择其中占用空间最大的一个, 主要是针对不用类型的变量而言；对于函数，其对应的符号相当于一个只读指针变量，而指针类型大小是固定的，不存在这个问题.

#### 强引用和弱引用
一些对外部文件的符号引用，如果在被链接成可执行文件时，没有找到相关的符号，那么链接器就会报符号未定义错误，这种被称为强应用 (Strong Reference).

与之相对应还有一种弱引用 (Weak Reference)，如果可以找到符号则连接，否则连接器会默认设置为 0 或者一个特殊值, 而不是报错.

> 注意，在声明弱引用时需要使用 static 关键字

> 对于弱符号（Weak Symbol）和弱引用，其都仅是GNU工具链GCC对C语言语法的扩展，并不是C本身的语言特性. 微软的MSVC就不支持弱符号和弱引用特性.

强引用和弱引用主要用于库的链接过程.

这种弱符号和弱引用对于库来说十分有用，比如库中定义的弱符号可以被用户定义的强符号所覆盖，从而使得程序可以使用自定义版本的库函数; 或者程序可以对某些扩展功能模块的引用定义为弱引用，当用户将扩展模块与程序链接在一起时，功能模块就可以正常使用；如果去掉了某些功能模块，那么程序也可以正常链接，只是缺少了相应的功能，这使得程序的功能更加容易裁剪和组合.

### 可执行文件(Executable File)
ELF 的第二种格式

这个格式和.o 文件大致相似,还是分成一个个的 section,并且被节头表描述. 只不过这些section 是多个.o 文件合并过的. 但是这个文件已经是可以加载到内存里面执行的文件了,因而这些 section 被分成了需要加载到内存里面的代码段、数据段和不需要加载到内存里面的部分,**将小的 section 合成了大的段 segment**,并且在最前面加一个段头表
(Segment Header Table). 在代码里面的定义为 `struct elf32_phdr 和 [struct elf64_phdr](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L255)`,这里面除了有对于段的描述之外,最重要的是 p_vaddr,这个是这个段加载到内存的虚拟地址. 在 ELF 头里面有一项 e_entry,也是个虚拟地址,是这个程序运行的入口.

```c
typedef struct elf64_phdr {
  Elf64_Word p_type;
  Elf64_Word p_flags;
  Elf64_Off p_offset;		/* Segment file offset */
  Elf64_Addr p_vaddr;		/* Segment virtual address */ 这个段加载到内存的虚拟地址
  Elf64_Addr p_paddr;		/* Segment physical address */
  Elf64_Xword p_filesz;		/* Segment size in file */
  Elf64_Xword p_memsz;		/* Segment size in memory */
  Elf64_Xword p_align;		/* Segment alignment, file & memory */
} Elf64_Phdr;
```

一个ELF文件被加载后，代码段、数据段以及堆栈便在虚拟地址空间的分配:
![](/misc/img/compile/3-3.png)

再细节一点就是下面这个图:
![](/misc/img/compile/3-3.png)

查看程序的虚拟地址空间: `cat /proc/self/maps | sort -n`. 更细节可参考: 《Linker && Loader》和《程序员的自我修养——链接、装载与库》.

#### 程序->进程
```c
// https://elixir.bootlin.com/linux/latest/source/include/linux/binfmts.h#L102
// kernel定义用来加载二进制文件的对象
struct linux_binfmt {
	struct list_head lh;
	struct module *module;
	int (*load_binary)(struct linux_binprm *);
	int (*load_shlib)(struct file *);
	int (*core_dump)(struct coredump_params *cprm);
	unsigned long min_coredump;	/* minimal dump size */
} __randomize_layout;

// https://elixir.bootlin.com/linux/latest/source/fs/binfmt_elf.c#L93
// 对于 ELF 文件格式，对应的linux_binfmt实现
static struct linux_binfmt elf_format = {
	.module		= THIS_MODULE,
	.load_binary	= load_elf_binary,
	.load_shlib	= load_elf_library,
	.core_dump	= elf_core_dump,
	.min_coredump	= ELF_EXEC_PAGESIZE,
};
```

![](/misc/img/compile/465b740b86ccc6ad3f8e38de25336bf6.jpg)

### 共享对象文件（Shared Object）

当一个动态链接库被链接到一个程序文件中的时候，最后的程序文件并不包括动态链接库中的代码，而仅仅包括对动态链接库的引用，并且不保存动态链接库的全路径，**仅仅保存动态链接库的名称.**

基于动态连接库创建出来的二进制文件格式:
- 多了一个.interp 的 Segment,这里面是 ld-linux.so(动态链接器), 即运行时的链接动作都是它做的
- ELF 文件中还多了两个 section,一个是.plt,过程链接表(Procedure Linkage Table,PLT),一个是.got.plt,全局偏移量表(Global Offset Table,GOT)

由于是运行时才去找,编译的时候,压根不知道被调用函数在哪里,所以就在 PLT 里面建立一项PLT[x]. 这一项也是一些代码,有点像一个本地的代理,在二进制程序里面,不直接调用
具体的被调函数,而是调用 PLT[x] 里面的代理代码,这个代理代码会在运行的时候找真正的被调函数.

查找具体被调函数的方法: GOT也会为被调函数创建一项 GOT[y], 它是运行时被调函数在内存中真正的地址. 如果这个地址在, 程序调用 PLT[x] 里面的代理代码, 代理代码调用 GOT 表中对应项 GOT[y],调用的就是加载到内存中的 xxx.so 里面的具体函数了.

GOT如何找到具体函数: GOT[y]开始时也没有地址, 但它又回调 PLT, 这个时候 PLT 会转而调用 PLT[0],也即第一项,PLT[0] 转而调用 GOT[2],这里面是 ld-linux.so 的入口函数,这个函数会找到加载到内存中的 xxx.so 里面的具体函数的地址,然后把这个地址放在 GOT[y] 里面. 下次,PLT[x] 的代理函数就能够直接调用了.

#### elf内存装载
一般32位可执行程序的默认加载首地址是：0x8048000，64位可执行程序的默认首加载地址是：0x400000, 可通过`ld --verbose`查看. `.text`会被加载到该地址, `.data`段紧随其后, 再之后是`.bss`.

### 静/动态链接(Shared Object File)
静态链接库一旦链接进去,代码和变量的 section 都合并了,因而程序运行的时候,就不依赖于这个库是否存在. 但是这样有一个缺点,就是相同的代码段,如果被多个程序使用的话,在内存里面就有多份,而且一旦静态链接库更新了,如果二进制执行文件不重新编译,也不随着更新.

因而就出现了另一种,动态链接库 (Shared Libraries),不仅仅是一组对象文件的简单归档,而
是多个对象文件的重新组合,可被多个程序共享.

> 动态链接库是 ELF 的第三种类型,共享对象文件 (Shared Object)
> 当运行有动态链接的程序时, 首先寻找动态链接库,然后加载它. 默认情况下,系统在 /lib 和/usr/lib 文件夹下寻找动态链接库. 如果找不到就会报错,我们也可以设定 LD_LIBRARY_PATH 环境
变量,程序运行时会在此环境变量指定的文件夹下寻找动态链接库.

### VMA和LMA
- VMA(virtual memory address): 程序区段在执行时期的地址
- LMA(load memory address): 某程序区段加载时的地址. 因为我们知道程序运行前要经过：编译、链接、装载、运行等过程, 那装载到哪里呢？ 实际上就是LMA对应的地址里.

一般情况下，LMA和VMA都是相等的，不等的情况主要发生在一些嵌入式系统上.

![](/misc/img/compile/cpp_lma_vma.jpg)

### DWARF(Debugging With Attributed Record Formats)
DWARF(Debugging With Attributed Record Formats, 最新版是实验性的v5, 常用是v4)是面向ELF文件的一种较新的调试信息格式，调试信息存储在对象文件的各个部分中，是可执行程序与源代码之间关系的简单表示.

`objdump -g code.o`可以查看调试信息的细节.

## FAQ
### 如何使用readelf和objdump解析elf
参考:
- [使用readelf和objdump解析目标文件](https://www.jianshu.com/p/863b279c941e)

在Linux下，使用gcc -c xxxx.c仅编译生成.o文件时, 便于解析.

目标文件只是ELF文件的可重定位文件(Relocatable file)，ELF文件一共有4种类型：Relocatable file、Executable file、Shared object file和Core Dump file.

> ELF文件结构信息定义在/usr/include/elf.h.

解析elf header: `readelf -h xxx` // Elf64_Ehdr, elf文件的信息
解析efl section: `readelf -S -W b.o`/`objdump -h xxx` // Elf64_Shdr, Section部分主要存放的是机器指令代码和数据
解析`.text`/`.data`/`.rodata`段: `objdump -s -d xxx`
解析`.bss`段: `objdump -x -s -d xxx` // 打印出目标文件的符号表，通过符号表我们可以知道各个变量的存放位置