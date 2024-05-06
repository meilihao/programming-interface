# elf
参考:
- [ELF Format](http://www.skyfree.org/linux/references/ELF_Format.pdf)
- [Linux内核之ELF格式解析](https://mudongliang.github.io/2015/10/31/linuxelf.html)
- [计算机那些事(4)——ELF文件结构](http://chuquan.me/2018/05/21/elf-introduce/)
- [ELF文件格式](https://www.cntofu.com/book/114/Theory/ELF.md)
- [**二进制文件是什么样的？**](https://www.tuicool.com/articles/QJBZ7br)

ELF (Executeable and Linkable Format,可执行与可链接格式)是linux 下二进制可执行可链接文件的格式, 可以分为三种类型可重定位的目标文件(Relocatable File,或者Object File)、可执行文件(Executable)和共享库(Shared Object,或者Shared Library).

目前常见的Linux、 Android可执行文件、共享库（.so）、目标文件（ .o）以及Core 文件（吐核）均为此格式, 可通过`readelf -a xxx`查看.

[GNU Binutils(GNU Binary Utilities)](https://www.gnu.org/software/binutils/binutils.html)软件包里包含了一系列生成、解析和处理ELF文件的命令行工具.

![程序编译过程](/misc/img/compile/1671100-20190512202937314-1323961004.jpg)

![可执行程序的ELF](/misc/img/process/v2-85a5b44f20d53e6e992269dccc20ac6b_1200x500.jpg)

![一个简单的程序被编译成目标文件后的结构](/misc/img/compile/cpp_segment_example.jpg)

![目标文件各个段在文件中的布局](/misc/img/compile/cpp_segment_layout.png)

![一个ELF格式的简单示意图](/misc/img/compile/3-1.png)
左边是链接器的视角，ELF文件由若干个节(section)组成。右边是加载器的视角，这些节被习惯性称之为段(segment).

> 编译时生成的 .o（目标文件）以及链接后的 .so （共享库）均可通过链接视图解析
> ELF 规格也允许定义一个解释器(ELF 程序头部的 PT_INTERP 元素)来运行程序. 如果定义了解释器,内核则基于指定解释器可执行文件的各段来构建进程映像,转而由解释器负责加载和执行程序.

elf文件可以分为elf header(eh)、Program Header Table、Sections和Section Header Table几个部分.

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
  Elf64_Word e_flags;   // 与文件关联的特定于处理器的标志, 格式是EF_<machin_flag>
  Elf64_Half e_ehsize;    // Size of this header
  Elf64_Half e_phentsize; // Size of program headers
  Elf64_Half e_phnum;     // Number of program headers
  Elf64_Half e_shentsize; // Size of section headers
  Elf64_Half e_shnum;     // Number of section headers
  Elf64_Half e_shstrndx;  // Section header string table index // .shstrtab在段表中的下标
} Elf64_Ehdr;
```

elf Magic(`7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00`) elf解析器(通常是程序加载器)用它来校验文件的类型是否是 elf, 解析:
- 0x7f : elf标志
- 0x454c46 : "elf"的ascii码
- 0x01/0x02 : elf的class, 即32/64位
- 0x01/0x02 : 字节序, 0x01表示小端
- 0x01 : elf文件的主版本号
- 后续的9个字节elf标准未定义, 一般用0填充, 或作为自定义的扩展标志

> [elf标准, v1.2后未更新](https://refspecs.linuxfoundation.org/elf/elf.pdf)

默认下gcc编译出的elf可执行文件被认为是linux/unix下程序， 因此在文件入口链接了`/lib64/ld-linux-x86-64.so.2`, 其入口是`_start`, 然后才是`_start`调用`main`.

### proghdr(ph)
字段  含义  备注
Type  段的类型，暗含了如何解析此段的内容 
Offset  本段内容在文件的位置  
FileSiz 本段内容在文件中的大小 
VirtAddr  本段内容的开始位置在进程空间中的虚拟地址  
MemSiz  本段内容在进程空间中的大小 
PhysAddr  本段内容的开始位置在进程空间中的物理地址  由于MMU的存在，物理地址不可知，大多时候等于虚拟地址
Flags 段的权限  R/W/E 分别表示 读/写/可执行
Align 大小对齐信息

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
- .text(代码段, 只读):放编译好的部分二进制可执行代码,比如各种函数, 进程代码段不仅包括`.text`, 还有`.init`, `.fini`等.

    这部分区域的大小在程序运行前就已经确定，并且内存区域通常属于只读，某些架构也允许代码段为可写，即允许修改程序.
- .data(数据段): 存储了**已初始化的全局变量，但初始化为0的全局变量出于编译优化的策略还是被保存在`.bss`中**, 属于静态内存分配.
- .bss(Block Started by Symbol): 存储了**未初始化的全局变量或者是默认初始化为0的全局变量**, 且不占空间, 属于静态内存分配, 因此比`.data`节省空间
    `.bss`的size仅表示程序加载器在加载程序时候, 需要为该段分配的空间大小

    可通过`objdump -h *.o`查看
- .rodata: 该段也叫常量区(只读)，用于存放常量数据, 比如字符串常量, 全局const变量和#define定义的常量.

    特殊:
    1. 部分立即数会直接存放在`.text`中
    1. 对于字符串常量，编译器会去掉重复的常量，让程序的每个字符串常量只有一份
    1. `.rodata`是在多个进程间是共享的，这可以提高空间利用率
    1. 在程序加载时， 每个常亮会获得一个固定的内存地址
- .symtab:符号表,保存了符号信息, 基本信息可通过`readelf -s xxx`查看, 具体信息可通过`nm`或`readelf -s xxx`查看

  symtab和strtabsection并不是必不可少的，它们是用来调试的，最终发布软件之前一般会将它们去除，执行`strip <elf_file>`就可以去掉它们.

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

elf_file由内核加载执行, 内核不是以section为单位加载的文件内容的，而是以segment为单位。一个segment可以包含0或多个section，由ph描述（很多情况下，可以将一个ph理解为一个segment）.

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

第一个ph是PHDR，本质上并不是一个segment，它仅包含了整个ph表的信息.
第二个ph是interpreter, 包含了interpreter section.
接着是若干LOAD segment，它们是加载可执行程序的关键

#### 程序->进程
内核支持的可执行文件不止elf一种，还包含script和aout等格式, 所有的格式都有一个linux_binfmt结构体变量与之对应，比 如elf_format、script_format和aout_format。必须调用register_binfmt等函数将它们链接到以formats变量为头的链表中，才能支持对应的格式.

```c
// https://elixir.bootlin.com/linux/latest/source/include/linux/binfmts.h#L102
// kernel定义用来加载二进制文件的对象
struct linux_binfmt {
	struct list_head lh; // 将其连接到formats链表中
	struct module *module;
	int (*load_binary)(struct linux_binprm *); //加载文件
	int (*load_shlib)(struct file *); // 加载共享库
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

// https://elixir.bootlin.com/linux/v6.6.30/source/include/linux/binfmts.h#L18
/*
 * This structure is used to hold the arguments that are used when loading binaries.
 */
struct linux_binprm {
#ifdef CONFIG_MMU
  struct vm_area_struct *vma; // 内存映射信息
  unsigned long vma_pages;
#else
# define MAX_ARG_PAGES  32
  struct page *page[MAX_ARG_PAGES];
#endif
  struct mm_struct *mm; // 管理内存
  unsigned long p; /* current top of mem */
  unsigned long argmin; /* rlimit marker for copy_strings() */
  unsigned int
    /* Should an execfd be passed to userspace? */
    have_execfd:1,

    /* Use the creds of a script (see binfmt_misc) */
    execfd_creds:1,
    /*
     * Set by bprm_creds_for_exec hook to indicate a
     * privilege-gaining exec has happened. Used to set
     * AT_SECURE auxv for glibc.
     */
    secureexec:1,
    /*
     * Set when errors can no longer be returned to the
     * original userspace.
     */
    point_of_no_return:1;
  struct file *executable; /* Executable to pass to the interpreter */
  struct file *interpreter;
  struct file *file; // 关联的文件
  struct cred *cred;  /* new credentials */ // 权限和身份
  int unsafe;   /* how unsafe this exec is (mask of LSM_UNSAFE_*) */
  unsigned int per_clear; /* bits to clear in current->personality */
  int argc, envc; // args, 参数数量; envc, 环境变量的数量
  const char *filename; /* Name of binary as seen by procps */
  const char *interp; /* Name of the binary really executed. Most
           of the time same as filename, but could be
           different for binfmt_{misc,script} */
  const char *fdpath; /* generated filename for execveat */
  unsigned interp_flags;
  int execfd;   /* File descriptor of the executable */
  unsigned long loader, exec;

  struct rlimit rlim_stack; /* Saved RLIMIT_STACK used during exec. */

  char buf[BINPRM_BUF_SIZE];
} __randomize_layout;
```

![](/misc/img/compile/465b740b86ccc6ad3f8e38de25336bf6.jpg)

linux_binprm （binary parameter, 以下简称bprm），表示加载二进制文件需要使用的参数信息.

文件的前128个字节包含它的身份标识，表明它的格式。一种格式在尝试加载文件前，需要先判断文件是否为期望的格式

exec系列函数:
```c
#include <unistd.h>

extern char **environ;

int execl( const char *path, const char *arg, ...);
int execlp( const char *file, const char *arg, ...);
int execle( const char *path, const char *arg , ..., char * const envp[]);
int execv( const char *path, char *const argv[]);
int execvp( const char *file, char *const argv[]);
int fexecve(int fd, char *const argv[], char *const envp[]);
int  execve  (const  char  *filename,  char *const argv [], char *const
       envp[]);
int execveat(int dirfd, const char *pathname,
                    char *const _Nullable argv[],
                    char *const _Nullable envp[],
                    int flags);
```

execve和execveat调用同名系统调用实现，二者的主要区别在于确定文件路径的方式不同。前者的filename参数直接给出文件路径，后者的filename以绝对路径给出时，dirfd被忽略，以相对路径给出时，dirfd 指出了路径所在的目录，dirfd等于AT_FDCWD表示进程的当前工作目录.

其他的函数都是glibc包装，通过调用execve实现的. fexecve需要调用execve，所以它需要将参数fd转换成filename。proc文件系统提供了一项便利，访问/proc/self/fd/目录下对应的fd文件等同于访问目标文件.

它的原理很简单, /proc/self是一个链接，目标目录是/proc进程id对应的目录.

exec函数族可以接受的文件除了可执行文件外还可以是脚本文件，也就是经常看到的以`#! interpreter [optional-arg]`开头的文件, interpreter必须指向一个可执行文件，相当于执行`interpreter
[optional-arg] filename arg...`

#### 系统调用
execve和execveat系统调用最终都调用__do_execve_file实现，在将处理权交给具体的fmt之前，`__do_execve_file`需要完成以下任务:
1. 调用kzalloc申请为bprm内存。
2. 调 用 prepare_bprm_creds 初 始 化 bprm 的 cred 字 段 ； 调 用do_open_execat打开文件，得到file，为bprm->file赋值；根据filename参数为bprm->filename赋值
3. 调用bprm_mm_init设置当前进程的栈：调用mm_alloc申请mm_struct对象，赋值给bprm->mm，然后申请vm_area_struct对象赋值给 bprm->vma 。 这 是 bprm->mm 的 第 一 个 vma ， 它 的 vm_end 等 于STACK_TOP_MAX，大小等于PAGE_SIZE。STACK_TOP_MAX在x86平台上是3G，bprm->p被赋值为`vma->vm_end - sizeof(void *)`，用户栈始于此。请注意，此处仅仅是将得到的mm_struct赋值给bprm->mm，并没有改变进程的mm。
4. 栈已准备就绪，接下来将计算参数的个数，调用prepare_arg_pages 为 bprm->argc 和 bprm->envc 赋 值 ， 然 后 调 用copy_strings将envp和argv等的内容按照顺序复制到用户栈中。
5. 调用prepare_binprm完成cred相关的操作，并读取文件的前128个字节存入bprm->buf。如果文件置位了S_ISUID或者S_ISGID标志，进程可以获得文件所有者的权限

准备工作已经完成，接下来需要各个fmt接手了 ，search_binary_handler函数会遍历formats链表，调用fmt的load_binary.
如果成功调用load_binary，进程会执行新的程序，不会返回，失败则返回错误，继续尝试下一个fmt。所以，exec函数族执行成功的情况下也是不会返回的。

exec函数族存在一个误区，很多人以为系统会创建一个新进程执行程序，但实际上并没有新进程出现，只不过原进程的栈、内存映射等被替换了而已，这个“新”应该理解为焕然一新.

分析elf文件，也就是elf_format的load_binary，load_elf_binary函数。该函数比较复杂，下面的代码中仅保留了处理可执行文件
（ET_EXEC）的逻辑，load_bias和total_size等变量在这种情况下等于0，但依然很复杂。理解elf文件的格式是理解它的前提，可以分为两部分理解。前6步是去旧, 7到14步是迎新.

TODO: use elf64

elfhdr是一个宏，32位平台上就是elf32_hdr，函数的loc变量包含了两个elf32_hdr，elf_ex对应当前可执行文件，interp_elf_ex对应解释器文件，也就是本例中的/lib/ld-linux.so.2。第1步，将bprm->buf赋值给loc->elf_ex.

第2步，检查文件的头，是否以"\177ELF"开头，确保文件是elf格式，除此之外还有平台和文件本身的检查，此处省略。
第3步，申请sizeof(struct elf_phdr) * elf_ex->e_phnum大小的内存，读 取 文 件 的 ph 表 存 入 其 中 ， 32 位 平 台 上 elf_phdr 就 是 上 节 介 绍 的
elf32_phdr。在本例中，也就是读取9个ph，成功后elf_phdata变量指向该ph数组。
第4步，遍历elf_phdata数组，找到描述解释器的segment，也就是本例中的INTERP，它的偏移量等于0x154，大小等于0x13。读取这部分 字 符 到 elf_interpreter 字 符 串 数 组 ， 得 到 “/lib/ld-linux.so.2” ，open_exec打开elf_interpreter文件得到interpreter，读取它的elf32_hdr，存入loc->interp_elf_ex。当然，如果可执行文件是静态链接的，是没有解释器的，也就不需要这步。
第5步，如果需要解释器文件，确保它是elf格式的，然后读取它的ph表，成功后interp_elf_phdata变量指向它的ph数组。
第6步，去旧。调用de_thread注销线程组内其他线程（进程），替换进程的内存描述符current->mm为bprm->mm，替换mm->exe_file为bprm->file，调用do_close_on_exec将置位了O_CLOEXEC标志的文件
关闭。

第7步，调用flush_signal_handlers还原进程处理信号的策略，如果某信号当前的处理策略不是SIG_IGN，则更改为SIG_DFL.
第8步，调用commit_creds使bprm->cred生效。第9步，调整进程的用户栈，我们在do_execveat_common的第3和第 4 项 任 务 中 ， 已 经 确 定 了 栈 起 始 于`STACK_TOP_MAX-
sizeof(void*) `， 并 已 将 参 数 等 复 制 到 栈 中 ， 此 处 做 最 终 的 调 整 。randomize_stack_top在进程的PF_RANDOMIZE标志置1的情况下，在STACK_TOP（32位平台上与STACK_TOP_MAX相等）的基础上做一个随机的改动（一般为8M以内），作为栈的最终起始地址stack_top。随机的栈，是为了防止被黑客攻击。

setup_arg_pages 会 根 据 新 的 stack_top ， 更 新 bprm->p ， 如 果stack_top 和 原 来 的 栈 起 始 地 址 不 同 ， 需 要 调 用 shift_arg_pages 更 新bprm->vma以及和它相关的页表，保证新旧栈的内容不变（不需要重新复制参数信息）。

第10步，真正的加载过程开始了，只需要加载LOAD segment.

所谓的加载，实际就是内存映射，将segment映射到固定的虚拟地址上（MAP_FIXED）. 当然，elf_map在映射的过程中要考虑对齐问题.

第 11 步 ， 设 置 进 程 brk 的 起 点 current->mm->start_brk 和 current->mm->brk，它们的值决定于elf_brk，以PAGE_SIZE对齐的情况下，在本例中它的值等于0x0804B000（进1法）。elf_bss是未初始化段的起始地址，它虽然没有占用elf_f的物理空间，但也是占用内存的，比如u_init_data占用了0x804a024地址，elf_brk接着未初始化段结束的位置的下一区域（取决于对齐方式，默认是下一页）。

第12步，加载解释器，由load_elf_interp函数完成。在本例中，加载的是/lib/ld-linux.so.2文件，它是一个共享文件，不需要固定映射，
映射成功后返回文件被映射的起始地址elf_entry。interp_load_addr的含义 与 load_addr 类 似 ， elf_entry将值赋值给它后会加上loc->interp_elf_ex.e_entry，也就是/lib/ld-linux.so.2在内存中的地址加上入口代码的偏移量，这样elf_entry最终等于解释器的入口地址.

如果不需要解释器，elf_entry则等于loc->elf_ex.e_entry.

第13步，保存必要信息到用户栈，包括argc的值、argv和envp的值，此处保存的值才是读者熟悉的main函数中使用的参数。do_execveat_common的第4步中保存的是参数的内容，注意区分。另外还包括一些键值对，它们是程序正确执行的关键,可以在/proc/{进程号}/auxv文件中得到这些信息，使用xxd命令即可.

AT_ENTRY保存的是elf_f的入口地址，这点尤为重要。在第12步中可以看到，本例中，开始执行的代码并不是elf_f的，而是解释器的
入口代码，解释器执行完毕如何切换到elf_f执行的呢？依靠的就是此处保存到栈中的信息。

第14步，万事俱备，start_thread使进程执行新的代码，该函数是平台相关的。x86平台上，它会修改pt_regs的ip为elf_entry，sp为bprm->p，这样系统调用返回用户空间的时候执行的就是elf_entry处的代码了。bprm->p此时的值是栈顶，也就是argc的位置.

解释器的代码可以在glibc中找到，在本例中解释器的入口是_start，由汇编语言完成，它加载elf_f所需的库，然后找到elf_f的入
口，执行elf_f.

解释器找到elf_f入口的过程并不神秘，解释器执行时，esp寄存器的值等于bprm->p，也就是栈顶，将esp的值作为参数调用_dl_start函数，最终只需要根据该参数解析栈的布局即可得到elf_f入口了。glibc
中的DL_FIND_ARG_COMPONENTS宏就是这个作用, 其中cookie就是栈顶，auxp就是键值对表，查找AT_ENTRY键对应的值即可.

### 共享对象文件（Shared Object）

当一个动态链接库被链接到一个程序文件中的时候，最后的程序文件并不包括动态链接库中的代码，而仅仅包括对动态链接库的引用，并且不保存动态链接库的全路径，**仅仅保存动态链接库的名称.**

基于动态连接库创建出来的二进制文件格式:
- 多了一个.interp 的 Segment,这里面是 ld-linux.so(动态链接器), 即运行时的链接动作都是它做的

  interpsection为程序指定解释器（interpreter），从编译和运行程序的角度讲，可执行程序和库有两种关系，一种是静态链接（编译的时候加入-static），编译时将所有依赖库包含在内，运行时不需要加载其他库. 另一种是动态链接，需要指定解释器在运行时加载需要的库.

  interpsection实际存储的是一个字符串，指出解释器的路径, 比如`/lib64/ld-linux-x86-64.so.2`.

interpsection实际存储的是一个字符串，指出解释器的路径
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

### raw binary
bin文件即raw binary, 可使用objcopy处理elf文件得到. 它去除了elf文件一些不必要的格式信息,可以直接执行, elf文件则需要加载器(loader)解析执行. 许多设备的固件(firmware)以及引导代码都是bin文件,上电后即开始执行代码,不需要完整的操作系统.