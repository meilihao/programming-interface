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
    gcc -S hello.i -o hello.s # 包含预处理
    ```

    > 目前gcc将预处理和编译合成了一个步骤, c语言由`/usr/lib/gcc/x86_64-linux-gnu/8/cc1`处理, c++由同目录的cc1plus处理.
1. 汇编
    汇编器将编译器输出的汇编代码翻译成符合一定格式的机器代码，在Linux系统上一般表现为ELF目标文件(OBJ文件即`.o`文件)

    ```
    gcc -c hello.s -o hello.o <=> as hello.s -o hello.o # 包含预处理和编译
    ```

    在汇编阶段, 每个目标文件中指令和符号的地址都是相对于自身目标文件而言, 是一个临时的, 属于每个目标文件范围内的地址. 只有在链接时, 链接器将多个目标文件链接后, 才会为所有的符号同一分配地址.

    汇编器在目标文件中创建了一个重定位表给链接器, 链接器通过它读取需要重定位的位置以及如何填充位置处的信息, 见`readelf -r xxx.o`

    填充类型:
    1. R_X86_64_16: 填写操作数地点16位绝对地址
    1. R_X86_64_PC16: PC16表示相对指令指针的地址, 宽度是16位的
1. 链接
    将汇编器生成的OBJ文件和系统库的OBJ文件、库文件链接起来，最终生成可以在特定平台运行的可执行文件

    ```
    gcc hello.o -o hello # 包含预处理, 编译和汇编
    ```

    链接器的任务之一是为函数调用找到匹配的函数的可执行代码的位置.

    链接涉及到函数库.

    在例子中并没有定义“printf”的函数实现，且在预编译中包含进去的“stdio.h”中也只有该函数的声明，而没有定义函数的实现，那么“printf”函数是在哪里实现的呢？
    答案是：系统把这些函数实现都做到了名为libc.so.6的库文件中去了，在没有特别指定时，gcc会到系统默认的搜索路径“/usr/lib”下进行查找，也就是链接到libc.so.6库函数中去，这样就等于实现了函数“printf”，而这也就是链接的作用.

    > 链接过程其实是确定模块间符号的引用(比如`.c`文件间的引用)的过程. 其本质是把不同目标文件粘在一起，对不同目标文件中定义或引用的相同名字进行决议resolve和绑定binding.

    链接器合并.o的方法是相似段合并, 采用该方法的链接器一般都采用两步链接(two-pass linking):
    1. 空间与地址分配

        扫描所有的输入目标文件，并且获得它们各个段的长度、属性和位置，并且将输入目标文件中的符号表中的所有符号定义和符号引用收集起来，统一放到一个全局符号表。这一步，链接器能够获得所有输入目标段长度，并且将它们合并，计算出输出文件中的各个段合并后的长度与位置，并建立映射关系.
    1. 符号解析与重定位, 是链接的核心

        使用上面一步收集到的所有信息，读取输入段的数据、重定位信息，并且进行符号解析与重定位、调整代码中的地址.

    例子:
    ```c
    /*a.c*/
    extern int shared;
    int main()
    {
        int a=100;
        swap(&a, &shared);

        return 0;
    }

    /*b.c*/
    int shared = 1;
    void swap(int *a, int *b)
    {
        *a ^= *b ^= *a ^= *b;
    }

    // gcc -c a.c  -fno-stack-protector # 一定要加“-fno-stack-protector”，不然默认会调函数“__stack_chk_fail”进行栈相关检查，当手动裸ld去链接，因为无法链接到“__stack_chk_fail”所在库文件而报错: undefined reference to `__stack_chk_fail'. 解决办法不是在链接过程中，而是在编译时加此参数，强制gcc不进行栈检查.
    // gcc -c b.c
    // ld a.o b.o -e main -o ab
    ```

    `ld a.o b.o -e main -o ab`, `-e main`表示将main函数作为程序人口, 因为ld默认以"_start"为入口.

    最初`objdump -h <a.o|b.0>`输出的VMA和LMA都是0, 因为虚拟地址还没有分配. 而链接后`objdump -h ab`开始有确定的值.

    > VMA表示虚拟地址，LMA表示加载地址，正常情况下两个值都是一样的，但是在有些嵌入式系统中，特别是程序放在ROM的系统中时，LMA和VMA是不相同. 当前我们只要关注VMA即可.

    > 在32位系统上，默认的text段虚拟地址通常是0x0804800, 64位是0x0401000.

    因为每个符号在段内的相对位置是固定的，所以其实“main”、“shared”和“swap”的地址已经是确定的了，只不过链接器需要在默认的text段虚拟地址基础上给每个符号增加上一个偏移量(通常是原有的段大小)，使它们能够调整到正确的虚拟地址.

    链接时, ld通过elf文件中的重定位表(relocation table, 保存了与重定位相关的信息)来确定和如何调整.o中需要调整的指令. 对于.o, 它必须包含重定位表(每个需要被重定位的段都对应一个重定位段, 这些段统称为重定位表), 用来描述如何修改相应的段里的内容. 可通过`objdump -r xxx.o`查看重定位表, 其输出的每个要被重定位的地方叫一个重定位入口(relocation entry), 定义在[elf64_rel](https://elixir.bootlin.com/linux/v5.9-rc7/source/include/uapi/linux/elf.h#L167), 其中r_info的低8bit表示类型, 高24bit表示该符号在符号表中的下标.

    section(节)是指在汇编源码中经由关键字 section 或 segment 修饰、逻辑划分的指令或数据区域, 汇编器会将这两个关键字修饰的区域在目标文件中编译成节, 也就是说“节”最初诞生于目标文件中.
    segment(段)是链接器根据目标文件中属性相同的多个 section 合并后的 section 集合. 链接器把目标文件链接成可执行文件, 因此段最终诞生于可执行文件中. 
    
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

## gnu c 内嵌汇编
ref:
- [最牛X的GCC 内联汇编](https://www.linuxprobe.com/gcc-how-to.html)
- [内联汇编很可怕吗？看完这篇文章，终结它！](https://www.cnblogs.com/sewain/p/14707347.html)
- [GNU C内嵌汇编语言](https://www.ituring.com.cn/book/tupubarticle/26323)

应用场景:
1. c语言不支持的指令

    操作某些特殊的CPU寄存器(比如lgdt), 操作主板上的某些IO端口
1. 对性能极为苛刻的场景下, 必须使用c内嵌汇编来满足需求.

```c
#define nop()         __asm__ __volatile__ ("nop    \n\t") // nop(空操作)函数的实现
```

`__asm__`是GNU C定义的关键字asm的宏定义`（#define __asm__ asm）`，用来声明一个内嵌汇编表达式. 所以任何一个内嵌汇编表达式都以它开头, 如果要编写符合ANSI C标准的代码（即与ANSI C兼容），那就建议使用关键字`__asm__`.

`__volatile__`是gcc关键字volatile的宏定义, 用于告诉编译器这段代码不能被优化，需保持原样. 因为如果经过编译器优化，这段汇编可能被修改导致无法达到预期的执行效果. 如果要编写符合ANSI C标准的代码（即与ANSI C兼容），那就建议使用关键字`__volatile__`.


一般而言，C语言里嵌入汇编代码片段都要比纯汇编语言写的代码复杂得多. 因为需确定寄存器的分配情况、与C代码融合等细节问题. 为了这个目的，必须要对所用的汇编语言做更多的扩充，增加对汇编语言的明确指示.

C语言里的内嵌汇编代码可分为四部分，以“：”号进行分隔，其一般形式为：`("汇编语句" [:[输出部分output operands]:[输入部分input operands]:[损坏部分clobbers]])`. 它们之间用冒号隔开，如果只有汇编代码部分，后面的冒号可以省略。但是有输入列表部分而没有输出列表部分的时候，输出列表部分的冒号就必须要写，否则 GCC 没办法判断，同样的道理对于其它部分也一样, 具体参考[gcc手册  GAS 相关的章节](https://www.gnu.org/manual/manual.html).



举例:
```asm
static inline void atomic_add(int i, atomic_t *v)
{
        __asm__ __volatile__("lock;" "addl %1,%0"
                     : "+m" (v->a_count)
                     : "ir" (i));
}
//"lock;" "addl %1,%0" 是汇编指令部分
//: "+m" (v->a_count) 是输出列表部分，“+m”表示(v->a_count)和内存地址关联
//: "ir" (i) 是输入列表部分，“ir” 表示i是和立即数或者寄存器关联
```

```c
uint32_t cr0;
asm volatile ("movl %%rc0, %0\n":"=r"(cr0));
cr0:=0x80000000;
asm volatile ("movl %0, %%cr0\n"::"r"(cr0));

//=>
movl %cr0, %ebx
movl %ebx, 12(%esp)
orl $-2147483648, 12(%esp)
movl 12(%esp), %eax
movl %eax, %cr0
```


如果将内嵌汇编表达式比作函数, 指令部分是函数中的代码, 输入部分是用于向函数传入的参数, 输出部分是函数的返回值:

- 指令部分

    汇编语言的语句本身，其格式与在汇编语言程序中使用的格式基本相同，但也有不同之处。指令部分是内嵌汇编的必须项，而其它各部分则视具体情况而定，如果不需要的话是可以忽略的，所以在最简单的情况下就与常规的汇编语句基本相同.

    指令部分的编写规则：
    - gcc使用AT&T汇编语法, 语句间使用`;`, `\n`, `\n\t`分隔. 当指令列表里面有多条指令时，可以全部写在一对双引号中，也可将汇编代码放在多对双引号中.
    - `%1,%0`是占位符，它表示输出、输入列表中变量或表态式，占位符的数字从输出部分开始依次增加, 最多10个. 这些变量或者表态式会被GCC处理成寄存器、内存、立即数放在指令中

- 输出部分

    紧接在指令部分后面的就是`输出部分`，用来指定当前内嵌汇编语句的输出信息, 让 GCC 能够处理 C 语言左值表达式与汇编代码的结合, 格式为：`["输出操作约束"（输出表达式）, ...]`, 输出操作约束与输出表达式需成对出现:

    - 括号内的输出表达式主要负责保存指令部分的执行结果. 通常情况下, 输出表达式是一个变量.
    - 双引号内的部分是输出操作约束, 也可简称`输出约束`. 在输出表达式内需要用（=）或（+）来进行修饰. 等号（=）与加号（+）是有区别的：**等号（=）表示当前表达式是一个纯粹的输出(只写)操作，而加号（+）则表示当前表达式不仅仅是一个输出操作，还是一个输入操作即可读可写**；不管是等号（=）还是加号（+）, 它们都只能用在输出部分, 不能出现在输入部分.
    - `&`: 被修饰的操作数只能作为输出

    `"+m" (*addr)`: `+`表示`(*addr)`可读可写, m表示使用系统支持的任一种内存方式, 不需要借助于寄存器
    `"=a" (result)`: 把处理结果放在寄存器eax 中, 然后复制给变量result

- 输入部分

    记录指令部分的输入信息, 让 GCC 能够处理 C 语言表达式、变量、常量，让它们能够输入到汇编代码中去, 格式为：`"输入操作约束"（输入表达式）[, ...]`, 输入操作约束与输入表达式也需成对出现. 输入表达式中的操作约束不允许指定等号（=）和加号（+）约束，因此输入部分是只读的.

- 损坏部分

    有的时候，当想通知GNU C 指令部分执行时可能会对某些寄存器或内存进行修改, 且这些修改并未在输出部分或输入部分出现过，希望GNU C在编译时能够将这一点考虑进去, 以便 GCC 在汇编代码运行前，生成保存它们的代码，并且在生成的汇编代码运行后，恢复它们（寄存器）的代码. 此时就可以在损坏部分声明这些寄存器或内存, 格式是`"损坏描述"[, ...]`.

    - 寄存器修改通知

    这种情况一般发生在寄存器出现在指令部分中，但又不是输入/输出操作表达式所指定的，更不是编译器为r或g约束选择的寄存器. 如果该寄存器被指令部分所修改, 则需在损坏部分加以描述, 比如：`__asm__("movl %0,%%ecx"::"a"(__tmp):"cx");`, 这段内嵌汇编语句中，%ecx出现在指令列表中，并且被指令修改了，但是却未被任何输入/输出操作表达式所记录, 所以必须要在损坏部分加以描述`cx`, 确保一旦编译器发现后续代码还要使用它时, 会在指令部分执行时做好保存与恢复的工作, 否则可能导致程序异常.

    注意：如果在输入/输出操作表达式中指定寄存器；或为一些输入/输出操作表达式使用q,r,g约束让编译器指派寄存器时, 编译器对这些寄存器的状态是非常清楚的，它知道这些寄存器是被修改的，根本不需要在损坏部分声明它们；但除此之外，编译器对剩下的寄存器中哪些会被当前内嵌汇编语句所修改却一无所知, 此时需要记录.

    - 内存修改通知

    除了寄存器的内容会被修改之外，内存的数据同样也会被修改. 如果一个内嵌汇编语句的指令部分修改了内存，或者在此内嵌汇编表达式中出现过，此时内存数据可能发生改变，并且被该内存未用`m`约束的情况下，需要在损坏部分使用字符串`memory`向编译器声明该变化.

    如果损坏部分已使用`memory`对内存加以约束, 那么编译器会保证在指令部分执行后, 会重新向寄存器装载已引用过的内存空间, 而非寄存器中的副本, 以防止内存与副本中的数据不一致.

    - 标志寄存器修改通知

    当一个内嵌汇编中包含影响标志寄存器r|eflags的指令时，必须在损坏部分中使用`cc`来向编译器声明该修改.

### 操作约束
每个输入/输出表达式都必须指定自身的操作约束. 操作约束的类型有：寄存器约束、内存约束、立即数约束. 在输出表达中, 还有限定寄存器操作的修饰符.

- 寄存器约束

    限定了表达式的载体是一个寄存器, 它可以明确指派或`模糊指派再由编译器自行分配`. 此时可使用寄存器全名也可用缩写, 比如:
    ```x86asm
    __asm__ __volatile__("movl %0,%%cr0"::"eax"(cr0)); # 推荐全名
    __asm__ __volatile__("movl %0,%%cr0"::"a"(cr0)); # 如果指定的是寄存器的缩写名称，那边编译器会根据指令部分的代码来决定实际宽度.
    ```

    常用的寄存器约束的缩写:
    - r ：使用任何可用的通用寄存器
    - q ：从rax, rbx, rcx, rdx中指派一个寄存器
    - g ：使用寄存器或内存空间或立即数
    - m : 内存空间
    - A : eax和edx组成一个64位寄存器
    - a : 使用rax/eax/ax/al寄存器
    - b : 使用rbx/ebx/bx/bl寄存器
    - c : 使用rcx/ecx/cx/cl寄存器
    - d : 使用rdx/edx/dx/dl寄存器
    - D : 使用rdi/edi/di寄存器
    - S : 使用rsi/esi/si寄存器
    - f : 使用浮点寄存器
    - i : 使用一个整数类型的立即数
    - F : 使用一个浮点类型的立即数

    arm64:
    k                SP寄存器
    w                浮点寄存器、SIMD、SVE寄存器
    Upl                使用P0到P7中任意一个SVE寄存器
    Upa                使用P0到P15中任意一个SVE寄存器
    Input            整数，常常用于ADD指令
    J                整数，常常用于SUB指令
    K                整数，常常用于32位逻辑指令
    L                整数，常常用于64位逻辑指令
    M                整数，常常用于32位的MOV指令
    N                整数，常常用于64位的MOV指令
    S                绝对符号地址或标签引用
    Y                浮点数，其值为0
    Z                整数，其值为0
    Ush                表示一个符号的PC相对偏移量的高位部分(大于等于12bit的部分)，这个PC相对偏移介于0-4GB
    Q                表示没有使用偏移量的单一寄存器的内存地址
    Ump                一个适用于SI/DI/SF和DF模式下的加载-存储指令的内存地址

- 内存约束

    限定了表达式的载体是一个内存空间, 使用约束名`m`表示. 例如：
    ```x86asm
    __asm__ __volatile__ ("sgdt %0":"=m"(__gdt_addr)::);  
    __asm__ __volatile__ ("lgdt %0"::"m"(__gdt_addr));
    ```

- 立即数约束

    只能用于输入部分即立即数在表达式中只能作为右值使用, 限定了表达式的载体是一个数值, 使用约束名`i`表示整数类型, 用`F`表示浮点类型. 如果不想借助于任何寄存器或内存，则可以使用立即数约束. 比如:
    ```x86asm
    __asm__ __volatile__("movl %0,%%ebx"::"i"(50));  
    ```

- 修饰符

    只能用于输出部分, 除了等号（=）和加号（+）外, 还有`&`. `&`只能写在约束部分的第二个字符的位置上即（=）或（+）之后, 它强制编译器为输入操作数与输出操作数分配不同的寄存器, 否则可能会导致输入和输出数据混乱.

    只有在输入约束中使用过模糊约束(使用q,r,g等约束缩写)时, 在输出约束中使用`&`才有意义. 即如果所有输入操作表达式都明确指派了寄存器, 那么输出约束再使用`&`就没意义了.

此外，若一个操作数要求使用与前面某个约束中所要求的是同一个寄存器，那就把与那个约束对应的操作数编号放在约束条件中.

#### 序号占位符
序号占位符是输入/输出操作约束的数值映射, 每个内嵌汇编表达式中最多只有10个输入/输出操作表达式，这些操作表达式按照他们被列出来的顺序，依次被赋予编号0至9. 如果指令部分想引用序号占位符, 必须使用`%`前缀加以修饰. 指令部分为了区分序号占位符和寄存器, 会特意使用`%%`修饰寄存器. 在编译时, 编译器会将每个序号占位符所代表的表达式替换到相应的寄存器或内存中.

指令部分在引用序号占位符时, 可根据需要指定操作位宽或指定操作的字节位置. 比如在`%`与序号占位符之间插入一个`b`表示操作最低字节，或插入一个`h`表示操作次低字节.

### example
```c
#include <stdio.h>

int main()
{
    /* val1+val2=val3 */
    unsigned int val1 = 1;
    unsigned int val2 = 2;
    unsigned int val3 = 0;
    printf("val1:%d,val2:%d,val3:%d\n", val1, val2, val3);
    asm volatile(
        "movl $0,%%eax\n\t"      /* clear %eax to 0 */
        "addl %1,%%eax\n\t"      /* %eax += val1 */
        "addl %2,%%eax\n\t"      /* %eax += val2 */
        "movl %%eax,%0\n\t"      /* val2 = %eax */ // %数字表示后面输出部分、输入部分、破坏性描述部分的编号. 按照这几部分里变量的出现顺序从0开始编号，具体来说%0表示变量val3，%1表示变量val1，%2表示变量val2
        : "=m" (val3)            /* output =m mean only write output memory variable */ // 输出部分
        : "c" (val1), "d" (val2) /* input c or d mean %ecx/%edx */ // 输入部分, c表示ecx寄存器，d表示edx寄存器
    );
    printf("val1:%d+val2:%d=val3:%d\n", val1, val2, val3);

    return 0;
}
```

## 函数调用约定
ref:
- [调用约定](https://www.ituring.com.cn/book/tupubarticle/26323)

在 gcc c语言中约定, 栈是实现函数的局部变量, 参数和返回地址的关键因素.

> C语言的调用约定也被称为ABI, 即应用程序二进制接口.

函数调用过程:
1. 在执行函数前, 会将函数参数按逆序压入栈中, 然后执行call 指令. call指令会做两件事: 1. 将下一条指令的地址即返回地址压入栈中, 然后将rip指向被调函数.

1. 被调函数通过`push rbp`保存当前的基址寄存器. 再用`mov rbp, rsp`, 即可用相对于rbp的固定索引来访问函数参数. 对当前栈帧而言rbp是常量. 之后开始执行被调函数的代码.

> 栈帧包含一个函数中使用过的所有栈变量, 包括参数, 局部变量和返回地址.

> 栈变量称为局部变量的原因: 当函数返回后, 栈帧已不存在.

1. 当被调函数执行完毕后, 会做三件事:
    1. 将函数返回值放入rax
    1. 将栈恢复到调用函数时的状态(移除当前栈帧, `mov rsp, rbp`(重置rsp(即丢弃当前栈帧), 经`pop rbp`和ret后变成上一个栈帧的栈顶) + `pop rbp`将rbp重置为上一个栈帧的rbp)
    1. 将控制权还给调用函数, 这通过ret指令实现: 将栈顶值弹出, 并将rip置为该值

> 调用函数时需要假设, 当前的所有寄存器的内容都会被覆盖(除了rbp, 因为其由被调函数在函数开始时保存), 因此需要注意现场保护.

### 具体函数调用约定
- stdcall

    1. 在进行函数调用的时候，函数的参数是从右向左依次放入栈中的

    如`int function（int first，int second）`, 这个函数的参数入栈顺序，首先是参数second，然后是参数first.

    2. 函数的栈平衡操作是由被调用函数执行的，使用的指令是 retn X，它可在函数返回时从栈中弹出X个字节. 例如上面的function函数，当我们把function的函数参数压入栈中后，当function函数执行完毕后，由function函数负责将传递给它的参数first和second从栈中弹出来.

    3. 编译时, 编译器会在函数名的前面用下划线修饰，在函数名的后面由`@+入栈字节数`来修饰. 如上面的function函数，会被编译器转换为_function@8.

- cdecl

    1. 在进行函数调用的时候，和stdcall一样，函数的参数是从右向左依次放入栈中的.

    2. 函数的栈平衡操作是由调用函数执行的，这点是与stdcall不同之处. stdcall使用retn X平衡栈，cdecl则使用leave、pop、向上移动栈指针等方法来平衡栈.

    3. 每个函数调用者都包含有清空栈的代码，所以编译产生的可执行文件会比调用stdcall约定产生的文件大.

    **cdecl是GCC的默认调用约定. 但是，GCC在x64位系统环境下，却使用寄存器作为函数调用的参数**, 按照从左向右的顺序依次将前六个整型参数放在寄存器RDI, RSI, RDX, RCX, R8和R9上，同时XMM0到XMM7用来保存浮点变量，而用RAX保存返回值，且由调用者负责平衡栈.

- fastcall

    1. 函数参数尽可能使用通用寄存器rcx, rdx来传递前两个int类型的参数或较小的参数, 其余参数按照从右向左的顺序入栈.

    2. 函数的栈平衡操作是由被调用函数负责.

还有很多调用规则，如：thiscall、naked call、pascal等

![调用栈](/misc/img/compile/20200112171557752.png)
参考: [汇编指令push,mov,call,pop,leave,ret建立与释放栈的过程](https://blog.csdn.net/liu_if_else/article/details/72794199)

#### [GCC的x86平台cdecl规范详解](http://blog.bytemem.com/post/linux-kernel-function-call-convention)
cdecl属于Caller clean-up类规范. 在调用子程序（callee）时，x87浮点寄存器ST0-ST7必须是空的，在退出子程序时，ST1-ST7必须是空的，如果没有浮点返回值，ST0也必须是空的. 从gcc 4.5开始，函数栈的地址必是16-byte对齐的，在此之前只要求4-byte对齐.

寄存器现场的保存：

    x86-32: 寄存器EAX, ECX, EDX由调用者自己保存（caller-saved），子程序可以改变这些寄存器的值而不用恢复，其他寄存器是callee-saved.
    x86-64: 寄存器RBX, RBP, 和 R12–R15 由子程序保存和恢复，其他寄存器由调用者自己保存.

函数返回值：

    x86-32: 如果是整数存放在EAX寄存器, 如果是浮点数存放在x87协处理器的ST0寄存器.
    x86-64: 64位返回值存放在RAX寄存器，128为返回值保存在RAX和RDX寄存器. 浮点返回值保存在XMM0和XMM1寄存器.

其函数参数传递方式在x86-32和x86-64上是不同的：

    x86-32: 所有函数参数都通过函数栈传递，并且参数入栈顺序是Right-to-Left，即最后一个参数先入栈，第一个参数最后入栈.
    x86-64: 由于AMD64架构提供了更多的可用寄存器，编译器充分利用寄存器来传递参数. 函数的前六个整数参数依次用寄存器RDI, RSI, RDX, RCX, R8, R9 (R10 is used as a static chain pointer in case of nested functions)传递，比如只有一个参数时，用RDI传递参数；如果参数是浮点数，则依次用寄存器XMM0, XMM1, XMM2, XMM3, XMM4, XMM5, XMM6 and XMM7传递. 额外的参数仍然通过函数栈传递. **对于可变参数的函数，实际浮点类型的参数的个数需保存在RAX寄存器**.

### 参数传递方式

函数参数的传递方式无外乎两种: 一种是通过寄存器传递，另一种是通过内存传递. 这两种传递方式在通常情况下都能满足开发需求, 因此它并不会被特别关注. 但是, 在写操作系统时有许多要求苛刻的场景，这使得我们不得不掌握这两种参数传递方式.

- 寄存器传递

    寄存器传递就是通过寄存器来传递函数的参数, 优点是执行速度快，编译后生成的代码量少. 但只有少部分调用约定默认是通过寄存器传递参数，绝大部分编译器是需要特殊指定使用寄存器传递参数的.

    在X86体系结构的linux kernel中，syscall一般会使用寄存器传递. 因为应用程序和kernel的空间是隔离的. 如果想从应用层把参数传递到内核层的话，最方便快捷的方法是通过寄存器传递参数，否则只能大费周章才能把数据传递过去.

- 内存传递

    内存传递参数很好理解，在大多数情况下参数传递都是通过压栈的形式实现的, 比如go的函数调用.

    在X86体系结构下的Linux内核中，中断或异常的处理会使用内存传递参数. 因为，在中断产生后，到中断处理的上半部，中间的过渡代码是用汇编实现的. 汇编跳转到C语言的过程中，C语言函数是使用栈来传递参数的，为了无缝衔接，汇编就需要把参数压入栈中，然后再跳转到C语言实现的中断处理函数中执行.

    > linux 2.6开始逐渐改用寄存器传递.

以上这些都是在X86体系结构下的参数传递方式. 而**在X64体系结构下，大部分编译器都使用的是寄存器传递参数**, 一般规则为， 当参数少于等于6个时, 参数从左到右放入寄存器: rdi, rsi, rdx, rcx, r8, r9, 当参数为 7 个及以上时， 前 6 个与前面一样， 但后面的依次从 "右向左" 放入栈中, 可以写一些c代码验证.

#### 命令行参数/环境变量传递
压栈(栈中存放的是指向具体字符串的指针), 以64-bit平台举例:
```
0x0     <------- 表示env结束
envp[M] <------ [rsp + (N+1)*8 +(M+1)*8]
.............
.............
envp[1] <------ [rsp + (N+1)*8 +24]
envp[0] <------ [rsp + (N+1)*8 +16]
0x0     <------ 压入一个全0的值, 用于区分环境变量和命令行参数
argv[N] <------ [rsp + (N+1)*8]
.............
.............
argv[1] <------  [rsp+16]
argv[0] <------  [rsp+8]
argc    <------  [rsp]  // 参数个数
```

验证:
```bash
gdb -q a.out                                                                                                                                                                                 22:31:13
Reading symbols from a.out...(no debugging symbols found)...done.
(gdb) set environment a=1 # 每次仅运行设置一个
(gdb) set environment b=2
(gdb) set args 5.int.txt 5.out.txt # 运行设置多个
(gdb) b *_start
Breakpoint 1 at 0x4000b0
(gdb) r
Starting program: /home/chen/git/learn_asm/example/a.out 5.int.txt 5.out.txt

Breakpoint 1, 0x00000000004000b0 in _start ()
(gdb) p /x $rsp
$1 = 0x7fffffffdee0
(gdb) x /100xg 0x7fffffffdee0
0x7fffffffdee0: 0x0000000000000003      0x00007fffffffe223 # 3 表示argc
0x7fffffffdef0: 0x00007fffffffe24a      0x00007fffffffe254
0x7fffffffdf00: 0x0000000000000000      0x00007fffffffe25e
0x7fffffffdf10: 0x00007fffffffe286      0x00007fffffffe29c
0x7fffffffdf20: 0x00007fffffffe2b0      0x00007fffffffe312
0x7fffffffdf30: 0x00007fffffffe329      0x00007fffffffe334
0x7fffffffdf40: 0x00007fffffffe36b      0x00007fffffffe37d
0x7fffffffdf50: 0x00007fffffffe3bc      0x00007fffffffe3e0
0x7fffffffdf60: 0x00007fffffffe40c      0x00007fffffffe425
0x7fffffffdf70: 0x00007fffffffe43a      0x00007fffffffe46e
0x7fffffffdf80: 0x00007fffffffe482      0x00007fffffffe492
0x7fffffffdf90: 0x00007fffffffe4be      0x00007fffffffe4cf
0x7fffffffdfa0: 0x00007fffffffe4de      0x00007fffffffe508
0x7fffffffdfb0: 0x00007fffffffe515      0x00007fffffffead1
0x7fffffffdfc0: 0x00007fffffffeae0      0x00007fffffffeb12
0x7fffffffdfd0: 0x00007fffffffeb2a      0x00007fffffffeb4c
0x7fffffffdfe0: 0x00007fffffffeb71      0x00007fffffffed42
0x7fffffffdff0: 0x00007fffffffed6b      0x00007fffffffed90
0x7fffffffe000: 0x00007fffffffeda4      0x00007fffffffedb9
0x7fffffffe010: 0x00007fffffffedcc      0x00007fffffffede0
0x7fffffffe020: 0x00007fffffffede8      0x00007fffffffee11
0x7fffffffe030: 0x00007fffffffee25      0x00007fffffffee39
0x7fffffffe040: 0x00007fffffffee55      0x00007fffffffee5f
0x7fffffffe050: 0x00007fffffffee81      0x00007fffffffee9c
0x7fffffffe060: 0x00007fffffffeecc      0x00007fffffffeeeb
0x7fffffffe070: 0x00007fffffffeefa      0x00007fffffffef2e
0x7fffffffe080: 0x00007fffffffef49      0x00007fffffffef5a
0x7fffffffe090: 0x00007fffffffef94      0x00007fffffffefa9
0x7fffffffe0a0: 0x00007fffffffefb4      0x00007fffffffefc9
0x7fffffffe0b0: 0x00007fffffffefcd      0x0000000000000000 # 再次遇到0x0表示env结束
0x7fffffffe0c0: 0x0000000000000021      0x00007ffff7ffd000
0x7fffffffe0d0: 0x0000000000000010      0x00000000bfebfbff
...
(gdb) x /s 0x00007fffffffe223
0x7fffffffe223: "/home/chen/git/learn_asm/example/a.out" # 程序路径
(gdb) p (char*)0x00007fffffffe223
0x7fffffffe223: "/home/chen/git/learn_asm/example/a.out"
(gdb) x /s 0x00007fffffffe24a
0x7fffffffe24a: "5.int.txt"
(gdb) x /s 0x00007fffffffe254
0x7fffffffe254: "5.out.txt"
(gdb) x /s 0x00007fffffffe25e
0x7fffffffe25e: "CHROME_DESKTOP=code-url-handler.desktop"
(gdb) x /s 0x00007fffffffefc9
0x7fffffffefc9: "a=1"
(gdb) x /s 0x00007fffffffefcd
0x7fffffffefcd: "b=2" # 后添加的env先入栈
```

## 优化
`barrier()`屏障可阻止编译器的优化, 即屏蔽前的语句和屏蔽后的不乱"串门".

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