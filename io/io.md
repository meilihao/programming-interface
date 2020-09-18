# io
io = in + output.

io的通路是总线, 目前最新标准是PCIe 5.0.

> **windows的句柄 = linux的文件描述符, 但windows区分文件句柄和socket句柄**.


实现io的三种方式:
1. 发起syscall再循环检测io是否完成
1. 中断通知
1. DMA(direct memory access)

### 分类
1. 读/写io : 读写一段**连续的**内容.
1. 大/小快io : 读取数据块的大小(大小是相对而言的)
1. 连续/随机io : 随机io是指本次io给出的逻辑地址与上一次io的地址相差较大, 需重新寻址; 如果地址相近或在上一次的结束位置后面则是连续io.
1. 顺序/并发io : 磁盘控制器每一次对raid系统发出的指令套（指完成一个事物所需要的指令或者数据）, 是一条还是多条. 如果是一条，则控制器缓存中的IO队列，只能一个一个的来，此时是顺序IO；如果控制器可以同时对raid中的多块磁盘，同时发出指令套，则每次就可以执行多个IO，此时就是并发IO模式. 并发IO模式提高了效率和速度。
1. 持续/间断io : 持续不断发送/接受io请求的数据流为持续io; io数据流时断时续是间断io
1. 稳定/突发io : 存储设备在一段时间内接受/发送的iops以及throughput(吞吐量)保持相对稳定是稳定io; 单位时间内的iops和throughput突增是突发io.
1. 实/虚io : io请求包含对应的实际数据地址为实io; 应用针对文件元数据的操作或对存储发送的非实际数据请求(比如控制性io)是虚io.

### 文件系统io
出于速度和效率考虑,系统 I/O 调用(即内核)和标准 C 语言库 I/O 函数(即 stdio 函数)在操作磁盘文件时会对数据进行缓冲.

通常来说，文件I/O可以分为两种：
- Buffer I/O : 为了性能

  缓存 I/O 又被称作标准 I/O，大多数文件系统的默认 I/O 操作都是缓存 I/O.  在 Linux 的缓存 I/O 机制中，这种访问文件的方式是通过两个系统调用实现的：read() 和 write().

  Buffer I/O 中引入一类特别的操作叫做内存映射文件（mmap），它的不同点在于，中间会减少一层数据从用户地址空间到操作系统地址空间的复制开销.
- Direct I/O(直接I/O)`/`raw I/O(裸I/O)`

  凡是通过直接 I/O 方式进行数据传输，数据均直接在用户地址空间的缓冲区和磁盘之间直接进行传输，中间少了页缓存的支持. 常见于数据库类应用, 其高速缓存和 I/O 优化机制均自成一体.

  因为直接 I/O(针对磁盘设备和文件)涉及对磁盘的直接访问,所以在执行 I/O 时,必须遵守一些限制:
  - 用于传递数据的缓冲区,其内存边界必须对齐为块大小的整数倍
  - 数据传输的开始点,亦即文件和设备的偏移量,必须是块大小的整数倍
  - 待传递数据的长度必须是块大小的整数倍

  direct i/o的最大好处是减少kernel和用户空间的数据复制次数, 降低文件读写带来的cpu负载和内存带宽占用.

![](/misc/img/io/20190320001938378.png)
![](/misc/img/io/io_buffer.jpg)

> 追加写使用 buffered io有优势(os可以将写io合并成更大的io).

具体分类:
- 同步io

  向os发起io请求后一直处于阻塞状态, 直到os将io结果返回. 即当前一个IO真正完成后，后一个IO才可以发出. 如posix的read, write等系统调用.
- 异步io

  向os发起io请求后os立即返回一个"已接受"信号, 此时发起者还可继续执行后续代码. 即每个IO请求，是否完成，都不会影响后续IO的发出. 如glibc的aio，以及linux的libaio等.
- 阻塞/非阻塞io

  阻塞: 一直等到io操作完成为止
  非阻塞: 调用操作io的接口后立即返回，但是并不保证IO操作成功, 此时外层需要一个循环来处理, 费cpu.
- direct io

  参照上面的`Direct I/O`

> 同步与异步的主要区别就在于：会不会导致请求进程（或线程）阻塞.
> read()和 write()系统调用在操作磁盘文件时不会直接发起磁盘访问,而是仅仅在用户空间缓冲区与内核缓冲区高速缓存(kernel buffer cache)之间复制. 采用这一设计,意在使 read()和 write()调用的操作更为快速,因为它们不需要等待(缓慢的)磁盘操作.
> Linux 内核对缓冲区高速缓存的大小没有固定上限. 内核会分配尽可能多的缓冲区高速缓存页,而仅受限于两个因素:可用的物理内存总量,以及出于其他目的对物理内存的需求(例如,需要将正在运行进程的文本和数据页保留在物理内存中).

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

- iowait : os接受io请求到完成的时间

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
io scheduler是linux下专用来对io进行优化的模块, 所有针对底层存储设备的io都要经过它的优化操作, 然后将优化后的io顺序放入底层存储控制器驱动的queue中.

```bash
$ cat /sys/block/sda/queue/scheduler
noop deadline [cfq] # `[]`表示选中
$ cat /sys/block/sda/queue/nr_requests # 调度器的队列深度, 越大io优化效果越好, 但每个io延迟会随之升高
254
```

算法:
- CFQ （Completely Fair Scheduler完全公平调度器）（cfq） ：它是许多 Linux 发行版的默认调度器；它将由进程提交的同步请求放到多个进程队列中，然后为每个队列分配时间片以访问磁盘.
- Noop 调度器（noop） ： 基于先入先出（FIFO）队列概念的 Linux 内核里最简单的 I/O 调度器. 此调度程序最适合于 SSD.
- 截止时间调度器（deadline） ： 尝试保证请求的开始服务时间, 力求将每次请求的延迟降到最低.

> 对于使用外部存储系统的机器, 使用cfq即可, 因为外部存储控制器越来越智能, 更适合优化io.

参考:
- [详解Linux 磁盘I/O优化（oracle RAC）]https://www.tuicool.com/articles/NFZZfqY
- [如何更改 Linux I/O 调度器来调整性能 for ssd](https://linux.cn/article-8179-1.html), 修改后需更新grub: `sudo update-grub`.

## 工具
iostat

## 磁盘Persistent命名
操作系统通过路径发送IO到存储，Linux系统SCSI磁盘路径有以下部分组成：
- HBA卡的PCI标示符
- HBA卡的管道号
- 存储端SCSI target地址
- LUN(Logical Unit Number)号

SCSI磁盘路径在Linux上有3中表现方式:
1. /dev/sd目录
1. 通过major:minor号
1. /dev/disk/by-path, 该目录是 /dev/sd设备的软连接

    Fibre Channel磁盘/dev/disk/by-path路径, `pci-0000:04:00.0-fc-0x21000024ff749e42-lun-0`, 其中:
    1. `0000:04:00.0` : pci标识
    1. `fc` : channel号
    1. `0x21000024ff749e42` : target fc port name即target wwpn
    1. `lun-0` : lun号

上面这三种SCSI磁盘路径都不是永久不变，当服务器新增或者删除新的PCI设备时候，路径就会发生变化，有时候即使是服务器重启也可能导致路径变成发生变化.

为了保证应用程序使用的磁盘路径能够永久不变，有以下几种方法:
1. WWID, **推荐**

  根据SCSI标准，每个SCSI磁盘都有一个WWID。类似于网卡的MAC地址，要求是独一无二。通过WWID标示SCSI磁盘就可以保证磁盘路径永久不变，Linux系统上/dev/disk/by-id目录包含每个SCSI磁盘WWID访问路径, 比如：

  scsi-3600508b400105e210000900000490000 -> ../../sda

  > Linux自带的device-mapper-multipath工具就是通过WWID来探测SCSI磁盘路径，可以将同一设备多条路径合并，并在/dev/mapper/下面创建新的设备路径. 通过`multipath -l`可以看到WWID与磁盘路径、Host:Channel:Target:Lun与/dev/sd以及major:minor对应关系

1. UUID

  UUID是有文件系统在创建时候生成的，用来标记文件系统，类似WWID一样也是独一无二的. 因此使用UUID来标示SCSI磁盘，也能保证路径是永久不变的. Linux上/dev/disk/by-uuid可以看到每个已经创建文件系统的磁盘设备以及与/dev/sd之间的映射关系.

  > 注意：Linux自带的md和LVM工具也会在SCSI磁盘上面写入UUID信息

1. UDEV

  UDEV是Linux提供的一种让用户对设备进行自定义命名的机制. 可以通过UDEV将WWID/UUID信息跟磁盘路径映射起来，这样也可以保证设备路径永久不变.

## 删除磁盘
1. sync

  - raw device : `blockdev --flushbufs`将脏数据写入磁盘, 这一步骤对于裸设备事情情况尤为重要，因为裸设备无法通过umount或者vgreduce将脏数据写入磁盘.
  - other device : sync/fsync
1. 删除

  方法有2种:
  1. 使用`echo 1 > /sys/block/device-name/device/delete`删除磁盘，device-name以sd开头，比如sda等
  1. 使用`echo 1 > /sys/class/scsi_device/h:c:t:l/device/delete`删除磁盘，h代表HBA卡号，c代表HBA卡channel，t代表SCSI target ID，i代表LUN ID. h:c:t:l这些信息可以通过lsscsi，scsi_id，multipath -l，`ls -l /dev/disk/by-*`方式查看

当系统使用多路径软件时候，可以在线删除一条路径而不影响业务使用. 操作步骤如下：
- 在client中删除磁盘路径
- server端使用`echo offline > /sys/block/sda/device/state将磁盘路径offline`, 多路径软件将会使用剩余路径处理IO
- server端使用`echo 1 > /sys/block/device-name/device/delete`删除磁盘
- server端`echo 1 > /sys/class/scsi_device/h:c:t:l/device/delete`删除磁盘

## FAQ
### Synchronized I/O file integrity completion和synchronized I/O data integrity completion
SUSv3 定义的两种不同类型的同步 I/O 完成,二者之间的区别涉及用于描述文件的元数据(关于数据的数据).

Synchronized I/O file integrity completion 是synchronized I/O data integrity completion 的超集. 该 I/O 完成模式的区别在于在对文件的一次更新过程中,要将所有发生更新的文件元数据都传递到磁盘上,即使有些在后续对文件数据
的读操作中并不需要.

针对 synchronized I/O data completion 状态,如果是诸如最近修改时间戳之类的元数据属性发生了变化,那么是无需传递到磁盘的)相比之下,fsync()调用会强制将元数据传递到磁盘上.

若内容发生变化的内核缓冲区在 30 秒内未经显式方式同步到磁盘上,则一条长期运行的
内核线程pdflush会确保将其刷新到磁盘上, 规避缓冲区与相关磁盘文件内容长期处于不一致状态(以至于在系统崩溃时发生数据丢失)的问题. 文件/proc/sys/vm/dirty_expire_centisecs 规定了在 pdflush 刷新之前脏缓冲区必须达到的时间(单位: 毫秒). 位于同一目录下的其他文件则控制了 pdflush 操作的其他方面.