# signal
ref:
- `man signal`
- [Linux信号(signal)机制](http://gityuan.com/2015/12/20/signal/)
- [第 33 章 信号 - 部分 III. Linux系统编程](http://akaedu.github.io/book/ch33.html)

## 概述
### 信号类型
Linux系统共定义了64种信号，分为两大类：可靠信号与不可靠信号，前32种信号为不可靠信号，后32种为可靠信号。


> - 不可靠信号： 也称为非实时信号，不支持排队，信号可能会丢失, 比如发送多次相同的信号, 进程只能收到一次. 信号值取值区间为1~31

> - 可靠信号： 也称为实时信号，支持排队, 信号不会丢失, 发多少次, 就可以收到多少次. 信号值取值区间为32~64

在终端，可通过`kill -l`可查看所有的signal信号:
<table class="table">
  <thead>
    <tr>
      <th>取值</th>
      <th>名称</th>
      <th>解释</th>
      <th>默认动作</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>SIGHUP</td>
      <td>挂起</td>
      <td></td>
    </tr>
    <tr>
      <td>2</td>
      <td>SIGINT</td>
      <td>中断</td>
      <td></td>
    </tr>
    <tr>
      <td>3</td>
      <td>SIGQUIT</td>
      <td>退出</td>
      <td></td>
    </tr>
    <tr>
      <td>4</td>
      <td>SIGILL</td>
      <td>非法指令</td>
      <td></td>
    </tr>
    <tr>
      <td>5</td>
      <td>SIGTRAP</td>
      <td>断点或陷阱指令</td>
      <td></td>
    </tr>
    <tr>
      <td>6</td>
      <td>SIGABRT</td>
      <td>abort发出的信号</td>
      <td></td>
    </tr>
    <tr>
      <td>7</td>
      <td>SIGBUS</td>
      <td>非法内存访问</td>
      <td></td>
    </tr>
    <tr>
      <td>8</td>
      <td>SIGFPE</td>
      <td>浮点异常</td>
      <td></td>
    </tr>
    <tr>
      <td>9</td>
      <td>SIGKILL</td>
      <td>kill信号</td>
      <td>不能被忽略、处理和阻塞</td>
    </tr>
    <tr>
      <td>10</td>
      <td>SIGUSR1</td>
      <td>用户信号1</td>
      <td></td>
    </tr>
    <tr>
      <td>11</td>
      <td>SIGSEGV</td>
      <td>无效内存访问</td>
      <td></td>
    </tr>
    <tr>
      <td>12</td>
      <td>SIGUSR2</td>
      <td>用户信号2</td>
      <td></td>
    </tr>
    <tr>
      <td>13</td>
      <td>SIGPIPE</td>
      <td>管道破损，没有读端的管道写数据</td>
      <td></td>
    </tr>
    <tr>
      <td>14</td>
      <td>SIGALRM</td>
      <td>alarm发出的信号</td>
      <td></td>
    </tr>
    <tr>
      <td>15</td>
      <td>SIGTERM</td>
      <td>终止信号</td>
      <td></td>
    </tr>
    <tr>
      <td>16</td>
      <td>SIGSTKFLT</td>
      <td>栈溢出</td>
      <td></td>
    </tr>
    <tr>
      <td>17</td>
      <td>SIGCHLD</td>
      <td>子进程退出</td>
      <td>默认忽略</td>
    </tr>
    <tr>
      <td>18</td>
      <td>SIGCONT</td>
      <td>进程继续</td>
      <td></td>
    </tr>
    <tr>
      <td>19</td>
      <td>SIGSTOP</td>
      <td>进程停止</td>
      <td>不能被忽略、处理和阻塞</td>
    </tr>
    <tr>
      <td>20</td>
      <td>SIGTSTP</td>
      <td>进程停止</td>
      <td></td>
    </tr>
    <tr>
      <td>21</td>
      <td>SIGTTIN</td>
      <td>进程停止，后台进程从终端读数据时</td>
      <td></td>
    </tr>
    <tr>
      <td>22</td>
      <td>SIGTTOU</td>
      <td>进程停止，后台进程想终端写数据时</td>
      <td></td>
    </tr>
    <tr>
      <td>23</td>
      <td>SIGURG</td>
      <td>I/O有紧急数据到达当前进程</td>
      <td>默认忽略</td>
    </tr>
    <tr>
      <td>24</td>
      <td>SIGXCPU</td>
      <td>进程的CPU时间片到期</td>
      <td></td>
    </tr>
    <tr>
      <td>25</td>
      <td>SIGXFSZ</td>
      <td>文件大小的超出上限</td>
      <td></td>
    </tr>
    <tr>
      <td>26</td>
      <td>SIGVTALRM</td>
      <td>虚拟时钟超时</td>
      <td></td>
    </tr>
    <tr>
      <td>27</td>
      <td>SIGPROF</td>
      <td>profile时钟超时</td>
      <td></td>
    </tr>
    <tr>
      <td>28</td>
      <td>SIGWINCH</td>
      <td>窗口大小改变</td>
      <td>默认忽略</td>
    </tr>
    <tr>
      <td>29</td>
      <td>SIGIO</td>
      <td>I/O相关</td>
      <td></td>
    </tr>
    <tr>
      <td>30</td>
      <td>SIGPWR</td>
      <td>关机</td>
      <td>默认忽略</td>
    </tr>
    <tr>
      <td>31</td>
      <td>SIGSYS</td>
      <td>系统调用异常</td>
      <td></td>
    </tr>
  </tbody>
</table>

对于signal信号，绝大部分的默认处理都是终止进程或停止进程，或dump内核映像转储.

### 产生
信号来源分为硬件类和软件类：

1. 硬件方式

	- 用户输入：比如在终端上按下组合键ctrl+C，产生SIGINT信号
	- 硬件异常：CPU检测到内存非法访问等异常，通知内核生成相应信号，并发送给发生事件的进程

1. 软件方式

	通过系统调用，发送signal信号：kill()，raise()，sigqueue()，alarm()，setitimer()，abort()

	- kernel,使用 kill_proc_info(）等
	- native,使用 kill() 或者raise()等
	- java,使用 Procees.sendSignal()等

### 信号注册和注销
1. 注册

	在进程task_struct结构体中有一个未决信号的成员变量 struct sigpending pending。每个信号在进程中注册都会把信号值加入到进程的未决信号集。

	非实时信号发送给进程时，如果该信息已经在进程中注册过，不会再次注册，故信号会丢失；
	实时信号发送给进程时，不管该信号是否在进程中注册过，都会再次注册。故信号不会丢失；

1. 注销

	非实时信号：不可重复注册，最多只有一个sigqueue结构；当该结构被释放后，把该信号从进程未决信号集中删除，则信号注销完毕
	实时信号：可重复注册，可能存在多个sigqueue结构；当该信号的所有sigqueue处理完毕后，把该信号从进程未决信号集中删除，则信号注销完毕

## 信号处理
内核处理进程收到的signal是在当前进程的上下文，故进程必须是Running状态. 当进程唤醒或者调度后获取CPU，则会从内核态转到用户态时检测是否有signal等待处理，处理完，进程会把相应的未决信号从链表中去掉.

### 处理时机
signal信号处理时机： 内核态 -> signal信号处理 -> 用户态：
- 在内核态，signal信号不起作用
- 在用户态，signal所有未被屏蔽的信号都处理完毕
- 当屏蔽信号，取消屏蔽时，会在下一次内核转用户态的过程中执行

### 处理方式
进程对信号的处理方式： 有3种

- 默认 :接收到信号后按默认的行为处理该信号. 这是多数应用采取的处理方式
- 自定义 : 用自定义的信号处理函数来执行特定的动作
- 忽略 : 接收到信号后不做任何反应

### 信号安装
进程处理某个信号前，需要先在进程中安装此信号. 安装过程主要是建立信号值和进程对相应信息值的动作。

信号安装函数:
- `signal(int signum, const struct sigaction *act, struct sigaction *oldact)`：不支持信号传递信息，主要用于非实时信号安装

	- signum：要操作的signal信号
	- act：设置对signal信号的新处理方式
	- oldact：原来对信号的处理方式
	- 返回值：0 表示成功，-1 表示有错误发生
- sigaction():支持信号传递信息，可用于所有信号安装

其中 sigaction结构体:

- sa_handler:信号处理函数
- sa_mask：指定信号处理程序执行过程中需要阻塞的信号；
- sa_flags：标示位

	- SA_RESTART：使被信号打断的syscall重新发起。
	- SA_NOCLDSTOP：使父进程在它的子进程暂停或继续运行时不会收到 SIGCHLD 信号。
	- SA_NOCLDWAIT：使父进程在它的子进程退出时不会收到SIGCHLD信号，这时子进程如果退出也不会成为僵 尸进程。
	- SA_NODEFER：使对信号的屏蔽无效，即在信号处理函数执行期间仍能发出这个信号。
	- SA_RESETHAND：信号处理之后重新设置为默认的处理方式。
	- SA_SIGINFO：使用sa_sigaction成员而不是sa_handler作为信号处理函数。

### 信号发送
- kill()：用于向进程或进程组发送信号；
sigqueue()：只能向一个进程发送信号，不能像进程组发送信号；主要针对实时信号提出，与sigaction()组合使用，当然也支持非实时信号的发送；
- alarm()：用于调用进程指定时间后发出SIGALARM信号；
- setitimer()：设置定时器，计时达到后给进程发送SIGALRM信号，功能比alarm更强大；
- abort()：向进程发送SIGABORT信号，默认进程会异常退出。
- raise()：用于向进程自身发送信号；


## signal
- SIGTERM : 当子进程结束的时候通过SIGTERM信号告诉父进程, 父进程可处理该信号.


	> nohup启动进程，该进程会忽略所有的**sighup**信号, 使得该进程不会随着终端退出(会发送sighup)而结束.

	kill 命令的默认行为是向进程发送 SIGTERM即立即终止进程, 但是，可以在代码中处理、忽略或捕获此信号, 如果信号没有被进程捕获，则进程被杀死; SIGKILL是立即终止进程, 该信号不能被处理（捕获）、忽略或阻塞, `kill -9`即发送SIGKILL.

	升级场景(p, parent process;c, child process):
	1. p执行`nohup c > /dev/null 2>&1 &`进行升级
	2. c执行`rpm -e`卸载p所在package, 此时p会terminated(by kill), 导致c也被terminated.

	解决方法: [setsid](https://wangchujiang.com/linux-command/c/setsid.html). setsid命令 子进程从父进程继承了：SessionID、进程组ID和打开的终端。子进程如果要脱离这些，代码中可通过调用setsid来实现。，而命令行或脚本中可以通过使用命令setsid来运行程序实现。setsid帮助一个进程脱离从父进程继承而来的已打开的终端、隶属进程组和隶属的会话.

  以上方法不适合systemd+cgroup的系统, 原因在[这里](https://segmentfault.com/q/1010000041547027):
  
  与 systemd 利用cgroup进行层级管理有关系，systemd停止一个服务时，默认的KillMode是基于cgroup来识别的，换句话说systemd中管理的服务，下面fork出来的子进程，即使被你丢入后台，子进程脱离了父进程的关联，它的cgroup层级还是默认被关联在原来的服务下.

  具体可见man systemd.kill中的说明，然后调整一下apiserver的systemd配置, 改下默认的KillMode配置, 改为process试试.

  或参考truenas middlewared.service利用`RestartPreventExitStatus=SIGTERM`和`SendSIGKILL=no`

  > RestartPreventExitStatus表示当符合某些退出状态时不要进行重启.

  **推荐方法**: 通过systemd-run创建临时cgroup来解决: [`systemd-run --unit=my_system_upgrade --scope --slice=my_system_upgrade_slice -E  setsid nohup start-the-upgrade &> /tmp/some-logs.log &`](https://stackoverflow.com/questions/35200232/how-to-launch-a-process-outside-a-systemd-control-group)

  > [transient cgroup with systemd-run](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/resource_management_guide/chap-using_control_groups#sec-Creating_Transient_Cgroups_with_systemd-run)


## linux实现
signal_struct 和 sighand_struct , 分 别 对 应 task_struct 的 signal 和sighand字段(都是指针),二者都被同一个线程组中的线程共享.

signal_struct表示进程(线程组)当前的信号信息(状态).

```c
// https://elixir.bootlin.com/linux/v6.6.30/source/include/linux/sched/signal.h#L93
/*
 * NOTE! "signal_struct" does not have its own
 * locking, because a shared signal_struct always
 * implies a shared sighand_struct, so locking
 * sighand_struct is always a proper superset of
 * the locking of signal_struct.
 */
struct signal_struct {
	refcount_t		sigcnt;
	atomic_t		live;
	int			nr_threads;
	int			quick_threads;
	struct list_head	thread_head;

	wait_queue_head_t	wait_chldexit;	/* for wait4() */

	/* current thread group signal load-balancing target: */
	struct task_struct	*curr_target;

	/* shared signal handling: */
	struct sigpending	shared_pending; // 待处理的信号

	/* For collecting multiprocess signals during fork */
	struct hlist_head	multiprocess;

	/* thread group exit support */
	int			group_exit_code; // 退出码
	/* notify group_exec_task when notify_count is less or equal to 0 */
	int			notify_count;
	struct task_struct	*group_exec_task; // 等待线程组退出的进程

	/* thread group stop support, overloads group_exit_code too */
	int			group_stop_count;
	unsigned int		flags; /* see SIGNAL_* flags below */

	struct core_state *core_state; /* coredumping support */

	/*
	 * PR_SET_CHILD_SUBREAPER marks a process, like a service
	 * manager, to re-parent orphan (double-forking) child processes
	 * to this process instead of 'init'. The service manager is
	 * able to receive SIGCHLD signals and is able to investigate
	 * the process until it calls wait(). All children of this
	 * process will inherit a flag if they should look for a
	 * child_subreaper process at exit.
	 */
	unsigned int		is_child_subreaper:1;
	unsigned int		has_child_subreaper:1;

#ifdef CONFIG_POSIX_TIMERS

	/* POSIX.1b Interval Timers */
	unsigned int		next_posix_timer_id;
	struct list_head	posix_timers;

	/* ITIMER_REAL timer for the process */
	struct hrtimer real_timer;
	ktime_t it_real_incr;

	/*
	 * ITIMER_PROF and ITIMER_VIRTUAL timers for the process, we use
	 * CPUCLOCK_PROF and CPUCLOCK_VIRT for indexing array as these
	 * values are defined to 0 and 1 respectively
	 */
	struct cpu_itimer it[2];

	/*
	 * Thread group totals for process CPU timers.
	 * See thread_group_cputimer(), et al, for details.
	 */
	struct thread_group_cputimer cputimer;

#endif
	/* Empty if CONFIG_POSIX_TIMERS=n */
	struct posix_cputimers posix_cputimers;

	/* PID/PID hash table linkage. */
	struct pid *pids[PIDTYPE_MAX];

#ifdef CONFIG_NO_HZ_FULL
	atomic_t tick_dep_mask;
#endif

	struct pid *tty_old_pgrp;

	/* boolean value for session group leader */
	int leader;

	struct tty_struct *tty; /* NULL if no tty */

#ifdef CONFIG_SCHED_AUTOGROUP
	struct autogroup *autogroup;
#endif
	/*
	 * Cumulative resource counters for dead threads in the group,
	 * and for reaped dead child processes forked by this group.
	 * Live threads maintain their own counters and add to these
	 * in __exit_signal, except for the group leader.
	 */
	seqlock_t stats_lock;
	u64 utime, stime, cutime, cstime;
	u64 gtime;
	u64 cgtime;
	struct prev_cputime prev_cputime;
	unsigned long nvcsw, nivcsw, cnvcsw, cnivcsw;
	unsigned long min_flt, maj_flt, cmin_flt, cmaj_flt;
	unsigned long inblock, oublock, cinblock, coublock;
	unsigned long maxrss, cmaxrss;
	struct task_io_accounting ioac;

	/*
	 * Cumulative ns of schedule CPU time fo dead threads in the
	 * group, not including a zombie group leader, (This only differs
	 * from jiffies_to_ns(utime + stime) if sched_clock uses something
	 * other than jiffies.)
	 */
	unsigned long long sum_sched_runtime;

	/*
	 * We don't bother to synchronize most readers of this at all,
	 * because there is no reader checking a limit that actually needs
	 * to get both rlim_cur and rlim_max atomically, and either one
	 * alone is a single word that can safely be read normally.
	 * getrlimit/setrlimit use task_lock(current->group_leader) to
	 * protect this instead of the siglock, because they really
	 * have no need to disable irqs.
	 */
	struct rlimit rlim[RLIM_NLIMITS]; // 资源限制

#ifdef CONFIG_BSD_PROCESS_ACCT
	struct pacct_struct pacct;	/* per-process accounting information */
#endif
#ifdef CONFIG_TASKSTATS
	struct taskstats *stats;
#endif
#ifdef CONFIG_AUDIT
	unsigned audit_tty;
	struct tty_audit_buf *tty_audit_buf;
#endif

	/*
	 * Thread is the potential origin of an oom condition; kill first on
	 * oom
	 */
	bool oom_flag_origin;
	short oom_score_adj;		/* OOM kill score adjustment */
	short oom_score_adj_min;	/* OOM kill score adjustment min value.
					 * Only settable by CAP_SYS_RESOURCE. */
	struct mm_struct *oom_mm;	/* recorded mm when the thread group got
					 * killed by the oom killer */

	struct mutex cred_guard_mutex;	/* guard against foreign influences on
					 * credential calculations
					 * (notably. ptrace)
					 * Deprecated do not use in new code.
					 * Use exec_update_lock instead.
					 */
	struct rw_semaphore exec_update_lock;	/* Held while task_struct is
						 * being updated during exec,
						 * and may have inconsistent
						 * permissions.
						 */
} __randomize_layout;
```

shared_pending是线程组共享的信号队列,task_struct的pending字段也是sigpending结构体类型,表示线程独有的信号队列.

rlim表示进程当前对资源使用的限制,是rlimit结构体数组类型,可以通过getrlimit和setrlimit系统调用获取或者设置某一项限制。rlimit
结构体只有rlim_cur和rlim_max两个字段,表示当前值和最大值,使用setrlimit可以更改它们,增加rlim_max需要CAP_SYS_RESOURCE权
限,也可能会受到一些逻辑限制,比如RLIMIT_NOFILE(number of open files)对应的rlim_max不能超过sysctl_nr_open.

signal_struct在线程组内是共享的,这意味着以上字段的更改会影响到组内所有线程.

sighand_struct结构体表示进程对信号的处理方式,主要字段是action, 它是k_sigaction结构体数组类型,数组元素个数等于
_NSIG(64,系统当前可以支持的信号的最大值)。k_sigaction结构体的最重要字段是sa.sa_handler,表示用户空间传递的处理信号的函数,
函数原型为void(*)(int)。

sigpending结构体的signal字段是一个位图,表示当前待处理(pending)的信号,该位图的位数也等于_NSIG.

### 捕捉信号
signal、sigaction和rt_sigaction系统调用可以更改进程处理信号的方式,其中后两个的可移植性更好。用户可以直接使用glibc包装的
signal,函数原型如下,它一般调用rt_sigaction实现。自定义处理信号的函数,信号产生时函数被调用,被称为捕捉信号。

```c
#include <signal.h>

typedef void (*sighandler_t)(int);

sighandler_t signal(int signum, sighandler_t handler);
// ---
#include <signal.h>

int sigaction(int signum,
              const struct sigaction *_Nullable restrict act,
              struct sigaction *_Nullable restrict oldact);
```

三个系统调用最终都调用do_sigaction函数实现,区别仅在于传递给函数的act参数的值不同(集中在act->sa.sa_flags字段).

do_sigaction逻辑:
第1步,合法性检查,用户传递的sig必须在[1, 64]范围内,而且不能 更 改 sig_kernel_only 的 信 号. sig_kernel_only 包 含 SIGKILL 和
SIGSTOP两种信号,用户不可以自定义它们的行为.

第2步,设置p->sighand->action[sig-1],更改进程对信号的处理方式

  针对不同的信号,采用的处理方式也不相同.

  用户可以为信号设置自定义的处理方式,也可以恢复默认方式( sa_handler == SIG_DFL ) , 或 者 忽 略 信 号(sa_handler==SIG_IGN)。

第3步,更改了信号的处理方式后,如果信号之前的处理方式是忽略,则将之前收到的该信号从线程组中删除.

### 发送信号
kill、tgkill和tkill等系统调用可以用来给进程发送信号,用户空间可以直接使用kill或pthread_kill.

kill函数调用kill系统调用实现,后者初始化kernel_siginfo结构体,然后调用kill_something_info.

分三种情况,如果pid>0,信号传递给pid指定的进程:
1. 如果pid等于-1,信号会传递给当前进程有权限的所有进程,idle进程和同线程组的进程除外
1. 如果pid等于0,传递信号至当前进程组的每一个进程
1. 如果pid小于-1,传递至-pid指定的进程组的每一个进程

kill_pid_info和__kill_pgrp_info最终都是调用group_send_sig_info实现的(__kill_pgrp_info对进程组中每一个进程循环调用它), group_send_sig_info又调用do_send_sig_info.

do_send_sig_info函数最终会调用__send_signal发送信号,后者集中了发送信号的主要逻辑.

第1步,调用prepare_signal判断是否应该发送信号,判断依据和任务有以下几点。
(1) 如 果 进 程 处 于 coredump (>signal&SIGNAL_GROUP_COREDUMP)中,仅发送SIGKILL
(2)如果发送的信号属于sig_kernel_stop,清除线程组中所有的SIGCONT信号
(3)如果发送的信号等于SIGCONT,那么清除线程组种所有的属于sig_kernel_stop的信号, 并调用wake_up_state(t,__TASK_STOPPED)唤醒之前被停止的进程
(4)调用sig_ignored判断是否应该忽略信号,如果是则不发送信号.

首先,被block的信号不会被忽略,它们由task_struct的blocked和real_blocked两个字段表示,其中后者表示的block信号掩码是临时的, blocked字段可以由rt_sigprocmask系统调用设置。其次,当前处理方式不属于ignore的信号不可忽略.

第2步,根据group参数选择信号队列,t->signal->shared_pending是线程组共享的,t->pending是线程(进程)独有的,只有type参数等于PIDTYPE_PID时才会选择后者。使用kill系统调用发送信号时type等于PIDTYPE_MAX,所以它最终将信号挂载到线程组共享的信号队列中,也就是说它无法发送信号给一个特定的线程.

第3步,如果信号不是实时信号,且已经在信号队列中,则直接返回.

实 时 信 号 是 指 在 [SIGRTMIN, SIGRTMAX] 之 间 的 信 号 ,SIGRTMIN一般等于32,[1, 31]区间内的信号称作标准信号都有既定的含义,比如1是SIGHUP,31是SIGSYS。二者的区别如下.
首先,最大的区别是前者可以重复,后者不可,这就是第3步的逻辑。
其次,对实时信号而言,在调用sigaction自定义信号处理方式的时候,如果SA_SIGINFO标志置1,可以定义不同形式的handler,函数原型为void (*) (int, siginfo_t*, void*)。
最后,实时信号的顺序可以保证,同类型的信号,先发送先处理。

sigpending结构体的signal字段是位图,只能表达某个信号的有无,无法表示它的数量,标准信号可重复是如何实现的呢?答案在第4步。申请一个sigqueue对象,由它保存信号的信息,将它插入pending-
>list链表中,链表中当然可以存在相同的信号。

第5步,将信号在位图中对应的位置1,然后调用complete_signal查找一个处理信号的进程(线程):

第1步,查找一个可以处理信号的进程,在第3步唤醒它. wants_signal判断一个进程是否可以处理信号,依据如下:
(1)如果信号被进程block(p->blocked),不处理
(2)如果进程正在退出(PF_EXITING),不处理
(3)信号等于SIGKILL,必须处理
(4)处于__TASK_STOPPED | __TASK_TRACED状态的进程,不 处 理 。 __TASK_STOPPED 状 态 的 进 程 正 在 等 待 SIGCONT ,prepare_signal已经处理了这种情况.
(5)进程当前正在执行(task_curr),或者进程没有待处理的信号(!signal_pending),处理.

如果目标进程不可以处理信号,查找它的线程组内是否有线程可以处理它,如果暂时找不到,先返回等待进程在其他情况下处理它。

第2步,如果信号属于sig_fatal,并满足代码中的一系列条件,在信号不属于sig_kernel_coredump的情况下(该情况另行处理),发送
SIGKILL到线程组中的每一个线程,整个组的进程退出。

哪些信号属于sig_fatal?首先信号不属于sig_kernel_ignore,其次信号不属于sig_kernel_stop,最后信号的处理方式是SIG_DFL。也就是默认行为是terminate,且用户未改变处理方式的信号。

kill不能发送信号至指定线程,pthread_kill可以,它是通过tgkill系统调用实现,和kill不同的是它调用do_send_sig_info时传递的type参数等于PIDTYPE_PID.

### 处理信号
信号是何时、如何处理的?handler是如何被调用的?进程执行完handler后如何回到之前的工作的?这几个问题的答案都是平台相关的,以x86为例.

进程在处理完毕中断或系统调用等将要返回用户空间(resume_userspace)时,会检查是否还有工作要做,处理信号就是其中一项。如果需要(_TIF_SIGPENDING),则调用do_signal处理信号 。 do_signal先调用get_signal获取待处理的信号,然后调用handle_signal处理它.

get_signal调用get_signal_to_deliver实现,它的核心是一个循环.

第1步,优先处理优先级高的信号(synchronous信号),没有则调用dequeue_signal查找其他待处理的信号,p->pending信号队列优先 , 信号值小的优先,如果在该队列找不到, 再查找p->signal->shared_pending队列。也就是说优先处理发送给它自己的信号,然后处理发送给线程组的信号,这也意味着发送给线程组的信号被哪个线程处理是不确定的。

dequeue_signal找到信号后,也会填充它的信息(info),查看sigpending.list链表上是否有与它对应的sigqueue对象(q->info.si_signo
== sig)。如果没有,说明__send_signal的时候跳过了申请sigqueue或者申请失败,从sigpending.signal位图删除信号(sigdelset (&pending-
>signal, sig)),初始化info然后返回;如果只有1个,从位图删除信号,将sigqueue对象从链表删除,复制q->info的信息然后返回;如果
多于1个(实时信号),将sigqueue对象从链表删除,复制q->info的信息然后返回,此时信号还存在,不能从位图中删除。

第2步,如果用户自定义了处理信号的方式,返回信号交由handle_signal处理。如果sa_flags的标志SA_ONESHOT置位,表示用户的更改仅生效一次,恢复处理方式为SIG_DFL。
第3步,采取信号的默认处理方式,sig_kernel_ignore的信号被忽略,sig_kernel_stop的信号触发do_signal_stop,sig_kernel_coredump的信号触发do_coredump,以上都不属于的信号会触发do_group_exit导致线程组退出.

如果进程是因为系统调用进入内核,handle_signal可能会改变系统调用的返回结果,取决于处理信号前系统调用本身的结果.

系统调用返回值被信号改变表:
|系统调用的结果|改变后|
|-ERESTART_RESTARTBLOCk|-ENTR|
|-ERESTARTNOHAND|-ENTR|
|-ERESTARTSYS|-ENTR if(!sa_flags & SA_RESTART)|
|-ERESTARTNOINTR|regs->ax=regs->orig_ax; regs->ip-=2; 重新启动系统调用|
|-ERESTARTSYS且sa_flags & SA_RESTART|同上|

handle_signal的核心是setup_rt_frame,它可以解答第二个问题handler是如何被调用的,这是一个平台相关又比较复杂的话题,我们直接分析它的核心__setup_rt_frame函数(x86上其中一种情况).

函数的第4个参数regs就是保持在内核栈高端,进程返回用户空间使用的pt_regs.

rt_sigframe结构体比较复杂,在此我们只需要明确两点即可。第一,它的第一个字段为pretcode。第二,它可以保存pt_regs.

第1步,调用get_sigframe获得frame,不需要深究过程,只需要注意得到的frame是属于用户空间的,不是内核空间的。
第2步,给pretcode字段赋值为restorer,若SA_RESTORER没有置位, restorer 处 存 放的是代码 __kernel_rt_sigreturn 的地址。直 接使用signal系统调用的情况下,用户没有定制restorer的机会,如果使用glibc包装的signal,glibc可能会提供restorer.
第3步,保存当前的pt_regs。
第4步,更改pt_regs,regs->sp = (unsigned long)frame更改进程用户栈,regs->ip =(unsigned long)ksig->ka.sa.sa_handler,返回用户空间后,执行handler,第二个问题分析完毕.

rt_sigframe结构体的第一个字段为pretcode,handler执行完毕函数返回后,接下来执行的代码就是pretcode指向的代码,以__kernel_rt_sigreturn为例.

它直接触发了rt_sigreturn系统调用.

regs->sp-sizeof(long) 就 是 __setup_rt_frame 中 的 frame ,restore_sigcontext恢复之前保存在frame中的pt_regs的值,rt_sigreturn系统调用返回的时候即可回到处理信号之前的工作了.