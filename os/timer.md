# timer
自版本 2.6.21 开始,Linux 内核可选择是否支持高分辨率定时器. 如果选择支持(通过内核配置选项 CONFIG_HIGH_RES_TIMERS),那么本章各种定时器以及休眠接口的的精度则不再受内核 jiffy(软件时钟周期)的影响,可以达到底层硬件所支持的精度(达到微秒甚至纳秒级).

将 clock_nanosleep()与 nanosleep()区分开来的特性在于,可以选择不同的时钟来测量休眠间隔时间, 以达到更高精度.

> 由 fork()创建的子进程不会继承 POSIX 定时器. 调用 exec()期间亦或进程终止时将停止并删除定时器.
> 调用 fork()期间,子进程会继承 timerfd_create()所创建文件描述符的拷贝

Linux 2.6 所实现的 POSIX.1b 扩展为高精度时钟和定时器定义了一套 API. POSIX.1b 定时器比传统(settimer())UNIX 定时器更具优势,可以:创建多个定时器;选择定时器到期时的通知信号;获取定时器溢出计数,以便判断自上次到期通知后定时器是否又发生了多次到
期;选择通过执行线程函数而非递送信号来获取定时器通知.

Linux 特有的 timerfd API 提供了一组创建定时器的接口,与 POSIX 定时器 API 相类似,但
允许从文件描述符中读取定时器通知, 还可使用 select()、poll()和 epoll()来监控这些描述符.

## kernel timer
LINUX内核定时器是内核用来控制在未来某个时间点（基于jiffies）调度执行某个函数的一种机制. 其实现位于 [`<linux/timer.h>`(即`include/linux/timer.h`)](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/timer.h) 和 [kernel/time/timer.c](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/time/timer.c) 文件中。被调度的函数肯定是异步执行的，它类似于一种"软件中断"，而且是处于非进程的上下文中，所以调度函数必须遵守以下规则：
1. 没有 current 指针、不允许访问用户空间. 因为没有进程上下文，相关代码和被中断的进程没有任何联系
1. 不能执行休眠（或可能引起休眠的函数）和调度
1. 任何被访问的数据结构都应该针对并发访问进行保护，以防止竞争条件

内核定时器的调度函数运行过一次后就不会再被运行了（相当于自动注销），但可以通过在被调度的函数中重新调度自己来周期运行.