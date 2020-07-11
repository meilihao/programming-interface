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
    
接下来，来看一下信号处理最常见的流程. 这个过程主要是分成两步，第一步是注册信号处理函数. 第二步是发送信号.

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