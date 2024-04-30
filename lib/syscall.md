# syscall
参考:
- [Linux 系统调用权威指南(2016)](http://arthurchiao.art/blog/system-call-definitive-guide-zh/) 翻译自[The Definitive Guide to Linux System Calls](https://blog.packagecloud.io/eng/2016/04/05/the-definitive-guide-to-linux-system-calls/)
- [Linux系统分析实验（二）：Linux内核5.0系统调用处理过程](https://www.zybuluo.com/windmelon/note/1428811)
- [为 Linux 添加系统调用](https://blog.gloriousdays.pw/2017/11/25/add-linux-system-call/)
- [Searchable Linux Syscall Table for x86 and x86_64](https://filippo.io/linux-syscall-table/)或通过`man 2 syscalls`查看当前os支持的syscalls

OS向程序提供的内核服务接口, 调用过程: 用户态 - 系统调用 - 保存寄存器 - 内核态执行系统调用 - 恢复寄存器 - 返回用户态，然后接着运行.

最大目的: 屏蔽硬件层, 比如open()不需要知道文件具体在磁盘的哪个扇区.

> 系统调用的组成是固定的,每个系统调用都由一个唯一的数字来标识.
> 调用系统调用的优选方式是使用VDSO，VDSO是映射在每个进程地址空间中的存储器的一部分，其允许更有效地使用系统调用. int 0x80是一种调用系统调用的传统方法，应该避免.
> 在 Linux 上,系统调用syscall遵循的惯例是调用成功则返回非负值. 发生错误时,例程会对相应 errno 常量取反,返回一负值.

其实对于 Linux 内核, 应用程序会调用库函数，在库函数中调用 API 入口函数，触发中断进入 Linux 内核执行系统调用，完成相应的功能服务.

在 Linux 内核之上，使用最广泛的 C 库是 glibc，其中包括 C 标准库的实现，也包括所有和系统 API 对应的库接口函数。几乎所有 C 程序都要调用 glibc 的库函数，所以 glibc 是 Linux 内核上 C 程序运行的基础.

glibc 是 Linux 下使用的开源的标准 C 库即(libc). 它为程序员提供丰富的API, 除了例如字符串处理、数学运算等用户态服务之外, 最重要的是封装了系统调用以便于使用(每个api至少封装了一个syscall). 通常使用strace命令来跟踪进程执行时系统调用和所接收的信号.

系统调用在内核中的实现函数要有一个声明, 在[`include/linux/syscalls.h`](https://elixir.bootlin.com/linux/latest/source/include/linux/syscalls.h)里. 真正的实现这个系统调用，一般在一个.c 文件里面，例如 sys_open 的实现在 [fs/open.c](https://elixir.bootlin.com/linux/latest/source/fs/open.c#L1168) 里面.

> 查看glic版本: `$ /lib/x86_64-linux-gnu/libc-2.23.so`

在编译的过程中，根据 syscall_32.tbl 和 syscall_64.tbl 生成自己的 syscalls_32.h 和 syscalls_64.h 文件. 生成方式在 arch/x86/entry/syscalls/Makefile 文件中, 这里面会使用两个脚本，即 syscallhdr.sh、syscalltbl.sh，它们最终生成的 syscalls_32.h 和 syscalls_64.h 两个文件中就保存了系统调用号和系统调用实现函数之间的对应关系.

Linux 内核一共有 `__NR_syscall_max` 个系统调用, 而系统调用号从 0 开始到 `__NR_syscall_max` 结束.

Linux 内核有 400 多个系统调用，它使用了一个函数指针数组，存放所有的系统调用函数的地址，通过数组下标就能索引到相应的系统调用。这个数组叫 sys_call_table，即 Linux 系统调用表.

定义一个系统调用函数，需要使用专门的宏. 根据参数不同选用不同的宏，对于无参数的系统调用函数，应该使用 SYSCALL_DEFINE0 宏来定义.

添加syscall demo见[这里](https://time.geekbang.org/column/article/407343).

### 32 bit syscall
参考:
- [趣谈Linux操作系统.09.32 位系统调用过程](https://time.geekbang.org/column/article/90394)

传参要求: glibc/sysdeps/unix/sysv/linux/i386/sysdep.h

![](/misc/img/lib/566299fe7411161bae25b62e7fe20506.jpg)

> [linux x86的系统调用](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/syscalls/syscall_32.tbl)

### 64 bit syscall

传参要求: glibc/sysdeps/unix/sysv/linux/x86_64/sysdep.h

在64 bit上, 用syscall指令取代了`int $0x80`. syscall 指令还使用了一种特殊的寄存器，叫特殊模块寄存器（Model Specific Registers，简称 MSR）. 这种寄存器是 CPU 为了完成某些特殊控制功能为目的的寄存器，其中就包括系统调用.

在系统初始化的时候，trap_init() 除了初始化中断模式，还会调用 cpu_init->syscall_init来初始化syscall所需的MSR.

当 syscall 指令调用的时候，会从MSR寄存器里面拿出函数地址来调用，也就是调用entry_SYSCALL_64即`arch/x86/entry/entry_64.S`的`SYM_CODE_START(entry_SYSCALL_64)`->[do_syscall_64](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/common.c#L283).


在 do_syscall_64 里面，从 rax 里面拿出系统调用号，然后根据系统调用号，在系统调用表 sys_call_table 中找到相应的函数进行调用，并将寄存器中保存的参数取出来，作为函数参数.

> 无论是 32 位，还是 64 位，都会到系统调用表 sys_call_table 这里来.

64 位的系统调用返回的时候，执行的是 [USERGS_SYSRET64](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/irqflags.h#L147), 即返回用户态的指令变成了 sysretq.

![](/misc/img/lib/1fc62ab8406c218de6e0b8c7e01fdbd7.jpg)

> [linux x86_64的系统调用](https://elixir.bootlin.com/linux/latest/source/arch/x86/entry/syscalls/syscall_64.tbl): 格式: `系统调用号:?:系统调用的名字:系统调用在内核的实现函数(以 sys_ 开头)`

### 关联syscall及其实现
在编译的过程中，需要根据 syscall_32.tbl 和 syscall_64.tbl 生成自己的 unistd_32.h 和 unistd_64.h, 生成方式在 arch/x86/entry/syscalls/Makefile 中. 该过程会使用两个脚本，其中 arch/x86/entry/syscalls/syscallhdr.sh，会在文件中生成 #define __NR_open；而arch/x86/entry/syscalls/syscalltbl.sh，会在文件中生成 __SYSCALL(__NR_open, sys_open). 这样，unistd_32.h 和 unistd_64.h 是对应的系统调用号和系统调用实现函数之间的对应关系. 在文件 `arch/x86/entry/syscall_32.c|arch/x86/entry/syscall_64.c`，定义了这样一个表，里面 include 了这个头文件，从而所有的 sys_ 系统调用都在这个表里面了.

## fork
![](/misc/img/5uugf8fxqg.png)

fork使用了copy-on-write. 它的实际开销是复制父进程的页表以及给子进程创建进程描述符.
linux通过clone()系统调用来实现fork(). 其可通过一系列参数标志来指明父子进程需要共享的资源. fork(), vfork(), __clone()都是根据自身需要的参数来调用clone(), clone()->调用`do_fork()`(定义在`kernel/fork.c`中)->调用copy_process()

copy_process()执行内容:
1. 调用dup_task_struct()为新进程创建一个内核栈, thread_info, task_struct, 此时父子进程的描述符完全相同.
1. 检查并确保创建的子进程没有超出当前用户的进程数量限制
1. 子进程开始区分自己和父进程, 即重置进程描述符.
1. 设置TASK_UNINTERRUPTIBLE, 保证不会运行
1. call copy_flags(), 更新 task_struct的flags成员. 表明进程是否拥有超级用户权限的PF_SUPERPRIV标志被清0, 表明进程未call exec()的PF_FORKNOEXEC()标志被设置
1. call alloc_pid()分配pid
1. 根据传递给clone的参数标志, copy_process()拷贝或共享打开的文件, 文件系统信息, 信号处理函数, 进程地址空间和命名空间. 在一般情况下, 这些资源会被给定进程的所有线程共享(因为创建的是线程???); 否则, 这些资源对每个进程是不同的, 因此会被拷贝到这里.
1. 扫尾并返回一个指向子进程的指针

最后, 新成功创建的子进程会被优先执行, 因为一般子进程会立即call exec(), 以避免写时拷贝的额外开销. 因为父进程先执行, 有可能会向地址空间写入.

除了不拷贝父进程的页表项外，vfork()与fork()相同.

> fork会拷贝父进程的数据段, 而vfork在子程序没有调用exec和exit前, 子进程和父进程共享数据段. fork不限制父子进程的执行顺序; vfork则子进程先运行,此时父进程挂起, 直到子进程调用了exec或exit后，才不再限制父子进程的执行顺序.

### fork
是POSIX中创建**进程**的唯一方法.

> windows的CreateProcess不同于`fork`, 它会load真正的程序.
> Linux kernel 并**不提供直接创建新进程的系统调用**,剩下的所有进程都是 init 进程通过**fork**机制建立的

在linux中, 新进程(即子进程, child process)由老进程(即父进程, parent process)fork而来.

子进程从父进程处继承数据段、栈段、堆段的副本后,可以修改这些内容,不会影响父进程的“原版”内容(在内
存中被标记为只读的程序文本段则由父、子进程共享). 然后,子进程要么去执行与父进程共享代码段中的另一组不同函数,或者,更为常见的
情况是使用系统调用 execve()去加载并执行一个全新程序. execve()会销毁现有的文本段、数
据段、栈段及堆段,并根据新程序的代码,创建新段来替换它们.

> 当程序调用fork（）函数时，系统会创建新的进程，为其分配资源，例如存储数据和代码的空间, 然后把原来的进程的所有值都复制到新的进程中，只有少量数值与原来的进程值不同，相当于克隆了一个自己.
> 孤儿进程的检查方法是自己通过调用 `getppid()`(取得父进程的pid) 并判断是否返回 1.
> fork创建的子进程不会与父进程共享内存, 不继承父进程的定时器.

fork()的神奇之处在于它仅仅被调用一次，却能够返回两次（父进程与子进程各返回一次）, 通过返回值的不同就可以进行区分父进程与子进程. 它可能有三种不同的返回值：
- 在父进程中，fork返回新创建子进程的进程ID
- 在子进程中，fork返回0
- 如果出现错误，fork返回一个负值

fork后父子进程从相同的位置开始执行, 有了返回值再通过if-else 语句判断,如果是父进程,还接着做原来应该做的事情;如果是子进程就调用另一个系统调用execve来执行另一个程序,这个时候,子进程和父进程就分开各自执行自己的逻辑了.

有时候,父进程要留意子进程的运行情况, 通过系统调用waitpid即可,父进程将子进程的pid作为参数传给它,这样父进程就知道子进程运行结束与否了.

> 现代 UNIX 采用写时复制技术来实现 fork(),其效率较之于早期的 fork()实现要高出许多,进而将对 vfork()的需求剔除殆尽.
> 不应对 fork()之后执行父、子进程的特定顺序做任何假设.

二者的区别,仅在于传递给_do_fork的clone_flags参数不同,vfork多了CLONE_VFORK|CLONE_VM标志. CLONE_VFORK意味着调用vfork创建的新进程会在当前进程之前`运行`,而fork不保证执行顺序.

CLONE_VM意味着新进程与当前进程共享mm_struct,而fork会复制当前进程的内存信息(页表、mmap等),这意味着调用vfork创建
进程需要的代价更小.

vfork的设计意图是新进程并不依赖当前进程的内存空间,它一般会直接执行exec或者exit,在执行exec之前,一直运行在当前进程的内存空间中,不当的使用会有意想不到的问题.

如果CLONE_PARENT标志被置位,当前进程并不一定是新进程的父进程。fork和vfork都没有置位该标志,所以调用它们创建的新进程确
实是子进程.

## wait
阻塞直至有子进程终止并获取到其返回值(`exit()`或`main return`). 该调用可以消灭僵尸进程.

waitpid与wait类似, 但不会阻塞.

## clone
在 Linux 下，通过 pthread_create 创建一个线程.

![](/misc/img/qvgisc7uyy.png)

## exec
fork之后执行, 改变进程正在执行的程序正文, 重新创建地址空间、栈、数据段以及堆, 最后开始执行该新程序, 即旧程序被完全替代了.

> 系统只要检查文件的开头两个字节即可判断是否为可执行文件.
> execve()如果检测到传入的文件以两字节序列`#!`开始, 就会通过解析器程序执行该文件.
> execlp和execvp与execl和execv类似, 但它们通过检索shell的PATH环境变量来获取路径.

## exit
停止并清理进程, 比如关闭所有已打开的文件. 它会释放进程的大部分数据结构并向父进程发送SIGCHLD信号, 直到父进程通过`wait()`得知子进程已终止, 由父进程移除子进程的所有数据结构, 并释放进程描述符.

库函数 exit()位于系统调用_exit()之上.

在调用 fork()之后,父、子进程中一般只有一个会通过调用 exit()退出,而另一进程则应使用_exit()终止.

exit()会执行的动作如下:
1. 调用退出处理程序(通过 atexit()和 on_exit()注册的函数),其执行顺序与注册顺序相反
1. 刷新 stdio 缓冲区.
1. 使用由 status 提供的值执行_exit()系统调用

## wait
实现进程同步的方法. `wait()`会暂停父进程, 直至(第一个)子进程执行完成后, 等待的父进程才会继续执行.

wait的参数`非NULL`时是指向子进程退出时的状态信息.

> 子进程终止时会产生SIGCHLD信号.

## waitpid
系统调用 wait()存在诸多限制,而设计 waitpid()则意在突破这些限制:
- 如果父进程已经创建了多个子进程,使用 wait()将无法等待某个特定子进程的完成,只能按顺序等待下一个子进程的终止
- 如果没有子进程退出,wait()总是保持阻塞. 有时候会希望执行非阻塞的等待:是否有子进程退出,立判可知.
- 使用 wait()只能发现那些已经终止的子进程. 对于子进程因某个信号(如 SIGSTOP 或SIG TTIN)而停止,或是已停止子进程收到 SIGCONT 信号后恢复执行的情况就无能为力了.

参数 pid 用来表示需要等待的具体子进程,意义如下:
- pid 大于 0,表示等待进程 ID 为 pid 的子进程
- pid 等于 0,则等待与调用进程(父进程)同一个进程组(process group)的所有子进程
- pid 小于-1,则会等待进程组标识符与 pid 绝对值相等的所有子进程
- pid 等于-1,则等待任意子进程. wait(&status)的调用与 waitpid(-1, &status, 0)等价

## setpgrp
将目前进程所属的组识别码设为目前进程的进程识别码. 此函数相当于调用`setpgid(0,0)`.

该函数的调用者会把自己加入一个新的process group, 并且process group leader就是自己, 即process group id就是自己的进程号.

当某个用户退出系统时,则相应的 shell 进程所启动的全部进程都要被强行终止, 系统是根据进程的组标识符来选定应该终止的进程的.

## getenv/putenv
- getenv : 获取环境变量
- putenv : 设置进程的环境变量

## abot
向调用进程发送一个信号, 产生一个非正常终止即有core dump.

## kill
参数pid:
- `0` : 当前进程所在的进程组的所有进程
- `-1` : 信号按pid从大到小的顺序发送给全部的进程(有权限限制)
- `< -1` : pid绝对值的进程组里所有进程

## alarm
按给定的秒数设置报警时钟, 到点后向进程发送`SIGALRM`信号. 重复调用会覆盖之前设置.

## setjmp/longjmp
跳转执行位置.

setjmp会保存程序的当前位置(通过保存堆栈环境来实行), 到时要返回.
longjmp不返回.

## dup2
复制一个打开的文件描述符 oldfd,并返回一个新描述符(用newfd参数指定新描述符的数值, 原有的newfd会被关闭),二者都指向同一打开的文件句柄.

与`close`和`dup`的联动操作类似, 但保证了操作的独立性和完整性,不会被外来信号打断.

dup2()系统调用会为 oldfd 参数所指定的文件描述符创建副本,其编号由 newfd 参数指定. 如果由 newfd 参数所指定编号的文件描述符之前已经打开, 那么 dup2()会首先将其关闭.
dup2()调用会默然忽略 newfd 关闭期间出现的任何错误故此, 编码时更为安全的做法是:在调用dup2()之前,若 newfd 已经打开,则应显式调用 close()将其关闭.

> dup3()系统调用完成的工作与 dup2()相同,只是新增了一个附加参数 flag, 该flag仅支持O_CLOEXEC(让内核为新文件描述符设置 close-on-exec标志(FD_CLOEXEC).
> fcntl(oldfd, F_DUPFD, startfd)与dup()和 dup2()功能类似.

## open
打开文件, 返回文件描述符

flags:
- O_CREAT : 必要时创建文件
- O_DIRECT : 使用Direct I/O
- O_TRUNC : 清空文件
- O_APPEND : 已追加方式打开文件
- O_RDONLY : 只读打开
- O_WRONLY : 只写打开
- O_RDWR : 读写打开
- O_RSYNC : 与 O_SYNC 标志或 O_DSYNC 标志配合一起使用的,将这些标志对写操作的作用结合到读操作中(在读操作前完成写操作).
- O_DSYNC : 要求写操作按照 synchronized I/O data integrity completion 来执行(类似于fdatasync())
- O_SYNC : 同步打开, 遵从 synchronized I/O file integrity completion(类似于 fsync()函数), 但对性能的影响极大.
- O_NONBLOCK : 非阻塞打开
	
	若 open()调用未能立即打开文件,则返回错误,而非陷入阻塞. 有一种情况属于例外,调用 open()操作 FIFO 可能会陷入阻塞.
	调用 open()成功后,后续的 I/O 操作也是非阻塞的. 若 I/O 系统调用未能立即完成,则可能会只传输部分数据,或者系统调用失败,并返回 EAGAIN 或 EWOULDBLOCK 错误.
	管道、 FIFO、套接字、设备(比如终端、伪终端)都支持非阻塞模式.(因为无法通过 open()来获取管道和套接字的文件描述符,所以要启用非阻塞标志,就必须fcntl()的F_SETFL)
	由于内核缓冲区保证了普通文件 I/O 不会陷入阻塞,故而打开普通文件时一般会忽略 O_NONBLOCK 标志. 然而,当使用强制文件锁时, O_NONBLOCK标志对普通文件也是起作用的.

在glibc/intl/loadmsgcat.c, open 只是宏，实际工作的是 `__open_nocancel` 函数，其中会用 INLINE_SYSCALL_CALL 宏经过一系列替换，最终根据参数的个数替换成相应的 `internal_syscall##nr 宏`.

## sync()
使包含更新文件信息的所有内核缓冲区(即数据块、指针块、元数据等)刷新到磁盘上. 在 Linux 实现中,sync()调用仅在所有数据已传递到磁盘上(或者至少高速缓存)时返回.

## fsync()
将使缓冲数据和与打开文件描述符 fd 相关的所有元数据都刷新到磁盘上. 调用 fsync()会强制使文件处于 [Synchronized I/O file integrity completion](/io/io.md) 状态.

## fdatasync()
类似于fsync(), 只是强制文件处于 synchronized I/O dataintegrity completion 的状态.

## posix_fadvise()
调用允许进程就自身访问文件数据时可能采取的模式通知内核:
- POSIX_FADV_NORMAL

	无特别建议(默认行为). 在 Linux 中,该操作将文件预读窗口大小置为默认值(128KB).
- POSIX_FADV_SEQUENTIAL

	进程预计会从低偏移量到高偏移量顺序读取数据. 在 Linux 中,该操作将文件预读窗口大小置为默认值的两倍.
- POSIX_FADV_RANDOM

	进程预计以随机顺序访问数据. 在 Linux 中,该选项会禁用文件预读.
- POSIX_FADV_WILLNEED

	进程预计会在不久的将来访问指定的文件区域. 内核将由 offset 和 len 指定区域的文件数据预先填充到缓冲区高速缓存中. 后续对该文件的 read()调用将不会阻塞磁盘 I/O,只需从缓冲区高速缓存中抓取数据即可.
- POSIX_FADV_DONTNEED

	进程预计在不久的将来将不会访问指定的文件区域, 这一操作给内核的建议是释放相关的高速缓存页面(如果存在的话). 在 Linux 中会先尝试保存变更的页面, 成功后kernel才尝试释放, 因此	变通的方法之一是在 POSIX_FADV_DONTNEED 操作之前对指定的参数 fd 调用 sync()或 fdatasync().
- POSIX_FADV_NOREUSE

	进程预计会一次性地访问指定文件区域,不再复用. 这等于提示内核对指定区域访问一次后即可释放页面. 在 Linux 中,该操作目前不起作用.

## sigaction
用于处理信号, 替代`signal()`, 更稳定.

## fcntl
对一个打开的文件描述符执行一系列控制操作.

fcntl()的 F_SETFL 命令来修改打开文件的某些状态标志, 允许更改的标志有
O_APPEND、O_NONBLOCK、O_NOATIME、O_ASYNC 和 O_DIRECT. 系统将忽略对其他
标志的修改操作. 但有些其他的 UNIX 实现允许 fcntl()修改其他标志,如 O_SYNC.

## pread()和 pwrite()
完成与 read()和 write()相类似的工作,只是前两者会在 offset 参数所指定的位置进行文件 I/O 操作,而非始于文件的当前偏移量处,且它们不会改变文件的当前偏移量.

对 pread()和 pwrite()而言,fd 所指代的文件必须是可定位的(即允许对文件描述符执行lseek()调用).

它们对多线程应用有为有用.

## lseek
`lseek(fd, -5, SEEK_CUR)` # 将文件指针从当前位置前移5个B

## preadv()和 pwritev()

所执行的任务与 readv()和 writev()相同,但执行 I/O 的位置将由 offset 参数指定(类似于 pread()和 pwrite()系统调用).

## truncate()和 ftruncate()
若文件当前长度大于参数 length,调用将丢弃超出部分,若小于参数 length,在linux上调用将在文件尾部添加一系列空字节.

ftruncate()不会修改文件偏移量.

## mkstemp()和 tmpfile()
mkstemp()函数基于模板(模板参数采用路径名形式,其中最后 6 个字符必须为 XXXXXX, 这 6 个字符将被替换, 以保证文件名的唯一性,且修改后的字符串将通过 template 参数传回)生成一个唯一文件名并打开该文件,返回一个可用于 I/O 调用的文件描述符.

文件拥有者对 mkstemp()函数建立的文件拥有读写权限(其他用户则没有任何操作权限),且打开文件时使用了 O_EXCL 标志,以保证调用者以独占方式访问文件.

通常,打开临时文件不久,程序就会使用 unlink 系统调用将其删除.

tmpfile()函数会创建一个名称唯一的临时文件,并以读写方式将其打开(打开该文件时使用了 O_EXCL 标志,以防一个可能性极小的冲突).
tmpfile()函数执行成功,将返回一个文件流供 stdio 库函数使用。文件流关闭后将自动删除临时文件.

## pathconf()和 fpathconf()
pathconf()和 fpathconf()之间唯一的区别在于对文件或目录的指定方式. pathconf()采用路
径名方式来指定,而 fpathconf()则使用(之前已经打开的)文件描述符.

## stat()、lstat()以及 fstat()
获取与文件有关的信息.

lstat()所返回的信息针对的是符号链接自身(而非符号链接所指向的文件).

系统调用stat()和lstat()无需对其所操作的文件本身拥有任何权限,但针对指定pathname 的父目录要有执行(搜索)权限.

## utimensat()系统调用 和 futimens()库函数
纳秒级精度设置时间戳.

## sigaction()
创建信号处理器

## sigprocmask
从信号掩码中添加或者移除信号

## sigpending()
获取等待信号集(用以描述多个不同信号的数据结构)

## kill()
参数pid:
- pid 大于 0,那么会发送信号给由 pid 指定的进程
- pid 等于 0,那么会发送信号给与调用进程同组(同一进程组)的每个进程,包括调用进程自身
- pid 小于−1,那么会向组 ID 等于该 pid 绝对值的进程组内所有下属进程发送
信号.
- pid 等于−1,那么信号的发送范围是:调用进程有权将信号发往的每个目标进程, 除去 init(进程 ID 为 1)和调用进程自身. 如果特权级进程发起这一调用,那么会发送信号给系统中的所有进程,上述两个进程除外. 显而易见,有时也将这种信号发送方式称之为广播信号.

kill()将参数 sig 指定为 0(即所谓空信号),则无信号发送. 相反,kill()仅会去执行错误检查,查看是否可以向目标进程发送信号, 即检查目标进程是否存在.

## raise
进程需要向自身发送信号

调用 raise()相当于:
- 单线程程序, `kill(gitpid(),sig)`
- 支持线程的程序　: `pthread_kill(pthread_self(),sig)`

## abort
终止其调用进程,并生成核心转储. 避免进程对栈的扩展突破了对栈大小的限制时, 无法处理signal.

SUSv3 规定将常量 SIGSTKSZ作为划分备选栈大小的典型值,而将 MINSSIGSTKSZ 作为调用信号处理器函数所需的最小值.
在 Linux/x86-32 系统上,分别将这两个值定义为 8192 和 2048.

## sigaltstack()
告之内核该备选信号栈的存在.

## sigwaitinfo()和 sigtimedwait()
同步等待一个信号. 这省去了对信号处理器的设计和编码工作.

## 限制值
参考[unix环境高级编程 - 2.5.4 函数sysconf、 pathconf 和fpathconf]

### 扩展
- [使用posix_spawn : 是时候淘汰对操作系统的 fork() 调用了](https://www.infoq.cn/article/BYGiWI-fxHTNvSohEUNW)

## 堆内存分配
### brk
需求大小较小

### mmap
需求大小较大

```c
// from man 2 mmap
#include <sys/mman.h>

       void *mmap(void *addr, size_t length, int prot, int flags,
                  int fd, off_t offset);
       int munmap(void *addr, size_t length);
```
mmap可将fd从偏移offset开始长度为length的一块映射到内存区域中, 从而把文件的某一段映射到进程的地址空间, 实现通过访问内存的方式去访问文件.

应用访问文件一般有两种方法:
1. mmap直接访问虚拟地址空间
1. read/write进行寻址访问

> mmap2()与mmap的区别是mmap2中文件的偏移以页为单位.

## time
### gettimeofday
获取当前时间, 以timeval(从epoch到现在的秒数)和timezone结构体的形式返回. 它通过vDSO实现.

```c
// https://elixir.bootlin.com/linux/v5.10.2/source/arch/x86/entry/vdso/vclock_gettime.c#L17
extern int __vdso_gettimeofday(struct __kernel_old_timeval *tv, struct timezone *tz);
extern __kernel_old_time_t __vdso_time(__kernel_old_time_t *t);

int __vdso_gettimeofday(struct __kernel_old_timeval *tv, struct timezone *tz)
{
	return __cvdso_gettimeofday(tv, tz);
}

int gettimeofday(struct __kernel_old_timeval *, struct timezone *)
	__attribute__((weak, alias("__vdso_gettimeofday")));

// https://elixir.bootlin.com/linux/v5.10.2/source/arch/x86/include/asm/vdso/gettimeofday.h#L82
static __always_inline
long gettimeofday_fallback(struct __kernel_old_timeval *_tv,
			   struct timezone *_tz)
{
	long ret;

	asm("syscall" : "=a" (ret) :
	    "0" (__NR_gettimeofday), "D" (_tv), "S" (_tz) : "memory");

	return ret;
}

// https://elixir.bootlin.com/linux/v5.10.2/source/kernel/time/time.c
SYSCALL_DEFINE2(gettimeofday, struct __kernel_old_timeval __user *, tv,
		struct timezone __user *, tz)
{
	if (likely(tv != NULL)) {
		struct timespec64 ts;

		ktime_get_real_ts64(&ts);
		if (put_user(ts.tv_sec, &tv->tv_sec) ||
		    put_user(ts.tv_nsec / 1000, &tv->tv_usec))
			return -EFAULT;
	}
	if (unlikely(tz != NULL)) {
		if (copy_to_user(tz, &sys_tz, sizeof(sys_tz)))
			return -EFAULT;
	}
	return 0;
}

// https://elixir.bootlin.com/linux/v5.10.2/source/kernel/time/timekeeping.c#L794
/**
 * ktime_get_real_ts64 - Returns the time of day in a timespec64.
 * @ts:		pointer to the timespec to be set
 *
 * Returns the time of day in a timespec64 (WARN if suspended).
 */
void ktime_get_real_ts64(struct timespec64 *ts)
{
	struct timekeeper *tk = &tk_core.timekeeper;
	unsigned int seq;
	u64 nsecs;

	WARN_ON(timekeeping_suspended);

	do {
		seq = read_seqcount_begin(&tk_core.seq);

		ts->tv_sec = tk->xtime_sec;
		nsecs = timekeeping_get_ns(&tk->tkr_mono);

	} while (read_seqcount_retry(&tk_core.seq, seq));

	ts->tv_nsec = 0;
	timespec64_add_ns(ts, nsecs);
}
EXPORT_SYMBOL(ktime_get_real_ts64);

// https://elixir.bootlin.com/linux/v5.10.2/source/kernel/time/timekeeping.c#L379
static inline u64 timekeeping_delta_to_ns(const struct tk_read_base *tkr, u64 delta)
{
	u64 nsec;

	nsec = delta * tkr->mult + tkr->xtime_nsec; // 当前时间=上次更新时间xtime_nsec+ (上次更新时间到现在的时钟周期数delta) * 时钟频率
	nsec >>= tkr->shift; // 各时钟源频率不同, kernel通过tkr->mult和tkr->shift提供时钟周期与纳秒的互转

	/* If arch requires, add in get_arch_timeoffset() */
	return nsec + arch_gettimeoffset();
}

static inline u64 timekeeping_get_ns(const struct tk_read_base *tkr)
{
	u64 delta;

	delta = timekeeping_get_delta(tkr);
	return timekeeping_delta_to_ns(tkr, delta);
}

// https://elixir.bootlin.com/linux/v5.10.2/source/lib/vdso/gettimeofday.c#L323
static __maybe_unused int
__cvdso_gettimeofday(struct __kernel_old_timeval *tv, struct timezone *tz)
{
	return __cvdso_gettimeofday_data(__arch_get_vdso_data(), tv, tz);
}

// https://elixir.bootlin.com/linux/v5.10.2/source/lib/vdso/gettimeofday.c#L295
static __maybe_unused int
__cvdso_gettimeofday_data(const struct vdso_data *vd,
			  struct __kernel_old_timeval *tv, struct timezone *tz)
{

	if (likely(tv != NULL)) {
		struct __kernel_timespec ts;

		if (do_hres(&vd[CS_HRES_COARSE], CLOCK_REALTIME, &ts))
			return gettimeofday_fallback(tv, tz); // __NR_gettimeofday兜底

		tv->tv_sec = ts.tv_sec;
		tv->tv_usec = (u32)ts.tv_nsec / NSEC_PER_USEC;
	}

	if (unlikely(tz != NULL)) {
		if (IS_ENABLED(CONFIG_TIME_NS) &&
		    vd->clock_mode == VDSO_CLOCKMODE_TIMENS)
			vd = __arch_get_timens_vdso_data();

		tz->tz_minuteswest = vd[CS_HRES_COARSE].tz_minuteswest;
		tz->tz_dsttime = vd[CS_HRES_COARSE].tz_dsttime;
	}

	return 0;
}
```