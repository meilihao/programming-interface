# interrupt
ref:
- [Linux kernel中断子系统之（五）：驱动申请中断API](http://www.wowotech.net/irq_subsystem/request_threaded_irq.html)

	包含进程/中断上下文的图

广义中断包括:
1. 中断: 狭义中断/异步中断

	分可屏蔽中断和不可屏蔽中断, 来自于I/O设备的信号, 总是返回到下一步指令. 举例: 所有的IRQ, 电源掉电, 存储器奇偶校验
1. 异常: 同步中断

	分:
	- trap(陷阱): 主动的异常, 总是返回到下一步指令.  举例: 系统调用, 信号机制
	- fault(故障): 潜在可恢复的错误, 返回到当前指令.  举例: 缺页异常
	- abort(终止): 不可恢复的错误, 不返回. 举例: 硬件错误

概念:
- 中断请求(irq, interrupt request)是由可编程中断控制器(PIC, programmable interrupt controller)发起的, 其目的是为了中断cpu和执行中断服务程序(isr, interrupt service routine).
- 硬件中断: 当一个硬件设备想要告诉 CPU 某一需要处理的数据已经准备好后（例如：当键盘被按下或者一个数据包到了网络接口处），它将会发送一个中断请求（IRQ）来告诉 CPU 数据是可用的, 接下来会调用在内核启动时设备驱动注册的对应的中断服务程序（ISR）.
- 软件中断: 当在播放一个视频时，音频和视频是同步播放是相当重要的，这样音乐的速度才不会变化. 这是由软件中断实现的，由精确的计时器系统（称为 jiffies）重复发起的. 这个计时器会使得音乐播放器同步, 软件中断也可以被特殊的指令所调用，来读取或写入数据到硬件设备. 当系统需要**实时性时（例如在工业应用中），软件中断会变得重要**. 可参考[这里](https://www.linuxfoundation.org/blog/2013/03/intro-to-real-time-linux-for-embedded-developers/).


按中断来源分:
1. 内部中断: 来自cpu内部(软件中断指令, 溢出, 除法错误等)
1. 外部中断: 来自cpu外部, 由外设提供提供

按是否屏蔽分:
1. 可屏蔽中断: 可通过设置中断控制器寄存器等方法被屏蔽, 屏蔽后, 该中断不再得到响应
1. 不可屏蔽中断(NMI): 不能被屏蔽

根据中断入口跳转方法的不同, 中断分:
1. 向量中断: 不同的中断分配不同的中断号. 由硬件提供服务程序入口地址
1. 非向量中断: 多个中断共享一个入口, 进入该入口后, 再通过软件判断中断标志来识别具体是哪个中断. 由软件提供中断服务程序入口地址.

os已注册的中断: `cat /proc/interrupts`, 从左到右各列的含义依次为：`中断向量号、每个 CPU（0~n）中断发生次数、硬件来源、硬件源通道信息、以及造成中断请求的设备名`. 在表的末尾，有一些非数字的中断, 它们是特定于体系结构的中断，如LOC(local timer interrupt, 本地计时器中断)的中断请求（IRQ）号为 236, 其中一些在 Linux 内核源树中的[Linux IRQ 向量布局](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/irq_vectors.h)中指定.

中断是一种异步的事件处理机制，用来提供系统的并发处理能力, 当中断事件发生，会触发执行中断处理程序.

中断处理程序分为上半部和下半部:
- 上半部(top half)：硬中断

	用来快速处理中断, 从而可以服务更多的中断请求. 它在中断禁止模式（关闭中断响应）下运行，主要处理跟硬件紧密相关的或时间敏感的工作
- 下半部(bottom half)：软中断，用来异步(延迟)处理上半部未完成的工作, 通常以内核线程的方式运行

	每个 CPU 都对应一个软中断内核线程，名字是`ksoftirqd/<CPU编号>`
	当软中断事件的频率过高时，内核线程也会因为 CPU 使用率过高而导致软中断处理不及时，进而引发网络收发延迟，调度缓慢等性能问题.

	相比上半部, 它可以被新中断打断.

	linux实现下半部的机制主要有tasklet, 工作队列, 软中断和线程化irq(threaded_irq).

> in_interrupt()可用于判断是否处于中断上下文.

> 代码在`kernel/irq`, 与arch相关的在`arch/<arch>/kernel/irq.c`. 软中断和tasklet的实现在kernel/softirq.c

## cpu识别中断
x86处理器通过INTR和NMI 两个引脚分别接收外部中断请求信号: INTR接收可屏蔽中断, NMI接收不可屏蔽中断请求. 标志寄存器EFLAGS中的IF标志决定是否屏蔽可屏蔽中断请求.

传统的中断控制器是PIC(可编程中断控制器, 比如8259A), 在单核时够用了.

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

所有的中断处理程序必须即能正确处理中断, 又能保证处理完毕后可以返回中断前的程序继续执行. 因此, 中断处理必须完成3个任务:
1. 保存现场以便恢复
1. 调用已注册的中断服务例程(isr)处理中断
1. 恢复现场继续原有程序

中断处理程序和中断服务例程的区别: 中断处理程序包括中断处理的整个过程, 而中断服务例程是该过程中对产生中断的设备的处理逻辑. 并不是所有的中断处理程序都需要对应的中断服务例程, 中断服务例程是为了方便外设驱动利用内核提供的函数编程. **有了中断服务例程, dirver只需要专注处理设备自身的中断而不需要关系整个过程**.

## 中断服务例程
中断服务例程涉及两个关键的struct, 即irq_desc和irqaction, 两者是1:n的关系, 但并不是每个irq_desc都一定有与之对应的irqaction. irq_desc与irq号对应, irqaction与一个设备对应, 共享同一个irq号的多个设备的irqaction对应同一个irq_desc.

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

irq_desc和irqaction都有处理中断的fn, 执行顺序是: irq_desc上的handle_irq是一定执行的, irqaction的函数一般由handle_irq调用, 并由handle_irq的策略决定调用与否.

linux使用request_irq()和free_irq()来申请和释放中断. devm_request_irq()的`devm_`开通指kernel managed的资源, 一般不需要出错处理和remove()接口里显示的是否. 比如at86rf230驱动改用devm_request_irq()后就删除了free_irq().

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
1. 不能在handler中调用disable_irq这类需要等待当前中断执行完毕的函数, 中断处理中调用一个需要等待当前中断结束的函数, 会导致死锁. 实际上, handler执行的时候, 一般外部中断依然是在禁止的状态, 不需要disable_irq.

> 死锁: 两个及以上task在执行中, 因竞争资源而处于相互等待的现象.

request_irq通过将request_threaded_irq的thread_fn设为NULL来实现. 它们的区别是:
1. request_irq的handler直接在当前中断上下文中执行, request_threaded_irq的thread_fn在独立线程中执行.
1. 根据前面的第一条原则, handler不能进行复杂操作, 操作由工作队列等来进行, 即工作队列实际上也在进程上下文. 执行thread_fn线程的优先级比工作队列线程的优先级高

request_threaded_irq创建新线程时, 会调用sched_setscheduler_nocheck(t, SCHED_FIFO, ...)将线程设置为实时的. 对用户体验影响比较大, 要求快速响应的设备的驱动中, 采用中断模式的情况下, 使用request_threaded_irq有利于提高用户体验; 反之, 要求不高的设备的驱动中, 使用request_irq更合适.

### 中断处理
ref:
- [**硬核长文丨深入理解Linux中断机制**](https://zhuanlan.zhihu.com/p/551615380)
- [Kernel Exploring](https://richardweiyang-2.gitbook.io/kernel-exploring/00-start_from_hardware/05-interrupt_handler)
- [QEMU 如何模拟中断](https://martins3.github.io/qemu/interrupt.html)

	asm_common_interrupt的来源

```c
// https://elixir.bootlin.com/linux/v6.6.25/source/arch/x86/include/asm/idtentry.h#L498
/*
 * ASM code to emit the common vector entry stubs where each stub is
 * packed into IDT_ALIGN bytes.
 *
 * Note, that the 'pushq imm8' is emitted via '.byte 0x6a, vector' because
 * GCC treats the local vector variable as unsigned int and would expand
 * all vectors above 0x7F to a 5 byte push. The original code did an
 * adjustment of the vector number to be in the signed byte range to avoid
 * this. While clever it's mindboggling counterintuitive and requires the
 * odd conversion back to a real vector number in the C entry points. Using
 * .byte achieves the same thing and the only fixup needed in the C entry
 * point is to mask off the bits above bit 7 because the push is sign
 * extending.
 */
	.align IDT_ALIGN
SYM_CODE_START(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept NR_EXTERNAL_VECTORS
	UNWIND_HINT_IRET_REGS
0 :
	ENDBR
	.byte	0x6a, vector // `.byte	0x6a`=push
	jmp	asm_common_interrupt // 没找到asm_common_interrupt在哪
	/* Ensure that the above is IDT_ALIGN bytes max */
	.fill 0b + IDT_ALIGN - ., 1, 0xcc
	vector = vector+1
    .endr
SYM_CODE_END(irq_entries_start)

// https://elixir.bootlin.com/linux/v6.6.25/source/arch/x86/include/asm/idtentry.h#L640
// idtentry.h 会分别被 c 源文件和 asm 源文件 include, 所以其定义也分别有两种
/* Device interrupts common/spurious */
DECLARE_IDTENTRY_IRQ(X86_TRAP_OTHER,	common_interrupt);

// https://elixir.bootlin.com/linux/v6.6.25/source/arch/x86/include/asm/idtentry.h#L176
/**
 * DECLARE_IDTENTRY_IRQ - Declare functions for device interrupt IDT entry
 *			  points (common/spurious)
 * @vector:	Vector number (ignored for C)
 * @func:	Function name of the entry point
 *
 * Maps to DECLARE_IDTENTRY_ERRORCODE()
 */
#define DECLARE_IDTENTRY_IRQ(vector, func)				\
	DECLARE_IDTENTRY_ERRORCODE(vector, func)

// https://elixir.bootlin.com/linux/v6.6.25/source/arch/x86/include/asm/idtentry.h#L432
#define DECLARE_IDTENTRY_ERRORCODE(vector, func)			\
	idtentry vector asm_##func func has_error_code=1

// https://elixir.bootlin.com/linux/v6.6.25/source/arch/x86/include/asm/idtentry.h#L176
/* Entries for common/spurious (device) interrupts */
#define DECLARE_IDTENTRY_IRQ(vector, func)				\
	idtentry_irq vector func

// https://elixir.bootlin.com/linux/v6.6.25/source/arch/x86/entry/entry_64.S#L431
/*
 * Interrupt entry/exit.
 *
 + The interrupt stubs push (vector) onto the stack, which is the error_code
 * position of idtentry exceptions, and jump to one of the two idtentry points
 * (common/spurious).
 *
 * common_interrupt is a hotpath, align it to a cache line
 */
.macro idtentry_irq vector cfunc
	.p2align CONFIG_X86_L1_CACHE_SHIFT
	idtentry \vector asm_\cfunc \cfunc has_error_code=1
.endm

// https://elixir.bootlin.com/linux/v6.6.25/source/arch/x86/kernel/irq.c#L247
/*
 * common_interrupt() handles all normal device IRQ's (the special SMP
 * cross-CPU interrupts have their own entry points).
 */
DEFINE_IDTENTRY_IRQ(common_interrupt)
{
	struct pt_regs *old_regs = set_irq_regs(regs);
	struct irq_desc *desc;

	/* entry code tells RCU that we're not quiescent.  Check it. */
	RCU_LOCKDEP_WARN(!rcu_is_watching(), "IRQ failed to wake up RCU");

	desc = __this_cpu_read(vector_irq[vector]);
	if (likely(!IS_ERR_OR_NULL(desc))) {
		handle_irq(desc, regs);
	} else {
		apic_eoi();

		if (desc == VECTOR_UNUSED) {
			pr_emerg_ratelimited("%s: %d.%u No irq handler for vector\n",
					     __func__, smp_processor_id(),
					     vector);
		} else {
			__this_cpu_write(vector_irq[vector], VECTOR_UNUSED);
		}
	}

	set_irq_regs(old_regs);
}

// https://martins3.github.io/qemu/interrupt.html
asm_common_interrupt
  call  error_entry
  movq  %rsp, %rdi /* pt_regs pointer into 1st argument*/
  movq  ORIG_RAX(%rsp), %rsi  /* get error code into 2nd argument*/
  call  common_interrupt
  jmp error_return
```

中断处理函数定义在 irq_entries_start 函数数组里, 它定义了 FIRST_SYSTEM_VECTOR 到 FIRST_EXTERNAL_VECTOR 项, 每一项都是中断处理函数, 会跳到 asm_common_interrupt() 去执行，并最终调用 handle_irq(), 调用完毕后, 就从中断返回.


irq_entries_start和common_interrupt对vector的处理是为了与系统调用号区分, 处理信号的时候会判断处理信号之前是否处于系统调用过程中, 判断的依据就是regs->orig_ax不小于0, vector也存储在该位置.

common_interrupt完成中断处理的主要逻辑.

handle_irq开启了处理之旅. handle_irq函数完成溢出检查、找到irq对应的irq_desc对象，并调用它的handle_irq字段定义的回调函数.

举例, 键盘和鼠标共享中断引脚，连接到GPIO上，通过GPIO来实现中断，GPIO的中断直接连接到处理器上，假设GPIO的irq号为50，键盘和鼠标的irq号为200.

硬件上，键盘的中断引脚并不直接连接处理器，当它需要中断时，GPIO会检测到引脚电平的变化，进而去中断处理器。软件上，
GPIO的驱动一方面设置irq:50对应的irq_desc的handle_irq字段，另一方面为所有连接到它的外设（假设irq号在[180, 211]内）设置irq_desc.

> GPIO驱动参考[x3proto_gpio_chip](https://elixir.bootlin.com/linux/v6.6.25/source/arch/sh/boards/mach-x3proto/gpio.c)

键盘和鼠标驱动分别调用request_irq设置了irq_action，并将其链接到irq:200对应的irq_desc，这样irq_desc对象的action字段指向的链表
中就有键盘和鼠标两个设备的irq_action了.

因为irq是共享的，所以发生中断时，GPIO一般情况下并不知道是键盘还是鼠标触发了中断。这就需要外设处理中断前，首先必须根据自身状态寄存器判断是否是自己触发了中断，如果是才会继续执行。如果硬件或者驱动不支持这类判断，那么该外设不适合共享中断.

处理器检测到中断，通过handle_irq函数调用irq:50对应的中断服务例程. 它判断是哪个引脚引起了中断，然后将中断处理继续传递至连接到该引脚的设备, 即调用了irq:200的irq_desc的handle_irq字段，也就是GPIO驱动中设置的irq_set_chip_and_handler_name的handle参数, 比如handle_simple_irq.

> 其他handle有, handle_edge_irq对应的是边沿触发的设备，如果是电平触发，应该是handle_level_irq.

handle_edge_irq有两个重要功能，第一个功能与中断重入有关, 如果当前中断处理正在执行，这时候又触发了新一轮的同一个irq的中断，新中断并不会马上得到处理，而是执行desc->istate|=IRQS_PENDING操作。当前中断处理完毕后，会循环检测IRQS_PENDING是否被置位，如果是就继续处理中断。

从另一个角度来讲，即使处理当前中断时多个中断到来，完成当前处理后只会处理一次，这是合理的，毕竟对大多数硬件来讲最新时刻的状态更有意义。不过现实中，这种情况应该深入分析，因为这对某些需要跟踪轨迹设备的用户体验有较大影响。比如鼠标，如果中间几个点丢掉了，光标会从一个位置跳到另一个.

第二个功能就是处理当前中断，可以调用handle_irq_event函数实现。该函数会将IRQS_PENDING清零，调用irqd_set将irq_desc的
irq_data的IRQD_IRQ_INPROGRESS标记置位，表示正在处理中断, 然后调用handle_irq_event_percpu函数将处理权交给设备, 最后清除IRQD_IRQ_INPROGRESS标记.

handle_irq_event_percpu调用__handle_irq_event_percpu，后者遍历irqaction.

键盘和鼠标的驱动将它们的irqaction链接到了irq_desc，让函数回调它们的handler字段. 键盘的驱动调用了
request_threaded_irq，它的handler参数为NULL，系统默认设置成了irq_default_primary_handler函数，它直接返回IRQ_WAKE_THREAD， 所以对于键盘而言，会执行irq_wake_thread唤醒执行中断处理的线程，键盘的thread_fn=自定义的handle_keyboard_irq就是在该线程中执行的. 而对鼠标而 言，驱动调用的是request_irq，handler就是自定义的handle_mouse_irq函数，它使用工作队列完成I/O操作和数据报告等任务.

### 中断嵌套
一个中断发生的时候, 另一个中断到来, 需要先处理新中断, 处理完毕再返回原中断继续处理.

x86中，中断嵌套时，第一个中断占用了中断栈，那么后面的中断就只能在当前进程的栈中执行了. 另一点需要说明的是, x86中, common_interrupt是在当前进程中执行，调用irq_desc的handle_irq时才会切换到中断栈中; x64中, 直接切换到中断栈中执行common_interrupt.

### 中断栈
5.x内核版本中, 默认情况下, 中断处理都会优先在中断栈中执行.

32位系统中, 中断栈分为软中断栈和硬中断栈, 分别由每cpu变量hardirq_stack和softirq_stack表示; 64位系统中, 软硬中断使用同一个中断栈, 由每cpu变量irq_stack_ptr表示. 另外, 软中断也可以在单独的内核线程ksoftirqd中执行, 这种情况下就不需要使用软中断栈了.


### 使用和屏蔽中断
- disable_irq()
- disable_irq_nosync()

	与disable_irq()的区别在于disable_irq_nosync()立即返回, 而disable_irq()等待目前的中断处理完成.

	由于disable_irq()会等待指定的中断处理完, 因此如果在n号中断的上半部调用disable_irq(n), 会引起系统死锁, 此时只能用disable_irq_nosync().
- enable_irq()
``

`local_`开头的方法仅作用于本cpu:
```c
# define local_irq_save(flags) // 与local_irq_disable区别, local_irq_save会将目前的中断状态保留在flags中, 而local_irq_disable禁止中断而不保存状态
void local_irq_disable(void);
// 对应
# define local_irq_restore(flags)
void local_irq_enable(void);
```

## 中断共享
多个设备共享一根硬件中断线的情况在实际硬件系统中广泛存在, 使用方法:
1. 共享中断的多个设备在申请中断时, 都使用IRQF_SHARED, 而且一个设备以IRQF_SHARED申请某中断成功的前提是该中断未被申请, 或已被申请但是之前申请该中断的所有设备也都以IRQF_SHARED申请该中断
1. 尽管内核模块可访问的全局地址都可作为`request_irq(..., void *dev_id)`的最后一个参数dev_id, 但是设备结构体指针显然是可传入的最佳参数
1. 在中断到来时, 会遍历执行共享此中断的所有中断处理程序, 直到某个函数返回IRQ_HANDLED. 在中断处理程序上半部中, 应根据硬件寄存器中的信息对比dev_id判断是否为本设备的中断, 若不是, 迅速返回IRQ_NONE