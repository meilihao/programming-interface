# compile
参考:
- [GCC编译过程详解](http://chengqian90.com/C%E8%AF%AD%E8%A8%80/GCC%E7%BC%96%E8%AF%91%E8%BF%87%E7%A8%8B%E8%AF%A6%E8%A7%A3.html)
- [GCC编译链接过程](https://www.cnblogs.com/muahao/p/10346724.html)

编译是指将使用高级语言编写的源代码转换成机器语言的过程, 其中用于转换的工具被称为编译器(compiler).
> 函数调用会转换成`call`指令, 函数结束的处理则会转换成`return`指令

编译的四个阶段:
1. 预处理
    C/C++源文件中，以#开头的命令被称为预处理命令，如"#include"、宏定义命令"#define"、条件编译命令"#if、#ifdef"等.

    预处理是将包含(include)的文件插入原文件中、将宏定义展开、根据条件编译命令选择要使用的代码，最后将这些东西输出到一个.i文件中并等待进一步处理. 因此能在这一步检查宏定义和头文件是否正确
1. 编译
    编译器把C/C++代码(比如上述的.i文件)翻译成汇编代码, 即把预处理完的文件进行一系列词法分析、语法分析、语义分析及优化后生成相应的汇编代码文件. 这个过程是整个程序构建的核心部分，也是最复杂的部分之一
1. 汇编
    汇编器将编译器输出的汇编代码翻译成符合一定格式的机器代码，在Linux系统上一般表现为ELF目标文件(OBJ文件即`.o`文件)
1. 链接
    将汇编器生成的OBJ文件和系统库的OBJ文件、库文件链接起来，最终生成可以在特定平台运行的可执行文件

    连接器的任务之一是为函数调用找到匹配的函数的可执行代码的位置.

> 编译器优化选项分为4个级别，-O0表示没有优化，-O1为**缺省值**，建议使用-O2，-O3优化级别最高.

![GCC编译过程详解](/misc/img/GCC编译过程详解.png), 可通过`gcc -v main.c`的编译日志详细了解.

为什么要了解编译器如何工作:
- 优化程序性能 : 比如switch语句是否总是比一系列的if-else高效?
- 理解链接时出现的错误
- 避免安全漏洞 : 缓冲区溢出

include两种方式：
- `#include<>` : 引用的是编译器的类库路径里面的头文件
- `#include” “` : 引用的是程序目录的相对路径中的头文件

汇编代码:
```asm
	.file	"mstore.c"
	.text
	.globl	multstore
	.type	multstore, @function
multstore:
.LFB0:
	.cfi_startproc
	pushq	%rbx
	.cfi_def_cfa_offset 16
	.cfi_offset 3, -16
	movq	%rdx, %rbx
	call	mult2@PLT
	movq	%rax, (%rbx)
	popq	%rbx
	.cfi_def_cfa_offset 8
	ret
	.cfi_endproc
.LFE0:
	.size	multstore, .-multstore
	.ident	"GCC: (Debian 6.3.0-18+deb9u1) 6.3.0 20170516"
	.section	.note.GNU-stack,"",@progbits
```
`.xxx`是指导汇编器和链接器工作的伪指令, 通常可以忽略.

反汇编是指将机器代码转换为汇编代码，这在调试程序时常常用到. 比如使用`objdump -S file.o`查看`.o`文件的反汇编, 其内容描述如下:
1. 左侧第1列是地址偏移量
1. 左侧第2列是指令, x86_64指令是1~15字节. 常用指令/操作数较少的指令所需字节较少, 否则就较多
1. 左侧第3列是汇编代码, 等价于第2列的指令
    ```c
    // mstore.c
    long mult2(long, long);

    void multstore(long x, long y, long *dest) {
        long t = mult2(x,y);
        *dest = t;
    }
    ```
    `mult2(x,y)`在`mstore.s`里是`call	mult2@PLT`; 在`objdump -S mstore.o`时call里面只有相对位置`callq  27 <multstore+0x27>`(0x27是mult2在multstore中的相对位置); 在`objdump -S 完整程序`时为`callq  6ed <mult2>`(6ed是mult2可执行代码的位置)
1. 反汇编使用的指令命名与gcc生成的汇编代码有差别: 它省略了很多指令的后缀`q`, 且給call和ret指令添加了`q`后缀.

## FAQ
1. `#include <>`与`#include ""`区别

    `#include <>`, 引用系统提供的`.h`, 编译器仅在系统指定目录里查找或用`-I`附加其他位置.
    
    `#include ""`, 引用自定义的`.h`, 编译器优先在当前目录下寻找.

1. gcc编译选项
    - `-fPIC` : 位置无关码
    - `-shared` : 生成so

1. debugging symbols
符号是为了定位调试出错的代码行数

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

