# io
io = in + output.

io的通路是总线, 目前最新标准是PCIe 5.0.

> PCIE的地址总线和数据总线是分时复用的.

### 分类
1. 读/写io : 读写一段**连续的**内容.
1. 大/小快io : 读取数据块的大小(大小是相对而言的)
1. 连续/随机io : 随机io是指本次io给出的逻辑地址与上一次io的地址相差较大, 需重新寻址; 如果地址相近或在上一次的结束位置后面则是连续io.
1. 顺序/并发io : 磁盘控制器每一次对raid系统发出的指令套（指完成一个事物所需要的指令或者数据）, 是一条还是多条. 如果是一条，则控制器缓存中的IO队列，只能一个一个的来，此时是顺序IO；如果控制器可以同时对raid中的多块磁盘，同时发出指令套，则每次就可以执行多个IO，此时就是并发IO模式. 并发IO模式提高了效率和速度。
1. 持续/间断io : 持续不断发送/接受io请求的数据流为持续io; io数据流时断时续是间断io
1. 稳定/突发io : 存储设备在一段时间内接受/发送的iops以及throughput(吞吐量)保持相对稳定是稳定io; 单位时间内的iops和throughput突增是突发io.
1. 实/虚io : io请求包含对应的实际数据地址为实io; 应用针对文件元数据的操作或对存储发送的非实际数据请求(比如控制性io)是虚io.

### 文件系统io
通常来说，文件I/O可以分为两种：
- Buffer I/O : 为了性能

  缓存 I/O 又被称作标准 I/O，大多数文件系统的默认 I/O 操作都是缓存 I/O.  在 Linux 的缓存 I/O 机制中，这种访问文件的方式是通过两个系统调用实现的：read() 和 write().

  Buffer I/O 中引入一类特别的操作叫做内存映射文件（mmap），它的不同点在于，中间会减少一层数据从用户地址空间到操作系统地址空间的复制开销.
- Direct I/O

  凡是通过直接 I/O 方式进行数据传输，数据均直接在用户地址空间的缓冲区和磁盘之间直接进行传输，中间少了页缓存的支持. 常见于自缓存的数据库类应用.

![](/misc/img/io/20190320001938378.png)


具体分类:
- 同步io

  当前一个IO真正完成后，后一个IO才可以发出. 如posix的read, write等系统调用.
- 异步io

  每个IO请求，是否完成，都不会影响后续IO的发出. 如glibc的aio，以及linux的libaio等.
- 阻塞/非阻塞io

  阻塞: 一直等到io操作完成为止
  非阻塞: 调用操作io的接口后立即返回，但是并不保证IO操作成功
- direct io

  参照上面的`Direct I/O`

> 同步与异步的主要区别就在于：会不会导致请求进程（或线程）阻塞.

[Linux下5种IO模型的小结](https://www.cnblogs.com/ittinybird/p/4666044.html)
![linux io模型](/misc/img/io/212352514599938.png)


### 指标
- IO延迟

  控制器将io指令发出后, 直到io完成的所耗时间. 通常认为io延迟在20ms以内对于应用是可以接受的.
- IOPS

  单位时间内系统能处理的I/O请求数量，一般以每秒处理的I/O请求数量为单位，I/O请求通常为读或写数据操作请求.
- Queue Depth

  磁盘控制器所发出的批量指令的最大条数.
- 吞吐量 : iops * 平均io size

> IOPS=(Queue Depth）/（IO latency)

# 文件描述符
一个非负整数,kernel用来标识一个特定进程正在使用的文件.

# 标准输入 标准输出 标准错误
每当运行一个程序时,其所在的shell会为它打开3个文件描述符,即:
- 标准输入(standard input,0)
- 标准输入(standard output,1)
- 标准输入(standard error,2)
它们默认都链接到终端,可使用重定向方式来修改.

## 常用数值
- I/O Buffer 大小 8192 : unix环境高级编程v3 图3-6 linux上用不同缓存长度进行读操作的时间结果

## 磁盘IO调度算法
```bash
$ cat /sys/block/sda/queue/scheduler
noop deadline [cfq] # `[]`表示选中
```

算法:
- CFQ （Completely Fair Scheduler完全公平调度器）（cfq） ：它是许多 Linux 发行版的默认调度器；它将由进程提交的同步请求放到多个进程队列中，然后为每个队列分配时间片以访问磁盘.
- Noop 调度器（noop） ： 基于先入先出（FIFO）队列概念的 Linux 内核里最简单的 I/O 调度器. 此调度程序最适合于 SSD.
- 截止时间调度器（deadline） ： 尝试保证请求的开始服务时间, 力求将每次请求的延迟降到最低.

参考:
- [详解Linux 磁盘I/O优化（oracle RAC）]https://www.tuicool.com/articles/NFZZfqY
- [如何更改 Linux I/O 调度器来调整性能 for ssd](https://linux.cn/article-8179-1.html), 修改后需更新grub: `sudo update-grub`.
