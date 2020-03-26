# syscall
OS向程序提供的内核服务接口.

最大目的: 屏蔽硬件层, 比如open()不需要知道文件具体在磁盘的哪个扇区.

参考:
 - [linux x86_64的系统调用](linux-5.2/arch/x86/entry/syscalls/syscall_64.tbl): 格式: `系统调用号:?:系统调用的名字:系统调用在内核的实现函数(以 sys_ 开头)`

> 系统调用的组成是固定的,每个系统调用都由一个唯一的数字来标识.
> 调用系统调用的优选方式是使用VDSO，VDSO是映射在每个进程地址空间中的存储器的一部分，其允许更有效地使用系统调用. int 0x80是一种调用系统调用的传统方法，应该避免.
> 在 Linux 上,系统调用syscall遵循的惯例是调用成功则返回非负值. 发生错误时,例程会对相应 errno 常量取反,返回一负值.

glibc 是 Linux 下使用的开源的标准 C 库即(libc). 它为程序员提供丰富的API, 除了例如字符串处理、数学运算等用户态服务之外, 最重要的是封装了系统调用以便于使用(每个api至少封装了一个syscall). 通常使用strace命令来跟踪进程执行时系统调用和所接收的信号.

> syscall的函数声明在`include/linux/syscalls.h`里
> [linux x86的系统调用](linux-5.2/arch/x86/entry/syscalls/syscall_32.tbl)
> 查看glic版本: `$ /lib/x86_64-linux-gnu/libc-2.23.so`

## fork
![](/misc/img/5uugf8fxqg.png)

### fork
是POSIX中创建进程的唯一方法.

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
复制一个打开的文件描述符 oldfd,并返回一个新描述符(用newfd参数指定新描述符的数值),二者都指向同一打开的文件句柄.

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

## gettimeofday()
获取当前时间.

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
