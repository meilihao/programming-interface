# elf
参考:
- [ELF Format](http://www.skyfree.org/linux/references/ELF_Format.pdf)
- [Linux内核之ELF格式解析](https://mudongliang.github.io/2015/10/31/linuxelf.html)
- [计算机那些事(4)——ELF文件结构](http://chuquan.me/2018/05/21/elf-introduce/)

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

### 可重定位文件 (Relocatable File),
即编译时生成的`.o`文件, ELF 的一种类型

ELF 文件的头是用于描述整个文件的. 这个文件格式在内核中有定义,分别为 `struct elf32_hdr 和 struct elf64_hdr`.

节头部表(Section Header Table)是sections的元数据, 在代码里面的定义是`struct elf32_shdr 和 struct elf64_shdr`

文件的各section:
- .init : 程序初始化入口代码，在main()之前运行
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
- .symtab:符号表,保存了符号信息, 可通过`readelf -s xxx`查看
- .strtab:存储变量名，函数名, 是被`.symtab`引用的符号名称

    例如, `char* szPath="/root"`, `void func()`的变量名szPath和函数名func就存储在`.strtab`段.
- .shstrtab: 用于保存section header中用到的字符串. bss,text,data等段名存储在这里
- .comment :注释信息段
- .node.GUN-stack :堆栈提示段
- .debug: 一个调试符号表
- .eh_frame: 记录调试和异常处理时用到的信息
- 以`.rec`开头的 sections 里面装载了需要重定位的符号

    - .rel.text : 针对`.text`段的重定位表，还有rel.data(针对data段的重定位表).

> .data与.bss没有本质区别, 都是用于存放静态变量, 只是.data是已初始化过的静态数据, 而.bss程序是运行时会分配空间并置零的静态数据.

> static 声明的变量，无论它是全局变量还是在函数之中，只要是没有赋初值都存放在.bss段，如果赋了初值，则把它放在.data段.

![.o文件的ELF](/misc/img/compile/1671100-20190512203047832-334199166.jpg)

### 可执行文件(Executable File)
ELF 的第二种格式

这个格式和.o 文件大致相似,还是分成一个个的 section,并且被节头表描述. 只不过这些section 是多个.o 文件合并过的. 但是这个文件已经是可以加载到内存里面执行的文件了,因而这些 section 被分成了需要加载到内存里面的代码段、数据段和不需要加载到内存里面的部分,**将小的 section 合成了大的段 segment**,并且在最前面加一个段头表
(Segment Header Table). 在代码里面的定义为 `struct elf32_phdr 和 struct elf64_phdr`,这里面除了有对于段的描述之外,最重要的是 p_vaddr,这个是这个段加载到内存的虚拟地址. 在 ELF 头里面有一项 e_entry,也是个虚拟地址,是这个程序运行的入口.

基于动态连接库创建出来的二进制文件格式:
- 多了一个.interp 的 Segment,这里面是 ld-linux.so,这是动态链接器, 即运行时的链接动作都是它做的
- ELF 文件中还多了两个 section,一个是.plt,过程链接表(Procedure Linkage Table,PLT),一个是.got.plt,全局偏移量表(Global Offset Table,GOT)

由于是运行时才去找,编译的时候,压根不知道被调用函数在哪里,所以就在 PLT 里面建立一项PLT[x]. 这一项也是一些代码,有点像一个本地的代理,在二进制程序里面,不直接调用
具体的被调函数,而是调用 PLT[x] 里面的代理代码,这个代理代码会在运行的时候找真正的被调函数.

查找具体被调函数的方法: GOT也会为 被调 函数创建一项 GOT[y], 它是运行时 被调 函数在内存中真正的地址. 如果这个地址在, 程序 调用 PLT[x] 里面的代理代码,代理代码调用 GOT 表中对应项 GOT[y],调用的就是加载到内存中的 xxx.so 里面的 具体 函数了.

GOT如何找到具体函数: GOT[y]开始时也没有地址, 但它又回调 PLT, 这个时候 PLT 会转而调用 PLT[0],也即第一项,PLT[0] 转而调用 GOT[2],这里面是 ld-linux.so 的入口函数,这个函数会找到加载到内存中的 xxx.so 里面的 具体 函数的地址,然后把这个地址放在 GOT[y] 里面. 下次,PLT[x] 的代理函数就能够直接调用了.

一个ELF文件被加载后，代码段、数据段以及堆栈便在虚拟地址空间的分配:
![](/misc/img/compile/3-3.png)

再细节一点就是下面这个图:
![](/misc/img/compile/3-3.png)

查看程序的虚拟地址空间: `cat /proc/self/maps | sort -n`. 更细节可参考: 《Linker && Loader》和《程序员的自我修养——链接、装载与库》.

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