# syscall
OS向程序提供的内核服务接口.

参考:
 - [linux x86_64的系统调用](linux-5.2/arch/x86/entry/syscalls/syscall_64.tbl): 格式: `系统调用号:?:系统调用的名字:系统调用在内核的实现函数(以 sys_ 开头)`

glibc 是 Linux 下使用的开源的标准 C 库即(libc). 它为程序员提供丰富的API, 除了例如字符串处理、数学运算等用户态服务之外, 最重要的是封装了系统调用以便于使用(每个api至少封装了一个syscall). 通常使用strace命令来跟踪进程执行时系统调用和所接收的信号.

> syscall的函数声明在`include/linux/syscalls.h`里
> [linux x86的系统调用](linux-5.2/arch/x86/entry/syscalls/syscall_32.tbl)

## fork
![](/misc/img/5uugf8fxqg.png)

### fork
是POSIX中创建进程的唯一方法.

> windows的CreateProcess不同于`fork`, 它会load真正的程序.
> Linux kernel 并**不提供直接创建新进程的系统调用**,剩下的所有进程都是 init 进程通过**fork**机制建立的

在linux中, 新进程(即子进程, child process)由老进程(即父进程, parent process)fork而来.

> 当程序调用fork（）函数时，系统会创建新的进程，为其分配资源，例如存储数据和代码的空间, 然后把原来的进程的所有值都复制到新的进程中，只有少量数值与原来的进程值不同，相当于克隆了一个自己.
> 孤儿进程的检查方法是自己通过调用 `getppid()`(取得父进程的pid) 并判断是否返回 1.
> fork创建的子进程不会与父进程共享内存.

fork()的神奇之处在于它仅仅被调用一次，却能够返回两次（父进程与子进程各返回一次）, 通过返回值的不同就可以进行区分父进程与子进程. 它可能有三种不同的返回值：
- 在父进程中，fork返回新创建子进程的进程ID
- 在子进程中，fork返回0
- 如果出现错误，fork返回一个负值

fork后父子进程从相同的位置开始执行, 有了返回值再通过if-else 语句判断,如果是父进程,还接着做原来应该做的事情;如果是子进程就调用另一个系统调用execve来执行另一个程序,这个时候,子进程和父进程就分开各自执行自己的逻辑了.

有时候,父进程要留意子进程的运行情况, 通过系统调用waitpid即可,父进程将子进程的pid作为参数传给它,这样父进程就知道子进程运行结束与否了.

## wait
阻塞直至有子进程终止并获取到其返回值(`exit()`或`main return`). 该调用可以消灭僵尸进程.

waitpid与wait类似, 但不会阻塞.

## clone
在 Linux 下，通过 pthread_create 创建一个线程.

![](/misc/img/qvgisc7uyy.png)

## exec
fork之后执行, 改变进程正在执行的程序正文, 把内存段重置为预先定义的初始状态等, 最后开始执行该新程序, 即旧程序被完全替代了.

> 系统只要检查文件的开头两个字节即可判断是否为可执行文件.
> execlp和execvp与execl和execv类似, 但它们通过检索shell的PATH环境变量来获取路径.

## exit
停止并清理进程, 比如关闭所有已打开的文件. 它会释放进程的大部分数据结构并向父进程发送SIGCHLD信号, 直到父进程通过`wait()`得知子进程已终止, 由父进程移除子进程的所有数据结构, 并释放进程描述符. 

## wait
实现进程同步的方法. `wait()`会暂停父进程, 直至(第一个)子进程执行完成后, 等待的父进程才会继续执行.

wait的参数`非NULL`时是指向子进程退出时的状态信息.

> 子进程终止时会产生SIGCHLD信号.

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
与`close`和`dup`的联动操作类似, 但保证了操作的独立性和完整性,不会被外来信号打断.

## open
打开文件, 返回文件描述符

flags:
- O_CREAT : 必要时创建文件
- O_TRUNC : 清空文件
- O_APPEND : 已追加方式打开文件
- O_RDONLY : 只读打开
- O_WRONLY : 只写打开
- O_RDWR : 读写打开

## sigaction
用于处理信号, 替代`signal()`, 更稳定.

## 限制值
参考[unix环境高级编程 - 2.5.4 函数sysconf、 pathconf 和fpathconf]

### 扩展
- [使用posix_spawn : 是时候淘汰对操作系统的 fork() 调用了](https://www.infoq.cn/article/BYGiWI-fxHTNvSohEUNW)
