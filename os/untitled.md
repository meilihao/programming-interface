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