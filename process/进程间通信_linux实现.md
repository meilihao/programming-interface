# 进程间通信
使用System V IPC 进程间通信机制体系.

在System V IPC体系中，创建一个 IPC 对象都是 xxxget.

## 管道模型
管道是一种单向传输数据的机制，它其实是一段缓存，里面的数据只能从一端写入，从另一端读出. 如果想互相通信，就需要创建两个管道才行.

管道分为两种类型，shell命令中`|`表示的管道称为匿名管道，即这个类型的管道没有名字，竖线代表的管道随着命令的执行自动创建、自动销毁.

另外一种类型是命名管道. 这个类型的管道需要通过 mkfifo 命令显式地创建.

```bash
# mkfifo hello
```

管道以文件的形式存在，这也符合 Linux 里面一切皆文件的原则. 这个时候, 用ls 一下，可以看到，这个文件的类型是 p，就是 pipe 的意思.

> 当命令向管道写入内容且内容没有被读出时, 该命令会被阻塞.

## 消息队列模型
创建一个消息队列，使用 msgget 函数. 这个函数需要有一个参数 key，这是消息队列的唯一标识，且应该是唯一的.

可以指定一个文件，ftok 会根据这个文件的 inode，生成一个近乎唯一的 key. 只要在这个消息队列的生命周期内，这个文件不要被删除就可以了. 只要不删除，无论什么时刻，再调用 ftok，也会得到同样的 key.

发送消息主要调用 msgsnd 函数: 第一个参数是 message queue 的 id，第二个参数是消息的结构体，第三个参数是消息的长度，最后一个参数是 flag, IPC_NOWAIT 表示发送的时候不阻塞，直接返回.

收消息主要调用 msgrcv 函数: 第一个参数是 message queue 的 id，第二个参数是消息的结构体，第三个参数是可接受的最大长度，第四个参数是消息类型, 最后一个参数是 flag，IPC_NOWAIT 表示接收的时候不阻塞，直接返回.

## 共享内存模型
创建一个共享内存，调用 shmget: 第一个参数是 key，和 msgget 里面的 key 一样，都是唯一定位一个共享内存对象，也可以通过关联文件的方式实现唯一性, 第二个参数是共享内存的大小, 第三个参数如果是 IPC_CREAT，同样表示创建一个新的.

如果一个进程想要访问这一段共享内存，需要将这个内存加载到自己的虚拟地址空间的某个位置，通过 shmat 函数，就是 attach 的意思. 除非对于内存布局非常熟悉，否则可能会 attach 到一个非法地址. 所以，通常的做法是将 addr 设为 NULL，让内核选一个合适的地址, 返回值就是真正被 attach 的地方.

如果共享内存使用完毕，可以通过 shmdt 解除绑定，然后通过 shmctl，将 cmd 设置为 IPC_RMID，从而删除这个共享内存对象.

## 信号量
信号量和共享内存往往要配合使用. 信号量其实是一个计数器，主要用于实现进程间的互斥与同步，而不是用于存储进程间通信数据.

可以将信号量初始化为一个数值，来代表某种资源的总体数量. 对于信号量来讲，会定义两种原子操作，一个是 P 操作，称为申请资源操作. 这个操作会申请将信号量的数值减去 N，表示这些数量被他申请使用了，其他人不能用了. 另一个是 V 操作，我们称为归还资源操作，这个操作会申请将信号量加上 M，表示这些数量已经还给信号量了，其他人可以使用了.

所谓原子操作（Atomic Operation），就是任何一份资源，都只能通过 P 操作借给一个人，不能同时借给两个人.

如果想创建一个信号量，可以通过 semget 函数: 第一个参数 key 也是类似的，第二个参数 num_sems 不是指资源的数量，而是表示可以创建多少个信号量，形成一组信号量，也就是说，如果你有多种资源需要管理，可以创建一个信号量组.

无论是 P 操作还是 V 操作，都统一用 semop 函数: 第一个参数还是信号量组的 id，一次可以操作多个信号量, 第二个参数将这些操作放在一个数组中, 第三个参数 numops 就是有多少个操作.

数组的每一项是一个 struct sembuf，里面的第一个成员是这个操作的对象是哪个信号量(信号量组中对应的序号，0～sem_nums-1). 第二个成员就是要对这个信号量做多少改变(信号量值在一次操作中的改变量). 如果 sem_op < 0，就请求 sem_op 的绝对值的资源. 如果相应的资源数可以满足请求，则将该信号量的值减去 sem_op 的绝对值，函数成功返回. 当相应的资源数不能满足请求时，就要看 sem_flg 了, 如果把 sem_flg 设置为 IPC_NOWAIT，也就是没有资源也不等待，则 semop 函数出错返回 EAGAIN. 如果 sem_flg 没有指定 IPC_NOWAIT，则进程挂起，直到当相应的资源数可以满足请求。若 sem_op > 0，表示进程归还相应的资源数，将 sem_op 的值加到信号量的值上. 如果有进程正在休眠等待此信号量，则唤醒它们.

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/include/uapi/linux/sem.h#L40
/* semop system calls takes an array of these. */
struct sembuf {
	unsigned short  sem_num;	/* semaphore index in array */
	short		sem_op;		/* semaphore operation */
	short		sem_flg;	/* operation flags */
};
```

## 信号
信号没有特别复杂的数据结构，就是用一个代号一样的唯一数字id. 在 Linux 操作系统中，为了响应各种各样的事件，也是定义了几十种的信号，分别代表不同的意义. 信号之间依靠它们的值来区分. 信号可以在任何时候发送给某一进程，进程需要为这个信号配置信号处理函数. 当某个信号发生的时候，就默认执行这个函数就可以了.

可以通过 kill -l 命令，查看所有的信号, 再可以通过 man 7 signal 命令查看各信号的作用.

一旦有信号产生，就有下面这几种，用户进程对信号的处理方式
1. 执行默认操作

    Linux 对每种信号都规定了默认操作，例如, Term就是终止进程的意思. Core 的意思是 Core Dump，也即终止进程后，通过 Core Dump 将当前进程的运行状态保存在文件里面，方便开发者事后进行分析问题在哪里.
    
2.捕捉信号

    可以为信号定义一个信号处理函数, 当信号发生时，就执行相应的信号处理函数, 比如nginx的重启操作.

3.忽略信号

    当不希望处理某些信号的时候，就可以忽略该信号，不做任何处理. 有两个信号是应用进程无法捕捉和忽略的，即 SIGKILL 和 SEGSTOP，它们用于在任何时候中断或结束某一进程.
    
接下来，来看一下信号处理最常见的流程. 这个过程主要是分成两步，第一步是注册信号处理函数. 第二步是发送并处理信号.

如果不想让某个信号执行默认操作，一种方法就是对特定的信号注册相应的信号处理函数，设置信号处理方式的是 signal 函数. 这其实就是定义一个方法，并且将这个方法和某个信号关联起来. 当这个进程遇到这个信号的时候，就执行这个方法.

```c
// https://elixir.bootlin.com/glibc/latest/source/signal/signal.h#L72
/* Type of a signal handler.  */
typedef void (*__sighandler_t) (int);

// https://elixir.bootlin.com/glibc/latest/source/signal/signal.h#L185
#ifdef __USE_GNU
typedef __sighandler_t sighandler_t;
#endif

// https://linux.die.net/man/2/signal
#include <signal.h>

typedef void (*sighandler_t)(int);

sighandler_t signal(int signum, sighandler_t handler); 
```

如果在 Linux 下面执行 man signal 的话，会发现 Linux 不建议直接用这个方法，而是改用 sigaction. 定义如下：
```c
// https://elixir.bootlin.com/glibc/latest/source/signal/signal.h#L240
/* Get and/or set the action for signal SIG.  */
extern int sigaction (int __sig, const struct sigaction *__restrict __act,
		      struct sigaction *__restrict __oact) __THROW;
```

sigaction()与signal()的共同点是都将信号和一个动作进行关联; 区别是, sigaction()用一个结构 struct sigaction 表示了这个动作.

和 signal 类似的是，`struct sigaction`里面还是有 __sighandler_t. 但是，其他成员变量可以让开发者更加细致地控制信号处理的行为. 而 signal 函数没有提供设置这些的方法. 这里需要注意的是，signal 不是系统调用，而是 glibc 封装的一个函数. 这样就像 man signal 里面写的一样，不同的实现方式，设置的参数会不同，会导致行为的不同.

```c
// https://elixir.bootlin.com/glibc/latest/source/sysdeps/posix/sysv_signal.c#L37
/* Set the handler for the signal SIG to HANDLER,
   returning the old handler, or SIG_ERR on error.  */
__sighandler_t
__sysv_signal (int sig, __sighandler_t handler)
{
  struct sigaction act, oact;

  /* Check signal extents to protect __sigismember.  */
  if (handler == SIG_ERR || sig < 1 || sig >= NSIG)
    {
      __set_errno (EINVAL);
      return SIG_ERR;
    }

  act.sa_handler = handler;
  __sigemptyset (&act.sa_mask);
  act.sa_flags = SA_ONESHOT | SA_NOMASK | SA_INTERRUPT;
  act.sa_flags &= ~SA_RESTART;
  if (__sigaction (sig, &act, &oact) < 0)
    return SIG_ERR;

  return oact.sa_handler;
}
```

在这里面，sa_flags 进行了默认的设置. SA_ONESHOT 作用是这里设置的信号处理函数，仅仅起作用一次. 用完了一次后，就设置回默认行为. 这其实并不是我们想看到的, 毕竟一旦设置了一个信号处理函数，肯定希望它一直起作用，直到显式地关闭它.

另外一个设置就是 SA_NOMASK, 通过 __sigemptyset，将 sa_mask 设置为空. 这样的设置表示在这个信号处理函数执行过程中，如果再有其他信号，哪怕相同的信号到来的时候，这个信号处理函数会被中断. 如果一个信号处理函数真的被其他信号中断，其实问题也不大，因为当处理完了其他的信号处理函数后，还会回来接着处理这个信号处理函数的，但是对于相同的信号就有点尴尬了，这就需要这个信号处理函数写得比较有技巧了.

例如，对于这个信号的处理过程中，要操作某个数据结构，因为是相同的信号，很可能操作的是同一个实例，这样的话，同步、死锁这些都要想好. 其实一般的思路应该是，当某一个信号的信号处理函数运行的时候，我们暂时屏蔽这个信号, 屏蔽并不意味着信号一定丢失，而是暂存，这样能够做到信号处理函数对于相同的信号，处理完一个再处理下一个，这样信号处理函数的逻辑要简单得多.

还有一个设置就是设置了 SA_INTERRUPT，清除了 SA_RESTART, 信号的到来时间是不可预期的，有可能程序正在调用某个漫长的系统调用的时候（可以运行 man 7 signal 命令，在这里找 Interruption of system calls and library functions by signal handlers 的部分，里面说得非常详细），这个时候一个信号来了，会中断这个系统调用，去执行信号处理函数，那执行完了以后呢？系统调用怎么办呢？

这时候有两种处理方法，一种就是 SA_INTERRUPT，也即系统调用被中断了，就不再重试这个系统调用了，而是直接返回一个 -EINTR 常量，告诉调用方，这个系统调用被信号中断了，但是怎么处理你看着办. 如果是这样的话，调用方可以根据自己的逻辑，重新调用或者直接返回，这会使得我们的代码非常复杂，在所有系统调用的返回值判断里面，都要特殊判断一下这个值.

另外一种处理方法是 SA_RESTART. 这个时候系统调用会被自动重新启动，不需要调用方自己写代码. 当然也可能存在问题，例如从终端读入一个字符，这个时候用户在终端输入一个'a'字符，在处理'a'字符的时候被信号中断了，等信号处理完毕，再次读入一个字符的时候，如果用户不再输入，就停在那里了，需要用户再次输入同一个字符. 因此，建议使用 sigaction 函数，根据自己的需要定制参数.

glibc 里面有个文件 syscalls.list. 它里面定义了库函数调用哪些系统调用，在这里我们找到了 sigaction.

```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/sigaction.c#L30
#include <internal-signals.h>

int
__sigaction (int sig, const struct sigaction *act, struct sigaction *oact)
{
  if (sig <= 0 || sig >= NSIG || __is_internal_signal (sig))
    {
      __set_errno (EINVAL);
      return -1;
    }

  return __libc_sigaction (sig, act, oact);
}
libc_hidden_weak (__sigaction)
weak_alias (__sigaction, sigaction)

// https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/sigaction.c#L41
/* SPARC passes the restore function as an argument to rt_sigaction.  */
#ifndef STUB
# define STUB(act, sigsetsize) (sigsetsize)
#endif

/* If ACT is not NULL, change the action for SIG to *ACT.
   If OACT is not NULL, put the old action for SIG in *OACT.  */
int
__libc_sigaction (int sig, const struct sigaction *act, struct sigaction *oact)
{
  int result;

  struct kernel_sigaction kact, koact;

  if (act)
    {
      kact.k_sa_handler = act->sa_handler;
      memcpy (&kact.sa_mask, &act->sa_mask, sizeof (sigset_t));
      kact.sa_flags = act->sa_flags;
      SET_SA_RESTORER (&kact, act);
    }

  /* XXX The size argument hopefully will have to be changed to the
     real size of the user-level sigset_t.  */
  result = INLINE_SYSCALL_CALL (rt_sigaction, sig,
				act ? &kact : NULL,
				oact ? &koact : NULL, STUB (act, _NSIG / 8));

  if (oact && result >= 0)
    {
      oact->sa_handler = koact.k_sa_handler;
      memcpy (&oact->sa_mask, &koact.sa_mask, sizeof (sigset_t));
      oact->sa_flags = koact.sa_flags;
      RESET_SA_RESTORER (oact, &koact);
    }
  return result;
}
libc_hidden_def (__libc_sigaction)
```

内核代码注释里面会说，系统调用 signal 是为了兼容过去，系统调用 sigaction 也是为了兼容过去，连参数都变成了 struct compat_old_sigaction，所以说，我们的库函数虽然调用的是 sigaction，到了系统调用层，调用的可不是系统调用 sigaction，而是系统调用 [rt_sigaction](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L4237).

在 rt_sigaction 里面，将用户态的 struct sigaction 结构，拷贝为内核态的 k_sigaction，然后调用 [do_sigaction](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L4237). do_sigaction 也很简单，还记得进程内核的数据结构里，struct task_struct 里面有一个成员 sighand，里面有一个 action. 这是一个数组，下标是信号，内容就是信号处理函数，do_sigaction 就是设置 sighand 里的信号处理函数.

至此，信号处理函数的注册已经完成了.

![](/misc/img/process/7cb86c73b9e73893e6b0e0433d476928.png)

## 发送信号
在终端输入某些组合键的时候，会给进程发送信号，例如，Ctrl+C 产生 SIGINT 信号，Ctrl+Z 产生 SIGTSTP 信号.

有的时候，硬件异常也会产生信号. 比如，执行了除以 0 的指令，CPU 就会产生异常，然后把 SIGFPE 信号发送给进程. 再如，进程访问了非法内存，内存管理模块就会产生异常，然后把信号 SIGSEGV 发送给进程. 

这里同样是硬件产生的，对于中断和信号还是要加以区别. 中断要注册中断处理函数，但是中断处理函数是在内核驱动里面的，信号也要注册信号处理函数，信号处理函数是在用户态进程里面的.

对于硬件触发的，无论是中断，还是信号，肯定是先到内核的，然后内核对于中断和信号处理方式不同. 一个是完全在内核里面处理完毕，一个是将信号放在对应的进程 task_struct 里信号相关的数据结构里面，然后等待进程在用户态去处理. 当然有些严重的信号，内核会把进程干掉. 但是，这也能看出来，中断和信号的严重程度不一样，信号影响的往往是某一个进程，处理慢了，甚至错了，也不过这个进程被干掉，而中断影响的是整个系统. 一旦中断处理中有了 bug，可能整个 Linux 都挂了.

有时候，内核在某些情况下，也会给进程发送信号. 例如，向读端已关闭的管道写数据时产生 SIGPIPE 信号，当子进程退出时，要给父进程发送 SIG_CHLD 信号等.

最直接的发送信号的方法就是，通过命令 kill 来发送信号了. 例如，`kill -9 pid` 可以发送信号给一个进程，杀死它. 另外，还可以通过 kill 或者 sigqueue 系统调用，发送信号给某个进程，也可以通过 tkill 或者 tgkill 发送信号给某个线程. 虽然方式多种多样，但是最终都是调用了 do_send_sig_info 函数，将信号放在相应的 task_struct 的信号数据结构中.

调用链:
- kill->[kill_something_info](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L1556)->kill_proc_info->kill_pid_info->[group_send_sig_info](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L1403)->do_send_sig_info
- tkill->do_tkill->do_send_specific->do_send_sig_info
- tgkill->do_tkill->do_send_specific->do_send_sig_info
- rt_sigqueueinfo->do_rt_sigqueueinfo->kill_proc_info->kill_pid_info->[group_send_sig_info](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L1403)-> do_send_sig_info

[do_send_sig_info](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L1283) -> [send_signal](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L1208)，进而调用 [__send_signal](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L1070):
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L3643
/**
 *  sys_kill - send a signal to a process
 *  @pid: the PID of the process
 *  @sig: signal to be sent
 */
SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
	struct kernel_siginfo info;

	prepare_kill_siginfo(sig, &info);

	return kill_something_info(sig, &info, pid);
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L1556
static int __send_signal(int sig, struct kernel_siginfo *info, struct task_struct *t,
			enum pid_type type, bool force)
{
	struct sigpending *pending;
	struct sigqueue *q;
	int override_rlimit;
	int ret = 0, result;

	assert_spin_locked(&t->sighand->siglock);

	result = TRACE_SIGNAL_IGNORED;
	if (!prepare_signal(sig, t, force))
		goto ret;

	pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending;
	/*
	 * Short-circuit ignored signals and support queuing
	 * exactly one non-rt signal, so that we can get more
	 * detailed information about the cause of the signal.
	 */
	result = TRACE_SIGNAL_ALREADY_PENDING;
	if (legacy_queue(pending, sig))
		goto ret;

	result = TRACE_SIGNAL_DELIVERED;
	/*
	 * Skip useless siginfo allocation for SIGKILL and kernel threads.
	 */
	if ((sig == SIGKILL) || (t->flags & PF_KTHREAD))
		goto out_set;

	/*
	 * Real-time signals must be queued if sent by sigqueue, or
	 * some other real-time mechanism.  It is implementation
	 * defined whether kill() does so.  We attempt to do so, on
	 * the principle of least surprise, but since kill is not
	 * allowed to fail with EAGAIN when low on memory we just
	 * make sure at least one signal gets delivered and don't
	 * pass on the info struct.
	 */
	if (sig < SIGRTMIN)
		override_rlimit = (is_si_special(info) || info->si_code >= 0);
	else
		override_rlimit = 0;

	q = __sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit);
	if (q) {
		list_add_tail(&q->list, &pending->list);
		switch ((unsigned long) info) {
		case (unsigned long) SEND_SIG_NOINFO:
			clear_siginfo(&q->info);
			q->info.si_signo = sig;
			q->info.si_errno = 0;
			q->info.si_code = SI_USER;
			q->info.si_pid = task_tgid_nr_ns(current,
							task_active_pid_ns(t));
			rcu_read_lock();
			q->info.si_uid =
				from_kuid_munged(task_cred_xxx(t, user_ns),
						 current_uid());
			rcu_read_unlock();
			break;
		case (unsigned long) SEND_SIG_PRIV:
			clear_siginfo(&q->info);
			q->info.si_signo = sig;
			q->info.si_errno = 0;
			q->info.si_code = SI_KERNEL;
			q->info.si_pid = 0;
			q->info.si_uid = 0;
			break;
		default:
			copy_siginfo(&q->info, info);
			break;
		}
	} else if (!is_si_special(info) &&
		   sig >= SIGRTMIN && info->si_code != SI_USER) {
		/*
		 * Queue overflow, abort.  We may abort if the
		 * signal was rt and sent by user using something
		 * other than kill().
		 */
		result = TRACE_SIGNAL_OVERFLOW_FAIL;
		ret = -EAGAIN;
		goto ret;
	} else {
		/*
		 * This is a silent loss of information.  We still
		 * send the signal, but the *info bits are lost.
		 */
		result = TRACE_SIGNAL_LOSE_INFO;
	}

out_set:
	signalfd_notify(t, sig);
	sigaddset(&pending->signal, sig);

	/* Let multiprocess signals appear after on-going forks */
	if (type > PIDTYPE_TGID) {
		struct multiprocess_signals *delayed;
		hlist_for_each_entry(delayed, &t->signal->multiprocess, node) {
			sigset_t *signal = &delayed->signal;
			/* Can't queue both a stop and a continue signal */
			if (sig == SIGCONT)
				sigdelsetmask(signal, SIG_KERNEL_STOP_MASK);
			else if (sig_kernel_stop(sig))
				sigdelset(signal, SIGCONT);
			sigaddset(signal, sig);
		}
	}

	complete_signal(sig, t, type);
ret:
	trace_signal_generate(sig, info, t, type != PIDTYPE_PID, result);
	return ret;
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L1065
static inline bool legacy_queue(struct sigpending *signals, int sig)
{
	return (sig < SIGRTMIN) && sigismember(&signals->signal, sig);
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/uapi/asm/signal.h#L62
/* These should not be considered constants from userland.  */
#define SIGRTMIN	32
#define SIGRTMAX	_NSIG
```

在__send_signal中会看到 task_struct 里面的 sigpending. 在上面的代码里面，先是要决定应该用哪个 sigpending by `(type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending;`. 这就要看发送的信号，是给进程的还是线程的. 如果是 kill 发送的，也就是发送给整个进程的，就应该发送给 t->signal->shared_pending. 这里面是整个进程所有线程共享的信号；如果是 tkill 发送的，也就是发给某个线程的，就应该发给 t->pending. 这里面是这个线程的 task_struct 独享的. struct sigpending 里面有两个成员，一个是一个集合 sigset_t，表示都收到了哪些信号，还有一个链表，也表示收到了哪些信号. 它的结构如下：
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/signal_types.h#L30
struct sigpending {
	struct list_head list;
	sigset_t signal;
};

// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/signal.h#L25
typedef struct {
	unsigned long sig[_NSIG_WORDS];
} sigset_t;
```

如果都表示收到了信号，这两者有什么区别呢？接着往下看, 接下来要调用 [legacy_queue](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L1065). 如果满足条件，那就直接退出. 那 legacy_queue 里面判断的是什么条件呢？我们来看它的代码.

当信号小于 SIGRTMIN，也即 32 的时候，如果发现这个信号已经在集合里面了，就直接退出了. 这样会造成什么现象呢？就是信号的丢失. 例如，发送给进程 100 个 SIGUSR1（对应的信号为 10），那最终能够被我们的信号处理函数处理的信号有多少呢, 这就不好说了. 再比如总共 5 个 SIGUSR1，分别是 A、B、C、D、E. 如果这五个信号来得太密. A 来了，但是信号处理函数还没来得及处理，B、C、D、E 就都来了. 根据上面的逻辑，因为 A 已经将 SIGUSR1 放在 sigset_t 集合中了，因而后面四个都要丢失. 如果是另一种情况，A 来了已经被信号处理函数处理了，内核在调用信号处理函数之前，我们会将集合中的标志位清除，这个时候 B 再来，B 还是会进入集合，还是会被处理，也就不会丢.

这样信号能够处理多少，和信号处理函数什么时候被调用，信号多大频率被发送，都有关系，而且从后面的分析，我们可以知道，信号处理函数的调用时间也是不确定的. 看小于 32 的信号如此不靠谱，就称它为不可靠信号.

如果大于 32 的信号是什么情况呢？接着看, 接下来，__sigqueue_alloc 会分配一个 struct sigqueue 对象，然后通过 list_add_tail 挂在 struct sigpending 里面的链表上. 这样就靠谱多了是不是？如果发送过来 100 个信号，变成链表上的 100 项，都不会丢，哪怕相同的信号发送多遍，也处理多遍. 因此，大于 32 的信号我们称为可靠信号. 当然，队列的长度也是有限制的，如果执行 `ulimit -a` 命令，可以看到，这个限制 pending signals (-i) 31181. 当信号挂到了 task_struct 结构之后，最后需要调用 [complete_signal](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L989). 这里面的逻辑也很简单，就是说，既然这个进程有了一个新的信号，赶紧找一个线程处理一下吧.

complete_signal在找到了一个进程或者线程的 task_struct 之后，要调用 [signal_wake_up](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/sched/signal.h#L407)，来企图唤醒它，signal_wake_up 会调用 [signal_wake_up_state](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/signal.c#L989).

signal_wake_up_state 里面主要做了两件事情:
1. 就是给这个线程设置 TIF_SIGPENDING

    这就说明其实信号的处理和进程的调度是采取这样一种类似的机制. 当发现一个进程应该被调度的时候，并不直接把它赶下来，而是设置一个标识位 TIF_NEED_RESCHED，表示等待调度，然后等待系统调用结束或者中断处理结束，从内核态返回用户态的时候，调用 schedule 函数进行调度. 信号也是类似的，当信号来的时候，并不直接处理这个信号，而是设置一个标识位 TIF_SIGPENDING，来表示已经有信号等待处理. 同样等待系统调用结束，或者中断处理结束，从内核态返回用户态的时候，再进行信号的处理.

1. 就是试图唤醒这个进程或者线程

    wake_up_state 会调用 [try_to_wake_up](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/core.c#L2737) 方法. 这个函数学习进程的时候学过，就是将这个进程或者线程设置为 TASK_RUNNING，然后放在运行队列中，这个时候，当随着时钟不断的滴答，迟早会被调用. 如果 wake_up_state 返回 0，说明进程或者线程已经是 TASK_RUNNING 状态了，如果它在另外一个 CPU 上运行，则调用 kick_process 发送一个处理器间中断，强制那个进程或者线程重新调度，重新调度完毕后，会返回用户态运行. 这是一个时机会检查 TIF_SIGPENDING 标识位.

## 信号的处理
就是在从系统调用或者中断返回的时候，咱们讲调度的时候讲过，无论是从系统调用返回还是从中断返回，都会调用 [exit_to_usermode_loop](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/entry/common.c#L211)，只不过之前主要关注了 _TIF_NEED_RESCHED 这个标识位，这次重点关注 _TIF_SIGPENDING 标识位.

如果已经设置了 _TIF_SIGPENDING，就调用 [do_signal](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/signal.c#L806) 进行处理.

do_signal 会调用 [handle_signal](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/signal.c#L706). 按说，信号处理就是调用用户提供的信号处理函数，但是这事儿没有看起来这么简单，因为信号处理函数是在用户态的. 这时就要来回忆系统调用的过程了: 这个进程当时在用户态执行到某一行 Line A，调用了一个系统调用，在进入内核的那一刻，在内核 pt_regs 里面保存了用户态执行到了 Line A, 现在从系统调用返回用户态了，按说应该从 pt_regs 拿出 Line A，然后接着 Line A 执行下去，但是为了响应信号，就不能回到用户态的时候返回 Line A 了，而是应该返回信号处理函数的起始地址.

这个时候，就需要干预和定制 pt_regs 了. 这个时候，要看，是否从系统调用中返回. 如果是从系统调用返回的话，还要区分我们是从系统调用中正常返回，还是在一个非运行状态的系统调用中，因为会被信号中断而返回.

这里解析一个最复杂的场景: 从一个 tap 网卡中读取数据. 当时主要关注 schedule 那一行，也即如果当发现没有数据的时候，就调用 schedule，自己进入等待状态，然后将 CPU 让给其他进程. 具体的代码如下：
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/tap.c#L823
static ssize_t tap_do_read(struct tap_queue *q,
			   struct iov_iter *to,
			   int noblock, struct sk_buff *skb)
{
	DEFINE_WAIT(wait);
	ssize_t ret = 0;

	if (!iov_iter_count(to)) {
		kfree_skb(skb);
		return 0;
	}

	if (skb)
		goto put;

	while (1) {
		if (!noblock)
			prepare_to_wait(sk_sleep(&q->sk), &wait,
					TASK_INTERRUPTIBLE);

		/* Read frames from the queue */
		skb = ptr_ring_consume(&q->ring);
		if (skb)
			break;
		if (noblock) {
			ret = -EAGAIN;
			break;
		}
		if (signal_pending(current)) {
			ret = -ERESTARTSYS;
			break;
		}
		/* Nothing to read, let's sleep */
		schedule();
	}
	if (!noblock)
		finish_wait(sk_sleep(&q->sk), &wait);

put:
	if (skb) {
		ret = tap_put_user(q, skb, to);
		if (unlikely(ret < 0))
			kfree_skb(skb);
		else
			consume_skb(skb);
	}
	return ret;
}
```

此时关注和信号相关的部分, 这其实是一个信号中断系统调用的典型逻辑.

首先，把当前进程或者线程的状态设置为 TASK_INTERRUPTIBLE，这样才能使这个系统调用可以被中断.

其次，可以被中断的系统调用往往是比较慢的调用，并且会因为数据不就绪而通过 schedule 让出 CPU 进入等待状态. 在发送信号的时候，除了设置这个进程和线程的 _TIF_SIGPENDING 标识位之外，还试图唤醒这个进程或者线程，也就是将它从等待状态中设置为 TASK_RUNNING. 当这个进程或者线程再次运行的时候，根据进程调度第一定律，从 schedule 函数中返回，然后再次进入 while 循环. 由于这个进程或者线程是由信号唤醒的，而不是因为数据来了而唤醒的，因而是读不到数据的，但是在 signal_pending 函数中，我们检测到了 _TIF_SIGPENDING 标识位，这说明系统调用没有真的做完，于是返回一个错误 ERESTARTSYS，然后带着这个错误从系统调用返回.

然后，到了 exit_to_usermode_loop->do_signal->handle_signal. 在这里面，当发现出现错误 ERESTARTSYS 的时候，就知道这是从一个没有调用完的系统调用返回的，设置系统调用错误码 EINTR. 接下来，就开始折腾 pt_regs 了，主要通过调用 [setup_rt_frame](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/signal.c#L683)->[__setup_rt_frame](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/signal.c#L359).

frame 的类型是 [rt_sigframe](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/sigframe.h#L59). frame 的意思是帧. 只有在学习栈的时候，提到过栈帧的概念. 对的，这个 frame 就是一个栈帧. 在 [get_sigframe](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/signal.c#L232) 中会得到 pt_regs 的 sp 变量，也就是原来这个程序在用户态的栈顶指针，然后 get_sigframe 中，会将 sp 减去 sizeof(struct rt_sigframe)，也就是把这个栈帧塞到了栈里面，然后又在 __setup_rt_frame 中把 regs->sp 设置成等于 frame. 这就相当于强行在程序原来的用户态的栈里面插入了一个栈帧，并在最后将 regs->ip 设置为用户定义的信号处理函数 sa_handler. 这意味着，本来返回用户态应该接着原来的代码执行的，现在不了，要执行 sa_handler 了. 那执行完了以后呢？按照函数栈的规则，弹出上一个栈帧来，也就是弹出了 frame.

那如果假设 sa_handler 成功返回了，怎么回到程序原来在用户态运行的地方呢？玄机就在 frame 里面. 要想恢复原来运行的地方，首先，原来的 pt_regs 不能丢，这个没问题，是在 setup_sigcontext 里面，将原来的 pt_regs 保存在了 frame 中的 uc_mcontext 里面.

另外，很重要的一点，程序如何跳过去呢？在 __setup_rt_frame 中，还有一个不引起重视的操作，那就是通过 `unsafe_put_user(ksig->ka.sa.sa_restorer, &frame->pretcode, Efault)`，将 sa_restorer 放到了 frame->pretcode 里面，而且还是按照函数栈的规则. 函数栈里面包含了函数执行完跳回去的地址. 当 sa_handler 执行完之后，弹出的函数栈是 frame，也就应该跳到 sa_restorer 的地址.

这是什么地址呢？在 Glibc [__sigaction](https://elixir.bootlin.com/glibc/latest/source/signal/sigaction.c#L25) -> [__libc_sigaction](https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/sigaction.c#L42) -> [SET_SA_RESTORER](https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/x86_64/sigaction.c#L24)函数中它被赋值成了 restore_rt. 这其实就是 sa_handler 执行完毕之后，马上要执行的函数. 从名字就能感觉到，它将恢复原来程序运行的地方. 在 Glibc 中，可以找到它的定义，它竟然调用了一个系统调用，系统调用号为 [__NR_rt_sigreturn](https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/x86_64/64/arch-syscall.h#L239). 因此ra_restorer 会调用系统调用 rt_sigreturn 再次进入内核. rt_sigreturn的任务就是在内核中恢复原来的 pt_regs，重新指向 line A.

```c
// https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/x86_64/sigaction.c#L70
#define RESTORE(name, syscall) RESTORE2 (name, syscall)
# define RESTORE2(name, syscall) \
asm									\
  (									\
   /* `nop' for debuggers assuming `call' should not disalign the code.  */ \
   "	nop\n"								\
   ".align 16\n"							\
   ".LSTART_" #name ":\n"						\
   "	.type __" #name ",@function\n"					\
   "__" #name ":\n"							\
   "	movq $" #syscall ", %rax\n"					\
   "	syscall\n"							\
   ".LEND_" #name ":\n"							\
   ".section .eh_frame,\"a\",@progbits\n"				\
   ".LSTARTFRAME_" #name ":\n"						\
   "	.long .LENDCIE_" #name "-.LSTARTCIE_" #name "\n"		\
   ".LSTARTCIE_" #name ":\n"						\
   "	.long 0\n"	/* CIE ID */					\
   "	.byte 1\n"	/* Version number */				\
   "	.string \"zRS\"\n" /* NUL-terminated augmentation string */	\
   "	.uleb128 1\n"	/* Code alignment factor */			\
   "	.sleb128 -8\n"	/* Data alignment factor */			\
   "	.uleb128 16\n"	/* Return address register column (rip) */	\
   /* Augmentation value length */					\
   "	.uleb128 .LENDAUGMNT_" #name "-.LSTARTAUGMNT_" #name "\n"	\
   ".LSTARTAUGMNT_" #name ":\n"						\
   "	.byte 0x1b\n"	/* DW_EH_PE_pcrel|DW_EH_PE_sdata4. */		\
   ".LENDAUGMNT_" #name ":\n"						\
   "	.align " LP_SIZE "\n"						\
   ".LENDCIE_" #name ":\n"						\
   "	.long .LENDFDE_" #name "-.LSTARTFDE_" #name "\n" /* FDE len */	\
   ".LSTARTFDE_" #name ":\n"						\
   "	.long .LSTARTFDE_" #name "-.LSTARTFRAME_" #name "\n" /* CIE */	\
   /* `LSTART_' is subtracted 1 as debuggers assume a `call' here.  */	\
   "	.long (.LSTART_" #name "-1)-.\n" /* PC-relative start addr.  */	\
   "	.long .LEND_" #name "-(.LSTART_" #name "-1)\n"			\
   "	.uleb128 0\n"			/* FDE augmentation length */	\
   do_cfa_expr								\
   do_expr (8 /* r8 */, oR8)						\
   do_expr (9 /* r9 */, oR9)						\
   do_expr (10 /* r10 */, oR10)						\
   do_expr (11 /* r11 */, oR11)						\
   do_expr (12 /* r12 */, oR12)						\
   do_expr (13 /* r13 */, oR13)						\
   do_expr (14 /* r14 */, oR14)						\
   do_expr (15 /* r15 */, oR15)						\
   do_expr (5 /* rdi */, oRDI)						\
   do_expr (4 /* rsi */, oRSI)						\
   do_expr (6 /* rbp */, oRBP)						\
   do_expr (3 /* rbx */, oRBX)						\
   do_expr (1 /* rdx */, oRDX)						\
   do_expr (0 /* rax */, oRAX)						\
   do_expr (2 /* rcx */, oRCX)						\
   do_expr (7 /* rsp */, oRSP)						\
   do_expr (16 /* rip */, oRIP)						\
   /* libgcc-4.1.1 has only `DWARF_FRAME_REGISTERS == 17'.  */		\
   /* do_expr (49 |* rflags *|, oEFL) */				\
   /* `cs'/`ds'/`fs' are unaligned and a different size.  */		\
   /* gas: Error: register save offset not a multiple of 8  */		\
   "	.align " LP_SIZE "\n"						\
   ".LENDFDE_" #name ":\n"						\
   "	.previous\n"							\
   );
/* The return code for realtime-signals.  */
RESTORE (restore_rt, __NR_rt_sigreturn)
```
可以在内核里面找到 [__NR_rt_sigreturn](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/uapi/asm-generic/unistd.h#L442) 对应的系统调用[sys_rt_sigreturn](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/um/signal.c#L559).

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/include/uapi/asm-generic/unistd.h#L442
#define __NR_rt_sigreturn 139
__SC_COMP(__NR_rt_sigreturn, sys_rt_sigreturn, compat_sys_rt_sigreturn)
```

在这里面，把上次填充的那个 rt_sigframe 拿出来，然后 通过 copy_sc_from_user() 将 pt_regs 恢复成为原来用户态的样子. 从这个系统调用返回的时候，还是调用 exit_to_usermode_loop, 应用还误以为从上次的系统调用返回的呢. 至此，整个信号处理过程才全部结束.

![](/misc/img/process/1414775-20200422081220739-1799079084.png)

![](/misc/img/process/3dcb3366b11a3594b00805896b7731fb.png)