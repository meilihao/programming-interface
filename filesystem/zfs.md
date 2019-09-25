# zfs
参考:
- [在 Oracle® Solaris 11.2 中管理 ZFS 文件系统](https://docs.oracle.com/cd/E56344_01/html/E53918/index.html)
- [ZFS Administration](https://pthree.org/2012/12/04/zfs-administration-part-i-vdevs/)
- [FreeBSD Handbook's ZFS](https://www.freebsd.org/doc/handbook/)

```sh
# ubuntu 18.04
$ sudo apt install zfsutils-linux # 安装zfs
# 以下三种方式获取zfs version
$ modinfo zfs
$ cat /sys/module/zfs/version
$ dmesg | grep -i zfs
```

zfs有两个工具: zpool和zfs. zpool用于维护zfs pools, zfs负责维护zfs filesystems.

> ZFS可充当卷管理器
> zfs已实现零管理, 通过zfs daemon(zed,ZFS Event Daemon)实现, 因此无需将pool信息写入`/etc/fstab`, pool配置在`/etc/zfs/zpool.cache`里.
> zfs pool使用zfs_vdev_scheduler来调度io.

## 概念
pool : 存储设备的逻辑分组, 可将其存储空间分配给filesystem(数据集).
dataset/filesystem : zfs的文件系统
mirror : 一个虚拟设备存储相同的两个或两个以上的磁盘上的数据副本，在一个磁盘失败的情况下,相同的数据是可以用其他磁盘上的镜像.
resilvering ：在恢复设备时将数据从一个磁盘复制到另一个磁盘的过程

### zfs虚拟设备(zfs vdevs)
一个 VDEV 是一个meta-device，代表着一个或多个物理设备. ZFS 支持 7 中不同类型的 VDEV：
- disk, 默认, 比如HDD, SDD, PCIe NVME等等
- File : 预先分类的文件，为*.img的文件，可以作为一个虚拟设备载入ZFS
- Mirror : 标准的 RAID1 mirror
- ZFS 软件RAID : raidz=raidz1(raid5)/2(raid6)/3, 非标准的基于分布式奇偶校验的软件RAID. 速度: raid0 > raid1 > raidz1 > raidz2 > raidz3
- Hot Spare : 用于热备 ZFS 的软件 raid
- Cache : 用于2级自适应的读缓存的设备 (ZFS L2ARC), 提供在 memory 和 disk的缓冲层, 用于改善静态数据的随机读写性能
- Log : ZFS Intent Log(ZFS ZIL/SLOG, ZFS意图日志,一种对于 data 和 metadata 的日志机制，先写入然后再刷新为写事务), 用于崩溃恢复, 最好配置并使用快速的 SSD来存储ZIL, 以获得更佳性能.

VDEV始终是动态条带化的. 一个 device 可以被加到 VDEV, 但是不能移除.

## zpool
zfs支持分层组织filesystem, 每个filesystem仅有一个父级, 而且支持属性继承, 根为池名称.

> zpool描述了存储的物理特性, 因此必须在创建filesystem前创建池.
> 避免在存储池中使用磁盘分片而是应使用整块磁盘, 已避免潜在复杂性.
> 如果pool没有配置log device, pool会自行为ZIL预留空间.
> raid-z配置无法附加其他磁盘; 无法分离磁盘, 但将磁盘替换为备用磁盘或需要分离备用磁盘时除外; 无法移除除log device或cache device外的device

```sh
$ sudo zpool create pool-test /dev/sdb /dev/sdc /dev/sdd # 创建了一个零冗余的RAID-0存储池, ZFS 会在`/`中创建一个目录,目录名是pool name 
$ sudo zpool list # 显示系统上pools的列表
$ sudo zpool status <pool> # 查看pool的状态
$ sudo zpool destroy <pool> # 销毁pool
$ sudo zpool destroy <pool>/data-set # 销毁dataset
$ sudo zpool upgrade [<pool> | -a] # 更新 ZFS 时，就需要更新指定/全部池
$ sudo zpool add <pool> /dev/sdx # 将驱动器添加到池中
$ sudo zpool exprot <pool> # 如果要在另一个系统上使用该存储池，则首先需要将其导出. zpool命令将拒绝导入任何尚未导出的存储池
$ sudo zpool export oldname # 重命名已创建的zpool的过程分为2个步骤, export + import
$ sudo zpool import oldname newname
$ sudo zpool create <pool> mirror <device-id-1> <device-id-m1> mirror <device-id-2> <device-id-m2> # 创建RAID10
$ sudo zpool add <pool> log mirror <device-id-1> <device-id-2> # 添加 SLOG
$ sudo zpool add <pool> cache <device-id> # 添加L2ARC
$ sudo zpool iostat -v <pool> N # 每隔N秒输出一次pool的io状态
```

mirror/raidz设备不能从pool中删除, 但可增删不活动的hot spares(热备), cache, top-level, log device.

### zpool create
创建pool: `zpool create -f -m <mount> <pool> [raidz（2 | 3）| mirror] <ids>`

参数:
- f : 强制创建pool, 用于解决"EFI标签"错误
- m : 指定挂载点, 默认是`/`
- o : 设置kv属性, 比如ashift
- n : 仅模拟运行, 并输出日志, 用于检查错误, 但不是100%正确, 比如在配置中使用了相同的设备做mirror.

> `blockdev --getpbsz /dev/sdXY`获取扇区大小, zfs默认是512, 与设备扇区不一致会导致性能下降, 比如具有4KB扇区大小的高级格式磁盘，建议将ashift更改为12(2^12=4096). 更改[ashift](https://github.com/zfsonlinux/zfs/wiki/faq#advanced-format-disks)选项的唯一方法是重新创建池.

### zpool scrub
只要读取数据或zfs遇到错误，`zpool scrub`就会在可能的情况下进行静默修复.

```sh
$ zpool scrub <pool> # 检修 pool
$ zpool scrub -s <pool> # 取消正在运行的检修
```

> 建议是每周/月检修一次.

## zfs
```sh
$ sudo zfs get all <pool> # 获取pool的参数
$ sudo zfs set atime = off <pool> # 设置pool参数
$ sudo zfs set compression=gzip-9 mypool # 设置压缩的级别
$ sudo zfs inherit -rS atime  <pool> # 重置参数到default值
$ sudo zfs get keylocation <pool>/<filesystem> # 获取filesystem属性
$ sudo zfs set acltype = posixacl <pool> / <filesystem> # 使用ACL
$ sudo zfs set sharenfs=on <pool> # 通过nfs共享pool
$ sudo zfs set sharenfs=on <pool>/<filesystem> # 通过nfs共享filesystem
```

与大多数其他文件系统不同，ZFS具有可变的记录大小，或者通常称为块大小, 默认情况下，ZFS上的记录大小为128KiB，这意味着它将根据要写入的文件大小动态分配从512B到128KiB任意大小的块.

RDBMS倾向于实现自己的缓存算法，通常类似于ZFS自己的ARC. 因此最好禁用ZFS对数据库文件数据的缓存，并让数据库自己完成.

### zfs create
```sh
# zfs create <pool>/<filesystem>/... # 在pool下创建filesystem, filesystem除了快照外，还可以提高控制级别, 比如配额. 
```

> 为了能够创建和mount filesystem，zpool中不得预先存在相同名称的目录.

### zfs snapshot
ZFS snapshot（快照）是 ZFS 文件系统或卷的**只读**拷贝. 可以用于保存 ZFS 文件系统的特定时刻的状态，在以后可以用于恢复该快照，并回滚到备份时的状态.

```sh
$ sudo zfs snapshot -r mypool/projects@snap1 # 创建 mypool/projects 文件系统的快照
$ sudo zfs list -t snapshot # 查看所有的snapshots列表
$ sudo zfs rollback mypool/projects@snap1 # 回滚快照
$ sudo zfs destroy mypool/projects@snap1 # 移除snapshot
```

### zfs clone
一个ZFS clone是文件系统的可写的拷贝. 一个 ZFS clone 只能从 ZFS snapshot中创建，该 snapshot 不能被销毁，直到该 clones 也被销毁为止.

```sh
# 克隆 mypool/projects，首先创建一个 snapshot 然后 clone
$ sudo zfs snapshot -r mypool/projects@snap1
$ sudo zfs clone mypool/projects@snap1 mypool/projects-clone
```

### ZFS Send 和 Receive
ZFS send 发送文件系统的快照，然后流式传送到文件或其他机器. ZFS receive 接收该 stream 然后写到 snapshot 拷贝，作为 ZFS 文件系统. 这有利于通过拷贝或网络传送备份.

```sh
# 创建 snapshot 然后 save 到文件
$ sudo zfs snapshot -r mypool/projects@snap2
$ sudo zfs send mypool/projects@snap2 > ~/projects-snap.zfs
$ sudo zfs receive -F mypool/projects-copy < ~/projects-snap.zfs # 恢复
```

### ZFS Ditto Blocks，重复块
Ditto blocks 创建更多的冗余拷贝. 

对于只有一个设备的storage pool，ditto blocks are spread across the device, trying to place the blocks at least 1/8 of the disk apart. 对于多设备的 pool，ZFS 试图分布 ditto blocks 到多个独立的 VDEVs, 可设置1~3 份拷贝.

```sh
$ sudo zfs set copies=3 mypool/projects
```

### ZFS Deduplication，文件去重
ZFS dedup 将丢弃重复数据块，并以到现有数据块的引用来替代. 这将节约磁盘空间，这**需要大量的内存**. 内存中的去重记录表需要消耗大约 ~320 bytes/block. 表格尺寸越大，写入时就会越慢.

```
$ sudo zfs set dedup=on mypool/projects # 启用去重
```