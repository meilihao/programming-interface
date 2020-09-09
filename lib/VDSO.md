# vDSO
Linux 虚拟动态共享库(virtual Dynamic Shared Object, vDSO)技术, 这种技术允许在用户空间 执行内核代码, 通过`ldd`可查看程序是否引用了`linux-vdso.so.1`来确定其是否使用了该技术.

> vDSO 的代码位于 arch/x86/vdso/，由 一些汇编、C 和一个连接器脚本组成.

> vdso在kernel编译后存在`arch/x86/entry/vdso`

Linux vDSO 是一段内核代码，但映射到 用户空间，因而可以被用户空间程序直接调用. 其设计思想就是部分系统调用无需用户程序 进入内核就可以调用，比如gettimeofday.

> 内核会在程序加载时将 vDSO 地址写入 ELF 头，这就是为什么 vDSO 的地址永远出现在 AT_SYSINFO_EHDR 和 AT_SYSINFO 的原因.