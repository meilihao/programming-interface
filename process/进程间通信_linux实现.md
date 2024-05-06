# 进程间通信
使用System V IPC 进程间通信机制体系.

在System V IPC体系中，创建一个 IPC 对象都是 xxxget.

XSI IPC源于System V,System V的三种IPC机制,后来收录在Unix的XSI(X/Open System Interface,称作X/Open系统接口)接口
中,称之为XSI IPC.

与POSIX IPC类似,XSI IPC也有semaphore(信号量)、shared memory(共享内存)和message queue(消息队列)三种,不过它们的
接口风格与POSIX IPC的差别较大.

这三种通信方式有很多相似之处,基本的设计思路也相同,都使用特定的数据结构作为载体, 称之为ipc对象.

**与 System V IPC 接口不同，POSIX IPC 接口均为多线程安全接口**.

**POSIX 在无竞争条件下，不需要陷入内核，其实现是非常轻量级的; System V 则不同，无论有无竞争都要执行系统调用，因此性能落了下风**.

总体来说，System V IPC存在时间比较老，许多系统都支持，但是接口复杂，并且可能各平台上实现略有区别. **推荐用POSIX IPC**.

TODO: clean XSI IPC.

## 管道模型
管道是一种单向传输数据的机制，它其实是一段缓存，里面的数据只能从一端写入，从另一端读出. 如果想互相通信，就需要创建两个管道才行.

管道分为两种类型，shell命令中`|`表示的管道称为匿名管道，即这个类型的管道没有名字，竖线代表的管道随着命令的执行自动创建、自动销毁.

另外一种类型是命名管道. 这个类型的管道需要通过 mkfifo 命令显式地创建.

```bash
# mkfifo hello
```

代码用:
```c
#include <unistd.h>

int pipe(int pipefd[2]);

#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <fcntl.h>              /* Definition of O_* constants */
#include <unistd.h>

int pipe2(int pipefd[2], int flags);
```

创建pipe的函数(用户空间)的原型如下,二者的区别仅在于第二个参数,pipe对应pipe2的flags参数为0的情况. 先调用pipe,然后
再使用fcntl设置flags也是可以的,有效的标志仅限于O_CLOEXEC、O_NONBLOCK和O_DIRECT三种.

管道以文件的形式存在，这也符合 Linux 里面一切皆文件的原则. 这个时候, 用ls 一下，可以看到，这个文件的类型是 p，就是 pipe 的意思.

> 当命令向管道写入内容且内容没有被读出时, 该命令会被阻塞.

管道的创建，需要通过这个系统调用`int pipe(int fd[2])`. 创建了一个管道 pipe，会返回两个文件描述符，这表示管道的两端，一个是管道的读取端描述符 fd[0]，另一个是管道的写入端描述符 fd[1].

> 内核提供了pipe和pipe2两个系统调用,二者都调用do_pipe2实现.

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/pipe.c#L1028
SYSCALL_DEFINE2(pipe2, int __user *, fildes, int, flags)
{
	return do_pipe2(fildes, flags);
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/pipe.c#L1006
/*
 * sys_pipe() is the normal C calling standard for creating
 * a pipe. It's not the way Unix traditionally does this, though.
 */
static int do_pipe2(int __user *fildes, int flags)
{
	struct file *files[2];
	int fd[2];
	int error;

	error = __do_pipe_flags(fd, files, flags);
	if (!error) {
		if (unlikely(copy_to_user(fildes, fd, sizeof(fd)))) {
			fput(files[0]);
			fput(files[1]);
			put_unused_fd(fd[0]);
			put_unused_fd(fd[1]);
			error = -EFAULT;
		} else {
			fd_install(fd[0], files[0]);
			fd_install(fd[1], files[1]);
		}
	}
	return error;
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/pipe.c#L956
static int __do_pipe_flags(int *fd, struct file **files, int flags)
{
	int error;
	int fdw, fdr;

	if (flags & ~(O_CLOEXEC | O_NONBLOCK | O_DIRECT | O_NOTIFICATION_PIPE))
		return -EINVAL;

	error = create_pipe_files(files, flags);
	if (error)
		return error;

	error = get_unused_fd_flags(flags);
	if (error < 0)
		goto err_read_pipe;
	fdr = error;

	error = get_unused_fd_flags(flags);
	if (error < 0)
		goto err_fdr;
	fdw = error;

	audit_fd_pair(fdr, fdw);
	fd[0] = fdr;
	fd[1] = fdw;
	return 0;

 err_fdr:
	put_unused_fd(fdr);
 err_read_pipe:
	fput(files[0]);
	fput(files[1]);
	return error;
}
```

在内核中，主要的逻辑在 pipe2 系统调用中. do_pipe2()里面要创建一个数组 files，用来存放管道的两端的打开文件，另一个数组 fd 存放管道的两端的文件描述符. 如果调用 [__do_pipe_flags](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/pipe.c#L956) 没有错误，那就调用 fd_install，将两个 fd 和两个 struct file 关联起来. 这一点和打开一个文件的过程很像了.

__do_pipe_flags里面调用了 create_pipe_files，然后生成了两个 fd, 其中fd[0]是用于读的，fd[1]是用于写的.

创建的文件属于pipefs文件系统,它是一个只有简易结构的文件系统 , mount的时候(mount_pseudo函数)置位了SB_NOUSER标志
(super_block的s_flags字段),意味着pipefs不能挂载到 mountpoint上.

创建一个管道，大部分的逻辑其实都是在 [create_pipe_files](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/pipe.c#L912) 函数里面实现的. 命名管道是创建在文件系统上的, 但从这里可以看出，匿名管道，也是创建在文件系统上的，只不过是一种特殊的文件系统，创建一个特殊的文件，对应一个特殊的 inode，就是这里面的 [get_pipe_inode](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/pipe.c#L872).

从 get_pipe_inode 的实现，可以看出，匿名管道来自一个特殊的文件系统 pipefs. 这个文件系统被挂载后，就得到了 struct vfsmount *pipe_mnt, 然后挂载的文件系统的 superblock 就变成了：pipe_mnt->mnt_sb.

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/pipe.c#L1392
static struct file_system_type pipe_fs_type = {
	.name		= "pipefs",
	.init_fs_context = pipefs_init_fs_context,
	.kill_sb	= kill_anon_super,
};

static int __init init_pipe_fs(void)
{
	int err = register_filesystem(&pipe_fs_type);

	if (!err) {
		pipe_mnt = kern_mount(&pipe_fs_type);
		if (IS_ERR(pipe_mnt)) {
			err = PTR_ERR(pipe_mnt);
			unregister_filesystem(&pipe_fs_type);
		}
	}
	return err;
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/pipe.c#L1181
const struct file_operations pipefifo_fops = {
	.open		= fifo_open,
	.llseek		= no_llseek,
	.read_iter	= pipe_read,
	.write_iter	= pipe_write,
	.poll		= pipe_poll,
	.unlocked_ioctl	= pipe_ioctl,
	.release	= pipe_release,
	.fasync		= pipe_fasync,
};

// https://elixir.bootlin.com/linux/v6.6.30/source/include/linux/pipe_fs_i.h#L58
/**
 *	struct pipe_inode_info - a linux kernel pipe
 *	@mutex: mutex protecting the whole thing
 *	@rd_wait: reader wait point in case of empty pipe
 *	@wr_wait: writer wait point in case of full pipe
 *	@head: The point of buffer production
 *	@tail: The point of buffer consumption
 *	@note_loss: The next read() should insert a data-lost message
 *	@max_usage: The maximum number of slots that may be used in the ring
 *	@ring_size: total number of buffers (should be a power of 2)
 *	@nr_accounted: The amount this pipe accounts for in user->pipe_bufs
 *	@tmp_page: cached released page
 *	@readers: number of current readers of this pipe
 *	@writers: number of current writers of this pipe
 *	@files: number of struct file referring this pipe (protected by ->i_lock)
 *	@r_counter: reader counter
 *	@w_counter: writer counter
 *	@poll_usage: is this pipe used for epoll, which has crazy wakeups?
 *	@fasync_readers: reader side fasync
 *	@fasync_writers: writer side fasync
 *	@bufs: the circular array of pipe buffers
 *	@user: the user who created this pipe
 *	@watch_queue: If this pipe is a watch_queue, this is the stuff for that
 **/
struct pipe_inode_info {
	struct mutex mutex;
	wait_queue_head_t rd_wait, wr_wait;
	unsigned int head;
	unsigned int tail;
	unsigned int max_usage;
	unsigned int ring_size;
#ifdef CONFIG_WATCH_QUEUE
	bool note_loss;
#endif
	unsigned int nr_accounted;
	unsigned int readers; // 读者的数量
	unsigned int writers; // 写者的数量
	unsigned int files; // 引用该文件的file对象的数量
	unsigned int r_counter;
	unsigned int w_counter;
	bool poll_usage;
	struct page *tmp_page;
	struct fasync_struct *fasync_readers;
	struct fasync_struct *fasync_writers;
	struct pipe_buffer *bufs; // pipe_buffer数组
	struct user_struct *user;
#ifdef CONFIG_WATCH_QUEUE
	struct watch_queue *watch_queue;
#endif
};
```

pipe的核心在于pipe_inode_info结构体, get_pipe_inode调用alloc_pipe_info(), 申请一个它的对象, 再将它赋值给inode的i_pipe字段.

pipe_buffer结构体本身并不是buffer,在还没有确定用户需要多大的空间传递数据的情况下就申请16个buffer的设计也不合常理. 用到的时候一个pipe_buffer申请一页内存作为buffer,进程通信过程中写的是这些buffer,读的也是它们.

get_pipe_inode 调用 new_inode_pseudo 函数创建一个 inode后开始填写 inode 的成员，这里和文件系统的很像. 但值得注意的是 [struct pipe_inode_info](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/pipe_fs_i.h#L57)，这个结构里面有个成员是 struct pipe_buffer *bufs. 可以知道，所谓的匿名管道，其实就是内核里面的一串缓存. 另外一个需要注意的是 [pipefifo_fops](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/pipe.c#L1181)，将来对于文件描述符的操作，在内核里面都是对应这里面的操作.

回到 create_pipe_files 函数，创建完了 inode，还需创建一个 dentry 和它对应. dentry 和 inode 对应好了，就要开始创建 struct file 对象了. 先创建用于写入的，对应的操作为 pipefifo_fops；再创建读取的，对应的操作也为 pipefifo_fops. 然后把 private_data 设置为 pipe_inode_info. 这样从 struct file 这个层级上，就能直接操作底层的读写操作.

pipe2系统调用建立了fd和file的关联,用户空间只需要直接使用得到的fd即可。两个进程随后的通信实际上就是操作文件的读端和写
端,所以pipe的本质是两个进程读写同一个文件,只不过该文件没有路径,无法打开,只有“看得到”fd的进程才能使用它.

至此，一个匿名管道就创建成功了. 如果对于 fd[1]写入，调用的是 pipe_write，向 pipe_buffer 里面写入数据；如果对于 fd[0]的读入，调用的是 pipe_read，也就是从 pipe_buffer 里面读取数据.

使用pipe通信实际就是读写文件,也就是pipefifo_fops的read和write操作. pipefifo_fops的write操作最终调用的是pipe_write.

但是这个时候，两个文件描述符都是在一个进程里面的，并没有起到进程间通信的作用，怎么样才能使得管道是跨两个进程的呢？还记得创建进程调用的 fork 吗？在这里面，创建的子进程会复制父进程的 struct files_struct，在这里面 fd 的数组会复制一份，但是 fd 指向的 struct file 对于同一个文件还是只有一份，这样就做到了，两个进程各有两个 fd 指向同一个 struct file 的模式，两个进程就可以通过各自的 fd 写入和读取同一个管道文件实现跨进程通信了.

![](/misc/img/process/9c0e38e31c7a51da12faf4a1aca10ba3.png)

由于管道只能一端写入，另一端读出，所以上面的这种模式会造成混乱，因为父进程和子进程都可以写入，也都可以读出，通常的方法是父进程关闭读取的 fd，只保留写入的 fd，而子进程关闭写入的 fd，只保留读取的 fd，如果需要双向通行，则应该创建两个管道.

到这里，仅仅解析了使用管道进行父子进程之间的通信，但是在 shell 里面的不是这样的. 在 shell 里面运行 A|B 的时候，A 进程和 B 进程都是 shell 创建出来的子进程，A 和 B 之间不存在父子关系. 不过，有了上面父子进程之间的管道这个基础，实现 A 和 B 之间的管道就方便多了. 我们首先从 shell 创建子进程 A，然后在 shell 和 A 之间建立一个管道，其中 shell 保留读取端，A 进程保留写入端，然后 shell 再创建子进程 B. 这又是一次 fork，所以，shell 里面保留的读取端的 fd 也被复制到了子进程 B 里面. 这个时候，相当于 shell 和 B 都保留读取端，只要 shell 主动关闭读取端，就变成了一管道，写入端在 A 进程，读取端在 B 进程.

接下来要做的事情就是，将这个管道的两端和输入输出关联起来. 这就要用到 dup2 系统调用了, 方法是`int dup2(int oldfd, int newfd)`.

这个系统调用，将老的文件描述符赋值给新的文件描述符，让 newfd 的值和 oldfd 一样. 在 files_struct 里面，有这样一个表，下标是 fd，内容指向一个打开的文件 struct file.

在这个表里面，前三项是定下来的，其中第零项 STDIN_FILENO 表示标准输入，第一项 STDOUT_FILENO 表示标准输出，第三项 STDERR_FILENO 表示错误输出.

在 A 进程中，写入端可以做这样的操作：dup2(fd[1],STDOUT_FILENO)，将 STDOUT_FILENO（也即第一项）不再指向标准输出，而是指向创建的管道文件，那么以后往标准输出写入的任何东西，都会写入管道文件. 在 B 进程中，读取端可以做这样的操作，dup2(fd[0],STDIN_FILENO)，将 STDIN_FILENO 也即第零项不再指向标准输入，而是指向创建的管道文件，那么以后从标准输入读取的任何东西，都来自于管道文件. 至此，才将 A|B 的功能完成.

![](/misc/img/process/c042b12de704995e4ba04173e0a304e2.png)

接下来，看命名管道.

FIFO是pipe的兄弟,又被称作命名管道。pipe为通信的进程创建了没有路径的特殊文件,所以进程间的关系需要有血缘关系,FIFO解
决了这个问题,它创建有路径的(命名的)文件供进程使用.

命名管道需要事先通过命令 mkfifo，进行创建. 如果是通过代码创建命名管道，也有一个函数，但是这不是一个系统调用，而是 Glibc 提供的函数. 它的定义如下：
```c
#include <sys/types.h>
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);

#include <fcntl.h>           /* Definition of AT_* constants */
#include <sys/stat.h>

int mkfifoat(int dirfd, const char *pathname, mode_t mode);

// https://elixir.bootlin.com/glibc/latest/source/sysdeps/posix/mkfifo.c#L25
/* Create a named pipe (FIFO) named PATH with protections MODE.  */
int
mkfifo (const char *path, mode_t mode)
{
  dev_t dev = 0;
  return __xmknod (_MKNOD_VER, path, mode | S_IFIFO, &dev);
}

// https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/generic/xmknod.c#L32
/* Create a device file named PATH, with permission and special bits MODE
   and device number DEV (which can be constructed from major and minor
   device numbers with the `makedev' macro above).  */
int
__xmknod (int vers, const char *path, mode_t mode, dev_t *dev)
{
  unsigned long long int k_dev;

  if (vers != _MKNOD_VER)
    {
      __set_errno (EINVAL);
      return -1;
    }

  /* We must convert the value to dev_t type used by the kernel.  */
  k_dev = (*dev) & ((1ULL << 32) - 1);
  if (k_dev != *dev)
    {
      __set_errno (EINVAL);
      return -1;
    }

  return INLINE_SYSCALL (mknodat, 4, AT_FDCWD, path, mode,
                         (unsigned int) k_dev);
}
weak_alias (__xmknod, _xmknod)
libc_hidden_def (__xmknod)
```

并不存在mkfifo系统调用,它们是glibc通过mknod和mknodat实现的.

Glibc 的 mkfifo 函数会调用 mknodat 系统调用, 创建一个字符设备的时候，也是调用的 mknod. 因此命名管道也是一个设备，因而我们也用 mknod.

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/namei.c#L3611
SYSCALL_DEFINE4(mknodat, int, dfd, const char __user *, filename, umode_t, mode,
		unsigned int, dev)
{
	return do_mknodat(dfd, filename, mode, dev);
}

SYSCALL_DEFINE3(mknod, const char __user *, filename, umode_t, mode, unsigned, dev)
{
	return do_mknodat(AT_FDCWD, filename, mode, dev);
}
```

do_mknodat先是通过 user_path_create 对于这个管道文件创建一个 dentry，然后因为是 S_IFIFO，所以调用 vfs_mknod. 由于这个管道文件是创建在一个普通文件系统上的，假设是在 ext4 文件上，于是 vfs_mknod 会调用 ext4_dir_inode_operations 的 mknod，也即会调用 ext4_mknod.

在 ext4_mknod 中，ext4_new_inode_start_handle 会调用 __ext4_new_inode，在 ext4 文件系统上真的创建一个文件，但是会调用 init_special_inode，创建一个内存中特殊的 inode，inode 的 i_fop 变成指向 pipefifo_fops，这一点和匿名管道是一样的. 这样，管道文件就创建完毕了, 接下来，要打开这个管道文件，还是会调用文件系统的 open 函数. 还是沿着文件系统的调用方式，一路调用到 pipefifo_fops 的 open 函数，也就是 [fifo_open](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/pipe.c#L1056).

在 fifo_open 里面，创建 pipe_inode_info，这一点和匿名管道也是一样的. 这个结构里面有个成员是 struct pipe_buffer *bufs. 所谓的命名管道，其实是也是内核里面的一串缓存, 接下来，对于命名管道的写入，还是会调用 pipefifo_fops 的 pipe_write 函数，向 pipe_buffer 里面写入数据. 对于命名管道的读入，还是会调用 pipefifo_fops 的 pipe_read，也就是从 pipe_buffer 里面读取数据.

![](/misc/img/process/486e2bc73abbe91d7083bb1f4f678097.png)

可以看到最终被创建的是FIFO类型的文件。mknod能够成功的前提是目录所属的文件系统为目录的inode定义了mknod操作(inode-
>i_op->mknod)。一般文件系统定义该操作的时候,都要为特殊的文件(普通文件、目录、链接等文件除外的文件)提供特殊的文件操
作 , 常 见 的 实 现 是 调 用 init_special_inode 函 数.init_special_inode给FIFO文件的文件操作为pipefifo_fops,多么熟悉,
pipe的文件操作也是它.

一 句 话 总 结 , mkfifo 创 建 了 一 个 文 件 , 它 的 文 件 操 作 为
pipefifo_fops。

pipefifo_fops的读写已经分析过了,在这方面FIFO和pipe没有区
别。

二者不同的是,pipe创建后,文件已经打开读写两端,相关的pipe_inode_info对象和buf也会被创建;但创建了FIFO文件后,使用它
的进程还需要自行打开文件(fifo_open),pipe_inode_info对象和buf也是打开的过程中创建的。

关于fifo_open有几点需要说明。

首先,如果以只读方式打开FIFO文件,open会阻塞,等待有进程以写方式打开文件才返回(有可能被信号中断)。如果
O_NONBLOCK标志被置位,即使没有写进程不等待,也直接返回;虽 然 open 没 有 出 错 , 但 写 进 程 不 一 定 存 在 , 直 接 读 有 可 能 触 发
SIGPIPE,要确保有数据可读的情况下再读。

其次,如果以只写方式打开FIFO文件,open会阻塞,等待有进程以读方式打开文件才返回(有可能被信号中断)。如果O_NONBLOCK标志被置位又没有读进程的情况下,返回错误.

最后,以读写方式打开FIFO文件,不会出现阻塞.

## 消息队列模型
创建一个消息队列，使用 msgget 函数. 这个函数需要有一个参数 key，这是消息队列的唯一标识，且应该是唯一的.

可以指定一个文件，ftok 会根据这个文件的 inode，生成一个近乎唯一的 key. 只要在这个消息队列的生命周期内，这个文件不要被删除就可以了. 只要不删除，无论什么时刻，再调用 ftok，也会得到同样的 key.

发送消息主要调用 msgsnd 函数: 第一个参数是 message queue 的 id，第二个参数是消息的结构体，第三个参数是消息的长度，最后一个参数是 flag, IPC_NOWAIT 表示发送的时候不阻塞，直接返回.

收消息主要调用 msgrcv 函数: 第一个参数是 message queue 的 id，第二个参数是消息的结构体，第三个参数是可接受的最大长度，第四个参数是消息类型, 最后一个参数是 flag，IPC_NOWAIT 表示接收的时候不阻塞，直接返回.

消息队列有一系列系统调用与之对应:
```c
		#include <fcntl.h>           /* For O_* constants */
		#include <sys/stat.h>        /* For mode constants */
		#include <mqueue.h>

		mqd_t mq_open(const char *name, int oflag);
		mqd_t mq_open(const char *name, int oflag, mode_t mode,
						struct mq_attr *attr);
		int mq_close(mqd_t mqdes);
		int mq_unlink(const char *name);
		int mq_getattr(mqd_t mqdes, struct mq_attr *attr);
		int mq_setattr(mqd_t mqdes, const struct mq_attr *restrict newattr,
						struct mq_attr *restrict oldattr);

	    #include <signal.h>           /* Definition of SIGEV_* constants */
        int mq_notify(mqd_t mqdes, const struct sigevent *sevp);

	    int mq_send(mqd_t mqdes, const char msg_ptr[.msg_len],
                     size_t msg_len, unsigned int msg_prio);
		ssize_t mq_receive(mqd_t mqdes, char msg_ptr[.msg_len],
                          size_t msg_len, unsigned int *msg_prio);

		#include <time.h>
		#include <mqueue.h>

		int mq_timedsend(mqd_t mqdes, const char msg_ptr[.msg_len],
						size_t msg_len, unsigned int msg_prio,
						const struct timespec *abs_timeout);
		ssize_t mq_timedreceive(mqd_t mqdes, char *restrict msg_ptr[.msg_len],
                          size_t msg_len, unsigned int *restrict msg_prio,
                          const struct timespec *restrict abs_timeout);
```

mq_open由glibc封装,参数name必须以“/”开始,经过参数检查后,舍去name的第一个字符(也就是name+1)调用mq_open系统调
用。另外,内核并不接受name中出现“/”,所以传递给mq_open的函数只能是“/no_slash”形式.

Linux以文件作为消息队列的载体,mq_open会在需要的情况下创建文件,返回的消息队列描述符实际上就是文件描述符。需要说明的
是,POSIX并没有规定需要通过文件的方式实现消息队列,所以对消息队列描述符使用poll、select等虽然可行,但没有可移植性。

mq_open的第4个参数struct mq_attr *attr表示消息队列的属性,在open阶段只有mq_maxmsg和mq_msgsize字段起作用,前者限制队列上
消息的最大数目,后者限制消息的长度,发送长度超过mq_msgsize的消息会出错。它的mq_flags和mq_curmsgs字段open时会被忽略,后者表示队列上消息的当前数目。mq_getattr和mq_setattr可以用来获取或修改消息队列的属性,通过mq_getsetattr系统调用实现,mq_setattr功能有限,只能修改mq_flags的O_NONBLOCK标志,也会间接地更改文件的标志(file->f_flags)。

如果mq_open的attr为NULL,则使用系统默认的mq_maxmsg和mq_msgsize值.

mq_open创建或者打开的文件属于一个特殊文件系统,mqueue。它 定 义 了 一 个 内 嵌 inode 的 mqueue_inode_info 结 构 体 ( 以 下 简 称info ) , 创 建 文 件 的 时 候 由 mqueue_alloc_inode ( super_block->s_op->alloc_inode字段)返回它的对象.

```c
// https://elixir.bootlin.com/linux/v6.6.30/source/ipc/mqueue.c#L134
struct mqueue_inode_info {
	spinlock_t lock;
	struct inode vfs_inode;
	wait_queue_head_t wait_q; // 配合实现poll机制

	struct rb_root msg_tree; // 消息队列红黑树
	struct rb_node *msg_tree_rightmost;
	struct posix_msg_tree_node *node_cache;
	struct mq_attr attr;

	struct sigevent notify;
	struct pid *notify_owner;
	u32 notify_self_exec_id;
	struct user_namespace *notify_user_ns;
	struct ucounts *ucounts;	/* user who created, for accounting */
	struct sock *notify_sock;
	struct sk_buff *notify_cookie;

	/* for tasks waiting for free space and messages, respectively */
	struct ext_wait_queue e_wait_q[2]; // 2个等待队列, 一个等待空间(send), 一个等待消息(receive)

	unsigned long qsize; /* size of queue in memory (sum of all msgs) */
};
```

e_wait_q 字 段 表 示 两 个 由 ext_wait_queue 对 象 组 成 的 链 表 ,e_wait_q[0]和e_wait_q[1]是链表的头。ext_wait_queue结构体的task字
段表示等待的进程,msg字段表示发送给进程的消息。

看到msg_tree你应该恍然大悟,消息最终肯定是由这棵红黑树来管理的了。msg_tree红黑树直接管理posix_msg_tree_node对象,每一个对象都管理一组消息,它们的顺序由其priority字段决定。priority表示某个消息的优先级,值越大优先级越高,由mq_send发送消息的时候指定,优先级高的消息会被优先接收.

同一个优先级的多个消息会被插入posix_msg_tree_node的msg_list的链表中。

消息由msg_msg结构体表示,它的m_list字段将它插入msg_list链表,m_type字段表示消息的优先级,m_ts字段表示消息的长度,next
字段为msg_msgseg指针类型,是msg_msgseg组成的链表的头,msg_msgseg只有一个next字段指向下一个msg_msgseg对象。

消息的内容是紧接着msg_msg和msg_msgseg对象存放的,记消息的长度为len,使用alloc_msg(len)可获得msg_msg对象。alloc_msg函数
为消息申请内存,当len小于PAGE_SIZE-sizeof(structmsg_msg)(DATALEN_MSG宏)时,申请len+sizeof(structmsg_msg)个字节的
内存即可,不需要使用msg_msgseg。如果len大于DATALEN_MSG,申请的第一页内存存储msg_msg对象和消息前DATALEN_MSG个字节,接下来的页存储msg_msgseg对象和消息的其余字节,每一页最多存储PAGE_SIZE-sizeof(structmsg_msgseg)(DATALEN_SEG宏)个字节.

#### 发送消息
mq_send和mq_timedsend用来发送消息,前者调用后者实现,后者超时会出错,前者是不限时版本,最终都通过mq_timedsend系统调用实现,主要完成以下任务。

(1)参数检查,消息的优先级不能大于MQ_PRIO_MAX(32768),消息的长度不能大于info->attr.mq_msgsize等。

(2)调用load_msg复制消息,它先调用alloc_msg申请足够的内存存储msg_msg、msg_msgseg对象和消息,然后将消息从用户空间复制
到对应的内存中。

(3)如果队列上消息已满(info->attr.mq_curmsgs==info->attr.mq_maxmsg),文件置位了O_NONBLOCK标志的情况下返回错
误,没有置位的情况下则睡眠在info->e_wait_q[SEND].list链表上等待队列上消息被读取。

(4)如果队列上消息未满,有进程等待接收消息(info->e_wait_q[RECV].list链表不为空)的情况下,调用pipelined_send唤醒
链表上最后一个进程(e_wait_q的两个链表是有序的,按照进程优先级排序,高优先级在后)接收消息。没有进程等待接收消息的情况
下,将消息按照优先级插入info->msg_tree红黑树中。

需要说明的是第4条,如果发送消息时有进程等待接收,消息会被它直接“消化”掉,不算作队列中的消息,也就是说不会增加info->attr.mq_curmsgs字段的值。这是合理的,正如紧俏的商品,还没来得及放在货架上就被哄抢一空。

#### 接收消息
mq_receive和mq_timedreceive用来接收消息,二者的区别与mq_send和mq_timedsend的区别类似,它们最终都通过mq_timedreceive
系统调用实现,主要完成以下任务。

(1)参数检查,要读的消息的长度msg_len不能小于info->attr.mq_msgsize,msg_len是用户空间提供的,表示buffer的长度,过
短有可能无法读完一整条消息。

(2)如果队列上没有消息(info->attr.mq_curmsgs==0),文件置位了O_NONBLOCK标志的情况下返回错误,没有置位的情况下则睡
眠在info->e_wait_q[RECV].list链表上等待消息。如果成功等到消息(有可能超时或者被信号打断),则将消息复制到用户空间。

(3)如果队列上有消息,调用msg_get获得消息,然后将消息复制到用户空间。如果info->e_wait_q[SEND].list链表上有进程在等待,则找到最后一个进程(优先级最高),将它的消息插入队列并将其唤醒。



## 共享内存模型
Linux实现shared memory的方式与semaphore类似,没有特殊的系统调用,由glibc封装的shm_open和shm_unlink. 它们参数name的命名方式与semaphore相同,shm_open的逻辑与sem_open也类似,不同的地方在于shm_open只是打开(创建)文件,并没有调用mmap映射文件,返回的参数也不同,shm_open返回的是文件描述符。另外,shm_open会将文件的FD_CLOEXEC标志置1.

得到了文件描述符后,可以使用ftruncate和mmap操作文件获得共享内存.

创建一个共享内存，调用 `int shmget(key_t key, size_t size, int shmflag)`: 第一个参数是 key，和 msgget 里面的 key 一样，都是唯一定位一个共享内存对象，也可以通过关联文件的方式实现唯一性, 第二个参数是共享内存的大小, 第三个参数如果是 IPC_CREAT，同样表示创建一个新的.

如果一个进程想要访问这一段共享内存，需要将这个内存加载到自己的虚拟地址空间的某个位置，通过 `void *shmat(int  shm_id, const  void *addr, int shmflg)`，at就是 attach 的意思. 除非对于内存布局非常熟悉，否则可能会 attach 到一个非法地址. 所以，通常的做法是将 addr 设为 NULL，让内核自动选一个合适的地址, 返回值就是真正被 attach 的地方.

如果共享内存使用完毕，可以通过 `int shmdt(const  void *shmaddr)` 解除绑定，然后通过 shmctl，将 cmd 设置为 IPC_RMID，从而删除这个共享内存对象.

对于共享内存，需要指定一个大小 size，这个一般要申请多大呢？一个最佳实践是，将多个进程需要共享的数据放在一个 struct 里面，然后这里的 size 就应该是这个 struct 的大小. 这样每一个进程得到这块内存后，只要强制将类型转换为这个 struct 类型，就能够访问里面的共享数据了.

### 创建共享内存
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/shm.c#L726
long ksys_shmget(key_t key, size_t size, int shmflg)
{
	struct ipc_namespace *ns;
	static const struct ipc_ops shm_ops = {
		.getnew = newseg,
		.associate = security_shm_associate,
		.more_checks = shm_more_checks,
	};
	struct ipc_params shm_params;

	ns = current->nsproxy->ipc_ns;

	shm_params.key = key;
	shm_params.flg = shmflg;
	shm_params.u.size = size;

	return ipcget(ns, &shm_ids(ns), &shm_ops, &shm_params);
}

SYSCALL_DEFINE3(shmget, key_t, key, size_t, size, int, shmflg)
{
	return ksys_shmget(key, size, shmflg);
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/util.c#L639
/**
 * ipcget - Common sys_*get() code
 * @ns: namespace
 * @ids: ipc identifier set
 * @ops: operations to be called on ipc object creation, permission checks
 *       and further checks
 * @params: the parameters needed by the previous operations.
 *
 * Common routine called by sys_msgget(), sys_semget() and sys_shmget().
 */
int ipcget(struct ipc_namespace *ns, struct ipc_ids *ids,
			const struct ipc_ops *ops, struct ipc_params *params)
{
	if (params->key == IPC_PRIVATE)
		return ipcget_new(ns, ids, ops, params);
	else
		return ipcget_public(ns, ids, ops, params);
}
```

这里面调用了抽象的 ipcget, 参数分别为共享内存对应的 shm_ids、对应的操作 shm_ops 以及对应的参数 shm_params.

如果 params->key 设置为 IPC_PRIVATE 则永远创建新的，如果不是的话，就会调用 [ipcget_public](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/util.c#L396).

在 ipcget_public 中，会按照 key，去查找 struct kern_ipc_perm. 如果没有找到，那就看是否设置了 IPC_CREAT；如果设置了，就创建一个新的. 如果找到了，就将对应的 id 返回.

按照参数 shm_ops，会调用 [newseg](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/shm.c#L600) 创建新的共享内存.

newseg 函数的第一步，通过 kvmalloc 在直接映射区分配一个 struct shmid_kernel 结构. 这个结构就是用来描述共享内存的. 这个结构最开始就是上面说的 struct kern_ipc_perm 结构. 接下来就是填充这个 struct shmid_kernel 结构，例如 key、权限等.

newseg 函数的第二步，共享内存需要和文件进行关联. 这是因为虚拟地址空间可以和物理内存关联，但是物理内存是某个进程独享的. 虚拟地址空间也可以映射到一个文件，文件是可以跨进程共享的. 而共享内存需要跨进程共享，也应该借鉴文件映射的思路. 只不过不应该映射一个硬盘上的文件，而是映射到一个内存文件系统上的文件. mm/shmem.c 里面就定义了这样一个基于内存的文件系统. 这里一定要注意区分 shmem 和 shm 的区别，前者是一个文件系统，后者是进程通信机制. 在系统初始化的时候，[shmem_init](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/shmem.c#L3860) 注册了 shmem 文件系统 shmem_fs_type，并且挂在到了 shm_mnt 下面.

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/shmem.c#L3849
static struct file_system_type shmem_fs_type = {
	.owner		= THIS_MODULE,
	.name		= "tmpfs",
	.init_fs_context = shmem_init_fs_context,
#ifdef CONFIG_TMPFS
	.parameters	= shmem_fs_parameters,
#endif
	.kill_sb	= kill_litter_super,
	.fs_flags	= FS_USERNS_MOUNT,
};

int __init shmem_init(void)
{
	int error;

	shmem_init_inodecache();

	error = register_filesystem(&shmem_fs_type);
	if (error) {
		pr_err("Could not register tmpfs\n");
		goto out2;
	}

	shm_mnt = kern_mount(&shmem_fs_type);
	if (IS_ERR(shm_mnt)) {
		error = PTR_ERR(shm_mnt);
		pr_err("Could not kern_mount tmpfs\n");
		goto out1;
	}

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
	if (has_transparent_hugepage() && shmem_huge > SHMEM_HUGE_DENY)
		SHMEM_SB(shm_mnt->mnt_sb)->huge = shmem_huge;
	else
		shmem_huge = 0; /* just in case it was patched */
#endif
	return 0;

out1:
	unregister_filesystem(&shmem_fs_type);
out2:
	shmem_destroy_inodecache();
	shm_mnt = ERR_PTR(error);
	return error;
}
```

接下来，newseg 函数会调用 [shmem_kernel_file_setup](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/shmem.c#L4098)，其实就是在 shmem 文件系统里面创建一个文件.

__shmem_file_setup 会创建新的 shmem 文件对应的 dentry 和 inode，并将它们两个关联起来，然后分配一个 struct file 结构，来表示新的 shmem 文件，并且指向独特的 [shmem_file_operations](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/shmem.c#L3751).

shmem_file_setup -> [__shmem_file_setup](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/shmem.c#L4055)会创建新的 shmem 文件对应的 dentry 和 inode，并将它们两个关联起来，然后分配一个 struct file 结构，来表示新的 shmem 文件，并且指向独特的 shmem_file_operations.

newseg 函数的第三步，通过 ipc_addid 将新创建的 struct shmid_kernel 结构挂到 shm_ids 里面的基数树上，并返回相应的 id，并且将 struct shmid_kernel 挂到当前进程的 sysvshm 队列中.

至此，共享内存的创建就完成了.

### 将共享内存映射到虚拟地址空间
从上面的代码解析中，共享内存的数据结构 struct shmid_kernel，是通过它的成员 struct file *shm_file，来管理内存文件系统 shmem 上的内存文件的. 无论这个共享内存是否被映射，shm_file 都是存在的. 接下来，要将共享内存映射到虚拟地址空间中. 调用的是 shmat，对应的系统调用如下：

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/shm.c#L600
SYSCALL_DEFINE3(shmat, int, shmid, char __user *, shmaddr, int, shmflg)
{
	unsigned long ret;
	long err;

	err = do_shmat(shmid, shmaddr, shmflg, &ret, SHMLBA);
	if (err)
		return err;
	force_successful_syscall_return();
	return (long)ret;
}
```

在[do_shmat函数](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/shm.c#L1418)里面，shm_obtain_object_check 会通过共享内存的 id，在基数树中找到对应的 struct shmid_kernel 结构，通过它找到 shmem 上的内存文件.

接下来，要分配一个 struct shm_file_data，来表示这个内存文件. 将 shmem 中指向内存文件的 shm_file 赋值给 struct shm_file_data 中的 file 成员.

然后，创建了一个 struct file，指向的也是 shmem 中的内存文件.

为什么要再创建一个呢？这两个的功能不同，shmem 中 shm_file 用于管理内存文件，是一个中立的，独立于任何一个进程的角色. 而新创建的 struct file 是专门用于做内存映射的，就像一个硬盘上的文件要映射到虚拟地址空间中的时候，需要在 vm_area_struct 里面有一个 struct file *vm_file 指向硬盘上的文件，现在变成内存文件了，但是这个结构还是不能少.

新创建的 struct file 的 private_data，指向 struct shm_file_data，这样内存映射那部分的数据结构，就能够通过它来访问内存文件了. 新创建的 struct file 的 file_operations 也发生了变化，变成了 [shm_file_operations](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/shm.c#L554).

接下来，do_mmap_pgoff 函数 会分配一个 vm_area_struct 指向虚拟地址空间中没有分配的区域，它的 vm_file 指向这个内存文件，然后它会调用 shm_file_operations 的 mmap 函数，也即 shm_mmap 进行映射.

shm_mmap 中调用了 shm_file_data 中的 file 的 mmap 函数，这次调用的是 shmem_file_operations 的 mmap，也即 shmem_mmap.

这里面，vm_area_struct 的 vm_ops 指向 shmem_vm_ops. 等从 call mmap 中返回之后，shm_file_data 的 vm_ops 指向了 [shmem_vm_ops](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/shmem.c#L3823)，而 vm_area_struct 的 vm_ops 改为指向 [shm_vm_ops](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/shm.c#L581).

shm_vm_ops 和 shmem_vm_ops里面最关键的就是 fault 函数，也即访问虚拟内存的时候，访问不到应该怎么办. 当访问不到的时候，先调用 vm_area_struct 的 vm_ops，也即 shm_vm_ops 的 fault 函数 shm_fault, 然后它会转而调用 shm_file_data 的 vm_ops，也即 shmem_vm_ops 的 fault 函数 [shmem_fault](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/shmem.c#L1968).

虽然基于内存的文件系统，已经为这个内存文件分配了 inode，但是内存也却是一点儿都没分配，只有在发生缺页异常的时候才进行分配.

shmem_fault 会调用 shmem_getpage_gfp 在 page cache 和 swap 中找一个空闲页，如果找不到就通过 shmem_alloc_and_acct_page 分配一个新的页，它最终会调用内存管理系统的 alloc_page_vma 在物理内存中分配一个页.

至此，共享内存才真的映射到了虚拟地址空间中，进程可以像访问本地内存一样访问共享内存.

![](/misc/img/process/20e8f4e69d47b7469f374bc9fbcf7251.png)

## 信号量
信号量和共享内存往往要配合使用. 信号量其实是一个计数器，主要用于实现进程间的互斥与同步，而不是用于存储进程间通信数据. **信号量以集合的形式存在的**.

Linux实现的POSIX semaphore效率很高,主要得益于sem_wait和sem_post的实现方式:它们不涉及系统调用,而是通过利用原子操作直接操作内存实现的.

内核中并没有sem_xxx类的系统调用,POSIX semaphore的函数都是glibc封装的,sem_open的实现分两步.

第1步,根据传递的name(假设为“sem_target”)得到目标文件路径。为了可移植性,name命名的最佳方式为“/xxxx”,xxxx中不能出
现“/”符号,最低的要求是“/”只能出现在开头,类似“/a/b”是不允许的。

目标文件所在的目录并不是随意的,glibc会挑选tmpfs和shm类型的文件系统的挂载点,优先选择“/dev/shm”.

第 2 步 , 如 果 “/dev/shm/sem_target” 不 存 在 且 sem_open 置 位 了O_CREAT标志,创建文件,然后调用mmap将文件映射到内存,映射
后的虚拟地址就是信号量的地址。如果“/dev/shm/sem_target”文件已经存在,说明semaphore已经被创建且初始化,打开并映射文件即可。

sem_open实际上也是通过共享内存实现的,不过这段内存的载体是文件。究竟采用有名信号量还是内存信号量,由具体的应用场景决
定。线程间使用内存信号量比较合理,因为这种情况下只需要一个全局变量即可,不需要文件,也不需要内存映射。进程间使用有名信号
量比较方便,但如果是父子进程(fork得到子进程会继承父进程映射的空间),使用父进程映射过的内存可以省去内存映射.

可以将信号量初始化为一个数值，来代表某种资源的总体数量. 对于信号量来讲，会定义两种原子操作，一个是 P 操作，称为申请资源操作. 这个操作会申请将信号量的数值减去 N，表示这些数量被他申请使用了，其他人不能用了. 另一个是 V 操作，我们称为归还资源操作，这个操作会申请将信号量加上 M，表示这些数量已经还给信号量了，其他人可以使用了.

信号量的 P 操作和 V 操作对应 semaphore_p 函数和 semaphore_v 函数. semaphore_p 会调用 semop 函数将信号量的值减一，表示申请占用一个资源，当发现当前没有资源的时候，进入等待. semaphore_v 会调用 semop 函数将信号量的值加一，表示释放一个资源，释放之后，就允许等待中的其他进程占用这个资源.

所谓原子操作（Atomic Operation），就是任何一份资源，都只能通过 P 操作借给一个人，不能同时借给两个人.

如果想创建一个信号量集合，可以通过 `int semget(key_t key, int nsems, int semflg)` 函数: 第一个参数 key 也是类似的，第二个参数 nsems 不是指资源的数量，而是表示可以创建多少个信号量，形成一组信号量，也就是说，如果你有多种资源需要管理，可以创建一个信号量组.

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

![](/misc/img/process/469552bffe601d594c432d4fad97490b.png)

### 信号量的内核机制
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L600
long ksys_semget(key_t key, int nsems, int semflg)
{
	struct ipc_namespace *ns;
	static const struct ipc_ops sem_ops = {
		.getnew = newary,
		.associate = security_sem_associate,
		.more_checks = sem_more_checks,
	};
	struct ipc_params sem_params;

	ns = current->nsproxy->ipc_ns;

	if (nsems < 0 || nsems > ns->sc_semmsl)
		return -EINVAL;

	sem_params.key = key;
	sem_params.flg = semflg;
	sem_params.u.nsems = nsems;

	return ipcget(ns, &sem_ids(ns), &sem_ops, &sem_params);
}

SYSCALL_DEFINE3(semget, key_t, key, int, nsems, int, semflg)
{
	return ksys_semget(key, nsems, semflg);
}
```

解析过了共享内存，再看信号量，就顺畅很多了. 这里同样调用了抽象的 ipcget，参数分别为信号量对应的 sem_ids、对应的操作 sem_ops 以及对应的参数 sem_params.

ipcget 的代码已经解析过了. 如果 key 设置为 IPC_PRIVATE 则永远创建新的；如果不是的话，就会调用 ipcget_public.

在 ipcget_public 中，能会按照 key，去查找 struct kern_ipc_perm. 如果没有找到，那就看看是否设置了 IPC_CREAT. 如果设置了，就创建一个新的; 如果找到了，就将对应的 id 返回.

这里重点看，如何按照参数 sem_ops，创建新的信号量会调用 [newary](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L528).

newary 函数的第一步，通过 kvmalloc 在直接映射区分配一个 [struct sem_array](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L114) 结构. 这个结构是用来描述信号量的，这个结构最开始就是 struct kern_ipc_perm 结构. 接下来就是填充这个 struct sem_array 结构，例如 key、权限等. struct sem_array 里有多个信号量，放在 struct sem sems[]数组里面，在 [struct sem](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L95) 里面有当前的信号量的数值 semval.

struct sem_array 和 struct sem 各有一个链表 struct list_head pending_alter，分别表示对于整个信号量数组的修改和对于某个信号量的修改.

newary 函数的第二步，就是初始化这些链表.

newary 函数的第三步，通过 ipc_addid 将新创建的 struct sem_array 结构，挂到 sem_ids 里面的基数树上，并返回相应的 id.

信号量创建的过程到此结束，接下来, 来看，如何通过 semctl 对信号量数组进行初始化.

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L1649
static long ksys_semctl(int semid, int semnum, int cmd, unsigned long arg, int version)
{
	struct ipc_namespace *ns;
	void __user *p = (void __user *)arg;
	struct semid64_ds semid64;
	int err;

	if (semid < 0)
		return -EINVAL;

	ns = current->nsproxy->ipc_ns;

	switch (cmd) {
	case IPC_INFO:
	case SEM_INFO:
		return semctl_info(ns, semid, cmd, p);
	case IPC_STAT:
	case SEM_STAT:
	case SEM_STAT_ANY:
		err = semctl_stat(ns, semid, cmd, &semid64);
		if (err < 0)
			return err;
		if (copy_semid_to_user(p, &semid64, version))
			err = -EFAULT;
		return err;
	case GETALL:
	case GETVAL:
	case GETPID:
	case GETNCNT:
	case GETZCNT:
	case SETALL:
		return semctl_main(ns, semid, semnum, cmd, p);
	case SETVAL: {
		int val;
#if defined(CONFIG_64BIT) && defined(__BIG_ENDIAN)
		/* big-endian 64bit */
		val = arg >> 32;
#else
		/* 32bit or little-endian 64bit */
		val = arg;
#endif
		return semctl_setval(ns, semid, semnum, val);
	}
	case IPC_SET:
		if (copy_semid_from_user(&semid64, p, version))
			return -EFAULT;
		/* fall through */
	case IPC_RMID:
		return semctl_down(ns, semid, cmd, &semid64);
	default:
		return -EINVAL;
	}
}

SYSCALL_DEFINE4(semctl, int, semid, int, semnum, int, cmd, unsigned long, arg)
{
	return ksys_semctl(semid, semnum, cmd, arg, IPC_64);
}
```

这里重点看，case SETALL 操作调用的 [semctl_main](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L1402) 函数，以及 case SETVAL 操作调用的 [semctl_setval](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L1340) 函数.

对于 SETALL 操作来讲，传进来的参数为 union semun 里面的 unsigned short *array，会设置整个信号量集合.

在 semctl_main 函数中，先是通过 sem_obtain_object_check，根据信号量集合的 id 在基数树里面找到 struct sem_array 对象，发现如果是 SETALL 操作，就将用户的参数中的 unsigned short *array 通过 copy_from_user 拷贝到内核里面的 sem_io 数组，然后是一个循环，对于信号量集合里面的每一个信号量，设置 semval，以及修改这个信号量值的 pid.

对于 SETVAL 操作来讲，传进来的参数 union semun 里面的 int val，仅仅会设置某个信号量.

在 semctl_setval 函数中，我们先是通过 sem_obtain_object_check，根据信号量集合的 id 在基数树里面找到 struct sem_array 对象，对于 SETVAL 操作，直接根据参数中的 val 设置 semval，以及修改这个信号量值的 pid.

至此，信号量数组初始化完毕.

接下来看 P 操作和 V 操作. 无论是 P 操作，还是 V 操作都是调用 semop 系统调用.
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L2275
SYSCALL_DEFINE3(semop, int, semid, struct sembuf __user *, tsops,
		unsigned, nsops)
{
	return do_semtimedop(semid, tsops, nsops, NULL);
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L2235
long ksys_semtimedop(int semid, struct sembuf __user *tsops,
		     unsigned int nsops, const struct __kernel_timespec __user *timeout)
{
	if (timeout) {
		struct timespec64 ts;
		if (get_timespec64(&ts, timeout))
			return -EFAULT;
		return do_semtimedop(semid, tsops, nsops, &ts);
	}
	return do_semtimedop(semid, tsops, nsops, NULL);
}

SYSCALL_DEFINE4(semtimedop, int, semid, struct sembuf __user *, tsops,
		unsigned int, nsops, const struct __kernel_timespec __user *, timeout)
{
	return ksys_semtimedop(semid, tsops, nsops, timeout);
}
```

semop 会调用 [do_semtimedop](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L2275)，这是一个非常复杂的函数.

do_semtimedop 做的第一件事情，就是将用户的参数，例如，对于信号量的操作 struct sembuf，拷贝到内核里面来. 另外，如果是 P 操作，很可能让进程进入等待状态，是否要为这个等待状态设置一个超时，timeout 也是一个参数，会把它变成时钟的滴答数目.

do_semtimedop 做的第二件事情，是通过 sem_obtain_object_check，根据信号量集合的 id，获得 struct sem_array，然后，创建一个 struct sem_queue 表示当前的信号量操作. 为什么叫 queue 呢？因为这个操作可能马上就能完成，也可能因为无法获取信号量不能完成，不能完成的话就只好排列到队列上，等待信号量满足条件的时候. semtimedop 会调用 [perform_atomic_semop](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L644) 在实施信号量操作.

在 perform_atomic_semop 函数中，对于所有信号量操作都进行两次循环. 在第一次循环中，如果发现计算出的 result 小于 0，则说明必须等待，于是跳到 would_block 中，设置 q->blocking = sop 表示这个 queue 是 block 在这个操作上，然后如果需要等待，则返回 1. 如果第一次循环中发现无需等待，则第二个循环实施所有的信号量操作，将信号量的值设置为新的值，并且返回 0.

接下来，回到 do_semtimedop，来看它干的第三件事情，就是如果需要等待，应该怎么办？

如果需要等待，则要区分刚才的对于信号量的操作，是对一个信号量的，还是对于整个信号量集合的. 如果是对于一个信号量的，那就将 queue 挂到这个信号量的 pending_alter 中；如果是对于整个信号量集合的，那就将 queue 挂到整个信号量集合的 pending_alter 中.

接下来的 do-while 循环，就是要开始等待了. 如果等待没有时间限制，则调用 schedule 让出 CPU；如果等待有时间限制，则调用 schedule_timeout 让出 CPU，过一段时间还回来. 当回来的时候，判断是否等待超时，如果没有等待超时则进入下一轮循环，再次等待，如果超时则退出循环，返回错误. 在让出 CPU 的时候，设置进程的状态为 TASK_INTERRUPTIBLE，并且循环的结束会通过 signal_pending 查看是否收到过信号，这说明这个等待信号量的进程是可以被信号中断的，也即一个等待信号量的进程是可以通过 kill 杀掉的.

再来看，do_semtimedop 要做的第四件事情，如果不需要等待，应该怎么办？

如果不需要等待，就说明对于信号量的操作完成了，也改变了信号量的值. 接下来，就是一个标准流程. 通过 DEFINE_WAKE_Q(wake_q) 声明一个 wake_q，调用 [do_smart_update](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L1026)，看这次对于信号量的值的改变，可以影响并可以激活等待队列中的哪些 struct sem_queue，然后把它们都放在 wake_q 里面，调用 wake_up_q 唤醒这些进程. 其实，所有的对于信号量的值的修改都会涉及这三个操作，如果你回过头去仔细看 SETALL 和 SETVAL 操作，在设置完毕信号量之后，也是这三个操作.

来看 do_smart_update 是如何实现的. do_smart_update 会调用 [update_queue](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L946).

update_queue 会依次循环整个信号量集合的等待队列 pending_alter，或者某个信号量的等待队列. 试图在信号量的值变了的情况下，再次尝试 perform_atomic_semop 进行信号量操作. 如果不成功，则尝试队列中的下一个；如果尝试成功，则调用 unlink_queue 从队列上取下来，然后调用 wake_up_sem_queue_prepare，将 q->sleeper 加到 wake_q 上去. q->sleeper 是一个 task_struct，是等待在这个信号量操作上的进程.

接下来，wake_up_q 就依次唤醒 wake_q 上的所有 task_struct，调用的是 wake_up_process 方法.

至此，对于信号量的主流操作都解析完毕了.

其实还有一点需要强调一下，信号量是一个整个 Linux 可见的全局资源，而不像在线程同步中的都是某个进程独占的资源，好处是可以跨进程通信，坏处就是如果一个进程通过 P 操作拿到了一个信号量，但是不幸异常退出了，如果没有来得及归还这个信号量，可能所有其他的进程都阻塞了.

那怎么办呢？Linux 有一种机制叫 SEM_UNDO，也即每一个 semop 操作都会保存一个反向 struct sem_undo 操作，当因为某个进程异常退出的时候，这个进程做的所有的操作都会回退，从而保证其他进程可以正常工作. 如果写的程序里面的 semaphore_p 函数和 semaphore_v 函数，都把 sem_flg 设置为 SEM_UNDO，就是这个作用.

等待队列上的每一个 [struct sem_queue](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L130)，都有一个 [struct sem_undo](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L146)，以此来表示这次操作的反向操作.

在进程的 task_struct 里面对于信号量有一个成员 struct sysv_sem，里面是一个 struct sem_undo_list，将这个进程所有的 semop 所带来的 undo 操作都串起来.

![](/misc/img/process/6028c83b0aa00e65916988911aa01b7c.png)

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

>　可靠信号通过信号队列实现, 即没有处理过的信号会在队列中, 因此不会丢失.

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

### xxxget
消息队列、共享内存、信号量均在使用之前都要生成 key，然后通过 key 得到唯一的 id，并且都是通过 xxxget 函数.

在内核里面，这三种进程间通信机制是使用统一的机制管理起来的，都叫 ipcxxx. 为了维护这三种进程间通信进制，在内核里面，声明了一个有三项的数组.

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/idr.h#L19
struct idr {
	struct radix_tree_root	idr_rt;
	unsigned int		idr_base;
	unsigned int		idr_next;
};

// https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/ipc_namespace.h#L16
struct ipc_ids {
	int in_use;
	unsigned short seq;
	struct rw_semaphore rwsem;
	struct idr ipcs_idr;
	int max_idx;
	int last_idx;	/* For wrap around detection */
#ifdef CONFIG_CHECKPOINT_RESTORE
	int next_id;
#endif
	struct rhashtable key_ht;
};

struct ipc_namespace {
	refcount_t	count;
	struct ipc_ids	ids[3];

	int		sem_ctls[4];
	int		used_sems;

	unsigned int	msg_ctlmax;
	unsigned int	msg_ctlmnb;
	unsigned int	msg_ctlmni;
	atomic_t	msg_bytes;
	atomic_t	msg_hdrs;

	size_t		shm_ctlmax;
	size_t		shm_ctlall;
	unsigned long	shm_tot;
	int		shm_ctlmni;
	/*
	 * Defines whether IPC_RMID is forced for _all_ shm segments regardless
	 * of shmctl()
	 */
	int		shm_rmid_forced;

	struct notifier_block ipcns_nb;

	/* The kern_mount of the mqueuefs sb.  We take a ref on it */
	struct vfsmount	*mq_mnt;

	/* # queues in this ns, protected by mq_lock */
	unsigned int    mq_queues_count;

	/* next fields are set through sysctl */
	unsigned int    mq_queues_max;   /* initialized to DFLT_QUEUESMAX */
	unsigned int    mq_msg_max;      /* initialized to DFLT_MSGMAX */
	unsigned int    mq_msgsize_max;  /* initialized to DFLT_MSGSIZEMAX */
	unsigned int    mq_msg_default;
	unsigned int    mq_msgsize_default;

	/* user_ns which owns the ipc ns */
	struct user_namespace *user_ns;
	struct ucounts *ucounts;

	struct llist_node mnt_llist;

	struct ns_common ns;
} __randomize_layout;

// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/util.h#L124
#define IPC_SEM_IDS	0
#define IPC_MSG_IDS	1
#define IPC_SHM_IDS	2

// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L169
#define sem_ids(ns)	((ns)->ids[IPC_SEM_IDS])
// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/msg.c#L99
#define msg_ids(ns)	((ns)->ids[IPC_MSG_IDS])
// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/shm.c#L86
#define shm_ids(ns)	((ns)->ids[IPC_SHM_IDS])
```

根据代码中的定义，第 0 项用于信号量，第 1 项用于消息队列，第 2 项用于共享内存，分别可以通过 sem_ids、msg_ids、shm_ids 来访问.

这段代码里面有 ns，全称叫 namespace. 可能不容易理解，可以将它认为是将一台 Linux 服务器逻辑的隔离为多台 Linux 服务器的机制. 现在，就可以简单的认为没有 namespace，整个 Linux 在一个 namespace 下面，那这些 ids 也是整个 Linux 只有一份. 接下来，再来看 ipc_namespace 的 [struct ipc_ids](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/ipc_namespace.h#L16) 里面保存了什么.

首先，in_use 表示当前有多少个 ipc；其次，seq 和 next_id 用于一起生成 ipc 唯一的 id，因为信号量，共享内存，消息队列，它们三个的 id 也不能重复；ipcs_idr 是一棵基数树，一旦涉及从一个整数查找一个对象，它都是最好的选择.

也就是说，对于 sem_ids、msg_ids、shm_ids 各有一棵基数树. 那这棵树里面究竟存放了什么，能够统一管理这三类 ipc 对象呢？通过函数 [ipc_obtain_object_idr](https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/util.c#L593)，可以看出端倪. 这个函数根据 id，在基数树里面找出来的是 [struct kern_ipc_perm](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/ipc.h#L12).

如果看用于表示信号量、消息队列、共享内存的结构，就会发现，这三个结构的第一项都是 struct kern_ipc_perm.

```c
//
/* One sem_array data structure for each set of semaphores in the system. */
struct sem_array {
	struct kern_ipc_perm	sem_perm;	/* permissions .. see ipc.h */
	time64_t		sem_ctime;	/* create/last semctl() time */
	struct list_head	pending_alter;	/* pending operations */
						/* that alter the array */
	struct list_head	pending_const;	/* pending complex operations */
						/* that do not alter semvals */
	struct list_head	list_id;	/* undo requests on this array */
	int			sem_nsems;	/* no. of semaphores in array */
	int			complex_count;	/* pending complex operations */
	unsigned int		use_global_lock;/* >0: global lock required */

	struct sem		sems[];
} __randomize_layout;

// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/msg.c#L48
/* one msq_queue structure for each present queue on the system */
struct msg_queue {
	struct kern_ipc_perm q_perm;
	time64_t q_stime;		/* last msgsnd time */
	time64_t q_rtime;		/* last msgrcv time */
	time64_t q_ctime;		/* last change time */
	unsigned long q_cbytes;		/* current number of bytes on queue */
	unsigned long q_qnum;		/* number of messages in queue */
	unsigned long q_qbytes;		/* max number of bytes on queue */
	struct pid *q_lspid;		/* pid of last msgsnd */
	struct pid *q_lrpid;		/* last receive pid */

	struct list_head q_messages;
	struct list_head q_receivers;
	struct list_head q_senders;
} __randomize_layout;

// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/shm.c#L52
struct shmid_kernel /* private to the kernel */
{
	struct kern_ipc_perm	shm_perm;
	struct file		*shm_file;
	unsigned long		shm_nattch;
	unsigned long		shm_segsz;
	time64_t		shm_atim;
	time64_t		shm_dtim;
	time64_t		shm_ctim;
	struct pid		*shm_cprid;
	struct pid		*shm_lprid;
	struct user_struct	*mlock_user;

	/* The task created the shm object.  NULL if the task is dead. */
	struct task_struct	*shm_creator;
	struct list_head	shm_clist;	/* list by creator */
} __randomize_layout;

// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/sem.c#L475
/*
 * sem_lock_(check_) routines are called in the paths where the rwsem
 * is not held.
 *
 * The caller holds the RCU read lock.
 */
static inline struct sem_array *sem_obtain_object(struct ipc_namespace *ns, int id)
{
	struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&sem_ids(ns), id);

	if (IS_ERR(ipcp))
		return ERR_CAST(ipcp);

	return container_of(ipcp, struct sem_array, sem_perm);
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/msg.c#L101
static inline struct msg_queue *msq_obtain_object(struct ipc_namespace *ns, int id)
{
	struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&msg_ids(ns), id);

	if (IS_ERR(ipcp))
		return ERR_CAST(ipcp);

	return container_of(ipcp, struct msg_queue, q_perm);
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/ipc/shm.c#L156
static inline struct shmid_kernel *shm_obtain_object(struct ipc_namespace *ns, int id)
{
	struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&shm_ids(ns), id);

	if (IS_ERR(ipcp))
		return ERR_CAST(ipcp);

	return container_of(ipcp, struct shmid_kernel, shm_perm);
}
```

也就是说，完全可以通过 struct kern_ipc_perm 的指针，通过进行强制类型转换后，得到整个结构. 做这件事情的函数是xxx_obtain_object.

通过这种机制，就可以将信号量、消息队列、共享内存抽象为 ipc 类型进行统一处理. 这有点儿面向对象编程中抽象类和实现类的意思. C++ 中类的实现机制，其实也是这么干的.

![](/misc/img/process/082b742753d862cfeae520fb02aa41af.png)