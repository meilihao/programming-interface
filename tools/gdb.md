# gdb
## cmd
- start : 开始调试
- break : 设置断点. `break sum if n==0` : 设置sum函数中， n==0 时的数据断点
- backtrace : 查看函数调用的顺序
- frame ${N} : 切换栈编号为N的上下文中, N从`backtrace`获取
- disas : 反汇编代码. `disas main` : 显示main函数对应的汇编代码 
- info args : 查看当前函数参数的值
- info locals : 查看当前函数的局部变量
- info break : 查看断点信息
- info registers : 查看当前的寄存器 
- info frame : 查看当前的栈帧
- x/nfu : 查看内存内容
    - n表示要显示的内存单元的个数,正负表示读取方向
    - f表示显示方式:
        - x 按十六进制格式显示变量
        - d 按十进制格式显示变量
        - u 按十进制格式显示无符号整型
        - o 按八进制格式显示变量
        - t 按二进制格式显示变量
        - a按十六进制格式显示变量
        - i 指令地址格式
        - c 按字符格式显示变量
        - s 按字符串格式显示变量
        - f 按浮点数格式显示变量
    - u表示一个地址单元的长度(b, B; h, 2B, w, 4B; g, 8B)
```gdb
// gcc (Debian 6.3.0-18+deb9u1) + deepin 15.10.1_x64
```c
int add(int a, int b, int c, int d, int e, int f, int g, int h) { // 8 个参数相加
    int sum = a + b + c + d + e + f + g + h;
    return sum;
}

int main(void) {
    int i = 10;
    int j = 20;
    int k = i + j;
    int sum = add(11, 22,33, 44, 55, 66, 77, 88);
    int m = k; // 为了观察 %rax Caller Save 寄存器的恢复
    return 0;
}
```
(gdb) backtrace
#0  add (a=11, b=22, c=33, d=44, e=55, f=66, g=77, h=88) at demo2.c:4
#1  0x00005555555546f0 in main () at demo2.c:12 // 是该函数的rip值
(gdb) info registers
rax            0x1e	30
rbx            0x0	0
rcx            0x2c	44
rdx            0x21	33
rsi            0x16	22
rdi            0xb	11
rbp            0x7fffffffe0a0	0x7fffffffe0a0 // 当前帧的栈底
rsp            0x7fffffffe0a0	0x7fffffffe0a0 // 当前帧的栈顶
r8             0x37	55
r9             0x42	66
r10            0xc	12
r11            0x1	1
r12            0x555555554530	93824992232752
r13            0x7fffffffe1c0	140737488347584
r14            0x0	0
r15            0x0	0
rip            0x555555554678	0x555555554678 <add+24>
eflags         0x206	[ PF IF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
(gdb) info frame // = info frame 0
Stack level 0, frame at 0x7fffffffe0b0: // 当前帧的开始位置, 从返回地址的结束位置开始算(之前文章一直介绍返回地址属于前一帧, 可能是gdb计算方式不同而已)
 rip = 0x555555554678 in add (demo2.c:4); saved rip = 0x5555555546f0 // `rip`是当前指令指针的地址, 即当前执行位置; `saved rip`是caller的rip
 called by frame at 0x7fffffffe0f0 // 调用者func的帧开始位置
 source language c.
 Arglist at 0x7fffffffe0a0, args: a=11, b=22, c=33, d=44, e=55, f=66, g=77, h=88
 Locals at 0x7fffffffe0a0, Previous frame's sp is 0x7fffffffe0b0 // ;`Previous frame's sp`:上一帧的rsp, 与`Stack level 0, frame at 0x7fffffffe0b0`对应
 Saved registers: // 当前桢中保存caller的寄存器信息, 格式为`寄存器名称 @ 保存位置`
  rbp at 0x7fffffffe0a0, rip at 0x7fffffffe0a8 // `0x7fffffffe0a0`是保存caller rbp的位置, `0x7fffffffe0a8`是保持返回地址的位置, 其值是`saved rip`
(gdb) info frame 1
Stack frame at 0x7fffffffe0f0:
 rip = 0x5555555546f0 in main (demo2.c:12); saved rip = 0x7ffff7a5a2e1
 caller of frame at 0x7fffffffe0b0 // 被调func的帧开始位置
 source language c.
 Arglist at 0x7fffffffe0e0, args: 
 Locals at 0x7fffffffe0e0, Previous frame's sp is 0x7fffffffe0f0
 Saved registers:
  rbp at 0x7fffffffe0e0, rip at 0x7fffffffe0e8
(gdb) x/2xg 0x7fffffffe0a0 // 读取旧rbp和返回地址
0x7fffffffe0a0:	0x00007fffffffe0e0	0x00005555555546f0
(gdb) disassemble main
Dump of assembler code for function main:
   0x00005555555546a6 <+0>:	push   %rbp
   0x00005555555546a7 <+1>:	mov    %rsp,%rbp
   0x00005555555546aa <+4>:	sub    $0x20,%rsp
   0x00005555555546ae <+8>:	movl   $0xa,-0x4(%rbp)
   0x00005555555546b5 <+15>:	movl   $0x14,-0x8(%rbp)
   0x00005555555546bc <+22>:	mov    -0x4(%rbp),%edx
   0x00005555555546bf <+25>:	mov    -0x8(%rbp),%eax
   0x00005555555546c2 <+28>:	add    %edx,%eax
   0x00005555555546c4 <+30>:	mov    %eax,-0xc(%rbp)
   0x00005555555546c7 <+33>:	pushq  $0x58
   0x00005555555546c9 <+35>:	pushq  $0x4d
   0x00005555555546cb <+37>:	mov    $0x42,%r9d
   0x00005555555546d1 <+43>:	mov    $0x37,%r8d
   0x00005555555546d7 <+49>:	mov    $0x2c,%ecx
   0x00005555555546dc <+54>:	mov    $0x21,%edx
   0x00005555555546e1 <+59>:	mov    $0x16,%esi
   0x00005555555546e6 <+64>:	mov    $0xb,%edi
   0x00005555555546eb <+69>:	callq  0x555555554660 <add>
   0x00005555555546f0 <+74>:	add    $0x10,%rsp            // 发现返回地址是call后的下一个指令地址
   0x00005555555546f4 <+78>:	mov    %eax,-0x10(%rbp)
   0x00005555555546f7 <+81>:	mov    -0xc(%rbp),%eax
   0x00005555555546fa <+84>:	mov    %eax,-0x14(%rbp)
   0x00005555555546fd <+87>:	mov    $0x0,%eax
   0x0000555555554702 <+92>:	leaveq 
   0x0000555555554703 <+93>:	retq   
End of assembler dump.
(gdb) disassemble add
Dump of assembler code for function add:
   0x0000555555554660 <+0>:	push   %rbp
   0x0000555555554661 <+1>:	mov    %rsp,%rbp
   0x0000555555554664 <+4>:	mov    %edi,-0x14(%rbp)
   0x0000555555554667 <+7>:	mov    %esi,-0x18(%rbp)
   0x000055555555466a <+10>:	mov    %edx,-0x1c(%rbp)
   0x000055555555466d <+13>:	mov    %ecx,-0x20(%rbp)
   0x0000555555554670 <+16>:	mov    %r8d,-0x24(%rbp)
   0x0000555555554674 <+20>:	mov    %r9d,-0x28(%rbp)
=> 0x0000555555554678 <+24>:	mov    -0x14(%rbp),%edx
   0x000055555555467b <+27>:	mov    -0x18(%rbp),%eax
   0x000055555555467e <+30>:	add    %eax,%edx
   0x0000555555554680 <+32>:	mov    -0x1c(%rbp),%eax
   0x0000555555554683 <+35>:	add    %eax,%edx
   0x0000555555554685 <+37>:	mov    -0x20(%rbp),%eax
   0x0000555555554688 <+40>:	add    %eax,%edx
   0x000055555555468a <+42>:	mov    -0x24(%rbp),%eax
   0x000055555555468d <+45>:	add    %eax,%edx
   0x000055555555468f <+47>:	mov    -0x28(%rbp),%eax
   0x0000555555554692 <+50>:	add    %eax,%edx
   0x0000555555554694 <+52>:	mov    0x10(%rbp),%eax
   0x0000555555554697 <+55>:	add    %eax,%edx
   0x0000555555554699 <+57>:	mov    0x18(%rbp),%eax
   0x000055555555469c <+60>:	add    %edx,%eax
   0x000055555555469e <+62>:	mov    %eax,-0x4(%rbp)
   0x00005555555546a1 <+65>:	mov    -0x4(%rbp),%eax
   0x00005555555546a4 <+68>:	pop    %rbp
   0x00005555555546a5 <+69>:	retq   
End of assembler dump.
```
- show language : 查看当前的语言环境. 如果GDB不能识为你所调试的编程语言，那么C语言被认为是默认的环境
- info source : 查看程序的信息
- set language : 如果GDB没有检测出当前的程序语言，那么也可以手动设置当前的程序语言. 其不加参数时即等于`-h`