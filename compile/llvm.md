# llvm
LLVM这个名字源于Lower Level Virtual Machine， 但这个项目并不局限于创建一个虚拟机， 它已经发展成为当今炙手可热的编译器基础框架.

它是一系列紧密联系的底层工具链组件的统称（例如链接、编译、调试等）:
- `clang-10 -v` : c/c++编译前端
- `opt-10 --version` : 旨在IR级的优化器和分析器

	将LLVM源文件作为输入，对其运行指定的优化或分析，然后输出优化文件或分析结果. opt的功能取决于是否给出了-analyze选项

	```bash
	# opt –view-cfg const.bc
	# opt -view-callgraph file.bc
	```
- llvm-dis-10 : 将LLVM位码解码成LLVM汇编码
- llvm-as-10 : 将人工可读的LLVM IR文件（称为LLVM汇编码）转换为LLVM位码
- llvm-link-10 : 将几个LLVM位码链接在一起，以产生一个包含所有输入的LLVM位码
- llvm-mc-10 : 能够将汇编指令并生成诸如ELF、MachO和PE等对象格式的目标文件. 它也可以反汇编相同的对象，从而转储这些指令的相应的汇编信息和内部LLVM机器指令数据结构
- lli-10 : LLVM IP的解释器和JIT编译器
- llc-10 : 一个通过特定后端将LLVM位码转换成目标机器汇编语言文件或目标文件的工具, 可以通过传递参数来选择优化级别、打开调试选项以及启用或禁用特定于目标的优化

它的设计就是基于库的, 即一系列接口清晰的可重用库.

最流行的传统静态编译器（像大多数 C 编译器）设计就是三段式的设计:前端->优化器->后端.

参考:
- [<<LLVM COOKBOOK's samples>>](https://resources.oreilly.com/examples/9781785285981/tree/master/B04197_CodeBundle/B04197_CodeBundle)

LLVM的代码有3种表示形式：
1. 编译器内存中的IR
1. 存于磁盘的bitcode
1. 用户可读的汇编码

LLVM IR(Intermediate Representation, 中间码)是基于静态单赋值(Static Single Assignment, SSA)的， 并且提供了类型安全性、 底层操作性、 灵活性， 因此能够清楚表达绝大多数高级语言. 这种表示形式
贯穿LLVM编译的各个阶段. 事实上， LLVM IR致力于成为一种足够底层的通用IR，只有这样， 高级语言的诸多特性才能够得以实现.

> LLVM IR 被设计成在编译优化层用来做中间分析和转换的载体.

## opt
```bash
# cat << EOF > testfile.ll
define i32 @test1(i32 %A) {
    %B = add i32 %A, 0
    ret i32 %B
}

define internal i32 @test(i32 %X, i32 %dead) {
    ret i32 %X
}

define i32 @caller() {
    %A = call i32 @test(i32 123, i32 456)
    ret i32 %A
}
EOF
# opt-10 -S -instcombine testfile.ll -o output1.ll # 指令合并
# cat output1.ll 
; ModuleID = 'testfile.ll'
source_filename = "testfile.ll"

define i32 @test1(i32 %A) {
  ret i32 %A # 合并了代码
}

define internal i32 @test(i32 %X, i32 %dead) {
  ret i32 %X
}

define i32 @caller() {
  %A = call i32 @test(i32 123, i32 456)
  ret i32 %A
}
# opt-10 -S -deadargelim testfile.ll -o output2.ll # 进行无用参数消除（ dead-argument-elimination） 优化
# cat output2.ll 
; ModuleID = 'testfile.ll'
source_filename = "testfile.ll"

define i32 @test1(i32 %A) {
  %B = add i32 %A, 0
  ret i32 %B
}

define internal i32 @test(i32 %X) { # 删除了dead
  ret i32 %X
}

define i32 @caller() {
  %A = call i32 @test(i32 123)
  ret i32 %A
}
```

LLVM Pass 是"遍历一遍IR，可以同时对它做一些操作"的意思.

LLVM优化器为用户提供了不同的优化Pass, 但整体的编写风格一致. 对每个Pass的源码编译, 得到一个Object文件, 之后这些不同的文件再链接得到一个库. Pass之间耦合很小， 而Pass之间的依赖信息由
LLVM Pass管理器（ PassManager） 来统一管理， 在Pass运行的时候会进行解析.

![](/misc/img/compile/llvm/v2-d77b8bad0eb6a8868e0df97620f2af0f_720w.jpg)

上图描述了pass依赖关系及链接到一个库的经过.

与优化器相似, LLVM代码生成器（ code generator） 也采用了模块的设计理念， 它将代码生成问题分解为多个独立Pass： 指令选择、 寄存
器分配、 指令调度、 代码布局优化、 代码发射. 同样， 也有许多内建的Pass， 它们默认执行， 但用户可以选择只执行其中一部分.

## llvm工具链
### 将C源码转换为LLVM汇编码
```bash
$ cat << EOF > multiply.c
int mult() {
    int a =5;
    int b = 3;
    int c = a * b;
    return c;
}
EOF
$ clang-10 -emit-llvm -S multiply.c -o multiply.ll # 将C语言代码转换成LLVM IR, 可明白C语言代码是如何映射到LLVM IR的.
$ clang-10 -cc1 -emit-llvm multiply.c -o multiply.ll2 # 或通过cc1生成IR
```

工作原理:
将C语言代码编译为LLVM IR的过程从词法分析开始——将C语言源码分解成token流， 每个token可表示标识符、 字面量、 运算符等；
token流会传递给语法分析器， 语法分析器会在语言的CFG（ Context Free Grammar， 上下文无关文法） 的指导下将token流组织成AST（ 抽
象语法树）； 接下来会进行语义分析， 检查语义正确性， 然后生成IR.

llvm IR信息可参考[这里](http://llvm.org/docs/LangRef.html)

### 将LLVM IR转换为bitcode
LLVM bitcode（ 也称为字节码——bytecode） 由两部分组成： 位流（ bitstream， 可类比字节流） ， 以及将LLVM IR编码成位流的编码格式.

```bash
$ cat << EOF > test.ll
define i32 @mult(i32 %a, i32 %b) #0 {
  %1 = mul nsw i32 %a, %b
  ret i32 %1
}
EOF
$ llvm-as-10 test.ll -o test.bc # 把test.ll文件的LLVM IR转为bitcode格式
```

工作原理:
llvm-as即是LLVM的汇编器. 它会将LLVM IR转为bitcode（ 就像把普通的汇编码转成可执行文件）.

为了把LLVM IR转为bitcode， 我们引入了区块（ block） 和记录（ record） 的概念. 区块表示位流的区域， 例如一个函数体、 符号表等. 每个区块的内容都对应一个特定的ID， 例如LLVM IR中函数体的ID是12. 记录由一个记录码和一个整数值组成， 它们描述了在指令、 全局变量描述符、 类型描述中的实体.

LLVM IR的bitcode文件由一个简单的封装结构封装. 结构包括一个描述文件段落偏移量的简单描述头， 以及内嵌BC文件的大小.

llvm bitcode信息可参考[这里](http://llvm.org/docs/BitCodeFormat.html#abstract)

### 将LLVM bitcode转换为目标平台汇编码
```bash
$ llc-10 test.bc -o test.s # 把LLVM bitcode转换为汇编码
$ cat test.s
$ clang -S test.bc -o test2.s –fomit-frame-pointer # 或通过Clang从bitcode文件格式生成汇编码， 需要使用`-S`参数. 此时输出的test.s文件和之前样例的一样. 另外, 使用了fomit-framepointer参数， 是因为Clang默认不消除帧指针而llc却默认消除
```

工作原理
llc命令把LLVM 输入编译为特定架构的汇编语言，如果我们在之前的命令中没有为其指定任何架构， 那么默认生成本机的汇编码，即调用llc命令的主机. 如果想更进一步由汇编文件得到可执行文件， 还需要使用汇编器和链接器.

在以上命令中加入-march=architechture参数， 可以生成特定目标架构的汇编码. 使用-mcpu=cpu参数则可以指定其CPU，而`-reg alloc =basic greedy fast/pbqp`则可以指定寄存器分配类型

### 将LLVM bitcode转回为LLVM IR
```bash
$ llvm-dis-10 test.bc -o test3.ll # 把bitcode文件转换为llvm ir
```

工作原理
llvm-dis命令即是LLVM反汇编器, 它使用LLVM bitcode文件作为输入， 输出LLVM IR. 如果省略文件名, llvm-dis会从stdin读取输入.

### 转换LLVM IR
```bash
$ opt-10 -mem2reg -S multiply.ll -o multiply1.ll # 运行mem2reg Pass， 它会帮助我们明白C指令是如何映射到IR指令的
```

工作原理
opt是LLVM的优化和分析工具，采用llvm ir文件作为输入， 并且按照passname执行Pass. 在执行Pass之后的输出存于llvm ir文件中， 其中包含转换后的IR代码. opt工具可以使用多个Pass.

如果给opt工具传入-analyze参数， 它会在输入源码上执行不同的分析， 并且在标准输出流或错误流打印分析结果. 当然， 输出也能重定向到另一个程序或者一个文件. 如果不为opt工具传入-analyze参数， 它会对输入执行转换Pass， 以进行优化.

可作为参数传递给opt工具：
- adce： 入侵式无用代码消除
- bb-vectorize： 基本块向量化
- constprop： 简单常量传播
- dce： 无用代码消除
- deadargelim： 无用参数消除
- globaldce： 无用全局变量消除
- globalopt： 全局变量优化
- gvn： 全局变量编号
- inline： 函数内联
- instcombine： 冗余指令合并
- licm： 循环常量代码外提
- loop-unswitch： 循环外提
- loweratomic： 原子内建函数lowering
- lowerinvoke： invode指令lowering， 以支持不稳定的代码生成器
- lowerswitch： switch指令lowering
- mem2reg： 内存访问优化
- memcpyopt： MemCpy优化
- simplifycfg： 简化CFG
- sink： 代码提升
- tailcallelim： 尾调用消除

可在[llvm/test/Transforms目录](https://github.com/llvm/llvm-project/tree/master/llvm/test/Transforms)找到以上的每一个Pass的测试代码.

### 链接LLVM bitcode
```bash
$ cat << "EOF" > test1.c
int func(int a) {
    a = a*2;
    return a;
}
EOF
$ cat << EOF > test2.c
#include<stdio.h>
extern int func(int a);

int main() {
    int num = 5;
    num = func(num);
    printf("number is %d\n", num);
    return num;
}
EOF
$ clang-10 -emit-llvm -S test1.c -o test1.ll
$ clang-10 -emit-llvm -S test2.c -o test2.ll
$ llvm-as-10 test1.ll -o test1.bc
$ llvm-as-10 test2.ll -o test2.bc
$ llvm-link-10 test1.bc test2.bc -o output.bc # 使用llvm-link命令链接两个LLVM bitcode文件
```

工作原理
llvm-linkM工具的功能和传统的链接器一致： 如果一个函数或者变量在一个文件中被引用， 却在另一个文件中定义， 那么链接器就会解析这个文件中引用的符号. 但和传统的链接器不同， llvm-link不会链接
Object文件生成一个二进制文件， 它只链接bitcode文件.

> 在链接bitcode文件之后， 我们可以传递`-S`参数给llvm-link工具来输出IR文件

### 执行LLVM bitcode
```bash
$ lli output.bc
number is 10
```

工作原理
lli工具命令执行LLVM bitcode格式程序， 它使用LLVM bitcode格式作为输入并且使用即时编译器（ JIT） 执行. 当然， 如果当前的架构不存在JIT编译器， 会用解释器执行. 如果lli能够采用JIT编译器， 那么它能高效地使用所有代码生成器参数， 如llc.