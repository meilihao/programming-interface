# lock
## FAQ
### soft lockup
Soft Lockup 是 Linux 内核中的一种警告，表示某个 CPU 核心在内核模式下长时间未响应调度. 这通常是由于某个内核任务（如内核线程、中断处理程序或系统调用）占用了 CPU 核心，导致其他任务无法运行.