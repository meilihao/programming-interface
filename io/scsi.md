# scsi
Linux SCSI子系统包括三层：
- 上层由特定的设备类型驱动所组成，如磁盘驱动、磁带驱动和CD-ROM驱动，最靠近用户空间
- 中间层是SCSI核心代码，连接上层和下层
- 下层包括诸如QLogic和Emulex HBA这类驱动，最靠近硬件

按照内核版本的区别，驱动可编译进内核或以模块的形式加载到内核. sd是SCSI磁盘驱动，或块驱动，作为模块时命名为scsi_mod.

通常，在大多数版本中这些驱动都编译为模块，并在启动时作为初始化内存磁盘镜像文件（initrid image）的一部分被加载. 如果当前没有在启动时加载，而启动过程中要求加载时，那么就需要重新编译一个包含该驱动的初始化内存磁盘镜像文件. 2.6版本内核中，不仅需要修改/etc/modules.conf文件，还要执行mkinitrd命令来对改动进行更新.

要确认驱动是否编译为模块及当前是否被加载，只需查看lsmod命令输出中的sd_mod 和 scsi_mod(???高版本未知).

Linux主机LUN识别：系统在加载主机适配层驱动时通过扫描SCSI总线时识别, SCSI子系统已发现并识别的设备在/proc/scsi/scsi文件中列出.

> 注意: `/proc/scsi/scsi`列出的磁盘设备不是动态的，因此不受网络状态变更的影响.

从Linux2.6内核版本开始，**/proc文件系统迁移至改进后的/sys文件系统**. /sys文件系统中加入对**动态变更的支持**，如添加和删除LUN，而无需重新加载主机适配器驱动或重启主机. 通常，可通过/sys/class/scsi_host/hostN目录下的内容来查看哪些SCSI设备已被主机识别, N是主机适配器ID号. 用户空间有一个lsscsi工具， 使用/sys中的信息来显示所有识别设备的汇总清单.

## 动态SAN网络重配
参考:
- [Linux系统SCSI磁盘扫描机制解析及命令实例](https://blog.csdn.net/jiaping0424/article/details/51777043)

用户在网络中添加或删除磁盘时，需要对其重新识别. 可使用以下5种方法之一让Linux主机重新识别这些变更：
- 重启主机

    重启主机是检测新添加磁盘设备的可靠方式. 在所有I/O停止之后方可重启主机，同时静态或以模块方式连接磁盘驱动. 系统初始化时会扫描PCI总线，因此挂载其上的SCSI host adapter会被扫描到，并生成一个PCI device. 之后扫描软件会为该PCI device加载相应的驱动程序. 加载SCSI host驱动时，其探测函数会初始化SCSI host，注册中断处理函数，最后调用scsi_scan_host函数扫描scsi host adapter所管理的所有scsi总线.
- 卸载并重新加载HBA驱动

    通常情况下，HBA驱动在系统中以模块形式加载. 从而允许模块被卸载并重新加载，在该过程中SCSI扫描函数得以调用. 通常，在卸载HBA驱动之前，SCSI设备的所有I/O都应该停止，卸载文件系统，多路径服务应用也需停止. 如果有代理或HBA应用帮助模块，也应当中止.
- Echo /proc下的SCSI设备列表

    不推荐.
- 通过/sys下的属性设置运行SCSI扫描

    2.6内核中，HBA驱动将SCAN功能导出至/sys目录下，可用来重新扫描该接口下的SCSI磁盘设备, 命令是`echo “- - -” > /sys/class/scsi_host/hostH/scan`
- 通过HBA厂商脚本运行SCSI扫描


注意：???Linux内核不在/dev目录中为网络设备指定固定名称, 设备文件名在扫描总线时按照发现顺序指定，例如，一个LUN名为/dev/sda，在驱动重新加载后，这个LUN名称可能更改为/dev/sdce. 网络重认可能导致主机，总线，target和LUN ID值的偏移，因此通过/proc/scsi/scsi文件添加特定磁盘是不可靠的.

## Linux设备命名
内核驱动可使用特定的设备文件来控制磁盘设备. 映射到同一物理磁盘设备的设备文件可能不止一个. 例如，在多路径环境下某一设备配置四条路径，则会有四个不同的设备文件映射到同一设备.

 设备文件位于/dev目录下，通过major和minor编号访问. 光纤通道连接的设备通过sd驱动作为SCSI磁盘设备管理. 因此，每一个LUN在/dev目录下有一个对应的设备文件.

 SCSI磁盘设备有一个以`sd`为前缀的特定设备文件，具有如下命名格式：`/dev/sd[a-z][1-15]`

名称中不带数字表示整个磁盘，而名称中有数字则表示磁盘的一个分区. 依据惯例，一个SCSI磁盘设备最多可以有16个minor编号. 因此，对于一整块磁盘，每块磁盘最多有15个分区，使用一个**minor编号**来标示整块磁盘（例如/dev/sda），其他15个minor编号用来标示该磁盘的分区（例如/dev/sda1，/dev/sda2，等等）. 以下示例显示整块磁盘/dev/sda的设备文件，该设备major编号为8，minor编号为0，当前有4个分区.
```sh
$ ls -l /dev/sda*
brw-rw---- 1 root disk 8, 0 3月  16 09:02 /dev/sda
brw-rw---- 1 root disk 8, 1 3月  16 09:02 /dev/sda1
brw-rw---- 1 root disk 8, 2 3月  16 09:02 /dev/sda2
brw-rw---- 1 root disk 8, 3 3月  16 09:02 /dev/sda3
brw-rw---- 1 root disk 8, 4 3月  16 09:02 /dev/sda4
```
 
`/proc/partitions`文件列出所有SCSI磁盘驱动识别出的`sd`设备，包括sd名，major编号，minor编号，以及各磁盘设备的大小.

## Linux磁盘限制:定义磁盘设备数量
Linux内核使用静态的major和minor编号来进行SCSI寻址. major可看作设备驱动程序，同一设备驱动程序管理的设备由相同的major编号. 而minor编号代表被访问的具体设备. 内核为SCSI磁盘设备保留了一定数量的major编号. 因此，SCSI磁盘设备的数量受到可用的major编号的限制. 这些可用的major编号数量又随内核版本不同而有所区别.

通常，可使用以下公式来计算Linux主机系统可支持的最大设备数：
最大磁盘数=major编号数×minor编号数÷分区数

对于Linux2.6内核版本major数增长至12比特位而minor增长至20比特位，因此Linux 2.6内核支持上千块磁盘，每块磁盘最多15个分区的限制没有改变. 由于major并不仅仅用于磁盘（/proc/devices），所以不能以2^12个major来计算磁盘个数. 一个major可用的minor数就达到2^20=65536，而每个SCSI磁盘16个minor的限制仍然不变，每个major可支持2^20÷16=4096个. 

其他限制磁盘数量的因素: ???如果HBA驱动作为模块加载在Linux系统中，则内核对可配置磁盘总数有一定的限制。有可能小于内核支持的最大值（通常128或256）。系统加载的第一个模块发现磁盘数可配置为内核支持的最大磁盘数，之后的驱动则达不到这个值。所有这些驱动共享一个pool中的设备结构，这些设备结构是在第一个主机适配器驱动加载后静态分配的。

## SCSI 设备在 Linux 系统上的地址映射
SCSI设备在Linux系统中的逻辑地址映射由4个部分组成：
- SCSI adapter number [host]

    host说的是适配卡在计算机内部IO总线的位置(比如PCI，PCMCIA，ISA等），适配器也称为HBA，其编号由内核以0开始的升序分配
    host id，也就是scsi适配器的id，可以认为就是scsi适配器的序号. 顺序一般情况下和通过lspci列出来scsi适配器的顺序是一致的.
- channel number [bus]

    一个HBA可能控制一个或者多个SCSI总线.
    SCSI总线类型有很多，常见的如：
    - SAS（Serial Attached SCSI，服务器自带的RAID控制器基本上都是这用SCSI总线）
    - FC（Fibre Channel，光纤通道，常见的SAN都是使用的这种总线）

    channel id，对应不同的scsi总线类型。一个scsi适配器可以支持多种不同的总线类型，连接不同的scsi设备. 但是实际上估计很少这么做，都是连接单一类型的设备.
- id number [target]

    每一个SCSI总线可以有多个SCSI设备连接在上面. 在SCSI术语里，HBA称为“initiator“，占用一个SCSI id号.
    initiator是负责和target（也就是连接在总线上的SCSI设备）通信，所有的SCSI指令都通过它传递，从而完成数据I/O、甚至设备管理等操作.

    一根SCSI并行缆线的id号数量和其宽度有关，如：8bits缆线（窄线）有8个id号. 其中HBA占用1个。剩下7个给SCSI设备（当然，这个7个接口还可以接SCSI缆线）；16bits的缆线对应的可以接15个SCSI设备(targets)。每一个id号就是所谓的target id。

    target id，对应的是一个scsi target设备.

    对于SAN来说，一个target id基本上是对应磁盘阵列上的一个控制器.

    很多阵列都是双控制器的，所以当一块光纤卡通过光纤交换机连接到了2台磁盘阵列的4个控制器，那么在/proc/scsi/scsi中，就会发现对于同一个scsi host，会看到4个target id（0，1，2，3）.

- lun [lun]
    每一个SCSI设备本身（绝大部分情况下说的是虚拟存储，比如RAID）能够包含多个LUN.

    至于lun数目的限制，不同的存储设备情况不同，和磁盘阵列、光纤卡、光纤卡驱动都有关系.

这4部分通过`:`连在一起表示，即常见的4段式的地址映射，如`2:0:0:1`

### 映射地址的应用
因为SCSI设备的映射地址，以数字的形式表现，本身就具备了顺序性，因此映射地址的主要作用在：
1. 内核通过映射地址为设备命名
1. 用户通过映射地址确定系统设备文件和实际物理设备之间的对应关系

查看SCSI设备的映射地址主要有3中方法：
- cat /proc/scsi/scsi
- dmesg命令
- sg_map -x
 