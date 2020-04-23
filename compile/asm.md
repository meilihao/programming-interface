# asm
## 语法
### AT&T汇编和Intel汇编
gcc, objdump默认使用AT&T格式的汇编, 也叫GAS格式(Gnu ASembler GNU汇编器)；microsoft工具和intel的文档使用intel格式的汇编, [两者区别](http://timothyqiu.com/archives/difference-between-att-and-intel-asm-syntax/):
1. 书写: AT&T 汇编格式要求关键字必须小写; Intel则必须大写
1. 寄存器命名: 在 AT&T 汇编格式中要加上 '%' 作为前缀；Intel则不加
1. 源/目的操作数顺序: AT&T 语法先写源操作数，再写目标操作数；Intel则相反. 快速记忆: 记目标操作数在哪个位置即可.
1. 常数/立即数: AT&T 语法要在常数前加 `$`; Intel则不加
1. 操作数长度标识: AT&T 语法将操作数的大小表示在指令的后缀中（b、w、l, 分别是1,2,4字节）；Intel 语法将操作数的大小表示在操作数的前缀中（BYTE PTR、WORD PTR、DWORD PTR）
1. 内存寻址方式: AT&T 语法总体上是section:offset(base, index, width)的格式; Intel 语法总体上是section:[INDEX * WIDTH + BASE + OFFSET]的格式

    section用于指定段寄存器.
1. jump和call: 远跳转指令, 远调用指令, 远返回的操作码，在 AT&T 汇编格式中为 "ljump", "lcall",`lret`; 而在 Intel 汇编格式中则为 "jmp far", "call far", `ret`

    AT&T:

        lcall $section:$offset
        ljmp $section:$offset
        lret

    Intel:

        CALL FAR SECTION:OFFSET ; 目标地址由段地址和段内偏移组成
        JMP FAR SECTION:OFFSET ; 同上
        RET

> AT&T在变量标识符前加`$`表示引用其地址

### 函数调用约定
- stdcall

    1. 在进行函数调用的时候，函数的参数是从右向左依次放入栈中的

    如`int function（int first，int second）`, 这个函数的参数入栈顺序，首先是参数second，然后是参数first.

    2. 函数的栈平衡操作是由被调用函数执行的，使用的指令是 retn X，它可在函数返回时从栈中弹出X个字节. 例如上面的function函数，当我们把function的函数参数压入栈中后，当function函数执行完毕后，由function函数负责将传递给它的参数first和second从栈中弹出来.

    3. 编译时, 编译器会在函数名的前面用下划线修饰，在函数名的后面由`@+入栈字节数`来修饰. 如上面的function函数，会被编译器转换为_function@8.

- cdecl

    1. 在进行函数调用的时候，和stdcall一样，函数的参数是从右向左依次放入栈中的.

    2. 函数的栈平衡操作是由调用函数执行的，这点是与stdcall不同之处. stdcall使用retn X平衡栈，cdecl则使用leave、pop、向上移动栈指针等方法来平衡栈.

    3. 每个函数调用者都包含有清空栈的代码，所以编译产生的可执行文件会比调用stdcall约定产生的文件大.

    cdecl是GCC的默认调用约定. 但是，GCC在x64位系统环境下，却使用寄存器作为函数调用的参数, 按照从左向右的顺序依次将前六个整型参数放在寄存器RDI, RSI, RDX, RCX, R8和R9上，同时XMM0到XMM7用来保存浮点变量，而用RAX保存返回值，且由调用者负责平衡栈.

- fastcall

    1. 函数参数尽可能使用通用寄存器rcx, rdx来传递前两个int类型的参数或较小的参数, 其余参数按照从右向左的顺序入栈.

    2. 函数的栈平衡操作是由被调用函数负责.

还有很多调用规则，如：thiscall、naked call、pascal等

### 参数传递方式

函数参数的传递方式无外乎两种: 一种是通过寄存器传递，另一种是通过内存传递. 这两种传递方式在通常情况下都能满足开发需求, 因此它并不会被特别关注. 但是, 在写操作系统时有许多要求苛刻的场景，这使得我们不得不掌握这两种参数传递方式.

- 寄存器传递

    寄存器传递就是通过寄存器来传递函数的参数, 优点是执行速度快，编译后生成的代码量少. 但只有少部分调用约定默认是通过寄存器传递参数，绝大部分编译器是需要特殊指定使用寄存器传递参数的.

    在X86体系结构的linux kernel中，syscall一般会使用寄存器传递. 因为应用程序和kernel的空间是隔离的. 如果想从应用层把参数传递到内核层的话，最方便快捷的方法是通过寄存器传递参数，否则只能大费周章才能把数据传递过去.

- 内存传递

    内存传递参数很好理解，在大多数情况下参数传递都是通过压栈的形式实现的, 比如go的函数调用.

    在X86体系结构下的Linux内核中，中断或异常的处理会使用内存传递参数. 因为，在中断产生后，到中断处理的上半部，中间的过渡代码是用汇编实现的. 汇编跳转到C语言的过程中，C语言函数是使用栈来传递参数的，为了无缝衔接，汇编就需要把参数压入栈中，然后再跳转到C语言实现的中断处理函数中执行.

    > linux 2.6开始逐渐改用寄存器传递.

以上这些都是在X86体系结构下的参数传递方式. 而**在X64体系结构下，大部分编译器都使用的是寄存器传递参数**.

### NASM
nasm是可以在windows、linux等系统下使用的汇编器, 而masm是微软专门为windows下汇编而写的，故而**推荐使用nasm**.

nasm注意点:
1. 它使用Intel语法
1. 区分大小写
1. 任何不被方括号[]括起来的标签或者变量名都被当作地址，访问其内容必须用[]包裹
1. `$`表示当前行被编译后的地址即当前行的偏移地址.

    `jmp $`: 表示死循环, 因为跳到当前位置后还是`jmp $`. 具体汇编解释: `jmp $`转化成机器码是E9 FD FF，其中E9的意思是jmp，FD FF是个地址，但是在x86里面是小端排列的，所以要将数值转换为地址：FFFD，其实就是-3，这个jmp是相对跳转，跳转的地址就是执行完这条命令后，指令寄存器-3的地址，正好这条指令的长度就是3个字节，所以，又回到了这条指令重新执行.
1. `$$`表示一个节（section）的开始处被编译后的地址, 就是这个节的起始地址.

    一般写汇编程序的时候，使用一个section就够了，只有在写复杂程序的时候，才会用到多个section. section既可以是数据段，也可以是代码段. 所以，如果把section比喻成函数，还是不太恰当.

    `$-$$`表示本行程序距离节（section）开始处的相对距离. 如果只有一个节（section）的话，那么他就表示本行程序距离程序头的距离.

> as是GNU Binutils下的汇编器, 使用AT&T语法.

### tools
```
# objdump -d a.out # 默认以AT&T语法输出
# objdump -d -m i386:x86-64:intel a.out # 以Intel语法输出. i386即Intel 80386, 但i386通常被用来作为对Intel（英特尔）32位微处理器的统称.
# objdump -d -M intel a.out # 也是以Intel语法输出
# gcc -S m.c # 默认以AT&T语法输出
# gcc -S -masm=intel m.c # 以Intel语法输出, `-masm`表示以哪种asm方言进行输出
```

example:
```
$ vim t.c
int test()
{
    int i = 0;
    i =  1 + 2;
    return i;
}

int main()
{
    test();
    return 0;
}
$ objdump -d -M intel a.out # 以Intel输出
...
0000000000001129 <test>:
    1129:   f3 0f 1e fa             endbr64 
    112d:   55                      push   rbp ; 保存当前栈的栈底
    112e:   48 89 e5                mov    rbp,rsp ; 栈底, 栈顶是同一个位置
    1131:   c7 45 fc 00 00 00 00    mov    DWORD PTR [rbp-0x4],0x0 ; 一个WORD是2B, 因此DWORD是4B, 这里是分配一个4B的空间保存i的值, 即`int i = 0`
    1138:   c7 45 fc 03 00 00 00    mov    DWORD PTR [rbp-0x4],0x3 ; i =  1 + 2
    113f:   8b 45 fc                mov    eax,DWORD PTR [rbp-0x4] ; eax保存返回值3
    1142:   5d                      pop    rbp ; `pop rbp` = `mov rbp, QWORD PTR [rsp]` + `add rsp,0x8`
    1143:   c3                      ret    ; ret =  pop rip = `mov rip, QWORD PTR [rsp]` + `add rsp,0x8`

0000000000001144 <main>: ; `0000000000001144`的长度是64 bit, 在x86_64下面，其实虚拟地址只使用了48位, 对应了256TB的地址空间, 通常已够用.
    1144:   f3 0f 1e fa             endbr64 
    1148:   55                      push   rbp
    1149:   48 89 e5                mov    rbp,rsp
    114c:   48 83 ec 10             sub    rsp,0x10
    1150:   b8 00 00 00 00          mov    eax,0x0
    1155:   e8 cf ff ff ff          call   1129 <test> ; call会将下一条指令地址115a压入栈中作为调用的返回地址, 然后跳到`func test`执行. `call 1129 <test>` = `push QWORD 115a` + `jmp 1129 <test>` = `sub rsp,0x8`(栈向低地址生长) + `mov QWORD PTR [rsp], 115a` + `jmp 1129 <test>`
    115a:   89 45 fc                mov    DWORD PTR [rbp-0x4],eax
    115d:   b8 00 00 00 00          mov    eax,0x0
    1162:   c9                      leave  
    1163:   c3                      ret    
    1164:   66 2e 0f 1f 84 00 00    nop    WORD PTR cs:[rax+rax*1+0x0]
    116b:   00 00 00 
    116e:   66 90                   xchg   ax,ax
...
$ objdump -d a.out # 以AT&T输出
...
0000000000001129 <test>:
    1129:   f3 0f 1e fa             endbr64 
    112d:   55                      push   %rbp
    112e:   48 89 e5                mov    %rsp,%rbp
    1131:   c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%rbp)
    1138:   c7 45 fc 03 00 00 00    movl   $0x3,-0x4(%rbp)
    113f:   8b 45 fc                mov    -0x4(%rbp),%eax
    1142:   5d                      pop    %rbp
    1143:   c3                      retq   

0000000000001144 <main>:
    1144:   f3 0f 1e fa             endbr64 
    1148:   55                      push   %rbp
    1149:   48 89 e5                mov    %rsp,%rbp
    114c:   48 83 ec 10             sub    $0x10,%rsp
    1150:   b8 00 00 00 00          mov    $0x0,%eax
    1155:   e8 cf ff ff ff          callq  1129 <test>
    115a:   89 45 fc                mov    %eax,-0x4(%rbp)
    115d:   b8 00 00 00 00          mov    $0x0,%eax
    1162:   c9                      leaveq ; leaveq = movq %rbp, %rsp; popq %rbp
    1163:   c3                      retq   
    1164:   66 2e 0f 1f 84 00 00    nopw   %cs:0x0(%rax,%rax,1)
    116b:   00 00 00 
    116e:   66 90                   xchg   %ax,%ax
...
```


## 数据对齐
计算机系统对基本数据类型的合法地址做出限制: 其地址必须是某个K(通常是2, 4, 8)的倍数, 该设计简化了cpu与memory间的硬件接口设计, 提高内存系统的性能.

## asm指令
操作数类型:
- 立即数,即常量
- 寄存器
- 内存引用

![操作数格式](/misc/img/register_value_format.png)

`subq $8 %rsp // align stack frame`: 为了让栈顶(%rsp)16位对齐, 因为在64位Linux机器上，要求函数调用前%rsp是16位对齐的, 这是由arch的ABI(Application Bianry nterface)对齐要求决定的, 阅读时可忽略, 也可通过GNU的`-mpreferred-stack-boundary`选项调整(**不推荐**).

### 数据传送
参考:
- [简单的汇编学习笔记与总结](http://rinchannow.club/2018/11/08/%E7%AE%80%E5%8D%95%E7%9A%84%E6%B1%87%E7%BC%96%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E4%B8%8E%E6%80%BB%E7%BB%93/)

- mov(b|w|l|q|absq) : 传送指定长度数据
    b;w;l;q = B/2B/4B/8B

    x64中保存１～２字节到寄存器时会保持其他的字节不变; 保存４字节时指令会把高位４字节置为０.

    movq与movabsq:
    - movq : 只能以表示为32位补码数字的立即数作为源操作数, 然后将该值符号扩展到64位后再放到目的
    - movabsq : 能够以任意64位立即数作为源操作数, 但只能以寄存器作为目的

- movz(xx) : 将较小的源值复制到较大的目的时使用, 但使用零扩展（剩余填充0）
- movs(xx) : 将较小的源值复制到较大的目的时使用, 但使用符号扩展（剩余填充符号位）
    特殊:
    - cltq : 把%eax符号扩展到%rax

### 压入和弹出栈数据
push src ：把数据压入栈中: `subq $8, %rsp` + `movq src, (%rsp)`
pop dst ：弹出栈顶数据: `movq (%rsp), dst`+`addq $8, %rsp`

```
pushq	%rbp  // 保存调用者的%rbp = `subq $8, %rsp`(分配栈空间) + `movq %rbp, (%rsp)`(向分配的栈空间写入%rbp). 因为栈是从高地址到低地址增长, %rsp-=8后向%rsp处写入数据即是为其赋值.
popq	%rbp  // = `movq (%rsp), %rbp`(读数据)+`addq $8, %rsp`
```

### 算术和逻辑操作
![算术和逻辑操作](/misc/img/register_integer_operate.png)

sal和shl是一样的，因为左移不会涉及符号位
计算时会设置eflags.

### 加载有效地址
leaq指令 ： leaq Src, Dst : 直接将有效地址（即：把括号内的值，不读入对应内存的数据）写入到目的

    leaq 7(%rdi, %rsi, 4), %rax // offset(base, index, width) = %rsi * 4 + %rdi + 7

### 特殊算术操作
![特殊算术操作](/misc/img/register_special_value.png)

mulq/imulq(乘法)要求一个参数必须在%rax中, 另一个数是源操作数, 将乘积的高64位存在%rdx中，低64位存在%rax中.
divq/idivq(除法)会把R[%rdx]:R[%rax]作为被除数（128位），S为除数，将结果的商存在%rax中，余存在%rdx中

### 比较和控制指令
这两种指令不修改任何寄存器的值，只设置eflags.

- CMP (cmpb, cmpw, cmpl, cmpq)
    CMP S1, S2：就是计算S2 - S1,它与`SUB`指令的行为一致，以设置条件码得以看出比较的结果
    CF = 1: 发生了进位或借位（这里做减法一般是借位，借位了就表明S2 < S1）
    ZF = 1: S1 = S2
    SF = 1: S2 - S1 < 0（补码运算意义上的）
    OF = 1: (a > 0 && b < 0 && (a - b) < 0) || (a < 0 && b > 0 && (a - b) > 0)
- TEST (testb, testw, testl, testq)
    TEST S1, S2：就是计算S1 & S2, 它与`AND`指令的行为一致，以设置条件码
    ZF = 1: S1 & S2 = 0
    SF = 1: S1 & S2 < 0（补码运算意义上的）
    经常使用这个指令测试一个数是不是负数：testq %rax, %rax

### set指令
SET类的指令可以将一个字节的值设置为条件码的某种组合，这种指令的目的操作数是低位单字节寄存器之一或一个字节的内存位置（如%al），一般是配合比较和测试指令使用，下面列出常用的SET类指令：

指令	同义名	效果	设置条件
sete D	setz	D <– ZF	相等/零
setne D	setnz	D <– ~ZF	不等/非零
sets D		    D <– SF	负数
setns D		    D <– ~SF	非负数
setg D	setnle	D <– ~(SF ^ OF) & ~ZF	有符号> (greater)
setge D	setnl	D <– ~(SF ^ OF)	有符号 >=
setl D	setnge	D <– SF ^ OF	有符号<
setle D	setng	D <– (SF ^ OF) | ZF	有符号<=
seta D	setnbe	D <– ~CF & ~ZF	无符号> (above)
setae D	setnb	D <– ~CF	无符号>=
setb D	setnae	D <– CF	无符号< (below)
setbe D	setna	D <– CF | ZF	无符号<=

### 跳转指令
`jmp`切换到程序的另一个位置开始执行, 常与lable联用

![jump指令](/misc/img/asm_jmp.png)

用条件传送实现的条件分支比用条件控制实现的高效: 现代硬件对条件传送的分支预测更准确.

![条件传送](/misc/img/asm_cmovX.png)

### 转移控制
call + ret

## 指令集
- simd最新是avx版本.

## 扩展
### plan9
参考:
- [plan9 assembly 入门](https://gocn.vip/article/733)

在plan9汇编里还可以直接使用的amd64的通用寄存器，应用代码层面会用到的通用寄存器主要是: rax, rbx, rcx, rdx, rdi, rsi, r8~r15这14个寄存器，虽然rbp和rsp也可以用，不过bp和sp会被用来管理栈顶和栈底，最好不要拿来进行运算. plan9中使用寄存器不需要带r或e的前缀，例如rax，只要写AX即可.

通用通用寄存器的名字在 IA64 和 plan9 中的对应关系:
	X86_64	rax	rbx	rcx	rdx	rdi	rsi	rbp	rsp	r8	r9	r10	r11	r12	r13	r14	rip
	Plan9	AX	BX	CX	DX	DI	SI	BP	SP	R8	R9	R10	R11	R12	R13	R14	PC

plan9汇编指令与其他arch指令的映射关系在`/usr/local/go/src/cmd/asm/internal/arch/arch.go`里.