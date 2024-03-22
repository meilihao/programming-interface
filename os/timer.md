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
编程语言里的定时器最终依赖于硬件定时器来实现, 内核在时钟中断发生后检测各定时器是否到期, 到期后的定时器处理函数将作为软中断在下半部指向. 本质是, 时钟中断处理程序会唤起TIMER_SOFTIRQ软中断, 运行当前处理器上到期的所有定时器.

在kernel中, timer_list表示一个定时器, 通过`init_timer/TIMER_INITIALIZER()/DEFINE_TIMER/setup_timer`初始化定时器; add_timer()将定时器加入内核动态定时器链表中; del_timer()删除定时器, del_timer_sync()是del_timer()的同步版本, 其会等待它被处理完, 因此不能用在中断上下文; mod_timer()修改定时器的到期时间, 定时器的到期时间往往是当前jiffies的基础上加上一个时延.

> hrtimer(高精度定时器)范例见`sound/soc/fsl/imx-pcm-fiq.c`, 它有上述类似API. 

LINUX内核定时器是内核用来控制在未来某个时间点（基于jiffies）调度执行某个函数的一种机制. 其实现位于 [`<linux/timer.h>`(即`include/linux/timer.h`)](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/timer.h) 和 [kernel/time/timer.c](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/time/timer.c) 文件中。被调度的函数肯定是异步执行的，它类似于一种"软件中断"，而且是处于非进程的上下文中，所以调度函数必须遵守以下规则：
1. 没有 current 指针、不允许访问用户空间. 因为没有进程上下文，相关代码和被中断的进程没有任何联系
1. 不能执行休眠（或可能引起休眠的函数）和调度
1. 任何被访问的数据结构都应该针对并发访问进行保护，以防止竞争条件

内核定时器的调度函数运行过一次后就不会再被运行了（相当于自动注销），但可以通过在被调度的函数中重新调度自己来周期运行.

## 内核中延迟的工作delayed_work
delayed_work基于工作队列和定时器实现.

## 内核延时
### 短延时
1. ndelay() // 纳秒
2. udelay() // 微妙
3. mdelay() // 毫秒

	对于毫秒及以上一般不直接使用mdelay(), 避免cpu资源浪费. 通常使用(这些接口受进程调度影响, 精度有限):
	- 可打断:

		1. msleep_interruptible()
	- 不可打断:

		1. msleep()
		1. ssleep()

本质是忙等待, 它根据cpu频率进行一定次数的循环.

内核启动时, 会运行一个延迟循环校准(Delay Loop Calibration), 计算出lpj(Loops Per Jiffy).

### 长延时
time_before()/time_after(), 本质是比较两次jiffies的差值

### 睡眠延时
睡眠延时明显比忙等待好, cpu资源可被其他资源使用. schedule_timeout()可使当前任务休眠至指定的jiffies之后被重新调度, 它的实现原理是向系统添加一个定时器, 在定时器处理函数中唤醒与参数对应的进程.

msleep()/msleep_interruptible()本质是依靠包含了schedule_timeout()的schedule_timeout_uninterruptible()和schedule_timeout_interruptible()实现的. schedule_timeout_[un]interruptible()的区别是调用schedule_timeout()前设置进程状态为TASK_INTERRUPTIBLE/TASK_UNINTERRUPTIBLE.