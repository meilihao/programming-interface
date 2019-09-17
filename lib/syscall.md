# syscall
OS向程序提供的内核服务接口.

参考:
 - [linux x86_64的系统调用](linux-5.2/arch/x86/entry/syscalls/syscall_64.tbl): 格式: `系统调用号:?:系统调用的名字:系统调用在内核的实现函数(以 sys_ 开头)`

glibc 是 Linux 下使用的开源的标准 C 库即(libc). 它为程序员提供丰富的API, 除了例如字符串处理、数学运算等用户态服务之外, 最重要的是封装了系统调用以便于使用(每个api至少封装了一个syscall). 通常使用strace命令来跟踪进程执行时系统调用和所接收的信号.

> syscall的函数声明在`include/linux/syscalls.h`里
> [linux x86的系统调用](linux-5.2/arch/x86/entry/syscalls/syscall_32.tbl)

## fork
![](/misc/img/5uugf8fxqg.png)

创建进程

在linux中, 新进程(即子进程, child process)由老进程(即父进程, parent process)fork而来.

> 当程序调用fork（）函数时，系统会创建新的进程，为其分配资源，例如存储数据和代码的空间, 然后把原来的进程的所有值都复制到新的进程中，只有少量数值与原来的进程值不同，相当于克隆了一个自己.
> 孤儿进程的检查方法是自己通过调用 `getppid()`(取得父进程的pid) 并判断是否返回 1.

fork()的神奇之处在于它仅仅被调用一次，却能够返回两次（父进程与子进程各返回一次）, 通过返回值的不同就可以进行区分父进程与子进程. 它可能有三种不同的返回值：
- 在父进程中，fork返回新创建子进程的进程ID
- 在子进程中，fork返回0
- 如果出现错误，fork返回一个负值

有了返回值再通过if-else 语句判断,如果是父进程,还接着做原来应该做的事情;如果是子进程就调用另一个系统调用execve来执行另一个程序,这个时候,子进程和父进程就分开各自执行自己的逻辑了.

有时候,父进程要留意子进程的运行情况, 通过系统调用waitpid即可,父进程将子进程的pid作为参数传给它,这样父进程就知道子进程运行结束与否了.

## clone
在 Linux 下，通过 pthread_create 创建一个线程.

![](/misc/img/qvgisc7uyy.png)

## exec
fork之后执行, 改变进程正在执行的程序正文, 把内存段重置为预先定义的初始状态等, 最后开始执行该新程序.

## 限制值
参考[unix环境高级编程 - 2.5.4 函数sysconf、 pathconf 和fpathconf]

### 扩展
- [使用posix_spawn : 是时候淘汰对操作系统的 fork() 调用了](https://www.infoq.cn/article/BYGiWI-fxHTNvSohEUNW)