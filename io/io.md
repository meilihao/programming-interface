# io
io = in + output.

io的通路是总线, 目前最新标准是PCIe 5.0.

> PCIE的地址总线和数据总线是分时复用的.

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
