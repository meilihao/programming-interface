# syscall
参考:
 - [linux x64的系统调用](linux-5.2/arch/x86/entry/syscalls/syscall_64.tbl)

glibc 是 Linux 下使用的开源的标准 C 库即(libc). 它为程序员提供丰富的API, 除了例如字符串处理、数学运算等用户态服务之外, 最重要的是封装了系统调用以便于使用(每个api至少封装了一个syscall). 通常使用strace命令来跟踪进程执行时系统调用和所接收的信号.

## fork
创建进程

在linux中, 新进程(即子进程, child process)由老进程(即父进程, parent process)fork而来.

对于 fork 系统调用的返回值,如果当前进程是子进程,就返回 0;如果当前进程是父进程,就返回子进程的pid. 有了返回值再通过if-else 语句判断,如果是父进程,还接着做原来应该做的事情;如果是子进程就调用另一个系统调用execve来执行另一个程序,这个时候,子进程和父进程就分开各自执行自己的逻辑了.

有时候,父进程要留意子进程的运行情况, 通过系统调用waitpid即可,父进程将子进程的pid作为参数传给它,这样父进程就知道子进程运行结束与否了.
完了没有,成功与否

### 扩展
- [使用posix_spawn : 是时候淘汰对操作系统的 fork() 调用了](https://www.infoq.cn/article/BYGiWI-fxHTNvSohEUNW)