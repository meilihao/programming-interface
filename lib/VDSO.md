# vDSO
Linux 虚拟动态共享库(virtual Dynamic Shared Object, vDSO)技术, 这种技术允许在用户空间执行内核代码, 通过`ldd`可查看程序是否引用了`linux-vdso.so.1`来确定其是否使用了该技术.

> vDSO 的代码位于 arch/x86/vdso/，由一些汇编、C 和一个连接器脚本组成.

> vdso在kernel编译后存在`arch/x86/entry/vdso`

> `linux-vdso.so.1`不存在对应的文件, 可通过dump memory获取到, 见FAQ中的`提取vDSO`

Linux vDSO 是一段内核代码，但映射到用户空间，因而可以被用户空间程序直接调用. 其设计思想就是部分系统调用无需用户程序进入内核就可以调用，比如gettimeofday.

> 内核会在程序加载时将 vDSO 地址写入 ELF 头，这就是为什么 vDSO 的地址永远出现在 AT_SYSINFO_EHDR 和 AT_SYSINFO 的原因. 在 glibc 下，可以使用参数 AT_SYSINFO_EHDR 通过 getauxval() 获取 vDSO map 后的内存起始地址，再通过解析 ELF 符号表得到对应的函数指针, 还有一个辅助向量 AT_SYSINFO，它是系统调用函数在 vDSO 的入口地址，用于解决 CPU 的差异导致的系统调用指令不同的问题.

> [自 v4.5 起 Linux 采用了缺页中断的方式加载 vdso 的内容](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f872f5400cc01373d8e29d9c7a5296ccfaf4ccf3). map vdso 后，会设置 ELF 的[辅助向量](https://elixir.bootlin.com/linux/v5.0/source/fs/binfmt_elf.c#L245).

> vDSO不走syscall, 因此没法被strace追踪到.

![ARM64 下的 vDSO 原理图](/misc/img/lib/anatony-of-the-vDSO-on-arm64.png)
## vdso/vsyscall
参考:
- [gettimeofday的几种实现方法](https://cloud.tencent.com/developer/article/1400209)

vsyscall用来执行特定的系统调用，减少系统调用的开销. 某些系统调用并不会向内核提交参数，而仅仅只是从内核里请求读取某个数据，例如gettimeofday()，内核在处理这部分系统调用时可以把系统当前时间写在一个固定的位置(由内核在每个时间中断里去完成这个更新动作)，mmap映射到用户空间. 这样会更快速，避免了传统系统调用模式INT 0x80/SYSCALL造成的内核空间和用户空间的上下文切换.

vsyscall和vDSO是用于加速某些系统调用的两种机制. vDSO用来替换vsyscall.

传统的int 0x80有点慢, Intel和AMD分别实现了sysenter/sysexit和syscall/ sysret, 即所谓的快速系统调用指令, 使用它们更快, 但是也带来了兼容性的问题. 于是Linux实现了vsyscall, 程序统一调用vsyscall, 具体的选择由内核来决定. 而vsyscall的实现就在VDSO中.

vsyscall的局限:
1. 分配的内存较小
1. 只允许4个系统调用
1. vsyscall页面在每个进程中是静态分配了相同的地址, 有潜在的安全风险

    基地址：0xffffffffff600000
    - `#define VSYSCALL_ADDR_vgettimeofday   0xffffffffff600000`
    - `#define VSYSCALL_ADDR_vtime           0xffffffffff600400`
    - `#define VSYSCALL_ADDR_vgetcpu         0xffffffffff600800`

> kernel出于兼容性的考虑保留了 vsyscall, 但[glibc 在 v2.22 移除了对 vsyscall 的支持](https://sourceware.org/git/?p=glibc.git;a=commit;h=7cbeabac0fb28e24c99aaa5085e613ea543a2346)
 
vDSO提供和vsyscall相同的功能，同时解决了其局限:
1. vDSO是动态分配的，地址是随机的, 即利用了 ASLR（address space layout randomization）增强安全性
1. 可以提供超过4个系统调用, 比如`5.4.70`上是提供了5个.

性能测试:
```c
// from https://github.com/pizhenwei/tool/blob/master/gettimeofday-bench.c
// gcc gettimeofday-bench.c -o time
#define _GNU_SOURCE         /* See feature_test_macros(7) */
#include <unistd.h>
#include <sys/syscall.h>
#include <sys/time.h>
#include <stdio.h>

#define LOOP 10000000

static inline unsigned long rdtsc(void)
{
        unsigned long low, high;

        asm volatile("rdtsc" : "=a" (low), "=d" (high) );

        return ((low) | (high) << 32);
}

void show_tv(struct timeval *tv, char *prefix)
{
        printf("[%s]sec = %ld, usec = %ld\n", prefix, tv->tv_sec, tv->tv_usec);
}

int main()
{
        long loop;
        struct timeval tmp;
        unsigned long start, end, elapsed;
        int (*__gettimeofday)(struct timeval *, struct timezone *);
        __gettimeofday = (int (*)(struct timeval *, struct timezone *))0xffffffffff600000;

        gettimeofday(&tmp, NULL);
        show_tv(&tmp, "VDSO");

        syscall(SYS_gettimeofday, &tmp, NULL);
        show_tv(&tmp, "SYSCALL");

        __gettimeofday(&tmp, NULL);
        show_tv(&tmp, "VSYSCALL");

        start = rdtsc();
        for (loop = 0; loop < LOOP; loop++) {
                gettimeofday(&tmp, NULL);
        }
        end = rdtsc();

        elapsed = end - start;
        printf("VDSO gettimeofday test %ld cycles, everage %ld cycles\n", elapsed, elapsed / LOOP);

        start = rdtsc();
        for (loop = 0; loop < LOOP; loop++) {
                syscall(SYS_gettimeofday, &tmp, NULL);
        }
        end = rdtsc();

        elapsed = end - start;
        printf("SYSCALL gettimeofday test %ld cycles, everage %ld cycles\n", elapsed, elapsed / LOOP);

        start = rdtsc();
        for (loop = 0; loop < LOOP; loop++) {
                __gettimeofday(&tmp, NULL);
        }
        end = rdtsc();

        elapsed = end - start;
        printf("VSYSCALL gettimeofday test %ld cycles, everage %ld cycles\n", elapsed, elapsed / LOOP);

        return 0;
}
```

测试结果:
```
[VDSO]sec = 1608611084, usec = 296151
[SYSCALL]sec = 1608611084, usec = 296266
[VSYSCALL]sec = 1608611084, usec = 296270
VDSO gettimeofday test 624122992 cycles, everage 62 cycles
SYSCALL gettimeofday test 15983063410 cycles, everage 1598 cycles
VSYSCALL gettimeofday test 28437038884 cycles, everage 2843 cycles
```

因此使用vDSO即可.

## FAQ
### 查看vsyscall/VDSO内存映射
`cat /proc/`pgrep code | head -1`/maps | egrep 'vdso|vsyscall'`

### 提取vDSO
```bash
$ cat gettimeofday.c
#include <sys/time.h>
#include <stdio.h>

void show_tv(struct timeval *tv, char *prefix)
{
	printf("[%s]sec = %ld, usec = %ld\n", prefix, tv->tv_sec, tv->tv_usec);
}

int main()
{
	struct timeval tmp;

	printf("p=%p\n", tmp);
	gettimeofday(&tmp, NULL);
	show_tv(&tmp, "VDSO");

	return 0;
}
$ gcc -ggdb gettimeofday-bench.c -o out
$ gdb -q out
...
>>> b main
>>> r
>>> info program
    Using the running image of child process 15883.
...
>>> shell cat /proc/15883/maps | grep vdso
7ffff7fd4000-7ffff7fd5000 r-xp 00000000 00:00 0                          [vdso]
>>> dump memory vdso.so 0x7ffff7fd4000 0x7ffff7fd5000
>>> q
$ objdump -T vdso.so # print vdso symbols

vdso.so：     文件格式 elf64-x86-64

DYNAMIC SYMBOL TABLE:
00000000000008f0  w   DF .text  00000000000000a8  LINUX_2.6   clock_gettime
0000000000000850 g    DF .text  0000000000000087  LINUX_2.6   __vdso_gettimeofday
00000000000009a0  w   DF .text  0000000000000005  LINUX_2.6   clock_getres
00000000000009a0 g    DF .text  0000000000000005  LINUX_2.6   __vdso_clock_getres
0000000000000850  w   DF .text  0000000000000087  LINUX_2.6   gettimeofday
00000000000008e0 g    DF .text  0000000000000010  LINUX_2.6   __vdso_time
00000000000008e0  w   DF .text  0000000000000010  LINUX_2.6   time
00000000000008f0 g    DF .text  00000000000000a8  LINUX_2.6   __vdso_clock_gettime
0000000000000000 g    DO *ABS*  0000000000000000  LINUX_2.6   LINUX_2.6
00000000000009b0 g    DF .text  000000000000002a  LINUX_2.6   __vdso_getcpu
00000000000009b0  w   DF .text  000000000000002a  LINUX_2.6   getcpu
```

> 无前缀`__vdso_`的函数名是相应函数的弱符号别名（GCC 属性：`__attribute__ ((weak, alias ("<alias name>")));`）