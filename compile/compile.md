# compile
参考:
- [GCC编译过程详解](http://chengqian90.com/C%E8%AF%AD%E8%A8%80/GCC%E7%BC%96%E8%AF%91%E8%BF%87%E7%A8%8B%E8%AF%A6%E8%A7%A3.html)
- [GCC编译链接过程](https://www.cnblogs.com/muahao/p/10346724.html)

编程语言的本质是变量和函数, 具体编程语言是组织和表达变量和函数的形式.

编译是指将使用高级语言编写的源代码转换成机器语言的过程, 其中用于转换的工具被称为编译器(compiler).
> 函数调用会转换成`call`指令, 函数结束的处理则会转换成`return`指令

编译的四个阶段:

```c
// 演示例子(hello.c)
#include <stdio.h>

int main()  
{  
     printf("Hello World\n");
     return 0;
}
```

1. 预处理
    C/C++源文件中，以#开头的命令被称为预处理命令，如"#include"、宏定义命令"#define"、条件编译命令"#if、#ifdef"等(`#pragma`会被保留).

    预处理是预编译器cpp将包含(include)的文件插入原文件中、将宏定义展开、根据条件编译命令选择要使用的代码，最后将这些东西输出到一个.i文件中并等待进一步处理. 因此能在这一步检查宏定义和头文件是否正确

    ```
    gcc -E hello.c -o hello.i
    ```

    > 可通过`printf SYS_read | gcc -include sys/syscall.h -m32 -E -`, 查看`sys/syscall.h`涉及的相关头文件
1. 编译
    编译器把C/C++代码(比如上述的.i文件)翻译成汇编代码, 即把预处理完的文件进行一系列词法分析、语法分析、语义分析及优化后生成相应的汇编代码文件. 这个过程是整个程序构建的核心部分，也是最复杂的部分之一

    ```
    gcc -S hello.i -o hello.s
    ```

    > 目前gcc将预处理和编译合成了一个步骤, c语言由`/usr/lib/gcc/x86_64-linux-gnu/8/cc1`处理, c++由同目录的cc1plus处理.
1. 汇编
    汇编器将编译器输出的汇编代码翻译成符合一定格式的机器代码，在Linux系统上一般表现为ELF目标文件(OBJ文件即`.o`文件)

    ```
    gcc -c hello.s -o hello.o <=> as hello.s -o hello.o
    ```
1. 链接
    将汇编器生成的OBJ文件和系统库的OBJ文件、库文件链接起来，最终生成可以在特定平台运行的可执行文件

    ```
    gcc hello.o -o hello
    ```

    链接器的任务之一是为函数调用找到匹配的函数的可执行代码的位置.

    链接涉及到函数库.

    在例子中并没有定义“printf”的函数实现，且在预编译中包含进去的“stdio.h”中也只有该函数的声明，而没有定义函数的实现，那么“printf”函数是在哪里实现的呢？
    答案是：系统把这些函数实现都做到了名为libc.so.6的库文件中去了，在没有特别指定时，gcc会到系统默认的搜索路径“/usr/lib”下进行查找，也就是链接到libc.so.6库函数中去，这样就等于实现了函数“printf”，而这也就是链接的作用.

    > 链接过程其实是确定模块间符号的引用(比如`.c`文件间的引用)的过程. 其本质是把不同目标文件粘在一起，对不同目标文件中定义或引用的相同名字进行决议resolve和绑定binding.
    
**合并流程**为:
```
gcc hello.c -o hello
```

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
`.xxx`和`.cfi_xxx`是指导汇编器和链接器工作的伪指令, 通常可以忽略.

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

## 交叉编译
参考:
- [深入理解交叉编译(Cross Compile)](https://www.jianshu.com/p/62613863aed0)

交叉编译指的是我们能够在一个平台（ 例如x86） 编译并构建二进制文件， 而在另一个平台（ 例如ARM） 运行.

几乎所有构建系统都使用形如 CPU-供应商-操作系统-ABI，称为三元组的名称表示目标机器, 为什么一个“三元组”却包含四个部分. 这是历史遗留的：最早，三个部分就足以无歧义地描述一台机器, 但是随着新的机器和系统不断出现，最终证明三个部分是不够的. 然而，“三元组”这个术语保留了下来.

交叉编译中的build、host、target选项的含义:
- --build=编译该软件所使用的平台(你现在电脑的平台), config.guess中猜的
- --host=该软件将运行的平台(编译出来的库可以在哪个平台上运行)
- --target=该软件所处理的目标平台, 即告诉该软件编译出来的工具链生成的代码的运行平台, 即编译程序能够为其生成机器码的平台. 这个选项只有在建立交叉编译环境的时候用到, 比如compile cross-compiler, binutils，toolchain时.

为相同平台（ 主机与目标机器相同） 编译代码我们称为本机编译（ native assembler） ， 而当主机与目标机器为不同平台时编译代码则称为交叉编译（ cross-compiler）.

build和host比较好理解,但是target就不好办了.

一般来说,我们平时所说的交差编译用不到他target的,比如`./configure --build=i386-linux --host=arm-linux`就可以了,在386的平台上编译可以运行在arm板的程序.但是,一般我们都是编译程序,而不是编译工具,如果我们编译工具,比如gcc,这个target就有用了.如果我们需要在一台机器上为arm开发板编译一个可以处理mips程序的gcc,那么target就是mips了.

`./configure --build=powerpc --host=i686 --target=mips`: 在powerpc编译, 而编译出的工具在i686运行, 且该工具为mips平台生成机器码.

## 可重入(reentrant)
一个函数被重入, 表示该函数还没有执行完成, 由于外部因素或内部调用, 又一次进入该函数执行. 一个函数被重入只有两种情况:
1. 多个线程同时执行该函数
2. 存在函数本身调用本身

一个函数被称为可重入, 表明该函数被重入后不会产生任何不良后果. 可重入函数的特点:
- 不使用任何(局部)静态或全局的非const变量
- 不返回任何(局部)静态或全局的非const变量的指针
- 仅依赖调用方提供的参数
- 不依赖任何单个资源的锁
- 不调用任何不可重入的函数

函数可重入是并发安全的强力保障.

## volatile/biarrier
volatile关键字可阻止过度优化, 具体作用:
1. 阻止编译器为了提高速度将一个变量缓存到寄存器内而不写回
1. 阻止编译器调整操作volatile变量的指令顺序(cpu动态调度下, 还是可能乱序)

biarrier指令会阻止将该指令之前的指令交换到biarrier之后, 反之亦然.

## FAQ
### `#include <>`与`#include ""`区别

    `#include <>`, 引用系统提供的`.h`, 编译器仅在系统指定目录里查找或用`-I`附加其他位置.
    
    `#include ""`, 引用自定义的`.h`, 编译器优先在当前目录下寻找.

### gcc编译选项
    - `-fPIC` : 位置无关码
    - `-shared` : 生成so

### debugging symbols
符号是为了定位调试出错的代码行数

### 内联(inline)函数
函数会在它被调用的位置上展开, 以消除函数调用和返回所带来的开销(寄存器保存和恢复), 同时编译器会把函数代码和内联展开代码放在一起进行优化, 因此有了进一步优化的可能.

缺点: 占用更多内存和占用更多的指令缓存. kerne通常把对时间要求高, 本身长度短的函数定义为内联函数. 如何函数较大, 会被反复调用, 且没有特别的时间限制, 此时不建议做成内联函数.

> 内联函数必须在使用前就定义好, 否则无法展开.

> linux kernel中, 为了类型安全和易读性, 优先使用内联函数而不是复杂的宏.

### 内联汇编
通过`asm()`指令来支持.

linux kernel混合使用了c和汇编. 在偏近arch底层和对执行时间要求严格的地方会使用汇编.

### likely()/unlikely()
gcc对fi条件选择语句的优化.

场景:
-  likely  : 经常触发
- unlikely : 绝少触发

> 场景用错会导致性能下降

### 编译不报错, 但运行报错: 缺so
比如使用librocksdb.so:`Compression type Snappy is not linked with the binary`

原因: rocksdb编译所需的依赖lib未安装, 且rocksdb编译时未报错.
解决方案: 安装依赖lib, 重新编译rocksdb.