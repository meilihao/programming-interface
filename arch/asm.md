# asm
## 数据对齐
计算机系统对基本数据类型的合法地址做出限制: 其地址必须是某个K(通常是2, 4, 8)的倍数, 该设计简化了cpu与memory间的硬件接口设计, 提高内存系统的性能.

## asm指令
操作数类型:
- 立即数,即常量
- 寄存器
- 内存引用

![操作数格式](/images/register_value_format.png)

`subq $8 %rsp // align stack frame`: 为了让栈顶(%rsp)16位对齐, 因为在64位Linux机器上，要求函数调用前%rsp是16位对齐的, 阅读时可忽略, 可通过GNU的`-mpreferred-stack-boundary`选项调整.

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
![算术和逻辑操作](/images/register_integer_operate.png)

sal和shl是一样的，因为左移不会涉及符号位
计算时会设置eflags.

### 加载有效地址
leaq指令 ： leaq Src, Dst : 直接将有效地址（即：把括号内的值，不读入对应内存的数据）写入到目的

    leaq 7(%rdi, %rsi, 4), %rax // offset(base, index, width) = %rsi * 4 + %rdi + 7

### 特殊算术操作
![特殊算术操作](/images/register_special_value.png)

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

![jump指令](/images/asm_jmp.png)

用条件传送实现的条件分支比用条件控制实现的高效: 现代硬件对条件传送的分支预测更准确.

![条件传送](/images/asm_cmovX.png)

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