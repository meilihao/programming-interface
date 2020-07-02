# task
在 Linux 里面，无论是进程，还是线程，到了内核里面，统一都叫任务（Task），由一个统一的结构 [task_struct](https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h#L632) 进行管理.

task_struct即进程描述符(process descriptor)也叫进程控制块(PCB))存在任务队列(task list, 双向循环链表)中.

## 成员
![task member](/misc/img/process/task.png)

### task id
```c
pid_t pid; // process id
pid_t tgid; // thread group ID, 可判断tast_struct代表的是一个进程还是一个线程
struct task_struct *group_leader;
```
任何一个进程,如果只有主线程,那pid是自己,tgid是自己,group_leader指向的还是自己; 但如果一个进程创建了其他线程, 那么线程有自己的pid,tgid就是进程的主线程的pid,group_leader指向的就是进程的主线程.

### signal处理
```c
// task_struct 中关于信号处理的字段
/* Signal handlers: */
struct signal_struct		*signal;
struct sighand_struct		*sighand; // 哪些信号正在通过信号处理函数进行处理: 处理的结果可以是忽略,可以是结束进程等等
sigset_t			blocked; // 哪些信号被阻塞暂不处理
sigset_t			real_blocked;
/* Restored if set_restore_sigmask() was used: */
sigset_t			saved_sigmask;
struct sigpending		pending; // 哪些信号尚等待处理
unsigned long			sas_ss_sp;
size_t				sas_ss_size;
unsigned int			sas_ss_flags;
```

信号处理函数默认使用用户态的函数栈, 也可以开辟新的栈专门用于信号处理,这就是sas_ss_xxx这三个变量的作用.

task_struct里的struct sigpending pending是本任务的; 而struct signal_struct *signal里的struct sigpending shared_pending是线程组共享的.

### task state
```c
/* -1 unrunnable, 0 runnable, >0 stopped: */
volatile long			state; // [state值的定义](https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h#L75)
int exit_state;
unsigned int flags;
```

state:
```c
// state是通过bitset的方式设置
/* Used in tsk->state: */
#define TASK_RUNNING			0x0000
#define TASK_INTERRUPTIBLE		0x0001
#define TASK_UNINTERRUPTIBLE		0x0002
#define __TASK_STOPPED			0x0004
#define __TASK_TRACED			0x0008
/* Used in tsk->exit_state: */
#define EXIT_DEAD			0x0010
#define EXIT_ZOMBIE			0x0020
#define EXIT_TRACE			(EXIT_ZOMBIE | EXIT_DEAD)
/* Used in tsk->state again: */
#define TASK_PARKED			0x0040
#define TASK_DEAD			0x0080
#define TASK_WAKEKILL			0x0100
#define TASK_WAKING			0x0200
#define TASK_NOLOAD			0x0400
#define TASK_NEW			0x0800
#define TASK_STATE_MAX			0x1000

/* Convenience macros for the sake of set_current_state: */
#define TASK_KILLABLE			(TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
#define TASK_STOPPED			(TASK_WAKEKILL | __TASK_STOPPED)
#define TASK_TRACED			(TASK_WAKEKILL | __TASK_TRACED)

#define TASK_IDLE			(TASK_UNINTERRUPTIBLE | TASK_NOLOAD)

/* Convenience macros for the sake of wake_up(): */
#define TASK_NORMAL			(TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE)

/* get_task_state(): */
#define TASK_REPORT			(TASK_RUNNING | TASK_INTERRUPTIBLE | \
					 TASK_UNINTERRUPTIBLE | __TASK_STOPPED | \
					 __TASK_TRACED | EXIT_DEAD | EXIT_ZOMBIE | \
					 TASK_PARKED)
```

![state 流转](/misc/img/task_state_change.png)

TASK_RUNNING并不是说进程正在运行, 而是`运行中+在运行队列中等待执行`. 当处于这个状态的进程获得时间片的时候,就是在运行中;如果没有获得时间片,就说明它被其他进程抢占了,在等待再次分配时间片. 

在运行中的进程,一旦要进行一些I/O操作,需要等待I/O完毕,这个时候会释放CPU,进入睡眠状态.

在Linux中,有两种睡眠状态:
- TASK_INTERRUPTIBLE ,可中断的睡眠状态
	进程在睡眠(即被阻塞), 等待某些条件达成. 一旦这些条件达成, kernel就会把进程状态设置为TASK_RUNNING. 处于此状态的进程会被信号唤醒而随时准备投入运行.

    **这是一种浅睡眠的状态,虽然在睡眠,等待I/O完成, 进程被信号唤醒后,不是继续刚才的操作,而是进行信号处理**. 当然也可以根据自己的意愿,来写信号处理函数,例如收到某些信号,就放弃等待这个I/O操作完成,直接退出,也可收到某些信息,继续等待
- TASK_UNINTERRUPTIBLE ,不可中断的睡眠状态
    这是一种深度睡眠状态,**不可被信号唤醒**,只能死等I/O操作完成. 一旦I/O操作因为特殊原因不能完成,这个时候,谁也叫不醒这个进程了(kill本身也是一个信号,既然这个状态不可被信号唤醒,kill信号也被忽略了).除非重启电脑,没有其他办法. 因此,这其实是一个比较危险的事情,除非极其有把握,不然还是不要设置成该状态.

于是就有了一种新的进程睡眠状态,TASK_KILLABLE,可以终止的新睡眠状态. 进程处于这种状态中,它的运行原理类似TASK_UNINTERRUPTIBLE,只不过可以响应致命信号.从定义可以看出,TASK_WAKEKILL用于在接收到致命信号时唤醒进程,而TASK_KILLABLE相当于这两位都设置了.

TASK_STOPPED是在进程接收到SIGSTOP、SIGTTIN、SIGTSTP或者SIGTTOU信号之后进入该状态, 同时在调试期间接收到任何信号也会进入该状态. 进程在等待一个恢复信息比如SIGCONT.

TASK_TRACED表示进程被debugger(比如ptrace)等进程监视,进程执行被调试程序所停止. 当一个进程被另外的进程所监视,每一个信号都会让进程进入该状态.

**一旦一个进程要结束,先进入的是EXIT_ZOMBIE状态**,但是这个时候它的父进程还没有使用wait()等系统调用来获知它的终止信息及释放它的所有数据结构,此时进程就成了僵尸进程.

EXIT_DEAD是进程的最终状态.

EXIT_ZOMBIE和EXIT_DEAD也可以用于exit_state.

state是和进程的运行、调度有关系, 还有其他的一些状态称为标志, 放在flags字段中,这些字段都被定义称为宏 ,以PF开头:
```c
// https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h#L1466
/*
 * Per process flags
 */
#define PF_IDLE			0x00000002	/* I am an IDLE thread */
#define PF_EXITING		0x00000004	/* Getting shut down */ 表示正在退出. 当有这个flag的时候,在函数find_alive_thread中,找活着的线程时,遇到有这个flag的,就直接跳过
#define PF_EXITPIDONE		0x00000008	/* PI exit done on shut down */
#define PF_VCPU			0x00000010	/* I'm a virtual CPU */ // 表示进程运行在虚拟CPU上. 在函数account_system_time中,统计进程的系统运行时间,如果有这个flag,就调用account_guest_time,按照客户机的时间进行统计
#define PF_WQ_WORKER		0x00000020	/* I'm a workqueue worker */
#define PF_FORKNOEXEC		0x00000040	/* Forked but didn't exec */ // 表示fork完了,还没有exec. 在_do_fork函数里面调用copy_process,这个时候把flag设置为PF_FORKNOEXEC; 当exec中调用了load_elf_binary的时候又把这个flag去掉
#define PF_MCE_PROCESS		0x00000080      /* Process policy on mce errors */
#define PF_SUPERPRIV		0x00000100	/* Used super-user privileges */
#define PF_DUMPCORE		0x00000200	/* Dumped core */
#define PF_SIGNALED		0x00000400	/* Killed by a signal */
#define PF_MEMALLOC		0x00000800	/* Allocating memory */
#define PF_NPROC_EXCEEDED	0x00001000	/* set_user() noticed that RLIMIT_NPROC was exceeded */
#define PF_USED_MATH		0x00002000	/* If unset the fpu must be initialized before use */
#define PF_USED_ASYNC		0x00004000	/* Used async_schedule*(), used by module init */
#define PF_NOFREEZE		0x00008000	/* This thread should not be frozen */
#define PF_FROZEN		0x00010000	/* Frozen for system suspend */
#define PF_KSWAPD		0x00020000	/* I am kswapd */
#define PF_MEMALLOC_NOFS	0x00040000	/* All allocation requests will inherit GFP_NOFS */
#define PF_MEMALLOC_NOIO	0x00080000	/* All allocation requests will inherit GFP_NOIO */
#define PF_LESS_THROTTLE	0x00100000	/* Throttle me less: I clean memory */
#define PF_KTHREAD		0x00200000	/* I am a kernel thread */
#define PF_RANDOMIZE		0x00400000	/* Randomize virtual address space */
#define PF_SWAPWRITE		0x00800000	/* Allowed to write to swap */
#define PF_MEMSTALL		0x01000000	/* Stalled due to lack of memory */
#define PF_UMH			0x02000000	/* I'm an Usermodehelper process */
#define PF_NO_SETAFFINITY	0x04000000	/* Userland is not allowed to meddle with cpus_allowed */
#define PF_MCE_EARLY		0x08000000      /* Early kill for mce process policy */
#define PF_MEMALLOC_NOCMA	0x10000000	/* All allocation request will have _GFP_MOVABLE cleared */
#define PF_FREEZER_SKIP		0x40000000	/* Freezer should not count it as freezable */
#define PF_SUSPEND_TASK		0x80000000      /* This thread called freeze_processes() and should not be frozen */
```

设置state: `set_current_state`, 除非必要(在SMP系统上), 它会设置内存屏障来强制其他cpu做重新排序, 否则等价于`task->state=state`

### do_exit
定义于`kernel/exit.c`, 工作内容:
1. 将task_struct中的flags设为PF_EXITING
1. call del_timer_sync() 删除内核定时器
1. BSD进程记账功能开启时, call acct_update_integrals()来输出记账信息
1. call exit_mm()释放进程占用的mm_struct, 如果mm_struct没有被共享的话, 就彻底释放它
1. call sem__exit(), 如果进程在排队等待IPC信号, 则取消排队
1. call exit_files()和exit_fs(), 递减文件描述符和文件系统数据的引用计数. 如果某个引用计数将为0, 则释放它.
1. 把存放在task_struct的exit_code中的任务退出码置位exit()提供的退出码(供父进程随时检索), 或着去完成其他由内核机制规定的退出动作.
1. call exit_notify()向父进程发送信号, 将task_struct的exit_state设为EXIT_ZOMBIE.
1. call schedule()切换到新的进程. 因为处于EXIT_ZOMBIE状态的进程不再被调度.

> exit_notify()会call forget_original_parent()-> find_new_reaper()来寻找父进程.

> 处于EXIT_ZOMBIE状态的进程所占用的所有内存就是内核栈, thread_info和task_struct, 此时进程存在的唯一作用是向它的父进程提供信息. 父进程获取信息或通知内核那是无用信息后, 进程持有的所有剩余内存被释放.

wait()->wait4(), 工作内容:
1. 挂起调用它的进程, 直到其中一个子进程退出, 此时得到返回的该子进程的pid, 即它设置的退出码.
1. call release_task()
1. call __exit_signal()->_unhash_process()->detach_pid()从pidhash上删除该进程, 同时也在task list上删除该进程.
1. __exit_signal()释放所有剩余的资源, 并进行最终统计和记录
1. 如果这个进程是线程组的最后一个进程, 并且领头进程已死, 那么release_task()会通知僵死的领头进程的父进程.
1. call put_task_struct()释放进程内核栈和thread_info, 并释放task_struct所占用的slab高速缓存.

孤儿进程处理: 为子进程在当前线程组内找一个线程作为父亲, 如果不行就让init来作为父进程.

### 调度
```c
//是否在运行队列上
int on_rq;
//优先级
int prio;
int static_prio;
int normal_prio;
unsigned int rt_priority;
//调度器类
const struct sched_class *sched_class;
//调度实体
struct sched_entity se;
struct sched_rt_entity rt;
struct sched_dl_entity dl;
//调度策略
unsigned int policy;
//可以使用哪些CPU
int nr_cpus_allowed;
cpumask_t cpus_allowed;
struct sched_info sched_info;
```

### 统计运行信息
```c
u64 utime;//用戶态消耗的CPU时间
u64 stime;//内核态消耗的CPU时间
unsigned long nvcsw;//自愿(voluntary)上下文切换计数
unsigned long nivcsw;//非自愿(involuntary)上下文切换计数
u64 start_time;//进程启动时间,不包含睡眠时间
u64 real_start_time;//进程启动时间,包含睡眠时间
```

### 进程亲缘关系

```c
struct task_struct __rcu *real_parent; /* real parent process */
struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */ // 指向其父进程. 当它终止时,必须向它的父进程发送信号
struct list_head children; /* list of my children */ // 表示链表的头部. 链表中的所有元素都是它的子进程
struct list_head sibling; /* linkage in my parent's children list */ // 用于把当前进程插入到兄弟链表中
```

通常情况下,real_parent和parent是一样的,但是也会有另外的情况存在. 例如,bash创建一个进程,那进程的parent和real_parent就都是bash. 如果在bash上使用GDB来debug一个进程,这个时候GDB是real_parent,bash是这个进程的parent.

![task tree](/misc/img/task_parent.png)

### 进程权限
```c
/* Objective and real subjective task credentials (COW): */
const struct cred __rcu		*real_cred; // 说明谁能操作我这个进程

/* Effective (overridable) subjective task credentials (COW): */
const struct cred __rcu		*cred; // 说明我这个进程能够操作谁
```
Objective是被操作对象, Subjective是操作者.

[cred定义](https://elixir.bootlin.com/linux/latest/source/include/linux/cred.h#L111):
```c
struct cred {
	atomic_t	usage;
#ifdef CONFIG_DEBUG_CREDENTIALS
	atomic_t	subscribers;	/* number of processes subscribed */
	void		*put_addr;
	unsigned	magic;
#define CRED_MAGIC	0x43736564
#define CRED_MAGIC_DEAD	0x44656144
#endif
	kuid_t		uid;		/* real UID of the task */
	kgid_t		gid;		/* real GID of the task */
	kuid_t		suid;		/* saved UID of the task */
	kgid_t		sgid;		/* saved GID of the task */
	kuid_t		euid;		/* effective UID of the task */
	kgid_t		egid;		/* effective GID of the task */
	kuid_t		fsuid;		/* UID for VFS ops */
	kgid_t		fsgid;		/* GID for VFS ops */
	unsigned	securebits;	/* SUID-less security management */
	kernel_cap_t	cap_inheritable; /* caps our children can inherit */
	kernel_cap_t	cap_permitted;	/* caps we're permitted */
	kernel_cap_t	cap_effective;	/* caps we can actually use */
	kernel_cap_t	cap_bset;	/* capability bounding set */
	kernel_cap_t	cap_ambient;	/* Ambient capability set */
#ifdef CONFIG_KEYS
	unsigned char	jit_keyring;	/* default keyring to attach requested
					 * keys to */
	struct key	*session_keyring; /* keyring inherited over fork */
	struct key	*process_keyring; /* keyring private to this process */
	struct key	*thread_keyring; /* keyring private to this thread */
	struct key	*request_key_auth; /* assumed request_key authority */
#endif
#ifdef CONFIG_SECURITY
	void		*security;	/* subjective LSM security */
#endif
	struct user_struct *user;	/* real user ID subscription */
	struct user_namespace *user_ns; /* user_ns the caps and keyrings are relative to. */
	struct group_info *group_info;	/* supplementary groups for euid/fsgid */
	struct rcu_head	rcu;		/* RCU deletion hook */
} __randomize_layout;
```

从定义可以看出, 大部分是关于用户和用户所属的用户组信息:
1. uid和gid,注释是real user/group id. 一般情况下,谁启动的进程,就是谁的ID. 但是权限审核的时候,往往不比较这两个,也就是说不大起作用
1. euid和egid,注释是effective user/group id. 当这个进程要操作消息队列、共享内存、信号量等对象的时候,其实就是在比较这个用户和组是否有权限
1. fsuid和fsgid, 即filesystem user/group id. 这个是对文件操作会审核的权限.一般说来,fsuid、euid,和uid是一样的,fsgid、egid,和gid也是一样的. 因为谁启动的进程,就应该审核启动的用户到底有没有这个权限, 但是也有特殊的情况.比如`chmod u+s program`的情况即设置了 set-user-ID 的标识位, uid还是进程创建用户, 但euid和fsuid变成了`program`所有者的uid.

除了以用户和用户组控制权限(粗颗粒度),Linux还有另一个机制就是capabilities(细颗粒度), 它用位图表示权限,定义在[capability.h](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/capability.h#L113)里.

cap_permitted表示进程能够使用的权限, 但是真正起作用的是cap_effective. cap_permitted中可以包含cap_effective中没有的权限, 一个进程可以在必要的时候放弃自己的某些权限,这样更加安全.

cap_inheritable表示当可执行文件的扩展属性设置了inheritable位时,调用exec执行该程序会继承调用者的inheritable集合,并将其加入到permitted集合. 但在非root用户下执行exec时, 通常不会保留inheritable集合, 但是往往又是非root用户,才想保留权限,所以非常鸡肋.

cap_bset,也就是capability bounding set,是系统中所有进程允许保留的权限. 如果这个集合中不存在某个权限,那么系统中的所有进程都没有这个权限. 即使以超级用户权限执行的进程,也是一样的. 这样有很多好处。例如,系统启动以后,将加载内核模块的权限去掉,那所有进程都不能加载内核模块.

cap_ambient是比较新加入内核的,就是为了解决cap_inheritable鸡肋的状况,也就是,非root用户进程使用exec执行一个程序的时候,如何保留权限的问题. 当执行exec的时候,cap_ambient会被添加到cap_permitted中,同时设置到cap_effective中.

### 内存管理
```c
struct mm_struct *mm;
struct mm_struct *active_mm;
```

每个进程都有自己独立的虚拟内存空间, 用mm_struct表示.

### 文件和文件系统
```c
/* Filesystem information: */
struct fs_struct *fs;
/* Open file information: */
struct files_struct *files;
```

每个进程有一个文件系统的数据结构，还有一个打开文件的数据结构.

### 用户态/内核态的切换的依赖变量
```c
struct thread_info		thread_info;
void				*stack; // 内核栈

/* CPU-specific state of this task: */
struct thread_struct		thread; // 在 Linux 中，真的参与进程切换的寄存器很少，主要的就是栈顶寄存器. 于是，在 task_struct 里面保留了要切换进程的时候需要修改的寄存器
```

#### 用户态函数栈
在用户态中,程序的执行往往是一个函数调用另一个函数(通过指令跳转), 函数调用都是通过栈来进行的.

在进程的内存空间里面,栈是一个从高地址到低地址,往下增长的结构,也就是上面是栈底,下面是栈顶,入栈和出栈的操作都是从下面的栈顶开始的

![x86_64的栈示意图](/misc/img/stack_64.png)

对于64位操作系统, 其寄存器数目比32位多. rax用于保存函数调用的返回结果. 栈顶指针寄存器变成了rsp,指向栈顶位置. 堆栈的Pop和Push操作会自动调整rsp,栈基指针
寄存器变成了rbp,指向当前栈帧的起始位置. 

改变比较多的是参数传递. rdi、rsi、rdx、rcx、r8、r9这6个寄存器,用于传递存储函数调用时的6个参数. 如果超过6的时候,还是需要放到栈里面. 

然而,前6个参数有时候需要进行寻址,但是如果在寄存器里面,是没有地址的,因而还是会放到栈里面,只不过放到栈里面的操作是被调用函数做的

#### 内核态函数栈(kernel stack)
Linux 给每个 task 都分配了内核栈.
32位系统的THREAD_SIZE在[page_32_types.h](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/page_32_types.h), 是这样定义的: PAGE_SIZE 是 4K，左移一位就是乘以 2，也就是 8K.
64位系统的THREAD_SIZE在[page_64_types.h](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/page_64_types.h), 是这样定义的: 在PAGE_SIZE的基础上左移两位,也即16K,并且要求起始地址必须是8192的整数倍.

```c
#ifdef CONFIG_KASAN
#define KASAN_STACK_ORDER 1
#else
#define KASAN_STACK_ORDER 0
#endif

#define THREAD_SIZE_ORDER	(2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER) // HREAD_SIZE表示了整个内核栈的大小
```

> Linux内核在x86平台下，PAGE_SIZE为4KB(32位和64位相同).

> 用户态进程所用的栈，是在进程线性地址空间中；而内核栈是当进程从用户空间进入内核空间时，特权级发生变化，需要切换堆栈，那么内核空间中使用的就是这个内核栈. 因为内核控制路径使用很少的栈空间，所以只需要几千个字节的内核态栈.

![内核栈示意图](/misc/img/process/kernel_stack.png)

32/64位的工作模式:
- 在用户态,应用程序进行了至少一次函数调用. 32 位的就是用函数栈传参, 而64位的前6个参数用寄存器,其他的用函数栈
- 在内核态,32位和64位都使用内核栈,格式也稍有不同,主要集中在pt_regs结构上
- 在内核态,**32位和64位的内核栈和task_struct的关联关系不同, 32位主要靠thread_info,64位主要靠Per-CPU变量**

> thread_info结构是对task_struct结构的补充, 因为task_struct结构庞大但是通用,不同的体系结构就需要保存不同的东西,所以往往与体系结构有关的,都放在thread_info里面.

在内核代码里面有这样一个union: [thread_union](https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h#L1650),x86_64下它将task_struct和stack放在一起:
```c
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK // 仅ia64体系结构启用此选项, 因此x86_64下thread_union会包含task_struct
	struct task_struct task;
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK // 会为x86_64体系结构启用此选项, 因此thread_info会包含在task_struct中
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

在内核栈的最高地址端,存放的是另一个结构[pt_regs](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/ptrace.h#L56),定义如下:
```c
struct pt_regs {
/*
 * C ABI says these regs are callee-preserved. They aren't saved on kernel entry
 * unless syscall needs a complete, fully filled "struct pt_regs".
 */
	unsigned long r15;
	unsigned long r14;
	unsigned long r13;
	unsigned long r12;
	unsigned long bp;
	unsigned long bx;
/* These regs are callee-clobbered. Always saved on kernel entry. */
	unsigned long r11;
	unsigned long r10;
	unsigned long r9;
	unsigned long r8;
	unsigned long ax;
	unsigned long cx;
	unsigned long dx;
	unsigned long si;
	unsigned long di;
/*
 * On syscall entry, this is syscall#. On CPU exception, this is error code.
 * On hw interrupt, it's IRQ number:
 */
	unsigned long orig_ax;
/* Return frame for iretq */
	unsigned long ip;
	unsigned long cs;
	unsigned long flags;
	unsigned long sp;
	unsigned long ss;
/* top of stack page */
};
```

当系统调用从用户态到内核态的时候,首先要做的第一件事情,就是将用户态运行过程中的CPU上下文保存起来,其实主要就是保存在这个结构的寄存器变量里. 这样当从内核系统调用返回的时候,才能让进程在刚才的地方接着运行下去. 系统调用的时候,压栈的值的顺序和struct pt_regs中寄存器定义的顺序是一样的.

在内核态时,CPU的寄存器ESP或者RSP,已经指向内核栈的栈顶,在内核态里的调用都有和用户态相似的过程.

#### [通过task_struct找内核栈的方法](https://elixir.bootlin.com/linux/latest/source/include/linux/sched/task_stack.h#L19)
```c
/*
 * When accessing the stack of a non-current task that might exit, use
 * try_get_task_stack() instead.  task_stack_page will return a pointer
 * that could get freed out from under you.
 */
static inline void *task_stack_page(const struct task_struct *task)
{
	return task->stack; // stack是指向内核栈的指针
}
```

#### [通过task_struct得到相应的pt_regs的方法](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/processor.h#L844)
```c
#define task_pt_regs(task) \
({									\
	unsigned long __ptr = (unsigned long)task_stack_page(task);	\
	__ptr += THREAD_SIZE - TOP_OF_KERNEL_STACK_PADDING;		\
	((struct pt_regs *)__ptr) - 1;					\
})
```

先从task_struct找到内核栈的开始位置(即task的首地址). 然后这个位置加上THREAD_SIZE就到了最后的位置(即thread_union的结束位置),然后转换为struct pt_regs的指针再减一,就相当于减去一个pt_regs的大小,就得到了pt_regs的首地址.

[TOP_OF_KERNEL_STACK_PADDING](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/thread_info.h#L36):
```c
#ifdef CONFIG_X86_32
# ifdef CONFIG_VM86
#  define TOP_OF_KERNEL_STACK_PADDING 16
# else
#  define TOP_OF_KERNEL_STACK_PADDING 8
# endif
#else
# define TOP_OF_KERNEL_STACK_PADDING 0
#endif
``

也就是说,32 位机器上是 8, 其他是0. 这是因为压栈pt_regs有两种情况. 我们知道,CPU用ring来区分权限,从而Linux可以区分内核态和用户态.
1. 涉及从用户态到内核态的变化的系统调用来说. 因为涉及权限的改变,会压栈保存SS、ESP寄存器的,这两个寄存器共占用8个byte.
1. 不涉及权限的变化,就不会压栈这8个byte. 这样就会使得两种情况不兼容. 如果没有压栈还访问,就会报错,所以还不如预留在这里,保证安全. 在64位上,修改了这个问题,变成了定长的.

#### 获取当前在某个 CPU 上执行的进程的 task_struct
Linux内核引入percpu变量之后，逐渐通过percpu变量来实现current宏，并且从Linux 4.1开始，x86移除了kernel_stack(用current_task取代)，并逐渐开始简化thread_info结构体，直到Linux 4.9彻底不再通过thread_info获取task_struct指针，而是直接通过percpu(current_task)变量保存当前运行进程的task_struct, 可调用 this_cpu_read_stable 进行获取.

每个CPU运行的task_struct不再通过thread_info获取了,而是直接放在PerCPU 变量里面了.

多核情况下,CPU是同时运行的,但是它们共同使用其他的硬件资源的时候,我们需要解决多个CPU之间的同步问题.
PerCPU变量是内核中一种重要的同步机制. 顾名思义,PerCPU变量就是为每个CPU构造一个变量的副本,这样多个CPU各自操作自己的副本,互不干涉. 比如,**当前进程的变量current_task就被声明为PerCPU变量**:
```c
// 声明在https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/current.h#L11
DECLARE_PER_CPU(struct task_struct *, current_task);

static __always_inline struct task_struct *get_current(void)
{
	return this_cpu_read_stable(current_task);
}

#define current get_current() // currrent变量实际上是一个指向struct task_struct的指针

// 定义在https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/cpu/common.c#L1747
DEFINE_PER_CPU(struct task_struct *, current_task) = &init_task;
```

也就是说,系统刚刚初始化的时候,current_task都指向init_task

由于采用了percpu current_task来保存当前运行进程的task_struct，所以在当某个CPU上的进程进行切换的时候,current_task需被修改为将要切换到的目标进程. 例如,进程切换函数__switch_to就会改变current_task:
```c
// https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/process_64.c#L426
__visible __notrace_funcgraph struct task_struct *
__switch_to(struct task_struct *prev_p, struct task_struct *next_p)
{
......
this_cpu_write(current_task, next_p); // 切换时更新该变量
......
return prev_p;
}
```

当要获取当前的运行中的task_struct的时候,就需要调用this_cpu_read_stable进行读取.
```c
// https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/percpu.h#L392
#define this_cpu_read_stable(var)	percpu_stable_op("mov", var)
```
好了,现在如果你是一个进程,正在某个CPU上运行,就能够轻松得到task_struct了.

## 进程的创建
![](/misc/img/process/9d9c5779436da40cabf8e8599eb85558.jpeg)

fork 是一个系统调用，对应的是 sys_call_table 中找到相应的系统调用 sys_fork.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L2535
#ifdef __ARCH_WANT_SYS_FORK
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
	struct kernel_clone_args args = {
		.exit_signal = SIGCHLD,
	};

	return _do_fork(&args);
#else
	/* can not support in nommu mode */
	return -EINVAL;
#endif
}
#endif

// https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L2416
/*
 *  Ok, this is the main fork-routine.
 *
 * It copies the process, and if successful kick-starts
 * it and waits for it to finish using the VM if required.
 *
 * args->exit_signal is expected to be checked for sanity by the caller.
 */
long _do_fork(struct kernel_clone_args *args)
{
	u64 clone_flags = args->flags;
	struct completion vfork;
	struct pid *pid;
	struct task_struct *p;
	int trace = 0;
	long nr;

	/*
	 * Determine whether and which event to report to ptracer.  When
	 * called from kernel_thread or CLONE_UNTRACED is explicitly
	 * requested, no event is reported; otherwise, report if the event
	 * for the type of forking is enabled.
	 */
	if (!(clone_flags & CLONE_UNTRACED)) {
		if (clone_flags & CLONE_VFORK)
			trace = PTRACE_EVENT_VFORK;
		else if (args->exit_signal != SIGCHLD)
			trace = PTRACE_EVENT_CLONE;
		else
			trace = PTRACE_EVENT_FORK;

		if (likely(!ptrace_event_enabled(current, trace)))
			trace = 0;
	}

	p = copy_process(NULL, trace, NUMA_NO_NODE, args);
	add_latent_entropy();

	if (IS_ERR(p))
		return PTR_ERR(p);

	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	trace_sched_process_fork(current, p);

	pid = get_task_pid(p, PIDTYPE_PID);
	nr = pid_vnr(pid);

	if (clone_flags & CLONE_PARENT_SETTID)
		put_user(nr, args->parent_tid);

	if (clone_flags & CLONE_VFORK) {
		p->vfork_done = &vfork;
		init_completion(&vfork);
		get_task_struct(p);
	}

	wake_up_new_task(p);

	/* forking complete and child started to run, tell ptracer */
	if (unlikely(trace))
		ptrace_event_pid(trace, pid);

	if (clone_flags & CLONE_VFORK) {
		if (!wait_for_vfork_done(p, &vfork))
			ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
	}

	put_pid(pid);
	return nr;
}
```

_do_fork 里面做的第一件大事就是 [copy_process()](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1841).

它的[dup_task_struct(current, node)](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1937), 主要做了下面几件事情:
1. 调用 alloc_task_struct_node 分配一个 task_struct 结构
1. 调用 alloc_thread_stack_node 来创建内核栈，这里面调用 __vmalloc_node_range 分配一个连续的 THREAD_SIZE 的内存空间，赋值给 task_struct 的 void *stack 成员变量
1. 调用 arch_dup_task_struct(struct task_struct *dst, struct task_struct *src)，将 task_struct 进行复制，其实就是调用 memcpy
1. 调用 setup_thread_stack 设置 thread_info.

到这里，整个 task_struct 复制了一份，而且内核栈也创建好了.

之后是设置权限[`retval = copy_creds(p, clone_flags);`](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1970):
1. 调用 prepare_creds，准备一个新的 struct cred *new, 其实还是从内存中分配一个新的 struct cred 结构，然后调用 memcpy 复制一份父进程的 cred.
1. 接着 p->cred = p->real_cred = get_cred(new)，将新进程的“我能操作谁”和“谁能操作我”两个权限都指向新的 cred.

接下来，copy_process 重新设置进程运行的统计量:
```c
p->utime = p->stime = p->gtime = 0;
p->start_time = ktime_get_ns();
p->real_start_time = ktime_get_boot_ns();
```

接下来，copy_process 开始设置调度相关的变量:
```c
retval = sched_fork(clone_flags, p);
```

[sched_fork](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L2068) 主要做了下面几件事情：
1. 调用 __sched_fork，在这里面将 on_rq 设为 0，初始化 sched_entity，将里面的 exec_start、sum_exec_runtime、prev_sum_exec_runtime、vruntime 都设为 0, 这几个变量涉及进程的实际运行时间和虚拟运行时间, 是否到时间应该被调度了，就靠它们几个
1. 设置进程的状态 p->state = TASK_NEW
1. 初始化优先级 prio、normal_prio、static_prio
1. 设置调度类，如果是普通进程，就设置为 p->sched_class = &fair_sched_class
1. 调用调度类的 task_fork 函数，对于 CFS 来讲，就是调用 task_fork_fair. 在这个函数里，先调用 update_curr，对于当前的进程进行统计量更新，然后把子进程和父进程的 vruntime 设成一样，最后调用 place_entity，初始化 sched_entity. 这里有一个变量 sysctl_sched_child_runs_first，可以设置父进程和子进程谁先运行. 如果设置了子进程先运行，即便两个进程的 vruntime 一样，也要把子进程的 sched_entity 放在前面，然后调用 resched_curr，标记当前运行的进程 TIF_NEED_RESCHED，也就是说，把父进程设置为应该被调度，这样下次调度的时候，父进程会被子进程抢占. 

接下来，copy_process 开始初始化与文件和文件系统相关的变量:
```c
retval = copy_files(clone_flags, p);
retval = copy_fs(clone_flags, p);
```

[copy_files](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1460) 主要用于复制一个进程打开的文件信息. 这些信息用一个结构 files_struct 来维护，每个打开的文件都有一个文件描述符. 在 copy_files 函数里面调用 dup_fd，在这里面会创建一个新的 files_struct，然后将所有的文件描述符数组 fdtable 拷贝一份.
[copy_fs](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1440) 主要用于复制一个进程的目录信息. 这些信息用一个结构 fs_struct 来维护. 一个进程有自己的根目录和根文件系统 root，也有当前目录 pwd 和当前目录的文件系统，都在 fs_struct 里面维护. copy_fs 函数里面调用 copy_fs_struct，创建一个新的 fs_struct，并复制原来进程的 fs_struct.

接下来，copy_process 开始初始化与信号相关的变量:
```c
init_sigpending(&p->pending);
retval = copy_sighand(clone_flags, p);
retval = copy_signal(clone_flags, p);
```

copy_sighand 会分配一个新的 sighand_struct, 最主要的是维护信号处理函数，在 copy_sighand 里面会调用 memcpy，将信号处理函数 sighand->action 从父进程复制到子进程.

init_sigpending 和 copy_signal 用于初始化，并且复制用于维护发给这个进程的信号的数据结构. copy_signal 函数会分配一个新的 signal_struct，并进行初始化.

接下来，copy_process 开始复制进程内存空间. 
```c
retval = copy_mm(clone_flags, p);
```

进程都有自己的内存空间，用 mm_struct 结构来表示. copy_mm 函数中调用 dup_mm，分配一个新的 mm_struct 结构，调用 memcpy 复制这个结构. dup_mmap 用于复制内存空间中内存映射的部分. mmap 可以分配大块的内存，其实 mmap 也可以将一个文件映射到内存中，方便可以像读写内存一样读写文件，这个在内存管理那节我们讲.

好了，copy_process 要结束了，上面图中的组件也初始化的差不多了.

接下来，copy_process 开始分配 pid，设置 tid，group_leader，并且建立进程之间的亲缘关系.

[_do_fork 做的]第二件大事 [`wake_up_new_task`](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/sched/core.c#L3012). 

首先，我们需要将进程的状态设置为 TASK_RUNNING. [activate_task](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/sched/core.c#L3035) 函数中会调用 enqueue_task.

如果是 CFS 的调度类，则执行相应的 [enqueue_task_fair](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/sched/fair.c#L5459).

在 enqueue_task_fair 中取出的队列就是 cfs_rq，然后调用 enqueue_entity. 在 enqueue_entity 函数里面，会调用 update_curr，更新运行的统计量，然后调用 __enqueue_entity，将 sched_entity 加入到红黑树里面，然后将 se->on_rq = 1 设置在队列上.

回到 enqueue_task_fair 后，将这个队列上运行的进程数目加一, 然后，wake_up_new_task 会调用 check_preempt_curr，看是否能够抢占当前进程. 

在 check_preempt_curr 中，会调用相应的调度类的 rq->curr->sched_class->check_preempt_curr(rq, p, flags). 对于 CFS 调度类来讲，调用的是 check_preempt_wakeup.

在 check_preempt_wakeup 函数中，前面调用 task_fork_fair 的时候，设置 sysctl_sched_child_runs_first 了，已经将当前父进程的 TIF_NEED_RESCHED 设置了，则直接返回; 否则，check_preempt_wakeup 还是会调用 update_curr 更新一次统计量，然后 wakeup_preempt_entity 将父进程和子进程 PK 一次，看是不是要抢占，如果要则调用 resched_curr 标记父进程为 TIF_NEED_RESCHED.

如果新创建的进程应该抢占父进程，因为 fork 是一个系统调用，从系统调用返回的时候，是抢占的一个好时机，如果父进程判断自己已经被设置为 TIF_NEED_RESCHED，就让子进程先跑，抢占自己.

## 线程的创建
env: glibc 2.31

![](/misc/img/process/14635b1613d04df9f217c3508ae8524b.jpeg)

创建一个线程调用的是 pthread_create.

其实，线程不是一个完全由内核实现的机制，它是由内核态和用户态合作完成的. pthread_create 不是一个系统调用，是 Glibc 库的一个函数.

### 用户态创建线程
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L623
int
__pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
               void *(*start_routine) (void *), void *arg)
{
 ...
}
versioned_symbol (libpthread, __pthread_create_2_1, 
```

首先处理的是线程的属性参数. 例如写程序的时候设置的线程栈大小. 如果没有传入线程属性，就取默认值.
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L628
const struct pthread_attr *iattr = (struct pthread_attr *) attr;
struct pthread_attr default_attr;
bool free_cpuset = false;
bool c11 = (attr == ATTR_C11_THREAD);
if (iattr == NULL || c11)
{
  ......
  iattr = &default_attr;
}
```

接下来，就像在内核里一样，每一个进程或者线程都有一个 task_struct 结构，在用户态也有一个用于维护线程的结构，就是这个 pthread 结构:
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L659
struct pthread *pd = NULL;
```

凡是涉及函数的调用，都要使用到栈, 每个线程也有自己的栈, 那接下来就是创建线程栈了:
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L660
int err = ALLOCATE_STACK (iattr, &pd);
```

[ALLOCATE_STACK](https://elixir.bootlin.com/glibc/latest/source/nptl/allocatestack.c#L53) 是一个宏，我们找到它的定义之后，发现它其实就是一个[函数allocate_stack](https://elixir.bootlin.com/glibc/latest/source/nptl/allocatestack.c#L53).

allocate_stack 主要做了以下这些事情:
1. 如果在线程属性里面设置过栈的大小，需要把设置的值拿出来
1. 为了防止栈的访问越界，在栈的末尾会有一块空间 guardsize，一旦访问到这里就错误了
1. 其实线程栈是在进程的堆里面创建的. 如果一个进程不断地创建和删除线程，我们不可能不断地去申请和清除线程栈使用的内存块，这样就需要有一个缓存. get_cached_stack 就是根据计算出来的 size 大小，看一看已经有的缓存中，有没有已经能够满足条件的；如果缓存里面没有，就需要调用 __mmap 创建一块新的
1. 线程栈也是自顶向下生长的，每个线程要有一个 pthread 结构，这个结构也是放在栈的空间里面的, 在栈底的位置，其实是地址最高位.
1. 计算出 guard 内存的位置，调用 setup_stack_prot 设置这块内存的是受保护的
1. 接下来，开始填充 pthread 这个结构里面的成员变量 stackblock、stackblock_size、guardsize、specific. 这里的 specific 是用于存放 Thread Specific Data 的，也即属于线程的全局变量
1. 将这个线程栈放到 stack_used 链表中，其实管理线程栈总共有两个链表，一个是 stack_used，也就是这个栈正被使用；另一个是 stack_cache，就是上面说的，一旦线程结束，先缓存起来，不释放，等有其他的线程创建的时候，给其他的线程用. 搞定了用户态栈的问题，其实用户态的事情基本搞定了一半.

> 如果要在堆里面 malloc 一块内存，比较大的话，用 __mmap

### 内核态创建任务
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L688
pd->start_routine = start_routine;
```

start_routine 就是咱们给线程的函数，start_routine，start_routine 的参数 arg，以及调度策略都要赋值给 pthread. 接下来 __nptl_nthreads 加一，说明又多了一个线程.

真正创建线程的是调用 [create_thread](https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/createthread.c#L49) 函数，这个函数定义如下:
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L785
retval = create_thread (pd, iattr, &stopped_start,
			      STACK_VARIABLES_ARGS, &thread_ran);
```

create_thread里面有很长的 clone_flags，需要留意, 然后就是 [ARCH_CLONE](https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/createthread.c#L34)，其实调用的是 [__clone](https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/x86_64/clone.S#L50).

> x86_64只有__clone, 而不是__clone2.

我们能看到最后调用了 syscall，这一点 clone 和我们原来熟悉的其他系统调用几乎是一致的。但是，也有少许不一样的地方。

如果在进程的主线程里面调用其他系统调用，当前用户态的栈是指向整个进程的栈，栈顶指针也是指向进程的栈，指令指针也是指向进程的主线程的代码.此时此刻执行到这里，调用 clone 的时候，用户态的栈、栈顶指针、指令指针和其他系统调用一样，都是指向主线程的.

但是对于线程来说，这些都要变, 因为我们希望当 clone 这个系统调用成功的时候，除了内核里面有这个线程对应的 task_struct，当系统调用返回到用户态的时候，用户态的栈应该是线程的栈，栈顶指针应该指向线程的栈，指令指针应该指向线程将要执行的那个函数. 所以这些都需要我们自己做，将线程要执行的函数的参数和指令的位置都压到栈里面，当从内核返回，从栈里弹出来的时候，就从这个函数开始，带着这些参数执行下去.

内核里面对于 [clone 系统调用](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L2564)的定义是这样的:
```c
#ifdef __ARCH_WANT_SYS_CLONE
#ifdef CONFIG_CLONE_BACKWARDS
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 unsigned long, tls,
		 int __user *, child_tidptr)
#elif defined(CONFIG_CLONE_BACKWARDS2)
SYSCALL_DEFINE5(clone, unsigned long, newsp, unsigned long, clone_flags,
		 int __user *, parent_tidptr,
		 int __user *, child_tidptr,
		 unsigned long, tls)
#elif defined(CONFIG_CLONE_BACKWARDS3)
SYSCALL_DEFINE6(clone, unsigned long, clone_flags, unsigned long, newsp,
		int, stack_size,
		int __user *, parent_tidptr,
		int __user *, child_tidptr,
		unsigned long, tls)
#else
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 int __user *, child_tidptr,
		 unsigned long, tls)
#endif
{
	struct kernel_clone_args args = {
		.flags		= (lower_32_bits(clone_flags) & ~CSIGNAL),
		.pidfd		= parent_tidptr,
		.child_tid	= child_tidptr,
		.parent_tid	= parent_tidptr,
		.exit_signal	= (lower_32_bits(clone_flags) & CSIGNAL),
		.stack		= newsp,
		.tls		= tls,
	};

	if (!legacy_clone_args_valid(&args))
		return -EINVAL;

	return _do_fork(&args);
}
#endif
```

复杂的标志位设定的影响:
- 对于 [copy_files](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1460)，原来是调用 dup_fd 复制一个 files_struct 的，现在因为 CLONE_FILES 标识位变成将原来的 files_struct 引用计数加一
- 对于 [copy_fs](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1440)，原来是调用 copy_fs_struct 复制一个 fs_struct，现在因为 CLONE_FS 标识位变成将原来的 fs_struct 的用户数加一
- 对于 [copy_sighand](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1513)，原来是创建一个新的 sighand_struct，现在因为 CLONE_SIGHAND 标识位变成将原来的 sighand_struct 引用计数加一
- 对于 [copy_signal](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1562)，原来是创建一个新的 signal_struct，现在因为 CLONE_THREAD 直接返回了
- 对于 [copy_mm](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1393)，原来是调用 dup_mm 复制一个 mm_struct，现在因为 CLONE_VM 标识位而直接指向了原来的 mm_struct
- 对于亲缘关系的影响，毕竟要识别多个线程是不是属于一个进程:

	```c
	// https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L2172
	p->pid = pid_nr(pid);
		if (clone_flags & CLONE_THREAD) {
			p->exit_signal = -1;
			p->group_leader = current->group_leader;
			p->tgid = current->tgid;
		} else {
			if (clone_flags & CLONE_PARENT)
				p->exit_signal = current->group_leader->exit_signal;
			else
				p->exit_signal = args->exit_signal;
			p->group_leader = p;
			p->tgid = p->pid;
		}

		p->nr_dirtied = 0;
		p->nr_dirtied_pause = 128 >> (PAGE_SHIFT - 10);
		p->dirty_paused_when = 0;

		p->pdeath_signal = 0;
		INIT_LIST_HEAD(&p->thread_group);
		p->task_works = NULL;

		/*
		* Ensure that the cgroup subsystem policies allow the new process to be
		* forked. It should be noted the the new process's css_set can be changed
		* between here and cgroup_post_fork() if an organisation operation is in
		* progress.
		*/
		retval = cgroup_can_fork(p, args);
		if (retval)
			goto bad_fork_put_pidfd;

		/*
		* From this point on we must avoid any synchronous user-space
		* communication until we take the tasklist-lock. In particular, we do
		* not want user-space to be able to predict the process start-time by
		* stalling fork(2) after we recorded the start_time but before it is
		* visible to the system.
		*/

		p->start_time = ktime_get_ns();
		p->start_boottime = ktime_get_boottime_ns();

		/*
		* Make it visible to the rest of the system, but dont wake it up yet.
		* Need tasklist lock for parent etc handling!
		*/
		write_lock_irq(&tasklist_lock);

		/* CLONE_PARENT re-uses the old parent */
		if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) {
			p->real_parent = current->real_parent;
			p->parent_exec_id = current->parent_exec_id;
		} else {
			p->real_parent = current;
			p->parent_exec_id = current->self_exec_id;
		}

	```

	从上面的代码可以看出，使用了 CLONE_THREAD 标识位之后，使得亲缘关系有了一定的变化:
	- 如果是新进程，那这个进程的 group_leader 就是它自己，tgid 是它自己的 pid，自己是线程组的头. 如果是新线程，group_leader 是当前进程的，group_leader，tgid 是当前进程的 tgid，也就是当前进程的 pid，这个时候还是拜原来进程为老大
	- 如果是新进程，新进程的 real_parent 是当前的进程，在进程树里面又见一辈人; 如果是新线程，线程的 real_parent 是当前的进程的 real_parent，其实是平辈的.

- 对于信号的处理

	如何保证发给进程的信号虽然可以被一个线程处理，但是影响范围应该是整个进程的. 例如，kill 一个进程，则所有线程都要被干掉. 如果一个信号是发给一个线程的 pthread_kill，则应该只有线程能够收到.
	
	在 copy_process 的主流程里面，无论是创建进程还是线程，都会初始化 struct sigpending pending by [`init_sigpending(&p->pending)`](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1992)，也就是每个 task_struct，都会有这样一个成员变量, 这就是一个信号列表. 如果这个 task_struct 是一个线程，这里面的信号就是发给这个线程的；如果这个 task_struct 是一个进程，这里面的信号是发给主线程的.
	
	另外，上面 copy_signal 的时候，可以看到，在创建进程的过程中，会初始化 signal_struct 里面的 struct sigpending shared_pending by [`
init_sigpending(&sig->shared_pending);`](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1584). 但是，在创建线程的过程中，连 signal_struct 都共享了. 也就是说，整个进程里的所有线程共享一个 shared_pending，这也是一个信号列表，是发给整个进程的，哪个线程处理都一样.

至此，clone 在内核的调用完毕，要返回系统调用，回到用户态.

### 用户态执行线程
根据 __clone 的第一个参数，回到用户态也不是直接运行我们指定的那个函数，而是一个通用的 [start_thread](https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L378)，这是所有线程在用户态的统一入口.

```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/createthread.c#L22
#define START_THREAD_DEFN \
  static void __attribute__ ((noreturn)) start_thread (void)

// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L378
START_THREAD_DEFN
{
	...
}
```

在 start_thread 入口函数中，才真正的调用用户提供的函数，在用户的函数执行完毕之后，会释放这个线程相关的数据. 例如，线程本地数据 thread_local variables，线程数目也减一. 如果这是最后一个线程了，就直接退出进程，另外 [__free_tcb](https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L344) 用于释放 pthread.

__free_tcb 会调用 [__deallocate_stack](https://elixir.bootlin.com/glibc/latest/source/nptl/allocatestack.c#L788) 来释放整个线程栈，这个线程栈要从当前使用线程栈的列表 stack_used 中拿下来，放到缓存的线程栈列表 stack_cache 中.

好了，整个线程的生命周期到这里就结束了.