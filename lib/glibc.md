# glibc
## glibc 系统调用 wrappers
对于很多系统调用，glibc 只用到了一个简单的 wrapper 程序：将参数放到合适的寄存器 ，然后执行 syscall 或 int $0x80 指令，或者调用 __kernel_vsyscall. 这个过程 用到了一系列的列表，这些列表的核心内容定义在几个文本文件里，然后被脚本文件处理之 后生成 C 代码.

> `sysdeps/unix/syscalls.list`文件描述了一些常规系统调用.
> 要了解每一列代表什么，请查看这个文件里的注释： [sysdeps/unix/make-syscalls.sh](https://github.molgen.mpg.de/git-mirror/glibc/blob/glibc-2.15/sysdeps/unix/make-syscalls.sh)