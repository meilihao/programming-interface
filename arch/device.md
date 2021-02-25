# device
root可使用 mknod 命令创建设备文件.

常见的硬件设备及其文件名称:
- IDE 设备 /dev/hd[a-d]
- SCSI/SATA/U 盘 /dev/sd[a-p]
- 软驱 /dev/fd[0-1]
- 打印机 /dev/lp[0-15]
- 光驱 /dev/cdrom
- 鼠标 /dev/mouse
- 磁带机 /dev/st0 或/dev/ht0

> /dev 目录中 sda 设备之所以是 a，并**不是由插槽决定的，而是由系统内核的识别顺序来决定的**，而恰巧很多主板的插槽顺序就是系统内核的识别顺序. 因此在使用 iSCSI 网络存储设备时就会发现，明明主板上第二个插槽是空着的，但系统却能识别到/dev/sdb 这个设备就是原因.
> 分区的数字编码不一定是强制顺延下来的，也有可能是**手工指定**的. 因此sda3 只能表示是编号为 3 的分区，而不能判断 sda 设备上已经存在了 3 个分区.

## blocksize
参考:
- [Advanced Format](https://zh.wikipedia.org/wiki/%E5%85%88%E9%80%B2%E6%A0%BC%E5%BC%8F%E5%8C%96)
- [磁盘格式的512，512e，4kn是什么意思](https://www.reneelab.net/what-is-4kn-disk.html)
- [过渡到高级格式化 4K 扇区硬盘](https://www.seagate.com/cn/zh/tech-insights/advanced-format-4k-sector-hard-drives-master-ti/)
- [Advanced Sector Format of Block Devices](https://www.thomas-krenn.com/en/wiki/Advanced_Sector_Format_of_Block_Devices)
- [512e and 4Kn Disk Formats](https://i.dell.com/sites/csdocuments/Shared-Content_data-Sheets_Documents/en/512e_4Kn_Disk_Formats_120413.pdf)
- [机械硬盘避坑大法：一文搞懂 PMR 和 SMR 有什么区别](https://www.ithome.com/0/436/608.htm)

对于大多数现代磁盘，逻辑扇区大小小于物理扇区大小是正常的(通过`fdisk -l /dev/sd<N>`获取), 这就是最常用的[高级格式磁盘512e](https://en.wikipedia.org/wiki/Advanced_Format)的实现方式.

> Linux内核可在`/sys/block/sdX/queue/physical_block_size`获取有关物理扇区大小的信息，并在`/sys/block/sdX/queue/logical_block_size`获取有关逻辑扇区大小的信息.

> The old format gave a format efficiency of 88.7%, whereas Advanced Format results in a format efficiency of 97.3%.

> `fdisk -l`的`I/O size`: optimal I/O size(最佳io的大小) 默认是 minimal I/O size(磁盘最小io的大小).

disk Advanced Format(Logical Sector Size/Physical Sector Size):
- 512n : 512/512
- 512e : 512/4,096

    512e是因为当时部分操作系统识别不了4k物理扇区，而做的兼容方案. 作为过渡时期的产物，512e(e的意思是Emulation)表示由disk firmware将4K的磁盘模拟成512，让系统以为看见的是512格式.

    在disk支持的情况下, `hdparm --set-sector-size 4096`可修改Logical Sector Size.
- 4Kn : 4,096/4,096

    带"Advanced 4Kn Format" (4K native). 没有512-byte emulation layer, 性能更佳.

## 设备文件
在linux中, 硬件设备都以文件形式存在, 不同设备有不同的文件类型, 这些文件被称为设备文件. 设备文件让程序能够同系统的硬件和外围设备进行通信. 在Linux中,所有的设备以文件的形式出现在目录 /dev 中, 由`systemd-udevd`管理(随着kernel报告的硬件变化创建或删除设备文件).

设备用主设备号和次设备号表示其特征, 主设备号和次设备号统称为设备号. 主设备号用来查找一个特定的驱动程序即指明设备类型, 次设备号用来表示使用该驱动程序的各设备即指具体哪个设备.

> 设备文件的 i 节点中记录了设备文件的主、辅 ID. 每个设备驱动程序都会将自己与特定主设备号的关联关系向内核注册,藉此建立设备专用文件和设备
驱动程序之间的关系.

`crw-rw-rw-  1 root tty       4,     3 6月  26 21:34 tty3`, `4`是主设备号, `3`是次设备号.

实际上设备文件也分块设备文件(block special file) 和 字符设备文件(character special file).  块设备每次读取或写入一块数据(一组字节, 通常是512的倍数), 字符设备每次读取或写入一个字节.

还有一种有设备驱动但没有实际设备的虚拟设备称为伪设备, 比如pty(伪tty), /dev/null, /dev/random等.

常见设备:
- `/dev/sd${xxx}` : scsi磁盘
- `/dev/srx` : scsi光驱
- `/dev/stx` : scsi磁带
- `/dev/cdrom` : 指向光驱的符号连接.

## 块设备(block device) / 字符设备(character device)
- 块设备(**可寻址**) : 提供对设备(比如磁盘,蓝光光盘)带缓冲的访问, 每次访问以固定长度(块)为单位进行.
- 字符设备(**不可寻址**) : 接收或输出字符流的设备, 需**按顺序操作, 不支持随机操作**, 用于键盘, 串口, 打印机, 调制解调器等.

> Linux 专有文件/proc/partitions 记录了系统中每个磁盘分区的主辅设备编号、大小和名称.

## udev
在用户空间实现的一种设备管理fs(devtmpfs), 它会将设备文件自动放在/dev下, 即/dev完全由udev管理. 它依赖sysfs(/sys下的虚拟文件系统)来了解系统的设备变化, 并使用一系列udev特有的规则来指定其对设备的管理以及命令, 默认规则在`/lib/udev/rules.d`下, 自定义规则在`/etc/udev/rules.d`下.

```bash
# grep -ri '/dev/disk' /lib/udev/rules.d # 查找生成/dev/disk下信息的规则
/lib/udev/rules.d/60-persistent-storage.rules
```

> 部分相关代码: [udev-builtin-path_id.c](https://cgit.freedesktop.org/systemd/systemd/tree/src/udev/udev-builtin-path_id.c)

## `/dev/loopN`
一种伪设备，使得文件可以如同块设备一般被访问.

可通过`losetup -a`查看.

snapd的`loopN`可通过`sudo apt autoremove --purge snapd`解决.

## [块设备持久化命名和多路径](https://www.zybuluo.com/tony-yin/note/1214135)
持久化命名，顾名思义区别于一次性或者是短暂的命名，它是一种长久的并且稳定靠谱的命名方案. 与之形成鲜明对比的就是/dev/sda这种非持久化命名，这两种命名方案各有各的用处，持久化命名方案有四种：by-label、by-uuid、by-id和by-path. 对于那些使用GUID分区表（GPT）的磁盘，还有额外的两种方案：by-partlabel和by-partuuid.

by-label和by-uuid都和文件系统相关，by-label是通过读取设备中的内容获取，by-uuid则是随着每次文件系统的创建而创建，所以by-uuid的持久化程度更高一些；持久化程度最高的要属by-path和by-id了，因为它们都是根据物理设备的位置或者信息而和链接做对应的，by-path会因为路径的变化而变化；而by-id则不会因为路径或者系统的改变而改变，它只会在多路径的情况下发生改变. 这两个在通过虚拟设备名称寻找物理设备的场景下都十分有用.

多路径设备则帮助我们在SAN等场景下提高了数据传输的可用性，目前由于网络带宽的发展，它在iscsi场景下也频繁亮相.

### by-label
label表示标签的意思，几乎每一个文件系统都有一个标签. 所有有标签的分区都在/dev/disk/by-label目录中列出.

label是通过从设备中的内容（即数据）获取，所以如果将该内容拷贝至另一个设备中，我们也可以通过blkid来获取磁盘的label.

### by-uuid
UUID是给每个**文件系统**唯一标识的一种机制，这个标识是在**分区格式化时通过文件系统工具生成**，比如mkfs，这个唯一标识可以起到解决冲突的作用. 所有GNU/Linux文件系统（包括swap和原始加密设备的LUKS头）都支持UUID. FAT和NTFS文件系统并不支持UUID，但是在/dev/disk/by-uuid目录下还是存在着一个更为简单的UID（唯一标识）.

    $ ls -l /dev/disk/by-uuid/
    total 0
    lrwxrwxrwx 1 root root 10 May 27 23:31 0a3407de-014b-458b-b5c1-848e92a327a3 -> ../../sda2
    lrwxrwxrwx 1 root root 10 May 27 23:31 b411dc99-f0a0-4c87-9e05-184977be8539 -> ../../sda3
    lrwxrwxrwx 1 root root 10 May 27 23:31 CBB6-24F2 -> ../../sda1
    lrwxrwxrwx 1 root root 10 May 27 23:31 f9fe0b69-a280-415d-a03a-a32752370dee -> ../../sda4

使用UUID方法的优点是，名称冲突发生的可能性大大低于使用Label的方式. 更深层次地讲，它是在创建文件系统时自动生成的. 例如，即使设备插入到另一个系统(可能有一个标签相同的设备)，它仍然是唯一的.

缺点是uuid使得许多配置文件(例如fstab或crypttab)中的长代码行难以读取和破坏格式. 此外，每当一个分区被调整大小或重新格式化时，都会生成一个新的UUID，并且必须(手动)调整配置.

### by-path
该目录中的条目提供一个符号名称，该符号名称通过用于访问设备的硬件路径引用存储设备，首先引用PCI hierachy中的存储控制器，并包括SCSI host、channel、target和LUN号，以及可选的分区号. 虽然这些名字比使用major和minor号或sd名字更容易，但必须使用谨慎以确保target号不改变在光纤通道SAN环境中(例如，通过使用持久绑定)，如果一个主机适配器切换到到一个不同的PCI插槽的话这个路径也会随之改变. 此外，如果HBA无法探测，或者如果驱动程序以不同的顺序加载，或者系统上安装了新的HBA，那么SCSI主机号都有可能会发生变化.

上面说了很多种情况都会导致by-path的值可能发生变化，但是在同一时间来说，by-path的值是和物理设备是唯一对应的，也就是说不管怎么说by-path是对应物理机器上面的某个位置的，根据by-path可以获取对应物理位置的设备（此前megaraid通过逻辑磁盘获取物理磁盘位置就是根据这个原理）

对于iSCSI设备，路径/名称映射从目标名称和门户信息映射到sd名称.
应用程序通常不适合使用这些基于路径的名称. 这是因为这些路径引用可能会更改存储设备，从而可能导致将不正确的数据写入设备. 基于路径的名称也不适用于多路径设备，因为基于路径的名称可能被误认为是单独的存储设备，导致不协调的访问和数据的意外修改.

此外，基于路径的名称是特定于系统的. 当设备被多个系统访问时，例如在集群中，这会导致意外的数据更改.

### by-id

此目录中的条目提供一个符号名称，该符号名称通过唯一标识符(与所有其他存储设备不同)引用存储设备. 标识符是设备的属性，但不存储在设备的内容(即数据)中.

该id从设备的全局ID（WWID）或设备序列号中获取. /dev/disk/by-id条目也可能包含一个分区号.

World Wide Identifier（WWID）可用于可靠的识别设备, SCSI标准要求所有SCSI设备提供一个持久的、系统无关的ID. WWID标识符保证对每个存储设备都是唯一的，并且独立于用于访问设备的路径.

这个标识符可以通过发出SCSI查询来获取设备标识重要厂商数据(第0x83页)或单位序列号(第0x80页). 从这些wwid到当前/dev/sd名称的映射可以在/dev/disk/by-id/目录中维护的符号链接中看到.
例如，具有页0x83标识符的设备将具有:

    scsi-3600508b400105e210000900000490000 -> ../../sda

或者，具有页0x80标识符的设备将具有:

    scsi-SSEAGATE_ST373453LW_3HW1RHM6 -> ../../sda

Red Hat Enterprise Linux 5自动维护从基于wwid的设备名称到系统上当前/dev/sd名称的正确映射. 应用程序可以使用/dev/disk/by-id/的链接引用磁盘上的数据，即使设备的路径改变，甚至当从不同系统访问该设备时都是如此.

但是当设备被插入到硬件控制器的端口时，而这个端口又受另一个子系统控制（即多路径），by-id的值也会改变.

> 如果两个target设置相同的vpd_unit_serial(`/sys/kernel/config/target/core/iblock_xxx/${iblock_name}/wwwn/vpd_unit_serial`), 那么initiator挂载时因为/dev/disk/by-id下的路径可能相同会导致出现覆盖(即只能找到一个盘), 但/dev/disk/by-path下的路径因为生成规则不同而正常, 我在验证drbd+fc问题时遇到过该情况.

### by-partlabel && by-partuuid
这两个和上面提到的by-label和by-uuid类似，只不过是在GPT磁盘上.

### 多路径设备
多路径设备指的是从一个系统到一个设备存在多个路径，这种现象主要出现在光纤网络的SAN下，主要是做数据链路冗余以达到高可用的效果，即对应底层一个物理设备，可能存在多个路径表示它.

如果从一个系统到一个设备有多个路径，那么device-mapper-multipath使用WWID来检测它, 然后在/dev/mapper/wwid中显示一个“伪设备”，例如/dev/ mapper/3600508b400105df70000000ac0000.

Device-mapper-multipath显示映射到非持久标识符：`Host:Channel:Target:LUN`， `/dev/sd名称`，以及`major:minor`号.

    3600508b400105df70000e00000ac0000 dm-2 vendor,product 
    [size=20G][features=1 queue_if_no_path][hwhandler=0][rw] 
    \_ round-robin 0 [prio=0][active] 
     \_ 5:0:1:1 sdc 8:32  [active][undef] 
     \_ 6:0:1:1 sdg 8:96  [active][undef]
    \_ round-robin 0 [prio=0][enabled] 
     \_ 5:0:0:1 sdb 8:16  [active][undef] 
     \_ 6:0:0:1 sdf 8:80  [active][undef]

Device-mapper-multipath在系统上自动维护每个基于wwid的设备名称和其对应的/dev/sd名称的正确映射. 这些名称即使是在路径发生改变时也是持久的，并且当从不同的系统访问设备时它们仍然是一致的.

## tty
参考:
- [Linux TTY/PTS概述](https://segmentfault.com/a/1190000009082089)

```bash
# toe -a # 查看支持的tty
# tty # 查看当前终端绑定的tty
/dev/pts/1
# echo aaa > /dev/pts/1 # 往tty直接写入数据跟写标准输出是一样的效果
aaa
# stty -a # 查看当前tty的配置
```

当pts/1收到input的输入后，会检查当前前端进程组是哪一个，然后将输入放到进程组的leader的输入缓存中，这样相应的leader进程就可以通过read函数得到用户的输入. 因此这是有时看到`/proc/<pid>/fd/0 -> /dev/pts/1`, 但`echo aaa > /dev/pts/1`并不起作用的原因, 目前也没法方法跨pgid写pts.
