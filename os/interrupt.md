# interrupt
概念:
- 中断请求(irq, interrupt request)是由可编程中断控制器(PIC, programmable interrupt controller)发起的, 其目的是为了中断cpu和执行中断服务程序(isr, interrupt service routine).
- 硬件中断: 当一个硬件设备想要告诉 CPU 某一需要处理的数据已经准备好后（例如：当键盘被按下或者一个数据包到了网络接口处），它将会发送一个中断请求（IRQ）来告诉 CPU 数据是可用的, 接下来会调用在内核启动时设备驱动注册的对应的中断服务程序（ISR）.
- 软件中断: 当在播放一个视频时，音频和视频是同步播放是相当重要的，这样音乐的速度才不会变化. 这是由软件中断实现的，由精确的计时器系统（称为 jiffies）重复发起的. 这个计时器会使得音乐播放器同步, 软件中断也可以被特殊的指令所调用，来读取或写入数据到硬件设备. 当系统需要**实时性时（例如在工业应用中），软件中断会变得重要**. 可参考[这里](https://www.linuxfoundation.org/blog/2013/03/intro-to-real-time-linux-for-embedded-developers/).

os已注册的中断: `cat /proc/interrupts`, 从左到右各列的含义依次为：`中断向量号、每个 CPU（0~n）中断发生次数、硬件来源、硬件源通道信息、以及造成中断请求的设备名`. 在表的末尾，有一些非数字的中断, 它们是特定于体系结构的中断，如LOC(local timer interrupt, 本地计时器中断)的中断请求（IRQ）号为 236, 其中一些在 Linux 内核源树中的[Linux IRQ 向量布局](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/irq_vectors.h)中指定.

## cpu识别中断
x86处理器通过INTR和NMI 两个引脚分别接收外部中断请求信号: INTR接收可屏蔽中断, NMI接收不可屏蔽中断请求. 标志寄存器EFLAGS中的IF标志决定是否屏蔽可屏蔽中断请求.

在SMP结构中, 每个处理器都包含一个IOAPIC(高级可编程中断控制器, advanced programmable interrupt controller), 外部设备产生中断请求时, **通过该中断控制器并传递给cpu**.

> 在 MP 架构下，为支持将中断传递给多个 CPU，引入了 APIC，其包含 LAPIC 和 IOAPIC. 一般来说，所有 LAPIC 都连接到一个 I/O APIC 上，形成一个一对多的结构(不排除有多 IOAPIC 的架构), 有两种工作模式：
> 1. 8259A 模式: 禁用 LAPIC， APIC 直连 CPU
> 1. 标准模式: 启用 LAPIC，所有的外部中断通过 IOAPIC 接收后转发给对应的 LAPIC

有两个与IF标志相关的函数, local_irq_enable和local_irq_disable, 它们通过设置或清除本地cpu的EFLAGS的IF标志来控制是否使能本地中断.

## 中断处理程序
每个中断都可以有自己的中断处理程序, 这些程序集中存储构成了中断描述表(Interrupt Descriptor Table, IDT). 它包含了256个中断描述符, 用数组idt_table表示.

中断描述符由struct gate_desc表示, 有GATE_INTERRUPT, CATE_TRAP, GATE_CALL和GATE_TASK几种, idt_table就是gate_desc类型的数组. 系统启动时会为idt_table赋值, 然后将它的地址写入寄存器. 当中断发生时, cpu会根据该地址和中断号, 计算得到对应的中断处理程序的地址并执行.

在256个中断中, 前32个(`0x00~0x1F`)是x86预留的, X86_LOCAL_APIC使能的情况下, `0xec~255`供APIC使用, 其余的(非专用)则由os使用.

> 中断请求按照高级可编程中断控制器（APIC）中的优先级高低排序（0是最高优先级）.

kernel定义了多种专用的中断描述符, 多以idt_data数组的形式存在, 比如early_idts, def_idts, apic_idts, 系统初始化过程中会调用idt_setup_from_table将这些数组的元素信息转换为idt_table对应的元素信息.

内核在中断初始化的过程中可以给`0x20~0xeb`号中断指定具体的中断处理程序, 此时由system_vectors位图来表示一个中断是否已指定, 没有指定的会被设为默认值.

所有的中断处理程序必须即能正确处理中断, 有能保证处理完毕后可以返回中断前的程序继续执行. 因此, 中断处理必须完成3个任务:
1. 保存现场以便恢复
1. 调用已注册的中断服务例程(isr)处理中断
1. 恢复现场继续原有程序

中断处理程序和中断服务例程的区别: 中断处理程序包括中断处理的整个过程, 而中断服务例程是该过程中对产生中断的设备的处理逻辑. 并不是所有的中断处理程序都需要对应的中断服务例程, 中断服务例程是为了方便外设驱动利用内核提供的函数编程. **有了中断服务例程, dirver只需要专注处理设备自身的中断而不需要关系整个过程**.

## 中断服务例程
中断服务例程涉及两个关键的struct, 即irq_desc和irqaction, 两者是1:n的关系, 但并不是每个irq_desc都一定有与之对应的irqaction. irq_desc与irq对应, irqaction与一个设备对应, 共享同一个irq号的多个设备的irqaction对应同一个irq_desc.

![](/misc/img/os/interrupt/20190602232050759.png)

> irq_desc是irqaction单向链表的头.

```c
// https://elixir.bootlin.com/linux/v5.10.2/source/include/linux/interrupt.h#L94
/**
 * struct irqaction - per interrupt action descriptor
 * @handler:	interrupt handler function 中断处理函数
 * @name:	name of the device
 * @dev_id:	cookie to identify the device
 * @percpu_dev_id:	cookie to identify the device
 * @next:	pointer to the next irqaction for shared interrupts 将其链接到链表中
 * @irq:	interrupt number 与irq_desc对应的irq
 * @flags:	flags (see IRQF_* above)
 * @thread_fn:	interrupt handler function for threaded interrupts 在独立的线程中执行中断处理时, 真正处理中断的函数
 * @thread:	thread pointer for threaded interrupts 对应的线程, 不在独立线程中执行中断处理时为NULL
 * @secondary:	pointer to secondary irqaction (force threading)
 * @thread_flags:	flags related to @thread
 * @thread_mask:	bitmask for keeping track of @thread activity
 * @dir:	pointer to the proc/irq/NN/name entry
 */
struct irqaction {
	irq_handler_t		handler;
	void			*dev_id;
	void __percpu		*percpu_dev_id;
	struct irqaction	*next;
	irq_handler_t		thread_fn;
	struct task_struct	*thread;
	struct irqaction	*secondary;
	unsigned int		irq;
	unsigned int		flags;
	unsigned long		thread_flags;
	unsigned long		thread_mask;
	const char		*name;
	struct proc_dir_entry	*dir;
} ____cacheline_internodealigned_in_smp;
```

根据kernel config, irq_desc在内存中有数组和radix_tree两种形式, 通过[irq_to_desc](https://elixir.bootlin.com/linux/v5.10.2/source/kernel/irq/irqdesc.c#L351)由irq获得对应的irq_desc.

```c
/**
 * struct irq_desc - interrupt descriptor
 * @irq_common_data:	per irq and chip data passed down to chip functions
 * @kstat_irqs:		irq stats per cpu
 * @handle_irq:		highlevel irq-events handler 处理中断的函数
 * @action:		the irq action chain irqaction组成的链表的头
 * @status_use_accessors: status information
 * @core_internal_state__do_not_mess_with_it: core internal status information
 * @depth:		disable-depth, for nested irq_disable() calls
 * @wake_depth:		enable depth, for multiple irq_set_irq_wake() callers
 * @tot_count:		stats field for non-percpu irqs
 * @irq_count:		stats field to detect stalled irqs
 * @last_unhandled:	aging timer for unhandled count
 * @irqs_unhandled:	stats field for spurious unhandled interrupts
 * @threads_handled:	stats field for deferred spurious detection of threaded handlers
 * @threads_handled_last: comparator field for deferred spurious detection of theraded handlers
 * @lock:		locking for SMP
 * @affinity_hint:	hint to user space for preferred irq affinity
 * @affinity_notify:	context for notification of affinity changes
 * @pending_mask:	pending rebalanced interrupts
 * @threads_oneshot:	bitfield to handle shared oneshot threads
 * @threads_active:	number of irqaction threads currently running
 * @wait_for_threads:	wait queue for sync_irq to wait for threaded handlers
 * @nr_actions:		number of installed actions on this descriptor
 * @no_suspend_depth:	number of irqactions on a irq descriptor with
 *			IRQF_NO_SUSPEND set
 * @force_resume_depth:	number of irqactions on a irq descriptor with
 *			IRQF_FORCE_RESUME set
 * @rcu:		rcu head for delayed free
 * @kobj:		kobject used to represent this struct in sysfs
 * @request_mutex:	mutex to protect request/free before locking desc->lock
 * @dir:		/proc/irq/ procfs entry
 * @debugfs_file:	dentry for the debugfs file
 * @name:		flow handler name for /proc/interrupts output
 */
struct irq_desc {
	struct irq_common_data	irq_common_data;
	struct irq_data		irq_data; // 芯片相关信息
	unsigned int __percpu	*kstat_irqs;
	irq_flow_handler_t	handle_irq;
	struct irqaction	*action;	/* IRQ action list */
	unsigned int		status_use_accessors;
	unsigned int		core_internal_state__do_not_mess_with_it;
	unsigned int		depth;		/* nested irq disables */
	unsigned int		wake_depth;	/* nested wake enables */
	unsigned int		tot_count;
	unsigned int		irq_count;	/* For detecting broken IRQs */
	unsigned long		last_unhandled;	/* Aging timer for unhandled count */
	unsigned int		irqs_unhandled;
	atomic_t		threads_handled;
	int			threads_handled_last;
	raw_spinlock_t		lock;
	struct cpumask		*percpu_enabled;
	const struct cpumask	*percpu_affinity;
#ifdef CONFIG_SMP
	const struct cpumask	*affinity_hint;
	struct irq_affinity_notify *affinity_notify;
#ifdef CONFIG_GENERIC_PENDING_IRQ
	cpumask_var_t		pending_mask;
#endif
#endif
	unsigned long		threads_oneshot;
	atomic_t		threads_active;
	wait_queue_head_t       wait_for_threads;
#ifdef CONFIG_PM_SLEEP
	unsigned int		nr_actions;
	unsigned int		no_suspend_depth;
	unsigned int		cond_suspend_depth;
	unsigned int		force_resume_depth;
#endif
#ifdef CONFIG_PROC_FS
	struct proc_dir_entry	*dir;
#endif
#ifdef CONFIG_GENERIC_IRQ_DEBUGFS
	struct dentry		*debugfs_file;
	const char		*dev_name;
#endif
#ifdef CONFIG_SPARSE_IRQ
	struct rcu_head		rcu;
	struct kobject		kobj;
#endif
	struct mutex		request_mutex;
	int			parent_irq;
	struct module		*owner;
	const char		*name;
} ____cacheline_internodealigned_in_smp;
```

irq_desc和irqaction都有处理中断的fn, 执行顺序是: irq_desc上的handle_irq是一定执行的, irqaction的函数一般由handle_irq调用, 并有handle_irq的策略决定调用与否.

使用中断模式的设备, 在使能中断之前必须设置触发方式(电平/边沿触发等), irq号, 处理函数等信息. kernel提供了[request_irq(用于调用)](https://elixir.bootlin.com/linux/v5.10.2/source/include/linux/interrupt.h#L143)和[request_threaded_irq(用于实现)](https://elixir.bootlin.com/linux/v5.10.2/source/include/linux/interrupt.h#L128)两个函数可以方便地配置这些信息.

```c
// https://elixir.bootlin.com/linux/v5.10.2/source/include/linux/interrupt.h#L128
extern int __must_check
request_threaded_irq(unsigned int irq, irq_handler_t handler, // handler表示对中断的第一步处理. thread_fn表示在独立线程中执行的处理函数, 为NULL时表示不在独立线程中处理中断.
		     irq_handler_t thread_fn,
		     unsigned long flags, const char *name, void *dev); // flag表示触发方式, 中断共享等标志. dev是设备绑定的数据, 用作调用handler和thread_fn的入参.

/**
 * request_irq - Add a handler for an interrupt line
 * @irq:	The interrupt line to allocate
 * @handler:	Function to be called when the IRQ occurs.
 *		Primary handler for threaded interrupts
 *		If NULL, the default primary handler is installed
 * @flags:	Handling flags
 * @name:	Name of the device generating this interrupt
 * @dev:	A cookie passed to the handler function
 *
 * This call allocates an interrupt and establishes a handler; see
 * the documentation for request_threaded_irq() for details.
 */
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
{
	return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
```

request_threaded_irq主要工作:
1. 根据入参对irqaction对象的字段赋值
1. 将新的irqaction链接到irq对应的irq_desc的action指向的链表的尾部. 这里涉及中断共享, 当多个设备共享同一个irq时, 要求每个设备都在flags中设置IRQF_SHARED标志, 设置的触发方式一致, IRQF_ONESHOT等设置也要相同.
1. 如果thread_fn!=NULL, 则调用setup_irq_thread创建一个线程来处理中断.

handler和thread_fn的编写原则:
1. 中断处理打断了当前进程的执行, 因此需要进行一系列复杂的处理, 所以要快速返回, 不能在handler中做复杂操作, 比如I/O操作等, 这就是所谓的中断处理的上半段(top half). 如果需要复杂操作, 一般有两种常见的做法:

    1. 在函数中启动工作队列或者软中断(如tasklet)等, 由工作队列等来完成工作.
    1. 在thread_fn中执行, 这就是所谓的中断处理的下半段(bottom half)
1. handler不能进行任何sleep的操作, 调用sleep, 使用信号量, 互斥锁等可能导致sleep的机制都不行.
1. 不能在handler中调用disable_irq这类需要等待当前中断执行完毕的函数, 中断处理中调用一个需要等待当前中断结束的函数, 会导致死锁. 实际上, handler执行的时候, 一般外部中断依然是在禁止的状态, 不需要desable_irq.

> 死锁: 两个及以上task在执行中, 因竞争资源而处于相互等待的现象.

request_irq通过将request_threaded_irq的thread_fn设为NULL来实现. 它们的区别是:
1. request_irq的handler直接在当前中断上下文中执行, request_threaded_irq的thread_fn在独立线程中执行.
1. 根据前面的第一条原则, handler不能进行复杂操作, 操作由工作队列等来进行, 即工作队列实际上也在进程上下文.
1. 执行thread_fn线程的优先级比工作队列线程的优先级高

request_threaded_irq创建新线程时, 会调用sched_setscheduler_nocheck(t, SCHED_FIFO, ...)将线程设置为实时的. 对用户体验影响比较大, 要求快速响应的设备的驱动中, 采用中断模式的情况下, 使用request_threaded_irq有利于提高用户体验; 反之, 要求不高的设备的驱动中, 使用request_irq更合适.