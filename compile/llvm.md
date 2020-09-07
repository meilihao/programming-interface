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

## clang
```bash
cat << EOF > multiply.c
int mult() {
    int a =5;
    int b = 3;
    int c = a * b;
    return c;
}
EOF
```