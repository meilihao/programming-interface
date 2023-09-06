# process

## 程序和进程
程序(program)是存储上的可执行文件. 进程(process)是程序在内存执行时的实体即进程是由内核定义的抽象的实体.

在类unix中, 每个进程都有一个唯一的数字标识符，称为进程ID（process ID）,也叫pid, 进程ID是一个非负整数.

Linux 内核默认限制进程号需小于等于 32767. 新进程创建时,内核会按顺序将下一个可用的进程号分配给其使用. 每当进程号达到 32767 的限制时,内核将重置进程号计数器(到300, 因为低数值的进程号为系统进程和守护进程所长期占用),以便从小整数开始分配.

> 进程号的默认上限可以通过 Linux系统特有的/proc/sys/kernel/pid_max 文件来进行调整(其值=pid默认最大值+1), 但32 位平台
中, pid_max 文件的最大值为 32768; 64位则允许是2^22.
> pid默认最大值受`<linux/threads.h>`中定义的PID最大值的限制.

每个进程都有一个创建自己的父进程, 可通过`getppid()`获得, 因此存在进程tree的概念. 同样也可通过由 Linux 系统所特有的`/proc/${PID}/status` 文件所提供的 ppid 字段获得.

进程可认为是一个用户地址空间和内核描述该进程的一组内核数据结构共同组成: 其中用户内存空间包含了程序代码及代码所使用的变量,而内核数据结构则用于维护进程状态信息. 
记录在内核数据结构中的信息包括许多与进程相关的标识号(IDs)、虚拟内存表、打开文件的描述符表、信号传递及处理的有关信息、进程资源使用及限制、当前工作目录和大量的其他信息.

先看看linux正在运行的进程:
```sh
$ps -eo pid,comm,args
```

> `comm`是进程的简称(即执行程序的名称);`args`是进程所对应的程序(可能含有路径)以及运行时所带的参数

也可使用`pstree`命令来展示当前的进程树.

POSIX进程的内存布局, 其分为三大段:
- 正文/文本段(比如程序代码) : 只读
- 数据段(比如变量, 通过`brk()`增长)

	- 初始化数据段包含显式初始化的全局变量和静态变量
	- 未初始化数据段包含了未进行显式初始化的全局变量和静态变量

		程序启动之前,系统将本段内所有内存初始化为 0. 出于历史原因, 此段常被称为 BSS 段(block started by symbol).
- 堆栈段

	- 栈(stack)

		一个动态增长和收缩的段,由栈帧(stack frames)组成. 系统会为每个当前调用的函数分配一个栈帧. 栈帧中存储了函数的局部变量(所谓自动变量)、实参和返回值.

	- 堆(heap)

		可在运行时(为变量)动态进行内存分配的一块区域. 堆顶端称作 program break(可通过brk()和 sbrk()调整).

> 堆向上增长, 栈向下增长.
> `brk()`不属于POSIX.

程序的具体构成:
- 二进制格式标识

	每个程序文件都包含用于描述可执行文件格式的元信息(metainformation). 内核(kernel)会利用此信息来解释文件中的其他信息. 现在,大多数 UNIX 实现(包括 Linux)采用可执行连接格式(ELF)

- 机器语言指令 : 对程序算法进行编码
- 程序入口地址 : 标识程序开始执行时的起始指令位置
- 数据 : 程序文件包含的变量初始值和程序使用的字面常量(literal constant)值(比如字符串)
- 符号表及重定位表 : 描述程序中函数和变量的位置及名称. 这些表格有多种用途,其中包括调试和运行时的符号解析(动态链接)
- 共享库和动态链接信息 : 程序文件所包含的一些字段,列出了程序运行时需要使用的共享库,以及加载共享库的动态链接器的路径名
- 其他信息 : 程序文件还包含许多其他信息,用以描述如何创建进程

> 应用程序二进制接口(ABI)是一套规则,规定了二进制可执行文件在运行时应如何与某些服务(诸如内核或函数库所提供的服务)交换信息. ABI 特别规定了使用哪些寄存器和栈地址来交换信息以及所交换值的含义.
> 在大多数 UNIX 实现(包括 Linux)中 C 语言编程环境提供了 3 个全局符号(symbol):etext、edata 和 end,可在程序内使用这些符号以获取相应程序文本段、初始化数据段和非初始化数据段结尾处下一字节的地址.

## 进程控制
有三个用于进程控制的主要函数： fork, exec和waitpid, 其中exec函数有7种变体，但经常统称它们为exec函数.

### 状态
- 运行中(实际占用cpu)
- 就绪(可运行, 但还未分配到cpu)
- 阻塞中(需要某种条件, 无法运行)

![](/misc/img/os/961dcc4764f2e60c09fa5255a02ba71f.png)

Linux进程状态:
![](/misc/img/os/linux_process_state.png)

## 进程和线程
尽管在UNIX中，进程与线程是有联系但不同的两个东西，但**在Linux中，线程只是一种特殊的进程, 它们都被抽象为task**,多个线程之间可以共享内存空间和IO接口.所以，进程是Linux程序的唯一的实现方式.

> 在现代os中, 进程提供两种虚拟机制: 虚拟处理器和虚拟内存.

通常, 一个进程只有一个线程(thread, 进程中一个**单一顺序**的控制流), 但也允许有多个线程, 这样可以进行多任务的同时处理和充分利用多核处理器的**并行**能力.

一个进程内的所有线程会共享同一地址空间, 文件描述符, 栈以及与进程相关的属性, 因此访问共享数据时需要**同步措施来避免不一致性**, 但它也有独立的程序计数器, 栈和一组寄存器. 与进程相同, 线程也有ID, 但其仅在所属进程内有效.

在引入线程的操作系统中，线程是独立**调度**的基本单位，进程是**资源**拥有的基本单位. 在同一进程中，线程的切换不会引起进程切换; 在不同进程中进行线程切换,如从一个进程内的线程切换到另一个进程中的线程时，会引起进程切换.

线程:
1. 用户态(1:M) : java

   不需要内核支持而在用户程序中实现的线程, 其不依赖于操作系统核心，应用进程利用线程库提供创建、同步、调度和管理线程的函数来控制用户线程.

   优点: 线程切换简单, 进程可以有自定义的调度算法.
   缺点: 操作系统内核不知道多线程的存在，因此一个线程阻塞将使得整个进程（包括它的所有线程）阻塞, 即用户级线程是OS内核不可感知的; 用户级线程不能利用系统的多核处理，仅有一个用户级线程可以被执行
1. 内核级(1:1)

   由操作系统内核创建和撤销, 内核维护进程及线程的上下文信息以及线程切换.一个内核线程由于I/O操作而阻塞，不会影响其它线程的运行.

   优点: 可以利用多核
   缺点: 线程在用户态的运行，而线程的调度和管理在内核实现，在控制权从一个线程传送到另一个线程需要用户态到内核态再到用户态的模式切换，比较占用系统资源.

1. 混合式(M:N) : go的GPM模型

   内核只识别内核级线程, 其再被用户级线程多路复用.

三种线程模型之间最大的差异就在于用户线程与内核调度实体（KSE，Kernel Scheduling Entity即内核线程）之间的对应关系上:
- 一对一模型（1:1） : 一个用户线程映射到一个内核线程，用户线程在存活期都会绑定到一个内核线程，一旦退出，2个线程都会退出
    优点: 实现简单且实现了真正的并发，多个线程可同时跑在不同的CPU上
    缺点: 借助了操作系统内核来创建、销毁和以及多个线程之间的上下文切换和调度, 代价很大. 比如用户线程起多了，内核线程肯定不够用，那么就需要切换. kernel限制了内核线程的数量.

	> LinuxThreads与NPTL均采用一对一的线程模型.
- 多对一模型（M:1）　：　多个用户线程(一般从属于单个进程)被映射到一个内核线程
    优点: 多个用户线程切换非常的轻量快速，不需要内核线程上下文切换(即不会陷入到内核态)
    缺点: 1个进程的所有协程都绑定在1个线程上, 该程序用不了硬件的多核优势; 如果一个线程阻塞了，那么映射到同一个内核线程的用户线程将都无法运行(因为内部线程对CPU是不可见的，此时可以理解为阻塞的是用户进程)
- 多对多模型（M:N） :　综合以上两种模型，即用户调度器实现用户线程到KSE的"调度"，内核调度器实现KSE到CPU上的"调度", 但实现最为复杂．golang采用的就是这种方式.

> 内核调度实体 KSE 就是指可以被操作系统内核调度器调度的对象实体.
> 用户态线程也叫协程（co-routine）
> **类nix实现了POSIX1003.1c标准的线程模型即pthread, 是linux上多线程编程的唯一选择**, 比如c++的boost::thread/c++11的thread都是基于它.
> 内核线程只工作在内核态中；而用户线程则既可以运行在内核态（执行系统调用时），也可以运行在用户态
> 多线程进程: **每个线程有自己的线程控制块, 用户栈, 内核栈**.

ULT(user level thread, 用户态线程)和KLT(kernel level thread, 内核态线程即轻量级进程)比较:
- ULT优势

	1. 所有线程在用户态, 不用上下文切换, 减少切换开销
	1. 可以定制更合适的调度策略, 而不会干扰kernel的调度
	1. ult可在任何os中运行, 而不用修改kernel
- ULT劣势

	1. ult阻塞时会阻塞进程中的所有线程
	1. ult无法充分利用多核技术: kernel一次只能把一个进程分配给一个cpu核心.

> windows中所有管理线程的工作由kernel完成(kernel提供api), 因此它是纯KLT.

## 进程分类
- 系统进程: 进行内存分配, 进程切换等管理工作. 不受普通用户甚至root的干预
- 用户进程: 执行用户程序.

	细分:
	- 交互进程: 由shell启动的进程, 在执行过程中, 需要与用户进行交互, 可运行在前/后台
	- 批处理进程: 一个进程集合, 负责按顺序启动其他的进程
	- 守护进程: 一直运行的进程, 不与终端关联, 周期性执行某种任务或等待处理某些发生的事件.


## 上下文(context)
执行上下文(execution context)又称为进程状态(process state), 是os用来管理和控制进程所需的内部数据. 可简单理解为被运行中的进程加载到寄存器的数据集.

上下文切换(context switching) : 存储运行中的进程的上下文, 然后将下一个要运行的进程的上下文恢复到寄存器的过程.

> 存储上下文的位置: 进程描述符和内核模式堆栈区域

### 进程实现
task_struct]包括:
- 它打开的文件
- 进程的地址空间
- 挂起的信号
- 进程的状态
- 等等

linux使用slab分配task_struct, 便于对象复用和缓存着色(cache coloring).

进程在内核态运行时需要自己的堆栈信息, 因此linux内核为每个进程都提供了一个内核栈kernel stack.

task_struct包含一个指向父进程的parent指针和一个指向子进程的children 链表.

init进程的进程描述符是作为init_task静态分配的.

查找init: `for (task = current; task != &init_task; task=task->parent)`
对于给定进程, 获取task list中的下一个进程`list_entry(task->task.next, struct task_struct, tasks)`(by next_task(task)宏);获取上一个进程`list_entry(task->task.prev, struct task_struct, tasks)`(by prev_task(task)宏)
遍历task list: `for_each_process(task)`, 开销较大.

```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/sched.h#L738
// 简化版
struct task_struct {
    struct thread_info thread_info;//处理器特有数据 
    volatile long   state;       //进程状态 
    void            *stack;      //进程内核栈地址 
    refcount_t      usage;       //进程使用计数
    int             on_rq;       //进程是否在运行队列上
    int             prio;        //动态优先级
    int             static_prio; //静态优先级
    int             normal_prio; //取决于静态优先级和调度策略
    unsigned int    rt_priority; //实时优先级
    const struct sched_class    *sched_class;//指向其所在的调度类
    struct sched_entity         se;//普通进程的调度实体
    struct sched_rt_entity      rt;//实时进程的调度实体
    struct sched_dl_entity      dl;//采用EDF算法调度实时进程的调度实体
    struct sched_info       sched_info;//用于调度器统计进程的运行信息 
    struct list_head        tasks;//所有进程的链表
    struct mm_struct        *mm;  //指向进程内存结构
    struct mm_struct        *active_mm;
    pid_t               pid;            //进程id
    struct task_struct __rcu    *parent;//指向其父进程
    struct list_head        children; //链表中的所有元素都是它的子进程
    struct list_head        sibling;  //用于把当前进程插入到兄弟链表中
    struct task_struct      *group_leader;//指向其所在进程组的领头进程
    u64             utime;   //用于记录进程在用户态下所经过的节拍数
    u64             stime;   //用于记录进程在内核态下所经过的节拍数
    u64             gtime;   //用于记录作为虚拟机进程所经过的节拍数
    unsigned long           min_flt;//缺页统计 
    unsigned long           maj_flt;
    struct fs_struct        *fs;    //进程相关的文件系统信息
    struct files_struct     *files;//进程打开的所有文件
    struct vm_struct        *stack_vm_area;//内核栈的内存区
  };
```

一个 task_struct 结构体的实例变量代表一个 Linux 进程.

Linux 早期是这样创建 task_struct 结构体的实例变量的：找伙伴内存管理系统，分配两个连续的页面（即 8KB），作为进程的内核栈，再把 task_struct 结构体的实例变量，放在这 8KB 内存空间的开始地址处。内核栈则是从上向下伸长的，task_struct 数据结构是从下向上伸长的. 此时Linux 把 task_struct 结构和内核栈放在了一起 ，所以只要把 RSP 寄存器的值读取出来，然后将其低 13 位清零，就得到了当前 task_struct 结构体的地址.

随着 Linux 版本的迭代, task_struct 结构体的体积越来越大，从前 task_struct 结构体和内核栈放在一起的方式就不合适了, 最新的版本是分开放的.

```c
//https://elixir.bootlin.com/linux/v6.5.1/source/kernel/fork.c#L2958
/*
 * Create a kernel thread.
 */
pid_t kernel_thread(int (*fn)(void *), void *arg, const char *name,
		    unsigned long flags)
{
	struct kernel_clone_args args = {
		.flags		= ((lower_32_bits(flags) | CLONE_VM |
				    CLONE_UNTRACED) & ~CSIGNAL),
		.exit_signal	= (lower_32_bits(flags) & CSIGNAL),
		.fn		= fn,
		.fn_arg		= arg,
		.name		= name,
		.kthread	= 1,
	};

	return kernel_clone(&args);
}

/*
 * Create a user mode thread.
 */
pid_t user_mode_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
	struct kernel_clone_args args = {
		.flags		= ((lower_32_bits(flags) | CLONE_VM |
				    CLONE_UNTRACED) & ~CSIGNAL),
		.exit_signal	= (lower_32_bits(flags) & CSIGNAL),
		.fn		= fn,
		.fn_arg		= arg,
	};

	return kernel_clone(&args);
}

#ifdef __ARCH_WANT_SYS_FORK
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
	struct kernel_clone_args args = {
		.exit_signal = SIGCHLD,
	};

	return kernel_clone(&args);
#else
	/* can not support in nommu mode */
	return -EINVAL;
#endif
}
#endif
```

kernel_clone就是进程创建入口, 核心逻辑:kernel_clone -> copy_process -> dup_task_struct -> alloc_task_struct_node(node),分配task_struct结构体 + alloc_thread_stack_node(tsk, node),分配内核栈

Linux 用于mm_struct描述一个进程的地址空间.

```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/mm_types.h#L598
//简化
struct mm_struct {
        struct vm_area_struct *mmap; //虚拟地址区间链表VMAs, 用来描述一段虚拟地址空间
        struct rb_root mm_rb;   //组织vm_area_struct结构的红黑树的根
        unsigned long task_size;    //进程虚拟地址空间大小
        pgd_t * pgd;        //指向MMU页表
        atomic_t mm_users; //多个进程共享这个mm_struct
        atomic_t mm_count; //mm_struct结构本身计数 
        atomic_long_t pgtables_bytes;//页表占用了多个页
        int map_count;      //多少个VMA
        spinlock_t page_table_lock; //保护页表的自旋锁
        struct list_head mmlist; //挂入mm_struct结构的链表
        //进程应用程序代码开始、结束地址，应用程序数据的开始、结束地址 
        unsigned long start_code, end_code, start_data, end_data;
        //进程应用程序堆区的开始、当前地址、栈开始地址 
        unsigned long start_brk, brk, start_stack;
        //进程应用程序参数区开始、结束地址
        unsigned long arg_start, arg_end, env_start, env_end;
};
```

在 copy_process 函数中被调用copy_mm, copy_mm 函数调用 dup_mm 函数，把当前进程的 mm_struct 结构复制到 allocate_mm 宏分配的一个 mm_struct 结构中。这样，一个新进程的 mm_struct 结构就建立了.


在 Linux 系统中，可以说万物皆为文件，比如文件、设备文件、管道文件等。一个进程对一个文件进行读写操作之前，必须先打开文件，这个打开的文件就记录在进程的文件表中，它由 task_struct 结构中的 files 字段指向files_struct.
```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/fdtable.h#L49
/*
 * Open file table structure
 */
struct files_struct {
  /*
   * read mostly part
   */
	atomic_t count;//自动计数
	bool resize_in_progress;
	wait_queue_head_t resize_wait;

	struct fdtable __rcu *fdt;
	struct fdtable fdtab;
  /*
   * written part on a separate cache line in SMP
   */
	spinlock_t file_lock ____cacheline_aligned_in_smp; //自旋锁
	unsigned int next_fd; //下一个文件句柄
	unsigned long close_on_exec_init[1]; //执行exec()时要关闭的文件句柄
	unsigned long open_fds_init[1];
	unsigned long full_fds_bits_init[1];
	struct file __rcu * fd_array[NR_OPEN_DEFAULT]; //默认情况下打开文件的指针数组
};
```

`int fd = open("/tmp/test.txt");`时 Linux 会建立一个 struct file 结构体实例变量与文件对应，然后把 struct file 结构体实例变量的指针放入 fd_array 数组中.

copy_process 函数调用copy_files，copy_files 最终会复制当前进程的 files_struct 结构到一个新的 files_struct 结构实例变量中，并让新进程的 files 指针指向这个新的 files_struct 结构实例变量.

sched_entity是 Linux 进程调度系统的一部分，被嵌入到了 Linux 进程数据结构中，与调度器进行关联，能间接地访问进程，这种高内聚低耦合的方式，保证了进程数据结构和调度数据结构相互独立.

```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/sched.h#L548
struct sched_entity {
	/* For load-balancing: */
	struct load_weight		load; //表示当前调度实体的权重
	struct rb_node			run_node;//红黑树的数据节点
	struct list_head		group_node;// 链表节点，被链接到 percpu 的 rq->cfs_tasks
	unsigned int			on_rq;//当前调度实体是否在就绪队列上

	u64				exec_start;//当前实体上次被调度执行的时间
	u64				sum_exec_runtime;//当前实体总执行时间
	u64				vruntime;//当前实体的虚拟时间
	u64				prev_sum_exec_runtime;//截止到上次统计，进程执行的时间

	u64				nr_migrations;//实体执行迁移的次数

#ifdef CONFIG_FAIR_GROUP_SCHED
	int				depth;// 表示当前实体处于调度组中的深度
	struct sched_entity		*parent;
	/* rq on which this entity is (to be) queued: */
	struct cfs_rq			*cfs_rq;
	/* rq "owned" by this entity/group: */
	struct cfs_rq			*my_q;
	/* cached value of my_q->h_nr_running */
	unsigned long			runnable_weight;
#endif

#ifdef CONFIG_SMP
	/*
	 * Per entity load average tracking.
	 *
	 * Put into separate cache line so it does not
	 * collide with read-mostly values above.
	 */
	struct sched_avg		avg; // 记录当前实体对于CPU的负载
#endif
};

//https://elixir.bootlin.com/linux/v6.5.1/source/kernel/sched/sched.h#L950
/*
 * This is the main, per-CPU runqueue data structure.
 *
 * Locking rule: those places that want to lock multiple runqueues
 * (such as the load balancing or the thread migration code), lock
 * acquire operations must be ordered by ascending &runqueue.
 */
struct rq {
	/* runqueue lock: */
	raw_spinlock_t		__lock;

	/*
	 * nr_running and cpu_load should be in the same cacheline because
	 * remote CPUs use both these fields when doing load calculation.
	 */
	unsigned int		nr_running; //多少个就绪运行进程
#ifdef CONFIG_NUMA_BALANCING
	unsigned int		nr_numa_running;
	unsigned int		nr_preferred_running;
	unsigned int		numa_migrate_on;
#endif
#ifdef CONFIG_NO_HZ_COMMON
#ifdef CONFIG_SMP
	unsigned long		last_blocked_load_update_tick;
	unsigned int		has_blocked_load;
	call_single_data_t	nohz_csd;
#endif /* CONFIG_SMP */
	unsigned int		nohz_tick_stopped;
	atomic_t		nohz_flags;
#endif /* CONFIG_NO_HZ_COMMON */

#ifdef CONFIG_SMP
	unsigned int		ttwu_pending;
#endif
	u64			nr_switches;

#ifdef CONFIG_UCLAMP_TASK
	/* Utilization clamp values based on CPU's RUNNABLE tasks */
	struct uclamp_rq	uclamp[UCLAMP_CNT] ____cacheline_aligned;
	unsigned int		uclamp_flags;
#define UCLAMP_FLAG_IDLE 0x01
#endif

	struct cfs_rq		cfs; //作用于完全公平调度算法的运行队列
	struct rt_rq		rt; //作用于实时调度算法的运行队列
	struct dl_rq		dl; //作用于EDF调度算法的运行队列

#ifdef CONFIG_FAIR_GROUP_SCHED
	/* list of leaf cfs_rq on this CPU: */
	struct list_head	leaf_cfs_rq_list;
	struct list_head	*tmp_alone_branch;
#endif /* CONFIG_FAIR_GROUP_SCHED */

	/*
	 * This is part of a global counter where only the total sum
	 * over all CPUs matters. A task can increase this counter on
	 * one CPU and if it got migrated afterwards it may decrease
	 * it on another CPU. Always updated under the runqueue lock:
	 */
	unsigned int		nr_uninterruptible;

	struct task_struct __rcu	*curr; //这个运行队列当前正在运行的进程
	struct task_struct	*idle; //这个队列的空转进程
	struct task_struct	*stop; //这个队列的空转进程
	unsigned long		next_balance;
	struct mm_struct	*prev_mm; //这个运行队列上一次运行进程的mm_struct

	unsigned int		clock_update_flags; //时钟更新标志
	u64			clock; // 运行队列的时间
	/* Ensure that all clocks are in the same cache line */
	u64			clock_task ____cacheline_aligned;
	u64			clock_pelt;
	unsigned long		lost_idle_time;
	u64			clock_pelt_idle;
	u64			clock_idle;
#ifndef CONFIG_64BIT
	u64			clock_pelt_idle_copy;
	u64			clock_idle_copy;
#endif

	atomic_t		nr_iowait;

#ifdef CONFIG_SCHED_DEBUG
	u64 last_seen_need_resched_ns;
	int ticks_without_resched;
#endif

#ifdef CONFIG_MEMBARRIER
	int membarrier_state;
#endif

#ifdef CONFIG_SMP
	struct root_domain		*rd;
	struct sched_domain __rcu	*sd;

	unsigned long		cpu_capacity;
	unsigned long		cpu_capacity_orig;

	struct balance_callback *balance_callback;

	unsigned char		nohz_idle_balance;
	unsigned char		idle_balance;

	unsigned long		misfit_task_load;

	/* For active balancing */
	int			active_balance;
	int			push_cpu;
	struct cpu_stop_work	active_balance_work;

	/* CPU of this runqueue: */
	int			cpu;
	int			online;

	struct list_head cfs_tasks;

	struct sched_avg	avg_rt;
	struct sched_avg	avg_dl;
#ifdef CONFIG_HAVE_SCHED_AVG_IRQ
	struct sched_avg	avg_irq;
#endif
#ifdef CONFIG_SCHED_THERMAL_PRESSURE
	struct sched_avg	avg_thermal;
#endif
	u64			idle_stamp;
	u64			avg_idle;

	unsigned long		wake_stamp;
	u64			wake_avg_idle;

	/* This is used to determine avg_idle's max value */
	u64			max_idle_balance_cost;

#ifdef CONFIG_HOTPLUG_CPU
	struct rcuwait		hotplug_wait;
#endif
#endif /* CONFIG_SMP */

#ifdef CONFIG_IRQ_TIME_ACCOUNTING
	u64			prev_irq_time;
#endif
#ifdef CONFIG_PARAVIRT
	u64			prev_steal_time;
#endif
#ifdef CONFIG_PARAVIRT_TIME_ACCOUNTING
	u64			prev_steal_time_rq;
#endif

	/* calc_load related fields */
	unsigned long		calc_load_update;
	long			calc_load_active;

#ifdef CONFIG_SCHED_HRTICK
#ifdef CONFIG_SMP
	call_single_data_t	hrtick_csd;
#endif
	struct hrtimer		hrtick_timer;
	ktime_t 		hrtick_time;
#endif

#ifdef CONFIG_SCHEDSTATS
	/* latency stats */
	struct sched_info	rq_sched_info;
	unsigned long long	rq_cpu_time;
	/* could above be rq->cfs_rq.exec_clock + rq->rt_rq.rt_runtime ? */

	/* sys_sched_yield() stats */
	unsigned int		yld_count;

	/* schedule() stats */
	unsigned int		sched_count;
	unsigned int		sched_goidle;

	/* try_to_wake_up() stats */
	unsigned int		ttwu_count;
	unsigned int		ttwu_local;
#endif

#ifdef CONFIG_CPU_IDLE
	/* Must be inspected within a rcu lock section */
	struct cpuidle_state	*idle_state;
#endif

#ifdef CONFIG_SMP
	unsigned int		nr_pinned;
#endif
	unsigned int		push_busy;
	struct cpu_stop_work	push_work;

#ifdef CONFIG_SCHED_CORE
	/* per rq */
	struct rq		*core;
	struct task_struct	*core_pick;
	unsigned int		core_enabled;
	unsigned int		core_sched_seq;
	struct rb_root		core_tree;

	/* shared state -- careful with sched_core_cpu_deactivate() */
	unsigned int		core_task_seq;
	unsigned int		core_pick_seq;
	unsigned long		core_cookie;
	unsigned int		core_forceidle_count;
	unsigned int		core_forceidle_seq;
	unsigned int		core_forceidle_occupation;
	u64			core_forceidle_start;
#endif

	/* Scratch cpumask to be temporarily used under rq_lock */
	cpumask_var_t		scratch_mask;

#if defined(CONFIG_CFS_BANDWIDTH) && defined(CONFIG_SMP)
	call_single_data_t	cfsb_csd;
	struct list_head	cfsb_csd_list;
#endif
};

//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/rbtree_types.h#L26
/*
 * Leftmost-cached rbtrees.
 *
 * We do not cache the rightmost node based on footprint
 * size vs number of potential users that could benefit
 * from O(1) rb_last(). Just not worth it, users that want
 * this feature can always implement the logic explicitly.
 * Furthermore, users that want to cache both pointers may
 * find it a bit asymmetric, but that's ok.
 */
struct rb_root_cached {
	struct rb_root rb_root; //红黑树的root
	struct rb_node *rb_leftmost; //红黑树最左子节点
};

//https://elixir.bootlin.com/linux/v6.5.1/source/kernel/sched/sched.h#L543
/* CFS-related fields in a runqueue */
struct cfs_rq {
	struct load_weight	load; //cfs_rq上所有调度实体的负载总和
	unsigned int		nr_running;//cfs_rq上所有的调度实体不含调度组中的调度实体
	unsigned int		h_nr_running;      /* SCHED_{NORMAL,BATCH,IDLE} */ //cfs_rq上所有的调度实体包含调度组中所有调度实体
	unsigned int		idle_nr_running;   /* SCHED_IDLE */
	unsigned int		idle_h_nr_running; /* SCHED_IDLE */

	u64			exec_clock; //当前 cfs_rq 上执行的时间
	u64			min_vruntime; //最小虚拟运行时间
#ifdef CONFIG_SCHED_CORE
	unsigned int		forceidle_seq;
	u64			min_vruntime_fi;
#endif

#ifndef CONFIG_64BIT
	u64			min_vruntime_copy;
#endif

	struct rb_root_cached	tasks_timeline; //所有调度实体的根

	/*
	 * 'curr' points to currently running entity on this cfs_rq.
	 * It is set to NULL otherwise (i.e when none are currently running).
	 */
	struct sched_entity	*curr; //当前调度实体
	struct sched_entity	*next; //下一个调度实体
	struct sched_entity	*last;//上一次执行过的调度实体
	struct sched_entity	*skip;

#ifdef	CONFIG_SCHED_DEBUG
	unsigned int		nr_spread_over;
#endif

#ifdef CONFIG_SMP
	/*
	 * CFS load tracking
	 */
	struct sched_avg	avg;
#ifndef CONFIG_64BIT
	u64			last_update_time_copy;
#endif
	struct {
		raw_spinlock_t	lock ____cacheline_aligned;
		int		nr;
		unsigned long	load_avg;
		unsigned long	util_avg;
		unsigned long	runnable_avg;
	} removed;

#ifdef CONFIG_FAIR_GROUP_SCHED
	unsigned long		tg_load_avg_contrib;
	long			propagate;
	long			prop_runnable_sum;

	/*
	 *   h_load = weight * f(tg)
	 *
	 * Where f(tg) is the recursive weight fraction assigned to
	 * this group.
	 */
	unsigned long		h_load;
	u64			last_h_load_update;
	struct sched_entity	*h_load_next;
#endif /* CONFIG_FAIR_GROUP_SCHED */
#endif /* CONFIG_SMP */

#ifdef CONFIG_FAIR_GROUP_SCHED
	struct rq		*rq;	/* CPU runqueue to which this cfs_rq is attached */

	/*
	 * leaf cfs_rqs are those that hold tasks (lowest schedulable entity in
	 * a hierarchy). Non-leaf lrqs hold other higher schedulable entities
	 * (like users, containers etc.)
	 *
	 * leaf_cfs_rq_list ties together list of leaf cfs_rq's in a CPU.
	 * This list is used during load balance.
	 */
	int			on_list;
	struct list_head	leaf_cfs_rq_list;
	struct task_group	*tg;	/* group that "owns" this runqueue */

	/* Locally cached copy of our task_group's idle value */
	int			idle;

#ifdef CONFIG_CFS_BANDWIDTH
	int			runtime_enabled;
	s64			runtime_remaining;

	u64			throttled_pelt_idle;
#ifndef CONFIG_64BIT
	u64                     throttled_pelt_idle_copy;
#endif
	u64			throttled_clock;
	u64			throttled_clock_pelt;
	u64			throttled_clock_pelt_time;
	int			throttled;
	int			throttle_count;
	struct list_head	throttled_list;
#ifdef CONFIG_SMP
	struct list_head	throttled_csd_list;
#endif
#endif /* CONFIG_CFS_BANDWIDTH */
#endif /* CONFIG_FAIR_GROUP_SCHED */
};

//https://elixir.bootlin.com/linux/v6.5.1/source/kernel/sched/sched.h#L2207
struct sched_class {

#ifdef CONFIG_UCLAMP_TASK
	int uclamp_enabled;
#endif

	void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags); //向运行队列中添加一个进程，入队
	void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags); //向运行队列中删除一个进程，出队
	void (*yield_task)   (struct rq *rq);
	bool (*yield_to_task)(struct rq *rq, struct task_struct *p);

	void (*check_preempt_curr)(struct rq *rq, struct task_struct *p, int flags); //检查当前进程是否可抢占

	struct task_struct *(*pick_next_task)(struct rq *rq);  //从运行队列中返回可以投入运行的一个进程

	void (*put_prev_task)(struct rq *rq, struct task_struct *p);
	void (*set_next_task)(struct rq *rq, struct task_struct *p, bool first);

#ifdef CONFIG_SMP
	int (*balance)(struct rq *rq, struct task_struct *prev, struct rq_flags *rf);
	int  (*select_task_rq)(struct task_struct *p, int task_cpu, int flags);

	struct task_struct * (*pick_task)(struct rq *rq);

	void (*migrate_task_rq)(struct task_struct *p, int new_cpu);

	void (*task_woken)(struct rq *this_rq, struct task_struct *task);

	void (*set_cpus_allowed)(struct task_struct *p, struct affinity_context *ctx);

	void (*rq_online)(struct rq *rq);
	void (*rq_offline)(struct rq *rq);

	struct rq *(*find_lock_rq)(struct task_struct *p, struct rq *rq);
#endif

	void (*task_tick)(struct rq *rq, struct task_struct *p, int queued);
	void (*task_fork)(struct task_struct *p);
	void (*task_dead)(struct task_struct *p);

	/*
	 * The switched_from() call is allowed to drop rq->lock, therefore we
	 * cannot assume the switched_from/switched_to pair is serialized by
	 * rq->lock. They are however serialized by p->pi_lock.
	 */
	void (*switched_from)(struct rq *this_rq, struct task_struct *task);
	void (*switched_to)  (struct rq *this_rq, struct task_struct *task);
	void (*prio_changed) (struct rq *this_rq, struct task_struct *task,
			      int oldprio);

	unsigned int (*get_rr_interval)(struct rq *rq,
					struct task_struct *task);

	void (*update_curr)(struct rq *rq);

#ifdef CONFIG_FAIR_GROUP_SCHED
	void (*task_change_group)(struct task_struct *p);
#endif

#ifdef CONFIG_SCHED_CORE
	int (*task_is_throttled)(struct task_struct *p, int cpu);
#endif
};

//https://elixir.bootlin.com/linux/v6.5.1/source/kernel/sched/sched.h#L2310
extern const struct sched_class stop_sched_class;//停止调度类
extern const struct sched_class dl_sched_class;//DeadLine调度类
extern const struct sched_class rt_sched_class;//实时调度类
extern const struct sched_class fair_sched_class;//CFS调度类
extern const struct sched_class idle_sched_class;//空转调度类

//https://elixir.bootlin.com/linux/v6.5.1/source/kernel/sched/fair.c#L12714
/*
 * All the scheduling class methods:
 */
DEFINE_SCHED_CLASS(fair) = {

	.enqueue_task		= enqueue_task_fair,
	.dequeue_task		= dequeue_task_fair,
	.yield_task		= yield_task_fair,
	.yield_to_task		= yield_to_task_fair,

	.check_preempt_curr	= check_preempt_wakeup,

	.pick_next_task		= __pick_next_task_fair,
	.put_prev_task		= put_prev_task_fair,
	.set_next_task          = set_next_task_fair,

#ifdef CONFIG_SMP
	.balance		= balance_fair,
	.pick_task		= pick_task_fair,
	.select_task_rq		= select_task_rq_fair,
	.migrate_task_rq	= migrate_task_rq_fair,

	.rq_online		= rq_online_fair,
	.rq_offline		= rq_offline_fair,

	.task_dead		= task_dead_fair,
	.set_cpus_allowed	= set_cpus_allowed_common,
#endif

	.task_tick		= task_tick_fair,
	.task_fork		= task_fork_fair,

	.prio_changed		= prio_changed_fair,
	.switched_from		= switched_from_fair,
	.switched_to		= switched_to_fair,

	.get_rr_interval	= get_rr_interval_fair,

	.update_curr		= update_curr_fair,

#ifdef CONFIG_FAIR_GROUP_SCHED
	.task_change_group	= task_change_group_fair,
#endif

#ifdef CONFIG_SCHED_CORE
	.task_is_throttled	= task_is_throttled_fair,
#endif

#ifdef CONFIG_UCLAMP_TASK
	.uclamp_enabled		= 1,
#endif
};
```

在 task_struct 结构中，会包含至少一个 sched_entity 结构的变量.

只要通过 sched_entity 结构变量的地址，减去它在 task_struct 结构中的偏移（由编译器自动计算），就能获取到 task_struct 结构的地址。这样就能达到通过 sched_entity 结构，访问 task_struct 结构的目的.

Linux 定义了一个进程运行队列结构，每个 CPU 分配一个这样的进程运行队列结构实例变量.

rq其中 task_struct 结构指针是为了快速访问特殊进程，而 rq 结构并不直接关联调度实体，而是包含了 cfs_rq、rt_rq、dl_rq，通过它们来关联调度实体.

有三个不同的运行队列，是因为作用于三种不同的调度算法.

load、exec_clock、min_vruntime、tasks_timeline 字段是 CFS 调度算法得以实现的关键, 且所有的调度实体都是通过红黑树组织起来的，即 cfs_rq 结构中的 tasks_timeline 字段.

task_struct 结构中包含了 sched_entity 结构。sched_entity 结构是通过红黑树组织起来的，红黑树的根在 cfs_rq 结构中，cfs_rq 结构又被包含在 rq 结构，每个 CPU 对应一个 rq 结构. 这样，就把所有运行的进程组织起来了.

从rq 数据结构中，发现Linux 是同时支持多个进程调度器的，不同的进程挂载到不同的运行队列中，如 rq 结构中的 cfs、rt、dl，然后针对它们这些结构，使用不同的调度器.

为了支持不同的调度器，Linux 定义了调度器类数据结构sched_class，它定义了一个调度器要实现哪些函数.

这个 sched_class 结构定义了一组函数指针. Linux 系统一共定义了五个 sched_class 结构的实例变量，这五个 sched_class 结构紧靠在一起，形成了 sched_class 结构数组.

> 实现一个新的调度器，就是实现这些对应的函数

这些类是有优先级的，它们的优先级是：stop_sched_class > dl_sched_class > rt_sched_class > fair_sched_class > idle_sched_class.

CFS 的设计理念是在有限的真实硬件平台上模拟实现理想的、精确的多任务 CPU.

Linux 会使用 CFS 调度器调度普通进程，CFS 调度器与其它进程调度器的不同之处在于没有时间片的概念，它是分配 CPU 使用时间的比例.

首先，CFS 调度器下的优先级叫权重，权重表示进程的优先级，各个进程按权重的比例分配 CPU 时间. 进程的时间 = CPU 总时间 * 进程的权重 / 就绪队列所有进程权重之和.

进程对外的编程接口中使用的是一个 nice 值，大小范围是（-20～19），数值越小优先级越大，意味着权重值越大，nice 值和权重之间可以转换的。Linux 提供了[sched_prio_to_weight](https://elixir.bootlin.com/linux/v6.5.1/source/kernel/sched/core.c#L11495)数组，用于转换 nice 值和权重.

一个进程每降低一个 nice 值，就能多获得 10% 的 CPU 时间。1024 权重对应 nice 值为 0，被称为 NICE_0_LOAD。默认情况下，大多数进程的权重都是 NICE_0_LOAD.

调度延迟是保证每一个可运行的进程，都至少运行一次的时间间隔.

随着进程的增加，每个进程分配的时间在减少，进程调度次数会增加，调度器占用的时间就会增加。因此，CFS 调度器的调度延迟时间的设定并不是固定的.

在 CFS 默认设置中，最小调度粒度时间是 0.75ms，用变量 sysctl_sched_min_granularity 记录, 由 `__sched_period` 函数负责计算. 其当超过 sched_nr_latency 时，因为无法保证调度延迟，因此转为保证最小调度粒度.

调度实体中的 vruntime 就是用来表示虚拟时间. 虚拟时间 vruntime 和实际时间（wtime）转换公式是`vruntime = wtime*( NICE_0_LOAD/weight)`.

CFS 调度器记录了每个进程的执行时间，为保证每个进程运行时间的公平，哪个进程运行的时间最少，就会让哪个进程运行. CFS 只需要保证每个进程运行的虚拟时间是相等的, 在选择下一个即将运行的进程时，只需要找到虚拟时间最小的进程就行了。这个计算过程由 calc_delta_fair 函数完成. 它传递给`__calc_delta`的 weight 参数是 NICE_0_LOAD，lw 参数正是调度实体中的 load_weight 结构体.

在运行队列中用红黑树结构组织进程的调度实体，这里进程虚拟时间正是红黑树的 key，这样进程就以进程的虚拟时间被红黑树组织起来了。红黑树的最左子节点，就是虚拟时间最小的进程，随着时间的推移进程会从红黑树的左边跑到右，然后从右边跑到左边.

如果一个进程比其它进程的虚拟时间小，它就应该运行达到和其它进程的虚拟时间持平，直到它的虚拟时间超过其它进程，这时就要停下来，这样其它进程才能被调度运行.

虚拟时间的方案还存在问题: 虚拟时间就是一个数据，如果没有任何机制对它进行更新，就会导致一个进程永远运行下去，因为那个进程的虚拟时间没有更新，虚拟时间永远最小.

linux存在定时周期调度机制: Linux 启动会启动定时器，这个定时器每 1/1000、1/250、1/100 秒（根据配置不同选取其一），产生一个时钟中断，在中断处理函数中最终会调用一个 scheduler_tick 函数.

```c
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));//获取当前时间 
    u64 delta_exec;
    delta_exec = now - curr->exec_start;//间隔时间 
    curr->exec_start = now;
    curr->sum_exec_runtime += delta_exec;//累计运行时间 
    curr->vruntime += calc_delta_fair(delta_exec, curr);//计算进程的虚拟时间 
    update_min_vruntime(cfs_rq);//更新运行队列中的最小虚拟时间，这是新建进程的虚拟时间，避免一个新建进程因为虚拟时间太小而长时间占用CPU
}
static void entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
    update_curr(cfs_rq);//更新当前运行进程和运行队列相关的时间
    if (cfs_rq->nr_running > 1)//当运行进程数量大于1就检查是否可抢占
        check_preempt_tick(cfs_rq, curr);
}
#define for_each_sched_entity(se) \
        for (; se; se = NULL)
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &curr->se;//获取当前进程的调度实体 
    for_each_sched_entity(se) {//仅对当前进程的调度实体
        cfs_rq = cfs_rq_of(se);//获取当前进程的调度实体对应运行队列
        entity_tick(cfs_rq, se, queued);
    }
}
void scheduler_tick(void)
{
    int cpu = smp_processor_id();
    struct rq *rq = cpu_rq(cpu);//获取运行CPU运行进程队列
    struct task_struct *curr = rq->curr;//获取当进程
    update_rq_clock(rq);//更新运行队列的时间等数据
    curr->sched_class->task_tick(rq, curr, 0);//更新当前时间的虚拟时间
}

static void check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    unsigned long ideal_runtime, delta_exec;
    struct sched_entity *se;
    s64 delta;
    //计算当前进程在本次调度中分配的运行时间
    ideal_runtime = sched_slice(cfs_rq, curr);
    //当前进程已经运行的实际时间
    delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    //如果实际运行时间已经超过分配给进程的运行时间，就需要抢占当前进程。设置进程的TIF_NEED_RESCHED抢占标志。
    if (delta_exec > ideal_runtime) {
        resched_curr(rq_of(cfs_rq));
        return;
    }
    //因此如果进程运行时间小于最小调度粒度时间，不应该抢占
    if (delta_exec < sysctl_sched_min_granularity)
        return;
    //从红黑树中找到虚拟时间最小的调度实体
    se = __pick_first_entity(cfs_rq);
    delta = curr->vruntime - se->vruntime;
    //如果当前进程的虚拟时间仍然比红黑树中最左边调度实体虚拟时间小，也不应该发生调度
    if (delta < 0)
        return;
}
```

scheduler_tick 函数会调用进程调度类的 task_tick 函数，对于 CFS 调度器就是 task_tick_fair 函数。但是真正做事的是 entity_tick 函数，entity_tick 函数中调用了 update_curr 函数更新当前进程虚拟时间.

entity_tick 函数的最后，调用了 check_preempt_tick 函数，用来检查是否可以抢占调度.

，如果需要抢占就会调用 resched_curr 函数设置进程的抢占标志，但是这个函数本身不会调用进程调度器函数，而是在进程从中断或者系统调用返回到用户态空间时，检查当前进程的调度标志，然后根据需要调用进程调度器函数.

如果设计需要进行进程抢占调度，Linux 就会在适当的时机进行进程调度，进程调度就是调用进程调度器入口函数，该函数会选择一个最合适投入运行的进程，然后切换到该进程上运行.

进程调度器入口函数:
```c
static void __sched notrace __schedule(bool preempt)
{
    struct task_struct *prev, *next;
    unsigned long *switch_count;
    unsigned long prev_state;
    struct rq_flags rf;
    struct rq *rq;
    int cpu;
    cpu = smp_processor_id();
    rq = cpu_rq(cpu);//获取当前CPU的运行队列
    prev = rq->curr; //获取当前进程 
    rq_lock(rq, &rf);//运行队列加锁
    update_rq_clock(rq);//更新运行队列时钟
    switch_count = &prev->nivcsw;
    next = pick_next_task(rq, prev, &rf);//获取下一个投入运行的进程
    clear_tsk_need_resched(prev); //清除抢占标志
    clear_preempt_need_resched();
    if (likely(prev != next)) {//当前运行进程和下一个运行进程不同，就要进程切换
        rq->nr_switches++; //切换计数统计
        ++*switch_count;
        rq = context_switch(rq, prev, next, &rf);//进程机器上下文切换
    } else {
        rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);
        rq_unlock_irq(rq, &rf);//解锁运行队列
    }
}
void schedule(void)
{
    struct task_struct *tsk = current;//获取当前进程
    do {
        preempt_disable();//关闭内核抢占
        __schedule(false);//进程调用
        sched_preempt_enable_no_resched();//开启内核抢占
    } while (need_resched());//是否需要再次重新调用
}

static inline struct task_struct *pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
    const struct sched_class *class;
    struct task_struct *p;
    //这是对CFS的一种优化处理，因为大部分进程属于CFS管理
    if (likely(prev->sched_class <= &fair_sched_class &&
           rq->nr_running == rq->cfs.h_nr_running)) {
        p = pick_next_task_fair(rq, prev, rf);//调用CFS的对应的函数
        if (unlikely(p == RETRY_TASK))
            goto restart;
        if (!p) {//如果没有获取到运行进程
            put_prev_task(rq, prev);//将上一个进程放回运行队列中
            p = pick_next_task_idle(rq);//获取空转进程
        }
        return p;
    }
restart:
    for_each_class(class) {//依次从最高优先级的调度类开始遍历
        p = class->pick_next_task(rq);
        if (p)//如果在一个调度类所管理的运行队列中挑选到一个进程，立即返回
            return p;
    }
    BUG();//出错
}

struct task_struct *pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
    struct cfs_rq *cfs_rq = &rq->cfs;
    struct sched_entity *se;
    struct task_struct *p;
    if (prev)
        put_prev_task(rq, prev);//把上一个进程放回运行队列
    do {
        se = pick_next_entity(cfs_rq, NULL);//选择最适合运行的调度实体
        set_next_entity(cfs_rq, se);//对选择的调度实体进行一些处理
        cfs_rq = group_cfs_rq(se);
    } while (cfs_rq);//在没有调度组的情况下，循环一次就结束了
    p = task_of(se);//通过se获取包含se的进程task_struct
    return p;
}

struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq)
{
    struct rb_node *left = rb_first_cached(&cfs_rq->tasks_timeline);//先读取在tasks_timeline中rb_node指针
    if (!left)
        return NULL;//如果为空直接返回NULL
    //通过红黑树结点指针取得包含它的调度实体结构地址
    return rb_entry(left, struct sched_entity, run_node);
}
static struct sched_entity *__pick_next_entity(struct sched_entity *se)
{    //获取当前红黑树节点的下一个结点
    struct rb_node *next = rb_next(&se->run_node);
    if (!next)
        return NULL;//如果为空直接返回NULL
    return rb_entry(next, struct sched_entity, run_node);
}
static struct sched_entity *pick_next_entity(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    //获取Cfs_rq中的红黑树上最左节点上调度实体，虚拟时间最小
    struct sched_entity *left = __pick_first_entity(cfs_rq);
    struct sched_entity *se;
    if (!left || (curr && entity_before(curr, left)))
        left = curr;//可能当前进程主动放弃CPU，它的虚拟时间比红黑树上的还小，所以left指向当前进程调度实体
    se = left; 
    if (cfs_rq->skip == se) { //如果选择的调度实体是要跳过的调度实体
        struct sched_entity *second;
        if (se == curr) {//如果是当前调度实体
            second = __pick_first_entity(cfs_rq);//选择运行队列中虚拟时间最小的调度实体
        } else {//否则选择红黑树上第二左的进程节点
            second = __pick_next_entity(se);
            //如果次优的调度实体的虚拟时间，还是比当前的调度实体的虚拟时间大
            if (!second || (curr && entity_before(curr, second)))
                second = curr;//让次优的调度实体也指向当前调度实体
        }
        //判断left和second的虚拟时间的差距是否小于sysctl_sched_wakeup_granularity
        if (second && wakeup_preempt_entity(second, left) < 1)
            se = second;
    }
    if (cfs_rq->next && wakeup_preempt_entity(cfs_rq->next, left) < 1) {
        se = cfs_rq->next;
    } else if (cfs_rq->last && wakeup_preempt_entity(cfs_rq->last, left) < 1) {
             se = cfs_rq->last;
    }
    clear_buddies(cfs_rq, se);//需要清除掉last、next、skip指针
    return se;
}
```

之所以在循环中调用 `__schedule` 函数执行真正的进程调度，是因为在执行调度的过程中，有些更高优先级的进程进入了可运行状态，因此它就要抢占当前进程. `__schedule`函数中会更新一些统计数据，然后调用 pick_next_task 函数挑选出下一个进程投入运行。最后，如果当前进程和下一个要运行的进程不同，就要进行进程机器上下文切换，其中会切换地址空间和 CPU 寄存器.

pick_next_task 函数只是个框架函数，它的逻辑也很清楚，会依照优先级调用具体调度器类的函数完成工作，对于 CFS 则会调用 pick_next_task_fair 函数.

pick_next_entity 函数选择虚拟时间最小的调度实体，然后调用 set_next_entity 函数，对选择的调度实体进行一些必要的处理，主要是将这调度实体从运行队列中拿出来.

pick_next_entity具体逻辑:首先，它调用了相关函数，从运行队列上的红黑树中查找虚拟时间最少的调度实体，然后处理要跳过调度的情况，最后决定挑选的调度实体是否可以抢占并返回它

代码的调用路径最终会返回到 `__schedule` 函数中，这个函数中就是上一个运行的进程和将要投入运行的下一个进程，最后调用 context_switch 函数，完成两个进程的地址空间和机器上下文的切换，一次进程调度工作结束. 

CFS核心就是让虚拟时间最小的进程最先运行， 一旦进程运行虚拟时间就会增加，最后尽量保证所有进程的虚拟时间相等，谁小了就要多运行，谁大了就要暂停运行.

### 线程实现
linux将线程当做进程来实现.

线程仅是在call clone时传递一些参数标志来指明需要共享的资源: `clone(CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND, 0)`

> windows和sum solaris等提供专门的线程机制(也叫轻量级进程, lightweight process).
> [clone()参数标志](Linux内核设计与实现v3-3.4.1 创建线程), 它定义在`<linux/sched.h>`

内核线程和普通进程的区别是它没有独立的地址空间(即指向地址空间的mm属性被设为NULL), 它只运行在内核空间, 可被调度和抢占.
内核线程只能由其他内核线程创建, 并从kthreadd内核进程中衍生出来, 创建接口定义在`<linux/kthread.h>`中. 新创建的线程处于不可运行状态, 需要wake_up_process()唤醒.

## 进程组(process group)
每个进程除了有进程id外, 还隶属于一个进程组. 进程组是一个或多个**进程**(进程+子进程+后裔进程)的集合.进程组会有一个进程组领导进程 (process group leader)，领导进程的PID则成为进程组的ID (process group ID, PGID)，以识别进程组.
PGID不会因领导进程退出而受到影响，且fork调用也不会改变PGID.

多个进程属于进程组的情况是多个进程用管道“|”号连接进行执行。如果在命令行执行单个进程时这个进程组只有这一个进程.

> `pstree -pg`可查看进程的pid和pgid, 也可通过`ps`查看.
> linux和unix的新进程由fork而来, 因此原远程的pid就是当前进程的ppid.
> 当用户从键盘发出一个信号时, 该信号会被送给当前与键盘相关的进程组中的所有成员.

```sh
➜  ~ ps -ax -o pid,pgid,ppid,args| cat # `-ax`=`-A`
  PID  PGID  PPID COMMAND
15967 15967  4325 /usr/bin/zsh
16547 16547 15967 ps -o pid,pgid,ppid,args
16548 16547 15967 cat
```

## 会话(session)
一个或多个**进程组**的集合.新建会话时，当前进程（会话中唯一的进程）成为会话首进程(session leader)，也是当前进程组的组长进程，其进程号为该会话的SID(session id).它通常是登录 shell，也可以是调用`setsid`成功新建会话的孤儿进程.

会话的每个进程组称为一个工作(job).一个会话一般包含**一个会话首进程、至多一个前台进程组和任意个后台进程组，控制终端(由这些组共享)可有可无**.

每次用户登录终端时会产生一个会话（session）. 从用户登录开始到用户退出为止，这段时间内在该终端执行的进程都属于这一个会话.

打开控制终端会致使会话首进程成为终端的控制进程即会话首进程与终端挂钩了. 一旦断开了与终端的连接(比如,关闭了终端窗口),控制进程将会收到 SIGHUP 信号.

> TPGID= 控制终端进程组ID（由控制终端修改，用于指示当前前台进程组）

> 一个终端至多只能成为一个会话的控制终端.

前后台进场分工:
- 前台进程组 : 该进程组中的进程可以向终端设备进行读、写操作（属于该组的进程都可以从终端获得输入）.通常用该进程组的`PGID=TPGID`来判断前台进程组.
- 后台进程组 : 该进程组中的进程只能向终端设备进行写操作.

```sh
➜  ~ ps -A -o pid,pgid,ppid,sid,args,tty,tpgid
  PID  PGID  PPID   SID COMMAND                     TT       TPGID
 4326  4326  4321  4326 /usr/bin/zsh                pts/0     9125
 9119  9119  4326  4326 ping localhost -c 10        pts/0     9125
 9125  9125  4326  4326 ps -o pid,pgid,ppid,sid,arg pts/0     9125
```

**进程组领导进程不能成为新会话首进程，但新会话首进程必定成为进程组领导进程.**

一个命令可以通过在末尾加上&方式让它在后台运行:
```sh
➜  ~ ping localhost -c 4 & 
[1] 6954 # 1表示工作号,6954为PGID
```

结束工作:
```sh
$kill -SIGTERM -6954 #通过在PGID前面加-来表示是一个PGID而不是PID
```
或者
```sh
$kill -SIGTERM %1 #发送给工作1
```

查看后台工作:
```sh
$fg %1
```

将当前运行的程序放在后台:
```sh
<CTRL> + z
```

停止执行当前运行的程序:
```sh
<CTRL> + c
```

列出所有后台程序:
```sh
$ jobs
```

## env
每个进程都有一份kv形式的环境列表, 由 fork()创建时继承自父进程的环境副
本. 这也为父子进程间通信提供了一种机制. 当进程调用 exec()时可指定新的环境列表.

在绝大多数 shell 中,可使用 export 命令来创建环境变量. 当前shell的环境变量可通过`env`命令查看.

set,env,export的区别:
- set : 显示(设置)**shell变量**,包括的私有变量以及用户变量，不同类的shell有不同的私有变量(bash,ksh,csh每中shell私有变量都不一样)
- env : 显示(设置)**用户环境变量**
- export : 显示(设置)当前**导出成**用户环境变量的shell变量

## 命令行参数
argv[0]包含了调用程序的名称,可以利用这一特性玩个实用的小技巧. 首先为同一程序创建多个链接(即名称不同)
,然后让该程序查看 argv[0],并根据调用程序的名称来执行不同任务, poweroff, reboot,half命令均是该技术应用的一个例子.

SUSv3 规定使用 ARG_MAX 常量(定义于`<limits.h>`)或者调用`sysconf (_SC_ARG_MAX)`函数以确定argv容量的上限值.

## init进程
系统引导时,内核会创建一个名为 init 的特殊进程,即"所有进程之父",该进程的相应
程序文件为`/sbin/init`(当前主流是systemd). 系统的所有进程不是由 init(使用 frok())`亲自`创建,就是由其后代
进程创建. init 进程的进程号总为 1,且总是以超级用户权限运行. 谁(哪怕是超级用户)都
不能`杀死`init 进程,只有关闭系统才能终止该进程. init 的主要任务是创建并监控系统运
行所需的一系列进程.

## 终止
终止一个进程有两种方法:
1. 进程可使用_exit()系统调用(或相关的exit()库函数),请求退出
2. 向进程传递信号,将其`杀死`

无论以何种方式退出,进程都会生成`终止状态`,一个非负小整数,可供父进程的 wait()系统调用检测. 在调用_exit()的
情况下,进程会指明自己的终止状态; 若由信号来`杀死`进程,则会根据导致进程`死亡`的信号类型来设置进程的终止状态.
有时会将传递进_exit()的参数称为进程的`退出状态`,以示与被killed的终止状态有所不同,后者(终止状态)要么指传递给_exit()的参数值,要么表示“杀死”进程的信号.

根据惯例,终止状态为 0 表示进程`功成身退`,非 0 则表示有错误发生. 大多数 shell 会
将前一执行程序的终止状态保存于 shell 变量`$?`中.

## 孤儿进程
**父进程先于子进程结束**，这时的子进程应该称作`孤儿进程（Orphan）`，它将被`1`号进程（即init 进程,通常是systemd）接管，init 进程成为其父进程.

## 僵尸进程(Zombie)
**子进程先于父进程结束**，但是其父进程尚未对其进行善后处理（获取已终止子进程的有关信息、释放相关资源）的进程.

一个已经终止,但父进程没有函数调用`wait/waitpid`等待子进程结束，也没有注册`SIGCHLD`信号处理函数，结果使得子进程的进程列表信息无法回收，就变成了僵尸进程.

> unix提供了一种机制可以让父进程知道子进程结束时的状态信息. 这种机制就是: 在每个进程退出的时候,内核释放该进程所有的资源,包括打开的文件,占用的内存等, 但是仍然为其保留一定的信息(包括进程号the process ID,退出状态the termination status of the process,运行时间the amount of CPU time taken by the process等, 要直到父进程通过wait / waitpid来取时才释放.

> kill无法杀死僵尸进程, 因为它已死亡(确保了父进程总是可以执行 wait()方法), 可通过杀死父进程的方法解决; 如果父进程是init, 则需要重启系统.

## 守护进程(daemon)
一个在后台运行并且不受任何终端控制的进程.

> 因此守护进程不属于任何终端, 因此它通过系统调用`syslog`来输出日志. 

守护进程的特点： 
1. 自成进程组，自成会话，与控制终端脱关联
2. 守护进程的父进程是1号进程
3. 守护进程的命令一般以字符d结尾 
4. 守护进程的生命周期是7*24小时不掉线

### 创建守护进程为什么要fork两次
1. 第一次fork: 为了脱离终端控制的魔爪. 调用了fork后，子进程拷贝了父进程的会话期、进程组、控制终端等资源、虽然父进程退出了，但是这些资源并没有改变，因此，需要用`setsid`来使得该子进程完全独立出来，从而摆脱其他进程的控制.
2. 第二次fork(不是必须的): 子进程调用`setsid`后成立了会话首进程,有了打开控制终端的能力，再fork一次，孙子进程就不能打开控制终端了(没调用`setsid`,不是会话首进程).

## 守护进程与后台进程

通过&符号，可以把命令放到后台执行。它与守护进程是不同的：
- 守护进程与终端无关，是被init进程收养的孤儿进程；而后台进程的父进程是终端，仍然可以在终端打印
- 守护进程在关闭终端时依然坚挺；而后台进程会随用户退出而停止，除非加上nohup
- 守护进程改变了会话、进程组、工作目录和文件描述符，后台进程直接继承父进程（shell）的

> 换句话说：守护进程就是默默地奋斗打拼的有为青年，而后台进程是默默继承老爸资产的富二代.

## 查看进程的信息
```sh
ll /proc/$PID
```

output:
- cwd : 符号链接,指向进程执行目录
- exe : 符号连接,执行程序的绝对路径\
- cmdline : 程序运行时输入的命令行命令
- environ : 进程运行时的环境变量
- fd : 目录,进程打开或使用的文件的符号连接

> 内核进程的话`/proc/$PID/exe`不指向具体路径.
> top -p $PID中该内核进程的VIRT和RES都显示为0

## 进程时间
一个进程维护3个时间:
- real : 进程运行的总时间
- user : 用户态的cpu时间,即cpu执行用户指令所用时间
- sys : 内核态的cpu时间,即cpu执行内核程序所用时间

## 工作目录
进程的当前工作目录继承自其父进程. 进程可以用chdir函数更改其工作目录.

## 文件描述符(file descriptor)
文件描述符是一个小的非负整数, kenel用以标识一个特定进程正在访问的文件. 当内核打
开一个现有文件或创建一个新文件时, 它就返回一个文件描述符. 当读、写文件时，就可使
用它.

## 进程时间
即CPU时间, 用以度量进程使用的CPU资源，是用户CPU时间和系统CPU时间之和.

进程时间以时钟滴答计算, 通常每秒钟取为 50、60或100个滴答.

当度量一个进程的执行时间时(比如time命令), 类Unix系统会使用三个时间值:
- 时钟时间 : 进程运行的时间段
- 用户CPU时间 :　执行用户指令所用的时间量
- 系统CPU时间 :　为该进程执行内核指令所用的时间

> 时钟滴答可使用sysconf函数获取, example中有示例.

## 错误处理
在类unix系统中, 函数出错时通常会返回一个负值, 而且整型变量errno通常设置为具有特定信息的一个值.

文件`<errno.h>`中定义了变量errno以及赋与它的各种常量. 这些常量都以`E`开头.

在支持多线程的环境中, 每个线程都有属于自己的局部errno, 以避免相互干扰.

> Unix手册第 2部分的第 1页， intro(2) 列出了所有这些出错常量; 而linux是包含在errno(3)手册页中.

### 出错恢复
错误分为两类: 致命性和非致命性.

发生致命性错误后, 程序通常会输出错误信息并退出, 但也允许拦截该错误自行处理, 比如golang的`recover()`. 

常见的处理非致命错误的方法是: 等待再重试.

## stdin(0), stdout(1), stderr(2)
通常运行一个程序时, 所有的shell都为其打开三个文件描述符(更确切地说是程序继承了 shell 文件描述符的副本)：标准输入、标
准输出以及标准出错. 如果不做特殊处理，则这三个描述符都连向终端. 大多数shell都会提供一种方法，使其中任何一个或所有这三个描述符都能重新定向到某一个文件.

> 记忆: 没有(0)要输入(in), 有(1)要输出(out), 其他(2)是错误(err)
> 非守护进程都有一个关联的控制终端, 进程的stdin, stdout, stderr都默认连接到该终端.

## 进程关联的用户id/组id
- 实际用户id/实际组id : 来自登录回话, 表示`我实际上是谁`
- 有效用户id/有效组id/附属组id : 决定了文件访问权限
- 保存的设置用户id/保存的设置组id : 有exec函数使用, 比如`passwd`命令

## 能力(Capabilities)
从kernel 2.2起, Linux 把传统上赋予超级用户的权限划分为一组相互独立的单元即`能力`. 每次特权操作都与特定的能力相关,仅当进程具有特定能力时,才能执行相应操作.
传统意义上的超级用户进程(有效用户 ID 为 0)则相应开启了所有能力.

## 资源限制
使用系统调用 setrlimit(), 进程可为自己消耗的各类资源设定一个上限. 此类资源限制的每一项均有两个相关值:软限制
(soft limit)限制了进程可以消耗的资源总量, 硬限制(hard limit)限制软限制的调整上限. 非特权
进程在针对特定资源调整软限制值时,可将其设置为 0 到相应硬限制值之间的任意值,但硬限制值则只能调低,不能调高.

由 fork()创建的新进程,会继承其父进程对资源限制的设置.
使用 ulimit 命令可调整 shell 的资源限制. shell 为执行命令所创建的子进程会继承上述资源设置.

## 虚拟地址表
由于物理内存的大小是受限制的，所以进程运行在自身的内存沙盒内即“虚拟内存地址（virtual address space）”，也称作 虚拟内存（Virtual Memory）.

字节在这个虚拟地址空间内的地址不再和处理器放在地址总线上的地址相同. 因此必须建立转换数据结构和系统将虚拟地址空间中的字节映射到物理字节.

虚拟地址可参见下图(`/proc/$PID/maps`)
[](https://cdn-images-1.medium.com/max/1300/1*ImbY2Tb3wZaeuKblwarFTg.jpeg)

当 CPU 执行一个指令需要引用内存地址时。首先将在 VMA（Virtual Memory Areas）中的逻辑地址转换为线性地址。这个转换通过 MMU 完成.
[](https://cdn-images-1.medium.com/max/1040/1*xek5BQhJhWqsOPAaA5uROw.png)

由于逻辑地址太大几乎很难独立的管理，所以引入`页（pages）` 进行管理。当必要的分页操作被激活后， 虚拟地址空间被分成更小的称作页的区域 （大部分操作系统下是 4KB，可以修改）。页是虚拟内存中数据内存管理的最小单元。**虚拟内存不存储任何内容，而是简单的将程序地址空间映射到底层物理内存之上**.

如果堆上有足够的空间的满足我们代码的内存申请，内存分配器可以完成内存申请无需内核参与，否则将通过操作系统调用`brk`进行扩展堆，通常是申请一大块内存.

堆内存地址增长,见下图:
- [](https://cdn-images-1.medium.com/max/1040/1*mvi6PRy9wu0KmBcP9YT5Cw.png)

应用程序通过系统调用`brk （sbrk/mmap 等）`获得内存. 内核调用它只更新堆VMA.

> 如何减少内存碎片化呢？答案取决是使用哪种内存分配算法，也就是使用哪个底层库, 比如tcmalloc和jemlloc.

参考:
- [Go 内存分配器可视化指南](https://blog.learngoprogramming.com/a-visual-guide-to-golang-memory-allocator-from-ground-up-e132258453ed)

## example
1. 从标准输入中读取并写入标准输出
```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	reader := bufio.NewReader(os.Stdin)

	for {
		result, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println("stdin read error:", err)
		}
		if result == "\n" {
			fmt.Println("quit!", result)

			break
		}

		fmt.Fprintf(os.Stdout,"output: %s\n", result)
	}
}
```

1. 获取进程的pid
```go
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Fprintf(os.Stdout, "pid: %d\n", os.Getpid())
}
```

1. 从输入读取命令并执行
```go
package main

import (
	"bytes"
	"log"
	"os"
	"os/exec"
)

func main() {
	args := os.Args
	if len(args) != 2 {
		log.Println("usage: run cmd")
		os.Exit(1)
	}

	cmdPath, err := exec.LookPath(args[1])
	CheckErr(err)

	cmd := exec.Command(cmdPath)

	var b bytes.Buffer
	cmd.Stdout = &b
	cmd.Stderr = &b

	err = cmd.Start()
	CheckErr(err)

	log.Println("Waiting for command to finish...")
	err = cmd.Wait()
	CheckErr(err)
	log.Printf("Command finished with output: %s\n", string(b.Bytes()))
}

func CheckErr(err error) {
	if err != nil {
		log.Fatal(err)
	}
}
```

1. 获取每秒的时钟滴答数
```go
package main

import (
	"fmt"
)

/*
#include <unistd.h>
#include <sys/types.h>
#include <pwd.h>
#include <stdlib.h>
*/
import "C"

func main() {
	var sc_clk_tck C.long
	sc_clk_tck = C.sysconf(C._SC_CLK_TCK)
	fmt.Printf("SysConf Clock Ticks: %d\n", sc_clk_tck)
}
```

## 进程的地址空间
在操作系统中,每个进程都是独占内存,互相之间不干扰即有独立的进程内存空间. 可通过pamp命令查看进程的内存空间布局.

进程的地址空间构成:
1. 代码段(Code Segment)/文本段 : 放可执行代码
1. 数据段

   1. 数据 : 已初始化好的变量
   1. bss : 已初始化为零值的变量
1. 堆(heap) : 程序可从该区域(由`malloc()`)动态分配额外内存. 生长方向: 低 -> 高.
1. 栈段(Stack segment)

   存放: 局部变量, 函数参数, 函数调用的返回地址. 生长方向: 高 -> 低.

只有进程使用内存的时候,才会使用内存管理的系统调用来申请,但是这还不代表真的就有了对应的物理内存. 实际上只有真的写入数据的时候,发现没有对应物理内存才会触发一个缺页中断来分配
物理内存.

在堆里面分配内存的系统调用是brk和mmap:
1. 当分配的内存数量比较小的时候使用 brk,会和原来的堆的数据连在一起
1. 当分配的内存数量比较大的时候使用 mmap,会重新划分一块区域

> linux使用伙伴系统(buddy system)来维护空闲分页, 可通过`/proc/buddyinfo`查看伙伴系统的信息.

### 分页回收
分页回收(page reclaiming) : 进程请求不到有效的分页时, kernel将尝试释放一定数量的分页(之前使用但不再使用且基于某些原因仍被标记为活跃的分页), 然后将这些分页分配给该进程.

linux kernel通过内核线程kswapd和内核函数`try_to_free_page()`来回收分页.

## FAQ
### 进程备忘
- jbd(journaling block driver)是文件系统的日志功能，jbd2是ext4文件系统版本.  

	常用的文件系统使用日志功能来保证文件系统的完整性，在写入新的数据到磁盘之前，会先将元数据写入日志；这样做可以保证在写入真实数据前后，一旦发生错误的话, 日志功能将很容易回滚到之前的状态，确保不会发生文件系统崩溃的情况.
	对文件系统的操作太频繁，导致IO压力过大.