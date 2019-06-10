# task
无论进程还是线程, 在linux kernel里都是task, 对于的定义是[`task_struct`](https://sourcegraph.com/github.com/torvalds/linux@d1fdb6d8f6a4109a4263176c84b899076a5f8008/-/blob/include/linux/sched.h#L584:8).

## 成员
![task member](images/task.png)

### task id
```c
pid_t pid; // process id
pid_t tgid; // thread group ID, 可判断tast_struct代表的是一个进程还是一个线程
struct task_struct *group_leader;
```
任何一个进程,如果只有主线程,那pid是自己,tgid是自己,group_leader指向的还是自己; 但如果一个进程创建了其他线程, 那么线程有自己的pid,tgid就是进程的主线程的
pid,group_leader指向的就是进程的主线程.

### signal处理
```c
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
volatile long			state; // state可以取的值定义在include/linux/sched.h头文件中
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

![state 流转](images/task_state_change.png)

TASK_RUNNING并不是说进程正在运行,而是表示进程在时刻准备运行的状态. 当处于这个状态的进程获得时间片的时候,就是在运行中;如果没有获得时间片,就说明它被其他进程抢占了,在等待再次分配时间片.

在运行中的进程,一旦要进行一些I/O操作,需要等待I/O完毕,这个时候会释放CPU,进入睡眠状态.

在Linux中,有两种睡眠状态:
- TASK_INTERRUPTIBLE ,可中断的睡眠状态
    这是一种浅睡眠的状态,虽然在睡眠,等待I/O完成,但是这个时候一个信号来的时候,进程还是要被唤醒. 只不过唤醒后,不是继续刚才的操作,而是进行信号处理. 当然也可以根据自己的意愿,来写信号处理函数,例如收到某些信号,就放弃等待这个I/O操作完成,直接退出,也可收到某些信息,继续等待
- TASK_UNINTERRUPTIBLE ,不可中断的睡眠状态
    这是一种深度睡眠状态,不可被信号唤醒,只能死等I/O操作完成. 一旦I/O操作因为特殊原因不能完成,这个时候,谁也叫不醒这个进程了(kill本身也是一个信号,既然这个状态不可被信号唤醒,kill信号也被忽略了).除非重启电脑,没有其他办法. 因此,这其实是一个比较危险的事情,除非极其有把握,不然还是不要设置成该状态.

于是就有了一种新的进程睡眠状态,TASK_KILLABLE,可以终止的新睡眠状态. 进程处于这种状态中,它的运行原理类似TASK_UNINTERRUPTIBLE,只不过可以响应致命信号.从定义可以看出,TASK_WAKEKILL用于在接收到致命信号时唤醒进程,而TASK_KILLABLE相当于这两位都设置了.

TASK_STOPPED是在进程接收到SIGSTOP、SIGTTIN、SIGTSTP或者SIGTTOU信号之后进入该状态.

TASK_TRACED表示进程被debugger等进程监视,进程执行被调试程序所停止. 当一个进程被另外的进程所监视,每一个信号都会让进程进入该状态.

一旦一个进程要结束,先进入的是EXIT_ZOMBIE状态,但是这个时候它的父进程还没有使用wait()等系统调用来获知它的终止信息,此时进程就成了僵尸进程.

EXIT_DEAD是进程的最终状态.

EXIT_ZOMBIE和EXIT_DEAD也可以用于exit_state

state是和进程的运行、调度有关系, 还有其他的一些状态称为标志, 放在flags字段中,这些字段都被定义称为宏 ,以PF开头:
```c
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

![task tree](images/task_parent.png)

### 进程权限
```c
/* Objective and real subjective task credentials (COW): */
const struct cred __rcu		*real_cred; // 说明谁能操作我这个进程

/* Effective (overridable) subjective task credentials (COW): */
const struct cred __rcu		*cred; // 说明我这个进程能够操作谁
```
Objective是被操作对象, Subjective是操作者.

[cred定义](https://sourcegraph.com/github.com/torvalds/linux/-/blob/include/linux/cred.h):
```c
// 大部分是关于用户和用户所属的用户组信息
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

第一个是uid和gid,注释是real user/group id. 一般情况下,谁启动的进程,就是谁的ID. 但是权限审核的时候,往往不比较这两个,也就是说不大起作用
第二个是euid和egid,注释是effective user/group id. 当这个进程要操作消息队列、共享内存、信号量等对象的时候,其实就是在比较这个用户和组是否有权限
第三个是fsuid和fsgid,也就是filesystem user/group id. 这个是对文件操作会审核的权限.一般说来,fsuid、euid,和uid是一样的,fsgid、egid,和gid也是一样的. 因为谁启动的进程,就应该审核启动的用户到底有没有这个权限, 但是也有特殊的情况.比如`chmod u+s program`的情况, uid还是进程创建用户, 但euid和fsuid变成了`program`所有者的uid.

除了以用户和用户组控制权限(粗颗粒度),Linux还有另一个机制就是capabilities(细颗粒度), 它用位图表示权限,定义在[capability.h](https://sourcegraph.com/github.com/torvalds/linux/-/blob/include/uapi/linux/capability.h#L113:9)里.

cap_permitted表示进程能够使用的权限, 但是真正起作用的是cap_effective. cap_permitted中可以包含cap_effective中没有的权限, 一个进程可以在必要的时候放弃自己的某些权限,这样更加安全.

cap_inheritable表示当可执行文件的扩展属性设置了inheritable位时,调用exec执行该程序会继承调用者的
inheritable集合,并将其加入到permitted集合. 但在非root用户下执行exec时,通常不会保留inheritable集合,但是往往又是非root用户,才想保留权限,所以非常鸡肋.

cap_bset,也就是capability bounding set,是系统中所有进程允许保留的权限. 如果这个集合中不存在某个权限,那么系统中的所有进程都没有这个权限. 即使以超级用户权限执行的进程,也是一样的. 这样有很多好处。例如,系统启动以后,将加载内核模块的权限去掉,那所有进程都不能加载内核模块.

cap_ambient是比较新加入内核的,就是为了解决cap_inheritable鸡肋的状况,也就是,非root用户进程使用exec执行一个程序的时候,如何保留权限的问题. 当执行exec的时候,cap_ambient会被添加到cap_permitted中,同时设置到cap_effective中.

### 内存管理
```c
struct mm_struct *mm;
struct mm_struct *active_mm;
```

### 文件和文件系统
```c
/* Filesystem information: */
struct fs_struct *fs;
/* Open file information: */
struct files_struct *files;
```