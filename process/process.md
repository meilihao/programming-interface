# process

## 程序和进程
程序(program)是存储上的可执行文件. 进程(process)是程序在内存执行时的实体.

在类unix中, 每个进程都有一个唯一的数字标识符，称为进程ID（process ID）,也叫pid, 进程ID是一个非负整数.

进程可认为是一个地址空间和内核描述该进程的一组数据结构共同组成.

先看看linux正在运行的进程:
```sh
$ps -eo pid,comm,args
```

> `comm`是进程的简称(即执行程序的名称);`args`是进程所对应的程序(可能含有路径)以及运行时所带的参数

也可使用`pstree`命令来展示当前的进程树.

## 进程控制
有三个用于进程控制的主要函数： fork, exec和waitpid, 其中exec函数有7种变体，但经常统称它们为exec函数.

### fork
Linux kernel 并**不提供直接创建新进程的系统调用**,剩下的所有进程都是 init 进程通过**fork**机制建立的.新的进程要通过老的进程复制自身得到，这就是`fork`.

在c语言中, 父进程调用`fork()`产生(spawn)子进程并得到子进程pid后, 子进程会复制父进程的一切, 并同样从`fork()`处开始执行, 且子进程调用`fork()`时返回`0`.

## 进程和线程
尽管在UNIX中，进程与线程是有联系但不同的两个东西，但**在Linux中，线程只是一种特殊的进程**,多个线程之间可以共享内存空间和IO接口.所以，进程是Linux程序的唯一的实现方式.

通常, 一个进程只有一个线程(thread, 进程中一个**单一顺序**的控制流), 但也允许有多个线程, 这样可以进行多任务的同时处理和充分利用多核处理器的并行能力.

一个进程内的所有线程会共享同一地址空间, 文件描述符, 栈以及与进程相关的属性, 因此访问共享数据时需要同步措施来避免不一致性. 与进程相同, 线程也有ID, 但其仅在所属进程内有效.

## 进程组(process group)
一个或多个**进程**的集合.进程组会有一个进程组领导进程 (process group leader),有的地方也叫组长进程，领导进程的PID则成为进程组的ID (process group ID, PGID)，以识别进程组.
PGID不会因领导进程退出而受到影响，且fork调用也不会改变PGID.

> linux和unix的新进程由fork而来, 因此原远程的pid就是当前进程的ppid.

```sh
➜  ~ ps -o pid,pgid,ppid,args| cat
  PID  PGID  PPID COMMAND
15967 15967  4325 /usr/bin/zsh
16547 16547 15967 ps -o pid,pgid,ppid,args
16548 16547 15967 cat
```

## 会话(session)
一个或多个**进程组**的集合.新建会话时，当前进程（会话中唯一的进程）成为会话首进程(session leader)，也是当前进程组的组长进程，其进程号为该会话的SID(session id).它通常是登录 shell，也可以是调用`setsid`成功新建会话的孤儿进程.

会话的每个进程组称为一个工作(job).一个会话一般包含**一个会话首进程、至多一个前台进程组和任意个后台进程组，控制终端(由这些组共享)可有可无**.

前后台进场分工:
- 前台进程组 : 该进程组中的进程可以向终端设备进行读、写操作（属于该组的进程都可以从终端获得输入）.通常用该进程组的`PGID=TPGID`来判断前台进程组.
- 后台进程组 : 该进程组中的进程只能向终端设备进行写操作.

```sh
➜  ~ ps -o pid,pgid,ppid,sid,args,tty,tpgid
  PID  PGID  PPID   SID COMMAND                     TT       TPGID
 4326  4326  4321  4326 /usr/bin/zsh                pts/0     9125
 9119  9119  4326  4326 ping localhost -c 10        pts/0     9125
 9125  9125  4326  4326 ps -o pid,pgid,ppid,sid,arg pts/0     9125
```

**进程组领导进程不能成为新会话首进程，但新会话首进程必定成为进程组领导进程.**

一个命令可以通过在末尾加上&方式让它在后台运行:
```sh
➜  ~ ping localhost -c 4 & 
[1] 6954 # 1表示工作号,6954为PGID
```

结束工作:
```sh
$kill -SIGTERM -6954 #通过在PGID前面加-来表示是一个PGID而不是PID
```
或者
```sh
$kill -SIGTERM %1 #发送给工作1
```

查看后台工作:
```sh
$fg %1
```

将当前运行的程序放在后台:
```sh
<CTRL> + z
```

停止执行当前运行的程序:
```sh
<CTRL> + c
```

列出所有后台程序:
```sh
$ jobs
```

## 孤儿进程
**父进程先于子进程结束**，这时的子进程应该称作`孤儿进程（Orphan）`，它将被`1`号进程（即init 进程,通常是systemd）接管，init 进程成为其父进程.

## 僵尸进程(Zombie)
**子进程先于父进程结束**，但是其父进程尚未对其进行善后处理（获取终止子进程的有关信息、释放它仍占用的资源）的进程.

一个已经终止,但父进程没有函数调用`wait/waitpid`等待子进程结束，也没有注册`SIGCHLD`信号处理函数，结果使得子进程的进程列表信息无法回收，就变成了僵尸进程.

## 守护进程(daemon)
一个在后台运行并且不受任何终端控制的进程.

守护进程的特点： 
1. 自成进程组，自成会话，与控制终端脱关联
2. 守护进程的父进程是1号进程
3. 守护进程的命令一般以字符d结尾 
4. 守护进程的生命周期是7*24小时不掉线

### 创建守护进程为什么要fork两次
1. 第一次fork: 为了脱离终端控制的魔爪. 调用了fork后，子进程拷贝了父进程的会话期、进程组、控制终端等资源、虽然父进程退出了，但是这些资源并没有改变，因此，需要用`setsid`来使得该子进程完全独立出来，从而摆脱其他进程的控制.
2. 第二次fork(不是必须的): 子进程调用`setsid`后成立了会话首进程,有了打开控制终端的能力，再fork一次，孙子进程就不能打开控制终端了(没调用`setsid`,不是会话首进程).

## 守护进程与后台进程

通过&符号，可以把命令放到后台执行。它与守护进程是不同的：
- 守护进程与终端无关，是被init进程收养的孤儿进程；而后台进程的父进程是终端，仍然可以在终端打印
- 守护进程在关闭终端时依然坚挺；而后台进程会随用户退出而停止，除非加上nohup
- 守护进程改变了会话、进程组、工作目录和文件描述符，后台进程直接继承父进程（shell）的

> 换句话说：守护进程就是默默地奋斗打拼的有为青年，而后台进程是默默继承老爸资产的富二代.

## 查看进程的信息
```sh
ll /proc/$PID
```

output:
- cwd : 符号链接,指向进程执行目录
- exe : 符号连接,执行程序的绝对路径\
- cmdline : 程序运行时输入的命令行命令
- environ : 进程运行时的环境变量
- fd : 目录,进程打开或使用的文件的符号连接

## 进程时间
一个进程维护3个时间:
- real : 进程运行的总时间
- user : 用户态的cpu时间,即cpu执行用户指令所用时间
- sys : 内核态的cpu时间,即cpu执行内核程序所用时间

## 工作目录
进程可以用chdir函数更改其工作目录.

## 文件描述符(file descriptor)
文件描述符是一个小的非负整数, kenel用以标识一个特定进程正在访问的文件. 当内核打
开一个现有文件或创建一个新文件时, 它就返回一个文件描述符. 当读、写文件时，就可使
用它.

## 进程时间
即CPU时间, 用以度量进程使用的CPU资源，是用户CPU时间和系统CPU时间之和.

进程时间以时钟滴答计算, 通常每秒钟取为 50、60或100个滴答.

当度量一个进程的执行时间时(比如time命令), 类Unix系统会使用三个时间值:
- 时钟时间 : 进程运行的时间段
- 用户CPU时间 :　执行用户指令所用的时间量
- 系统CPU时间 :　为该进程执行内核指令所用的时间

> 时钟滴答可使用sysconf函数获取, example中有示例.

## 错误处理
在类unix系统中, 函数出错时通常会返回一个负值, 而且整型变量errno通常设置为具有特定信息的一个值.

文件`<errno.h>`中定义了变量errno以及赋与它的各种常量. 这些常量都以`E`开头.

在支持多线程的环境中, 每个线程都有属于自己的局部errno, 以避免相互干扰.

> Unix手册第 2部分的第 1页， intro(2) 列出了所有这些出错常量; 而linux是包含在errno(3)手册页中.

### 出错恢复
错误分为两类: 致命性和非致命性.

发生致命性错误后, 程序通常会输出错误信息并退出, 但也允许拦截该错误自行处理, 比如golang的`recover()`. 

常见的处理非致命错误的方法是: 等待再重试.

## stdin(0), stdout(1), stderr(2)
通常运行一个程序时, 所有的shell都为其打开三个文件描述符：标准输入、标
准输出以及标准出错. 如果不做特殊处理，则这三个描述符都连向终端. 大多数shell都会提供一种方法，使其中任何一个或所有这三个描述符都能重新定向到某一个文件.

> 记忆: 没有(0)要输入(in), 有(1)要输出(out), 其他(2)是错误(err)
> 非守护进程都有一个关联的控制终端, 进程的stdin, stdout, stderr都默认连接到该终端.

## 进程关联的用户id/组id
- 实际用户id/实际组id : 来自登录回话, 表示`我实际上是谁`
- 有效用户id/有效组id/附属组id : 决定了文件访问权限
- 保存的设置用户id/保存的设置组id : 有exec函数使用, 比如`passwd`命令

## /proc
ps和top均从`/proc`读取进程状态信息. 内核产生的所有状态信息和统计信息均在里面.

`/proc/${pid}`内容:
- cmdline : 进程的完整命令行(以null分隔)
- cwd : 链接到进程当前工作目录的符号链接
- environ : 进程的环境变量(以null分隔)
- exe : 链接到进程可执行文件的符号链接
- fd : 包含链接到进程每个打开文件的符号链接. 链接到的管道和socket没有关联的文件名, 但有具体id
- maps : 内存映射信息(共享段, lib等)
- root : 链接到进程的根目录(由chroot设置)的符号链接
- state : 进程的总状态信息(ps可解析该信息)
- statm : 内存使用情况的信息

## 虚拟地址表
由于物理内存的大小是受限制的，所以进程运行在自身的内存沙盒内即“虚拟内存地址（virtual address space）”，也称作 虚拟内存（Virtual Memory）.

字节在这个虚拟地址空间内的地址不再和处理器放在地址总线上的地址相同. 因此必须建立转换数据结构和系统将虚拟地址空间中的字节映射到物理字节.

虚拟地址可参见下图(`/proc/$PID/maps`)
[](https://cdn-images-1.medium.com/max/1300/1*ImbY2Tb3wZaeuKblwarFTg.jpeg)

当 CPU 执行一个指令需要引用内存地址时。首先将在 VMA（Virtual Memory Areas）中的逻辑地址转换为线性地址。这个转换通过 MMU 完成.
[](https://cdn-images-1.medium.com/max/1040/1*xek5BQhJhWqsOPAaA5uROw.png)

由于逻辑地址太大几乎很难独立的管理，所以引入`页（pages）` 进行管理。当必要的分页操作被激活后， 虚拟地址空间被分成更小的称作页的区域 （大部分操作系统下是 4KB，可以修改）。页是虚拟内存中数据内存管理的最小单元。**虚拟内存不存储任何内容，而是简单的将程序地址空间映射到底层物理内存之上**.

如果堆上有足够的空间的满足我们代码的内存申请，内存分配器可以完成内存申请无需内核参与，否则将通过操作系统调用`brk`进行扩展堆，通常是申请一大块内存.

堆内存地址增长,见下图:
- [](https://cdn-images-1.medium.com/max/1040/1*mvi6PRy9wu0KmBcP9YT5Cw.png)

应用程序通过系统调用`brk （sbrk/mmap 等）`获得内存. 内核调用它只更新堆VMA.

> 如何减少内存碎片化呢？答案取决是使用哪种内存分配算法，也就是使用哪个底层库, 比如tcmalloc和jemlloc.

参考:
- [Go 内存分配器可视化指南](https://blog.learngoprogramming.com/a-visual-guide-to-golang-memory-allocator-from-ground-up-e132258453ed)

## example
1. 从标准输入中读取并写入标准输出
```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	reader := bufio.NewReader(os.Stdin)

	for {
		result, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println("stdin read error:", err)
		}
		if result == "\n" {
			fmt.Println("quit!", result)

			break
		}

		fmt.Fprintf(os.Stdout,"output: %s\n", result)
	}
}
```

1. 获取进程的pid
```go
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Fprintf(os.Stdout, "pid: %d\n", os.Getpid())
}
```

1. 从输入读取命令并执行
```go
package main

import (
	"bytes"
	"log"
	"os"
	"os/exec"
)

func main() {
	args := os.Args
	if len(args) != 2 {
		log.Println("usage: run cmd")
		os.Exit(1)
	}

	cmdPath, err := exec.LookPath(args[1])
	CheckErr(err)

	cmd := exec.Command(cmdPath)

	var b bytes.Buffer
	cmd.Stdout = &b
	cmd.Stderr = &b

	err = cmd.Start()
	CheckErr(err)

	log.Println("Waiting for command to finish...")
	err = cmd.Wait()
	CheckErr(err)
	log.Printf("Command finished with output: %s\n", string(b.Bytes()))
}

func CheckErr(err error) {
	if err != nil {
		log.Fatal(err)
	}
}
```

1. 获取每秒的时钟滴答数
```go
package main

import (
	"fmt"
)

/*
#include <unistd.h>
#include <sys/types.h>
#include <pwd.h>
#include <stdlib.h>
*/
import "C"

func main() {
	var sc_clk_tck C.long
	sc_clk_tck = C.sysconf(C._SC_CLK_TCK)
	fmt.Printf("SysConf Clock Ticks: %d\n", sc_clk_tck)
}
```

## 内存
在操作系统中,每个进程都有自己的内存,互相之间不干扰即有独立的进程内存空间.

对于进程的内存空间来讲,放程序代码的部分称为代码段(Code Segment).

对于进程的内存空间来讲,放进程运行中产生数据部分称为数据段(DataSegment). 其中局部变量的部分,仅在当前函数执行的时候起作用,当进入另一个函数时,这个
变量就释放了;也有动态分配的,会较长时间保存,指明才销毁的部分称为堆(Heap).

只有进程使用内存的时候,才会使用内存管理的系统调用来申请,但是这还不代表真的就有了对应的物理内存. 实际上只有真的写入数据的时候,发现没有对应物理内存才会触发一个中断来分配
物理内存.

在堆里面分配内存的系统调用是brk和mmap:
1. 当分配的内存数量比较小的时候使用 brk,会和原来的堆的数据连在一起
1. 当分配的内存数量比较大的时候使用 mmap,会重新划分一块区域