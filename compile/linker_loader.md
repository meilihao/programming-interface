# linker and loader
linker和loader的最基本作用: 将更抽象的名字和更底层的名字绑定起来, 让人可使用更抽象的名字来编写程序.

因为程序共享内存的需要(通过os), 在操作系统将程序加载到内存之前是无法确定程序运行的确切地址的,并将最终的地址绑定从链接时推延到了加载时.

硬件体系有两个方面影响了链接器: 程序寻址和指令格式, 因为ld需要对数据和指令中的地址及偏移量做修正.

ABI(Application Binary Interface)会影响ld的过程调用标准(Procedure Call Standard), 比如arm的APCS.

> ABI定义了程序间二进制可移植性的标准.

```bash
$ cat m.c
extern void a(char *);
int main(int ac, char **av)
{
static char string[] = "Hello, world!\n";
a(string);
}
$ cat a.c
#include <unistd.h>
#include <string.h>
void a(char *s)
{
        write(1, s, strlen(s));
}
$ gcc -c m.c # 仅编译不链接
$ objdump -h m.o

m.o：     文件格式 elf64-x86-64

节：
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000026  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         0000000f  0000000000000000  0000000000000000  00000068  2**3 # string:"Hello, world!\n" 共15B
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  00000077  2**0
                  ALLOC
...
$ objdump -d -M intel m.o

m.o：     文件格式 elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:	f3 0f 1e fa          	endbr64 
   4:	55                   	push   rbp
   5:	48 89 e5             	mov    rbp,rsp
   8:	48 83 ec 10          	sub    rsp,0x10
   c:	89 7d fc             	mov    DWORD PTR [rbp-0x4],edi
   f:	48 89 75 f0          	mov    QWORD PTR [rbp-0x10],rsi
  13:	48 8d 3d 00 00 00 00 	lea    rdi,[rip+0x0]        # 1a <main+0x1a>
  1a:	e8 00 00 00 00       	call   1f <main+0x1f> # 没有具体地址, 仅有下一条指令地址
  1f:	b8 00 00 00 00       	mov    eax,0x0
  24:	c9                   	leave  
  25:	c3                   	ret                 	retq
$ gcc -c a.c
$ objdump -h a.o

a.o：     文件格式 elf64-x86-64

节：
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000033  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000000  0000000000000000  0000000000000000  00000073  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  00000073  2**0
                  ALLOC
...
$ objdump -d -M intel a.o

a.o：     文件格式 elf64-x86-64


Disassembly of section .text:

0000000000000000 <a>:
   0:	f3 0f 1e fa          	endbr64 
   4:	55                   	push   rbp
   5:	48 89 e5             	mov    rbp,rsp
   8:	48 83 ec 10          	sub    rsp,0x10
   c:	48 89 7d f8          	mov    QWORD PTR [rbp-0x8],rdi
  10:	48 8b 45 f8          	mov    rax,QWORD PTR [rbp-0x8]
  14:	48 89 c7             	mov    rdi,rax
  17:	e8 00 00 00 00       	call   1c <a+0x1c>
  1c:	48 89 c2             	mov    rdx,rax
  1f:	48 8b 45 f8          	mov    rax,QWORD PTR [rbp-0x8]
  23:	48 89 c6             	mov    rsi,rax
  26:	bf 01 00 00 00       	mov    edi,0x1
  2b:	e8 00 00 00 00       	call   30 <a+0x30>
  30:	90                   	nop
  31:	c9                   	leave  
  32:	c3                   	ret
$ gcc -o m.out a.c m.c
$ objdump -h m.out 

m.out：     文件格式 elf64-x86-64

节：
Idx Name          Size      VMA               LMA               File off  Algn
...
 15 .text         000001c5  0000000000001080  0000000000001080  00001080  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 16 .fini         0000000d  0000000000001248  0000000000001248  00001248  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 17 .rodata       00000004  0000000000002000  0000000000002000  00002000  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
...
 24 .data         0000001f  0000000000004000  0000000000004000  00003000  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 25 .bss          00000001  000000000000401f  000000000000401f  0000301f  2**0
                  ALLOC
...
$ objdump -d -M intel m.out
...
0000000000001169 <a>: # function a 被重定位到0x1169
    1169:	f3 0f 1e fa          	endbr64 
    116d:	55                   	push   rbp
    116e:	48 89 e5             	mov    rbp,rsp
    1171:	48 83 ec 10          	sub    rsp,0x10
    1175:	48 89 7d f8          	mov    QWORD PTR [rbp-0x8],rdi
    1179:	48 8b 45 f8          	mov    rax,QWORD PTR [rbp-0x8]
    117d:	48 89 c7             	mov    rdi,rax
    1180:	e8 eb fe ff ff       	call   1070 <strlen@plt>
    1185:	48 89 c2             	mov    rdx,rax
    1188:	48 8b 45 f8          	mov    rax,QWORD PTR [rbp-0x8]
    118c:	48 89 c6             	mov    rsi,rax
    118f:	bf 01 00 00 00       	mov    edi,0x1
    1194:	e8 c7 fe ff ff       	call   1060 <write@plt>
    1199:	90                   	nop
    119a:	c9                   	leave  
    119b:	c3                   	ret    

000000000000119c <main>:
    119c:	f3 0f 1e fa          	endbr64 
    11a0:	55                   	push   rbp
    11a1:	48 89 e5             	mov    rbp,rsp
    11a4:	48 83 ec 10          	sub    rsp,0x10
    11a8:	89 7d fc             	mov    DWORD PTR [rbp-0x4],edi
    11ab:	48 89 75 f0          	mov    QWORD PTR [rbp-0x10],rsi
    11af:	48 8d 3d 5a 2e 00 00 	lea    rdi,[rip+0x2e5a]        # 4010 <string.1916>
    11b6:	e8 ae ff ff ff       	call   1169 <a>
    11bb:	b8 00 00 00 00       	mov    eax,0x0
    11c0:	c9                   	leave  
    11c1:	c3                   	ret    
    11c2:	66 2e 0f 1f 84 00 00 	nop    WORD PTR cs:[rax+rax*1+0x0]
    11c9:	00 00 00 
    11cc:	0f 1f 40 00          	nop    DWORD PTR [rax+0x0]
...
```

## 加载
加载器将被加载内容放在内存x位置的三种方法:
1. 绝对加载

  被加载内容总被加载到内存的固定位置, 可在编译或汇编时完成. 该方法不利于程序共享内存.
  
  程序的变化也会导致地址的变化, 可用符号引用来解决.
1. 可重定位加载

  编译器或汇编器不使用绝对内存地址, 而是使用相对于某些已知起点(被加载内容的内存起点)的地址.
1. 动态运行时加载

  被加载内容的所有内存访问都以相对形式表示, 仅在指令运行时才计算绝对地址, 为不影响性能, 有硬件完成.

## 链接
动态链接(dynamic linking): 把某些外部模块的链接推迟到加载模块之后.

动态链接优势:
1. 便于升级
1. 便于共享

运行时动态链接(runtime dynamic linking): 把某些外部模块的链接推迟到运行时执行.

### 链接过程
链接的过程是由一个链接脚本（Linker Script）控制的，链接脚本决定了给每个段分配什么地址，如何对齐，哪个段在前，哪个段在后，哪些段合并到同一个Segment，另外链接脚本还要插入一些符号到最终生成的文件中，例如__bss_start、_edata、_end等. 如果用ld做链接时没有用-T选项指定链接脚本，则使用ld的默认链接脚本，默认链接脚本可以用`ld --verbose`命令查看.

ENTRY(_start)说明_start是整个程序的入口点，因此_start是入口点并不是规定死的，是可以改用其它函数做入口点的。

PROVIDE (__executable_start = 0x08048000); . = 0x08048000 + SIZEOF_HEADERS;是Text Segment的起始地址，这个Segment包含后面列出的那些段，.plt、.text、.rodata等等。每个段的描述格式都是“段名 : { 组成 }”，例如.plt : { *(.plt) }，左边表示最终生成的文件的.plt段，右边表示所有目标文件的.plt段，意思是最终生成的文件的.plt段由各目标文件的.plt段组成。

. = ALIGN (CONSTANT (MAXPAGESIZE)) - ((CONSTANT (MAXPAGESIZE) - .) & (CONSTANT (MAXPAGESIZE) - 1)); . = DATA_SEGMENT_ALIGN (CONSTANT (MAXPAGESIZE), CONSTANT (COMMONPAGESIZE));是Data Segment的起始地址，要做一系列的对齐操作，这个Segment包含后面列出的那些段，.got、.data、.bss等等。

Data Segment的后面还有其它一些Segment，主要是调试信息.