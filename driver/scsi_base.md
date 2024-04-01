# scsi
ref:
- [Linux Scsi子系统框架介绍](https://www.cnblogs.com/Linux-tech/p/13873882.html)

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

## scsi硬件拓扑
scsi硬件拓扑:
- host: 有scsi功能(发送和接收scsi命令)的控制器

    一个scsi Host可能拥有多个channel，每个channel拥有一条scsi总线

    host编号是linux scsi子系统管理

    scsi子系统内部针对每个host控制器对应了Scsi_Host(以scsi_host_template为模板), 其内部有两个structdevice结构体：shost_gendev, shost_dev.

    host是在pci子系统完成ID匹配时添加, 也可以手工添加比如UNH iSCSI启动器.
- 物理device: host连接的物理scsi设备

    对于并行scsi, channel(scsi总线编号)和id(设备标识符)都是物理的, 但对于SAS和iscsi而言, channel和id对scsi而言没有实质意义. 它们的编号是底层驱动自行管理的. linux scsi子系统创造了一个target(scsi_target)=host编号+channel编号+id编号的概念.

    每个scsi target可能拥有多个lun. scsi_device就是对lun的抽象. 为大多数磁盘产商仅实现了lun0.

    这些lun就是可以接受scsi命令的实体, 例如可以是硬盘, cdrom, 磁带等等, 也可以是一些可以接收特殊scsi命令的wlun; 也有一些lun不是物理实体但是能接收scsi命令，也被看作为lun

主要bus和class:
- `scsi` bus：所有host，target，lun都有对应的struct device放在这上面

    通用的scsi的磁盘驱动`sd`，光盘驱动`sr`，磁带驱动`osst`等驱动也在这个bus上面, 这些驱动通过struct device被激活

    shost_gendev在其上. 其scsi_bus_match不允许target不有driver，所以目前只有一些attribute可以在用户空间使用函数中的`if (dev->type != &scsi_dev_type)`不允许shost_gendev有driver对应, 所以目前只有一些attribute可以在用户空间使用

    在scsi内部针对每个target创建了一个名字为`targetk:m:n`的device结构体(其中k是host编号，m是channel编号，n是id编号), 它也在其上, 同样scsi_bus_match不允许target不有driver，所以目前只有一些attribute可以在用户空间使用

    sdev_gendev挂在`scsi` bus上，它会触发bus驱动，驱动会通过sdev_gendev->type字段，来判断该device是否和自己匹配. hdd的device会触发名为`sd`的驱动. sd驱动会给匹配上的lun，在用户空间创建对应的block设备节点，类似于sda，sdb这些（sda1,sda2是sda上GPT或者MBR搞出来的逻辑分区，不属于scsi内容）.

    在`scsi bus上挂着很多驱动, 比如sd, sr. 这些驱动都通过scsi_register_driver注册到`scsi` bus上. 这些公版驱动有针对硬盘的，磁带的，光驱的，扫描仪，ROM等等各种设备的驱动. 基本上这些驱动都会在自己的probe里面去查看sdev_gendev->type字段，判断该device是否和自己匹配, 比如[sd_probe](https://elixir.bootlin.com/linux/v6.6.12/source/drivers/scsi/sd.c#L3623), 只有符合指定类型的设备，才会触发对应的驱动程序.

    在scsi总线probe的过程中，scsi middle level会为每个lun抽象成scsi device，实现的核心函数为scsi_probe_and_add_lun().

    在SCSI子系统中，不论是Scsi_Host实例、还是scsi_target实例抑或scsi_device实例，都被看作”设备”被挂接到scsi_bus_type的`设备链表`中.
- `scsi_host` class: host有对应的device即shost_dev寄存在这上面，通过host的struct device的attr(group,type)获取到控制器的属性. 例如可以通过这上面的scan触发系统做对整个host做scan动作
- `scsi_device` class:所有lun的对应的struct device寄存在这上面. 操作它们的驱动是sg.c

    struct scsi_device有两个device对象，分别是sdev_gendev和sdev_dev. 它们的名字都是k:m:n:lunN，其中k是host编号，m是channel编号，n是id编号，lunN是lun编号.

    sdev_dev是挂在名为`scsi_device`的class上，用作它用. 其中比较重要的sg.c驱动，它在这个class上注册了interface(callback), 当有device挂在这个class上时, interface会被调用，从而间接的创建对应的char设备. sg比较特殊，它会不加区分的给所有进来的lun创建一个对应的字符设备到用户空间, 类似于sg0，sg1.

## 驱动
根据bus-device-driver模型，在SCSI子系统中，其核心类与对象大体可以分为以下6类：
1. 对总线的抽象->struct bus_type scsi_bus_type
1. 对总线控制器的抽象->struct scsi_host_template, struct Scsi_Host
1. 对总线控制器驱动的抽象->None
1. 对总线设备的抽象->struct scsi_target, struct scsi_device, struct scsi_disk
1. 对总线设备驱动的抽象->struct scsi_driver
1. 对命令的抽象->struct scsi_cmnd

核心的对象有：scsi_bus_type, sdev_class, shost_class, sd_disk_class, 这些对象在内核初始化SCSI子系统时候即创建好，包括总线级的实例scsi_bus_type以及为各种SCSI对象准备的class，它们负责将后续注册到内核的SCSI相关对象通过sysfs融入内核设备驱动框架中，进而被用户使用.

scsi_bus_type，封装了SCSI总线级的共性资源，比如总线名、匹配函数等等

shost_class/sdev_class/sdisk_class这些class是内核用来组织SCSI总线下的资源的，shost用于组织sysfs中的scsi_host对象，sdev_class用于组织内核中的scsi_device对象，sdisk_class用于组织内核中的scsi_disk对象.

sg和ses是通过`scsi_device->sdev_dev`->sdev_class来管理scsi设备.

scsi子系统代码在`drivers/scsi`下, 其主要作用:
1. 探测scsi设备, 在内存建立可供设备驱动使用的数据结构
1. 在sysfs构建scsi子系统的目录树
1. scsi高层驱动绑定scsi设备, 在内存中构建对应的数据结构
1. 提供错误恢复api, 在scsi命令错误和超时后被调用

scsi_host，scsi_target和scsi_device均使用hostdata指向供低层驱动使用的资源.

scsi_host_template的调用函数中, 异常处理占了很大部分. 对于一个I/O, 执行结果分为:
1. I/O命令执行完成, 且I/O执行成功
1. I/O命令执行完成, 但I/O未成功, 返回有错误

    再次执行I/O命令或交给scsi错误处理任务处理
1. I/O超时未返回

    必须给scsi错误处理任务处理.

    scsi错误处理任务通常首先abort出错的命令, 如果不行, reset硬盘设备; 还是不行, reset总线; 还是不行, reset整个芯片.   

```c
// https://elixir.bootlin.com/linux/v6.6.14/source/include/scsi/scsi_host.h#L42 
struct scsi_host_template {
    /*
     * Put fields referenced in IO submission path together in
     * same cacheline
     */

    /*
     * Additional per-command data allocated for the driver.
     */
    unsigned int cmd_size;

    /*
     * The queuecommand function is used to queue up a scsi
     * command block to the LLDD.  When the driver finished
     * processing the command the done callback is invoked.
     *
     * If queuecommand returns 0, then the driver has accepted the
     * command.  It must also push it to the HBA if the scsi_cmnd
     * flag SCMD_LAST is set, or if the driver does not implement
     * commit_rqs.  The done() function must be called on the command
     * when the driver has finished with it. (you may call done on the
     * command before queuecommand returns, but in this case you
     * *must* return 0 from queuecommand).
     *
     * Queuecommand may also reject the command, in which case it may
     * not touch the command and must not call done() for it.
     *
     * There are two possible rejection returns:
     *
     *   SCSI_MLQUEUE_DEVICE_BUSY: Block this device temporarily, but
     *   allow commands to other devices serviced by this host.
     *
     *   SCSI_MLQUEUE_HOST_BUSY: Block all devices served by this
     *   host temporarily.
     *
         * For compatibility, any other non-zero return is treated the
         * same as SCSI_MLQUEUE_HOST_BUSY.
     *
     * NOTE: "temporarily" means either until the next command for#
     * this device/host completes, or a period of time determined by
     * I/O pressure in the system if there are no other outstanding
     * commands.
     *
     * STATUS: REQUIRED
     */
    int (* queuecommand)(struct Scsi_Host *, struct scsi_cmnd *); // 将scsi命令排入lldd的队列. scsi中间层调用该函数向hba发送scsi命令

    /*
     * The commit_rqs function is used to trigger a hardware
     * doorbell after some requests have been queued with
     * queuecommand, when an error is encountered before sending
     * the request with SCMD_LAST set.
     *
     * STATUS: OPTIONAL
     */
    void (*commit_rqs)(struct Scsi_Host *, u16);

    struct module *module; // 指向实现了这个host模板的底层驱动模块
    const char *name; // scsi hda驱动名称

    /*
     * The info function will return whatever useful information the
     * developer sees fit.  If not provided, then the name field will
     * be used instead.
     *
     * Status: OPTIONAL
     */
    const char *(*info)(struct Scsi_Host *);

    /*
     * Ioctl interface
     *
     * Status: OPTIONAL
     */
    int (*ioctl)(struct scsi_device *dev, unsigned int cmd,
             void __user *arg); // ioctl接口


#ifdef CONFIG_COMPAT
    /*
     * Compat handler. Handle 32bit ABI.
     * When unknown ioctl is passed return -ENOIOCTLCMD.
     *
     * Status: OPTIONAL
     */
    int (*compat_ioctl)(struct scsi_device *dev, unsigned int cmd,
                void __user *arg); // ioctl接口, 在64位内核进行32位系统调用时用
#endif

    int (*init_cmd_priv)(struct Scsi_Host *shost, struct scsi_cmnd *cmd);
    int (*exit_cmd_priv)(struct Scsi_Host *shost, struct scsi_cmnd *cmd);

    /*
     * This is an error handling strategy routine.  You don't need to
     * define one of these if you don't want to - there is a default
     * routine that is present that should work in most cases.  For those
     * driver authors that have the inclination and ability to write their
     * own strategy routine, this is where it is specified.  Note - the
     * strategy routine is *ALWAYS* run in the context of the kernel eh
     * thread.  Thus you are guaranteed to *NOT* be in an interrupt
     * handler when you execute this, and you are also guaranteed to
     * *NOT* have any other commands being queued while you are in the
     * strategy routine. When you return from this function, operations
     * return to normal.
     *
     * See scsi_error.c scsi_unjam_host for additional comments about
     * what this function should and should not be attempting to do.
     *
     * Status: REQUIRED (at least one of them)
     */
    int (* eh_abort_handler)(struct scsi_cmnd *); // 错误恢复动作: 放弃给定的命令
    int (* eh_device_reset_handler)(struct scsi_cmnd *); // 错误恢复动作: scsi设备复位
    int (* eh_target_reset_handler)(struct scsi_cmnd *); // 错误恢复动作: target复位
    int (* eh_bus_reset_handler)(struct scsi_cmnd *); // 错误恢复动作: scsi总线复位
    int (* eh_host_reset_handler)(struct scsi_cmnd *); // 错误恢复动作: 主机适配器复位

    /*
     * Before the mid layer attempts to scan for a new device where none
     * currently exists, it will call this entry in your driver.  Should
     * your driver need to allocate any structs or perform any other init
     * items in order to send commands to a currently unused target/lun
     * combo, then this is where you can perform those allocations.  This
     * is specifically so that drivers won't have to perform any kind of
     * "is this a new device" checks in their queuecommand routine,
     * thereby making the hot path a bit quicker.
     *
     * Return values: 0 on success, non-0 on failure
     *
     * Deallocation:  If we didn't find any devices at this ID, you will
     * get an immediate call to slave_destroy().  If we find something
     * here then you will get a call to slave_configure(), then the
     * device will be used for however long it is kept around, then when
     * the device is removed from the system (or * possibly at reboot
     * time), you will then get a call to slave_destroy().  This is
     * assuming you implement slave_configure and slave_destroy.
     * However, if you allocate memory and hang it off the device struct,
     * then you must implement the slave_destroy() routine at a minimum
     * in order to avoid leaking memory
     * each time a device is tore down.
     *
     * Status: OPTIONAL
     */
    int (* slave_alloc)(struct scsi_device *); // 在扫描到一个新的scsi设备后调用, 可以为其分配私有的device结构. 大多数scsi设备都有专门的私有数据

    /*
     * Once the device has responded to an INQUIRY and we know the
     * device is online, we call into the low level driver with the
     * struct scsi_device *.  If the low level device driver implements
     * this function, it *must* perform the task of setting the queue
     * depth on the device.  All other tasks are optional and depend
     * on what the driver supports and various implementation details.
     * 
     * Things currently recommended to be handled at this time include:
     *
     * 1.  Setting the device queue depth.  Proper setting of this is
     *     described in the comments for scsi_change_queue_depth.
     * 2.  Determining if the device supports the various synchronous
     *     negotiation protocols.  The device struct will already have
     *     responded to INQUIRY and the results of the standard items
     *     will have been shoved into the various device flag bits, eg.
     *     device->sdtr will be true if the device supports SDTR messages.
     * 3.  Allocating command structs that the device will need.
     * 4.  Setting the default timeout on this device (if needed).
     * 5.  Anything else the low level driver might want to do on a device
     *     specific setup basis...
     * 6.  Return 0 on success, non-0 on error.  The device will be marked
     *     as offline on error so that no access will occur.  If you return
     *     non-0, your slave_destroy routine will never get called for this
     *     device, so don't leave any loose memory hanging around, clean
     *     up after yourself before returning non-0
     *
     * Status: OPTIONAL
     */
    int (* slave_configure)(struct scsi_device *); // 在收到scsi设备的INQUIRY响应后调用, 可以进行特定的设置

    /*
     * Immediately prior to deallocating the device and after all activity
     * has ceased the mid layer calls this point so that the low level
     * driver may completely detach itself from the scsi device and vice
     * versa.  The low level driver is responsible for freeing any memory
     * it allocated in the slave_alloc or slave_configure calls. 
     *
     * Status: OPTIONAL
     */
    void (* slave_destroy)(struct scsi_device *); // 在销毁scsi设备之前调用, 释放关联的私有device结构

    /*
     * Before the mid layer attempts to scan for a new device attached
     * to a target where no target currently exists, it will call this
     * entry in your driver.  Should your driver need to allocate any
     * structs or perform any other init items in order to send commands
     * to a currently unused target, then this is where you can perform
     * those allocations.
     *
     * Return values: 0 on success, non-0 on failure
     *
     * Status: OPTIONAL
     */
    int (* target_alloc)(struct scsi_target *); // 在发现新的scsi target后调用, 可分配私有target结构

    /*
     * Immediately prior to deallocating the target structure, and
     * after all activity to attached scsi devices has ceased, the
     * midlayer calls this point so that the driver may deallocate
     * and terminate any references to the target.
     *
     * Status: OPTIONAL
     */
    void (* target_destroy)(struct scsi_target *); // 在销毁scsi target 前调用, 可释放关联的私有target结构

    /*
     * If a host has the ability to discover targets on its own instead
     * of scanning the entire bus, it can fill in this function and
     * call scsi_scan_host().  This function will be called periodically
     * until it returns 1 with the scsi_host and the elapsed time of
     * the scan in jiffies.
     *
     * Status: OPTIONAL
     */
    int (* scan_finished)(struct Scsi_Host *, unsigned long); // 如果自定义了扫描逻辑, 需要实现这个回调

    /*
     * If the host wants to be called before the scan starts, but
     * after the midlayer has set up ready for the scan, it can fill
     * in this function.
     *
     * Status: OPTIONAL
     */
    void (* scan_start)(struct Scsi_Host *); // 如果自定义了扫描逻辑, 需要实现这个回调

    /*
     * Fill in this function to allow the queue depth of this host
     * to be changeable (on a per device basis).  Returns either
     * the current queue depth setting (may be different from what
     * was passed in) or an error.  An error should only be
     * returned if the requested depth is legal but the driver was
     * unable to set it.  If the requested depth is illegal, the
     * driver should set and return the closest legal queue depth.
     *
     * Status: OPTIONAL
     */
    int (* change_queue_depth)(struct scsi_device *, int); // 用于改变主机适配器队列深度的回调函数

    /*
     * This functions lets the driver expose the queue mapping
     * to the block layer.
     *
     * Status: OPTIONAL
     */
    void (* map_queues)(struct Scsi_Host *shost);

    /*
     * SCSI interface of blk_poll - poll for IO completions.
     * Only applicable if SCSI LLD exposes multiple h/w queues.
     *
     * Return value: Number of completed entries found.
     *
     * Status: OPTIONAL
     */
    int (* mq_poll)(struct Scsi_Host *shost, unsigned int queue_num);

    /*
     * Check if scatterlists need to be padded for DMA draining.
     *
     * Status: OPTIONAL
     */
    bool (* dma_need_drain)(struct request *rq);

    /*
     * This function determines the BIOS parameters for a given
     * harddisk.  These tend to be numbers that are made up by
     * the host adapter.  Parameters:
     * size, device, list (heads, sectors, cylinders)
     *
     * Status: OPTIONAL
     */
    int (* bios_param)(struct scsi_device *, struct block_device *,
            sector_t, int []); // 返回磁盘的bios参数(柱面数, 磁道数, 扇区树等)

    /*
     * This function is called when one or more partitions on the
     * device reach beyond the end of the device.
     *
     * Status: OPTIONAL
     */
    void (*unlock_native_capacity)(struct scsi_device *);

    /*
     * Can be used to export driver statistics and other infos to the
     * world outside the kernel ie. userspace and it also provides an
     * interface to feed the driver with information.
     *
     * Status: OBSOLETE
     */
    int (*show_info)(struct seq_file *, struct Scsi_Host *);
    int (*write_info)(struct Scsi_Host *, char *, int);

    /*
     * This is an optional routine that allows the transport to become
     * involved when a scsi io timer fires. The return value tells the
     * timer routine how to finish the io timeout handling.
     *
     * Status: OPTIONAL
     */
    enum scsi_timeout_action (*eh_timed_out)(struct scsi_cmnd *); // 在中间层发现scsi命令超时时, 将调用底层驱动的这个回调. 使底层有机会修正错误, 正常完成命令; 或延长超时时间, 重新计时; 或进行错误恢复
    /*
     * Optional routine that allows the transport to decide if a cmd
     * is retryable. Return true if the transport is in a state the
     * cmd should be retried on.
     */
    bool (*eh_should_retry_cmd)(struct scsi_cmnd *scmd);

    /* This is an optional routine that allows transport to initiate
     * LLD adapter or firmware reset using sysfs attribute.
     *
     * Return values: 0 on success, -ve value on failure.
     *
     * Status: OPTIONAL
     */

    int (*host_reset)(struct Scsi_Host *shost, int reset_type);
#define SCSI_ADAPTER_RESET  1
#define SCSI_FIRMWARE_RESET 2


    /*
     * Name of proc directory
     */
    const char *proc_name; // proc目录名

    /*
     * This determines if we will use a non-interrupt driven
     * or an interrupt driven scheme.  It is set to the maximum number
     * of simultaneous commands a single hw queue in HBA will accept.
     */
    int can_queue; // 可以同时接受的命令数, 必须大于零

    /*
     * In many instances, especially where disconnect / reconnect are
     * supported, our host also has an ID on the SCSI bus.  If this is
     * the case, then it must be reserved.  Please set this_id to -1 if
     * your setup is in single initiator mode, and the host lacks an
     * ID.
     */
    int this_id; // 在很多情况下, 尤其是支持断开/重连时, 主机适配器在scsi总线上占用一个id

    /*
     * This determines the degree to which the host adapter is capable
     * of scatter-gather.
     */
    unsigned short sg_tablesize; // 反映了host支持聚散列表的能力
    unsigned short sg_prot_tablesize;

    /*
     * Set this if the host adapter has limitations beside segment count.
     */
    unsigned int max_sectors; // 单个scsi命令能访问扇区的最大数目

    /*
     * Maximum size in bytes of a single segment.
     */
    unsigned int max_segment_size;

    /*
     * DMA scatter gather segment boundary limit. A segment crossing this
     * boundary will be split in two.
     */
    unsigned long dma_boundary; // dma聚散段边界限制, 跨越这个边界的段将被分割为两个

    unsigned long virt_boundary_mask;

    /*
     * This specifies "machine infinity" for host templates which don't
     * limit the transfer size.  Note this limit represents an absolute
     * maximum, and may be over the transfer limits allowed for
     * individual devices (e.g. 256 for SCSI-1).
     */
#define SCSI_DEFAULT_MAX_SECTORS    1024

    /*
     * True if this host adapter can make good use of linked commands.
     * This will allow more than one command to be queued to a given
     * unit on a given host.  Set this to the maximum number of command
     * blocks to be provided for each device.  Set this to 1 for one
     * command block per lun, 2 for two, etc.  Do not set this to 0.
     * You should make sure that the host adapter will do the right thing
     * before you try setting this above 1.
     */
    short cmd_per_lun; // 允许排入连接到host的scsi设备的最大命令数目, 即队列深度

    /* If use block layer to manage tags, this is tag allocation policy */
    int tag_alloc_policy;

    /*
     * Track QUEUE_FULL events and reduce queue depth on demand.
     */
    unsigned track_queue_depth:1;

    /*
     * This specifies the mode that a LLD supports.
     */
    unsigned supported_mode:2; // 底层驱动支持的模式(启动器或目标器)

    /*
     * True for emulated SCSI host adapters (e.g. ATAPI).
     */
    unsigned emulated:1; // 是否仿真的host, 比如ATAPI

    /*
     * True if the low-level driver performs its own reset-settle delays.
     */
    unsigned skip_settle_delay:1; // 在host复位和总线复位, 底层驱动自行执行reset-settle延迟

    /* True if the controller does not support WRITE SAME */
    unsigned no_write_same:1;

    /* True if the host uses host-wide tagspace */
    unsigned host_tagset:1;

    /* The queuecommand callback may block. See also BLK_MQ_F_BLOCKING. */
    unsigned queuecommand_may_block:1;

    /*
     * Countdown for host blocking with no commands outstanding.
     */
    unsigned int max_host_blocked; // 如果host没有待处理的命令, 则暂时阻塞它， 以便在派发队列中累积足够多针对它的命令. 每次scsi策略例程处理请求时, 将减1, 报告host队列未准备好, 直到0才恢复正常操作, 即开始派发命令到底层驱动

    /*
     * Default value for the blocking.  If the queue is empty,
     * host_blocked counts down in the request_fn until it restarts
     * host operations as zero is reached.  
     *
     * FIXME: This should probably be a value in the template
     */
#define SCSI_DEFAULT_HOST_BLOCKED   7

    /*
     * Pointer to the SCSI host sysfs attribute groups, NULL terminated.
     */
    const struct attribute_group **shost_groups;

    /*
     * Pointer to the SCSI device attribute groups for this host,
     * NULL terminated.
     */
    const struct attribute_group **sdev_groups;

    /*
     * Vendor Identifier associated with the host
     *
     * Note: When specifying vendor_id, be sure to read the
     *   Vendor Type and ID formatting requirements specified in
     *   scsi_netlink.h
     */
    u64 vendor_id; // 产商标识符

    /* Delay for runtime autosuspend */
    int rpm_autosuspend_delay;
};


// https://elixir.bootlin.com/linux/v6.6.14/source/include/scsi/scsi_host.h#L535
struct Scsi_Host {
    /*
     * __devices is protected by the host_lock, but you should
     * usually use scsi_device_lookup / shost_for_each_device
     * to access it and don't care about locking yourself.
     * In the rare case of being in irq context you can use
     * their __ prefixed variants with the lock held. NEVER
     * access this list directly from a driver.
     */
    struct list_head    __devices; // scsi设备列表
    struct list_head    __targets; // target(目标节点)
    
    struct list_head    starved_list; // 由于host的处理能力, 发送到scsi设备(by starved_entry)的请求队列中的请求可能无法被立即派发, 这时, 将scsi设备链入`饥饿`链表, 等待合适时机在处理

    spinlock_t      default_lock; // 包含host的自旋锁
    spinlock_t      *host_lock; // 指向default_lock

    struct mutex        scan_mutex;/* serialize scanning activity */ // 异步扫描的互斥量

    struct list_head    eh_abort_list;
    struct list_head    eh_cmd_q; // 进入错误恢复的scsi命令链表的表头
    struct task_struct    * ehandler;  /* Error recovery thread. */ // 错误处理线程, 名称`scsi_eh_<host_no>`. 在scsi命令出现错误或超时时, 会被启动, 同时阻塞host, 不再派发scsi命令, 当所有scsi命令都已结束后, 开始恢复动作
    struct completion     * eh_action; /* Wait for specific actions on the
                          host. */ // .在发送一个用于错误恢复的scsi命令后, 需要等待其完成才能继续. 指向此同步目的的完成变量
    wait_queue_head_t       host_wait; // scsi设备错误恢复等待队列. 在scsi设备错误恢复过程中, 要操作scsi设备的进程将在此队列等待, 直至恢复完成
    const struct scsi_host_template *hostt; // 指向scsi_host_template
    struct scsi_transport_template *transportt; // 指向scsi传输层模板

    struct kref     tagset_refcnt;
    struct completion   tagset_freed;
    /* Area to keep a shared tag map */
    struct blk_mq_tag_set   tag_set;

    atomic_t host_blocked; // 阻塞计数器

    unsigned int host_failed;      /* commands that failed.
                          protected by host_lock */ // 失败的命令数
    unsigned int host_eh_scheduled;    /* EH scheduled without command */ // 一般情况下, 错误恢复都是因出现错误命令后调度的. 在没有错误命令情况下调度错误恢复的次数
    
    unsigned int host_no;  /* Used for IOCTL_GET_IDLUN, /proc/scsi et al. */ // 系统内标识host的唯一编号. 由全局scsi_host_next_hn控制

    /* next two fields are used to bound the time spent in error handling */
    int eh_deadline;
    unsigned long last_reset; // 如果有效, 它记录了上次复位的实际, 以jiffie为单位. 在提交命令到host前, 必须确保超过上次复位时间2s


    /*
     * These three parameters can be used to allow for wide scsi,
     * and for host adapters that support multiple busses
     * The last two should be set to 1 more than the actual max id
     * or lun (e.g. 8 for SCSI parallel systems).
     */
    unsigned int max_channel; // 连接到这个host的最大通道编号
    unsigned int max_id; // 连接到这个host的最大target编号
    u64 max_lun; // 连接到这个host的最大lun编号

    /*
     * This is a unique identifier that must be assigned so that we
     * have some way of identifying each detected host adapter properly
     * and uniquely.  For hosts that do not support more than one card
     * in the system at one time, this does not need to be set.  It is
     * initialized to 0 in scsi_register.
     */
    unsigned int unique_id; // host相互区别的唯一标识

    /*
     * The maximum length of SCSI commands that this host can accept.
     * Probably 12 for most host adapters, but could be 16 for others.
     * or 260 if the driver supports variable length cdbs.
     * For drivers that don't set this field, a value of 12 is
     * assumed.
     */
    unsigned short max_cmd_len; // 可以接受的scsi命令的最大长度

    int this_id; // scsi id
    int can_queue; // 可以同时接受的命令数, 必须大于零 
    short cmd_per_lun;
    short unsigned int sg_tablesize; // 支持聚散列表的能力
    short unsigned int sg_prot_tablesize;
    unsigned int max_sectors;
    unsigned int opt_sectors;
    unsigned int max_segment_size;
    unsigned long dma_boundary;
    unsigned long virt_boundary_mask;
    /*
     * In scsi-mq mode, the number of hardware queues supported by the LLD.
     *
     * Note: it is assumed that each hardware queue has a queue depth of
     * can_queue. In other words, the total queue depth per host
     * is nr_hw_queues * can_queue. However, for when host_tagset is set,
     * the total queue depth is can_queue.
     */
    unsigned nr_hw_queues;
    unsigned nr_maps;
    unsigned active_mode:2; // 当前激活模式(启动器或目标器)

    /*
     * Host has requested that no further requests come through for the
     * time being.
     */
    unsigned host_self_blocked:1; // 1, 底层驱动要求阻塞host即scsi中间层不要继续派发命令到其队列
    
    /*
     * Host uses correct SCSI ordering not PC ordering. The bit is
     * set for the minority of drivers whose authors actually read
     * the spec ;).
     */
    unsigned reverse_ordering:1; // 1, 按逆序(从max_id-1到0)扫描scsi总线

    /* Task mgmt function in progress */
    unsigned tmf_in_progress:1; // 1, 当前正在执行任务管理功能

    /* Asynchronous scan in progress */
    unsigned async_scan:1; // 1, 在执行异步扫描

    /* Don't resume host in EH */
    unsigned eh_noresume:1;

    /* The controller does not support WRITE SAME */
    unsigned no_write_same:1;

    /* True if the host uses host-wide tagspace */
    unsigned host_tagset:1;

    /* The queuecommand callback may block. See also BLK_MQ_F_BLOCKING. */
    unsigned queuecommand_may_block:1;

    /* Host responded with short (<36 bytes) INQUIRY result */
    unsigned short_inquiry:1;

    /* The transport requires the LUN bits NOT to be stored in CDB[1] */
    unsigned no_scsi2_lun_in_cdb:1;

    /*
     * Optional work queue to be utilized by the transport
     */
    char work_q_name[20]; // 工作队列名称, , 被传输层使用
    struct workqueue_struct *work_q; // 工作队列, 被传输层使用

    /*
     * Task management function work queue
     */
    struct workqueue_struct *tmf_work_q;

    /*
     * Value host_blocked counts down from
     */
    unsigned int max_host_blocked;

    /* Protection Information */
    unsigned int prot_capabilities; // 保护能力(是否支持DIF和或DIX)
    unsigned char prot_guard_type; // 护卫标签值计算方式(CRC16或IP校验和)

    /* legacy crap */
    unsigned long base; // MMIO基地址
    unsigned long io_port; // I/O端口编号
    unsigned char n_io_port; // 使用的I/O空间字节数
    unsigned char dma_channel; // DMA通道, 新的设备驱动不使用
    unsigned int  irq; // irq号
    

    enum scsi_host_state shost_state; // 状态

    /* ldm bits */
    struct device       shost_gendev, shost_dev; // shost_gendev链入scsi_bus_type的设备链表; shost_dev链入shost_class的设备链表

    /*
     * Points to the transport data (if any) which is allocated
     * separately
     */
    void *shost_data; // 指向分配的传输层数据(如果有)

    /*
     * Points to the physical bus device we'd use to do DMA
     * Needed just in case we have virtual hosts.
     */
    struct device *dma_dev; // 用来进行dma的物理总线设备, 仅为`虚拟`host所需

    /*
     * We should ensure that this is aligned, both for better performance
     * and also because some compilers (m68k) don't automatically force
     * alignment to a long boundary.
     */
    unsigned long hostdata[]  /* Used for storage of host specific stuff */ // 用以保存host的专有数据, 只被scsi底层驱动使用
        __attribute__ ((aligned (sizeof(unsigned long))));
};

// https://elixir.bootlin.com/linux/v6.6.14/source/include/scsi/scsi_device.h#L338
/* SCSI目标节点描述符 */
/*
 * scsi_target: representation of a scsi target, for now, this is only
 * used for single_lun devices. If no one has active IO to the target,
 * starget_sdev_user is NULL, else it points to the active sdev.
 */
struct scsi_target {
    struct scsi_device  *starget_sdev_user; // 用在scsi target一次只允许对一个逻辑单元进行I/O的场合. 如果没有I/O, 则为NULL; 否则指向正在进行I/O的scsi设备
    struct list_head    siblings; // 链入Scsi_Host.__targets
    struct list_head    devices; // scsi设备链表
    struct device       dev;
    struct kref     reap_ref; /* last put renders target invisible */ // 引用计数, 用于回收
    unsigned int        channel; // 所在的通道号
    unsigned int        id; /* target id ... replace
                     * scsi_device.id eventually */
    unsigned int        create:1; /* signal that it needs to be added */ // 1, 需要被添加
    unsigned int        single_lun:1;   /* Indicates we should only
                         * allow I/O to one of the luns
                         * for the device at a time. */ // 1, 表明一次只允许对target的一个lun进行I/O
    unsigned int        pdt_1f_for_no_lun:1;    /* PDT = 0x1f
                         * means no lun present. */ // 1, INQUIRY响应中外围设备类型(Peripheral Device Type), 0x1f表示没有连接逻辑单元
    unsigned int        no_report_luns:1;   /* Don't use
                         * REPORT LUNS for scanning. */
    unsigned int        expecting_lun_change:1; /* A device has reported
                         * a 3F/0E UA, other devices on
                         * the same target will also. */
    /* commands actually active on LLD. */
    atomic_t        target_busy; // 可以同时处理的命令数, 必须大于0
    atomic_t        target_blocked; // 阻塞计数器

    /*
     * LLDs should set this in the slave_alloc host template callout.
     * If set to zero then there is not limit.
     */
    unsigned int        can_queue;
    unsigned int        max_target_blocked; // 参考max_host_blocked
#define SCSI_DEFAULT_TARGET_BLOCKED 3

    char            scsi_level; // 支持的scsi规范级别
    enum scsi_target_state  state;
    void            *hostdata; /* available to low-level driver */ // 指向target专有数据(如果有的话)
    unsigned long       starget_data[]; /* for the transport */ // 用于传输层
    /* starget_data must be the last element!!!! */
} __attribute__((aligned(sizeof(unsigned long))));

// https://elixir.bootlin.com/linux/v6.6.14/source/include/scsi/scsi_device.h#L107
struct scsi_device {
    struct Scsi_Host *host; // 指向Scsi_Host
    struct request_queue *request_queue; // 指向这个scsi设备的请求队列

    /* the next two are protected by the host->host_lock */
    struct list_head    siblings;   /* list of all devices on this host */ // // 链入Scsi_Host.__devices
    struct list_head    same_target_siblings; /* just the devices sharing same target id */ // 链入scsi_target.devices

    struct sbitmap budget_map;
    atomic_t device_blocked;    /* Device returned QUEUE_FULL. */

    atomic_t restarts;
    spinlock_t list_lock; // 用于保护scsi_device的某些字段的自旋锁
    struct list_head starved_entry; // 链入host的`starved_list`的连接件
    unsigned short queue_depth; /* How deep of a queue we want */ // 队列深度即允许排入队列的最多命令数
    unsigned short max_queue_depth; /* max queue depth */
    unsigned short last_queue_full_depth; /* These two are used by */ // 记录上次报告队列满时设备中的活动命令数. 用于跟踪QUEUE_FULL事件以确定是否要, 以及何时调整设备队列深度
    unsigned short last_queue_full_count; /* scsi_track_queue_full() */ // 记录上次报告队列满时设备中的活动命令数未发生变化的次数. 用于跟踪QUEUE_FULL事件以确定是否要, 以及何时调整设备队列深度
    unsigned long last_queue_full_time; /* last queue full time */ // 记录上次报告队列满的时间(jiffie形式). 用于跟踪QUEUE_FULL事件以确定是否要, 以及何时调整设备队列深度
    unsigned long queue_ramp_up_period; /* ramp up period in jiffies */ // 如果设备能正常处理命令, 每经过一个间隔, 即以1的步长逐渐递增其命令深度, 直至达到最大值. 该过程叫"ramp up"
#define SCSI_DEFAULT_RAMP_UP_PERIOD (120 * HZ)

    unsigned long last_queue_ramp_up;   /* last queue ramp up time */ // 记录上次由于ramp up递增设备命令深度的时间(jiffie形式). 加上queue_ramp_up_period的间隔就是下一次可能调整的时间

    unsigned int id, channel; // target id,  通道id
    u64 lun; // lun编号
    unsigned int manufacturer;  /* Manufacturer of device, for using 
                     * vendor-specific cmd's */ // 制造商
    unsigned sector_size;   /* size in bytes */ // 硬件扇区长度(B), 通过Read Capacity命令读到

    void *hostdata;     /* available to low-level driver */ // 专有数据
    unsigned char type; // 类型
    char scsi_level; // 实现scsi规范的版本号, 利用INQUIRY命令获取
    char inq_periph_qual;   /* PQ from INQUIRY data */   // 标准的SCSI INQUIRY响应报文中的Peripheral Qualifier
    struct mutex inquiry_mutex;
    unsigned char inquiry_len;  /* valid bytes in 'inquiry' */ //  INQUIRY内容长度
    unsigned char * inquiry;    /* INQUIRY response data */ // 指向该设备的SCSI INQUIRY响应报文的内容
    const char * vendor;        /* [back_compat] point into 'inquiry' ... */ // INQUIRY响应报文中的产商标识符
    const char * model;     /* ... after scan; point to static string */ // INQUIRY响应报文中的产品标识符
    const char * rev;       /* ... "nullnullnullnull" before scan */ // INQUIRY响应报文中的产品修正号

#define SCSI_DEFAULT_VPD_LEN    255 /* default SCSI VPD page size (max) */
    struct scsi_vpd __rcu *vpd_pg0;
    struct scsi_vpd __rcu *vpd_pg83;
    struct scsi_vpd __rcu *vpd_pg80;
    struct scsi_vpd __rcu *vpd_pg89;
    struct scsi_vpd __rcu *vpd_pgb0;
    struct scsi_vpd __rcu *vpd_pgb1;
    struct scsi_vpd __rcu *vpd_pgb2;

    struct scsi_target      *sdev_target; // 指向所属的目标器, 只用于single_lun的目标器

    blist_flags_t       sdev_bflags; /* black/white flags as also found in
                 * scsi_devinfo.[hc]. For now used only to
                 * pass settings from slave_alloc to scsi
                 * core. */ // 记录scsi设备的额外标志
    unsigned int eh_timeout; /* Error handling timeout */

    /*
     * If true, let the high-level device driver (sd) manage the device
     * power state for system suspend/resume (suspend to RAM and
     * hibernation) operations.
     */
    unsigned manage_system_start_stop:1;

    /*
     * If true, let the high-level device driver (sd) manage the device
     * power state for runtime device suspand and resume operations.
     */
    unsigned manage_runtime_start_stop:1;

    /*
     * If true, let the high-level device driver (sd) manage the device
     * power state for system shutdown (power off) operations.
     */
    unsigned manage_shutdown:1;

    /*
     * If set and if the device is runtime suspended, ask the high-level
     * device driver (sd) to force a runtime resume of the device.
     */
    unsigned force_runtime_start_on_system_start:1;

    unsigned removable:1; // 可移除
    unsigned changed:1; /* Data invalid due to media change */ // 介质已发生改变, 数据无效
    unsigned busy:1;    /* Used to prevent races */ // 防止竞争
    unsigned lockable:1;    /* Able to prevent media removal */ // 可以被上锁, 以防止移除介质
    unsigned locked:1;      /* Media removal disabled */ // 设备已上锁, 不允许移除介质
    unsigned borken:1;  /* Tell the Seagate driver to be 
                 * painfully slow on this device */ // 设备握手问题, 需底层驱动做相应处理
    unsigned disconnect:1;  /* can disconnect */ // 可以断开连接
    unsigned soft_reset:1;  /* Uses soft reset option */ // 使用软复位选项
    unsigned sdtr:1;    /* Device supports SDTR messages */ // 设备支持同步数据传输, 只适用于SPI设备
    unsigned wdtr:1;    /* Device supports WDTR messages */ // 设备支持16位宽数据传输, 只适用于SPI设备
    unsigned ppr:1;     /* Device supports PPR messages */ // 设备支持PPR(并行协议请求)消息
    unsigned tagged_supported:1;    /* Supports SCSI-II tagged queuing */ // 支持SCSI-II Tagged Queuing
    unsigned simple_tags:1; /* simple queue tag messages are enabled */ // 启用simple Queue tag message
    unsigned was_reset:1;   /* There was a bus reset on the bus for // 刚做过复位
                 * this device */
    unsigned expecting_cc_ua:1; /* Expecting a CHECK_CONDITION/UNIT_ATTN
                     * because we did a bus reset. */ // 期望收到check condition/unit attention
    unsigned use_10_for_rw:1; /* first try 10-byte read / write */ // 首先参数10字节的读/写命令
    unsigned use_10_for_ms:1; /* first try 10-byte mode sense/select */ // 首先尝试10字节的mode sense/select命令
    unsigned set_dbd_for_ms:1; /* Set "DBD" field in mode sense */
    unsigned no_report_opcodes:1;   /* no REPORT SUPPORTED OPERATION CODES */
    unsigned no_write_same:1;   /* no WRITE SAME command */
    unsigned use_16_for_rw:1; /* Use read/write(16) over read/write(10) */
    unsigned use_16_for_sync:1; /* Use sync (16) over sync (10) */
    unsigned skip_ms_page_8:1;  /* do not use MODE SENSE page 0x08 */ // 不要使用mode sense命令的page 0x08
    unsigned skip_ms_page_3f:1; /* do not use MODE SENSE page 0x3f */ // 不要使用mode sense命令的page 0x3f
    unsigned skip_vpd_pages:1;  /* do not read VPD pages */
    unsigned try_vpd_pages:1;   /* attempt to read VPD pages */
    unsigned use_192_bytes_for_3f:1; /* ask for 192 bytes from page 0x3f */ // 对mode sense命令page 0x3f, 使用192字节
    unsigned no_start_on_add:1; /* do not issue start on add */ // 添加设备时, 不要自动启动它
    unsigned allow_restart:1; /* issue START_UNIT in error handler */ // 允许在错误处理函数中发送START_UNIT命令. 因为某些硬盘会自动spin down.
    unsigned no_start_on_resume:1; /* Do not issue START_STOP_UNIT on resume */
    unsigned start_stop_pwr_cond:1; /* Set power cond. in START_STOP_UNIT */ // 在START_STOP_UNIT命令中设置电源条件域
    unsigned no_uld_attach:1; /* disable connecting to upper level drivers */ // 禁止设备连接到高层驱动
    unsigned select_no_atn:1; // 设备被选中时, 不需要Assert ATN
    unsigned fix_capacity:1;    /* READ_CAPACITY is too high by 1 */ // READ_CAPACITY多了1个
    unsigned guess_capacity:1;  /* READ_CAPACITY might be too high by 1 */ // READ_CAPACITY可能多1个
    unsigned retry_hwerror:1;   /* Retry HARDWARE_ERROR */ // 即使底层报硬件错误, 也需要重试, 因为某些设备将任何(包括可恢复的)错误都报告为不可恢复的硬件错误
    unsigned last_sector_bug:1; /* do not use multisector accesses on
                       SD_LAST_BUGGY_SECTORS */ // 在访问最后一个扇区时, 按单个硬件扇区来处理. 这是因为某些设备(如SD卡)不能多扇区访问最后的部分扇区
    unsigned no_read_disc_info:1;   /* Avoid READ_DISC_INFO cmds */
    unsigned no_read_capacity_16:1; /* Avoid READ_CAPACITY_16 cmds */
    unsigned try_rc_10_first:1; /* Try READ_CAPACACITY_10 first */
    unsigned security_supported:1;  /* Supports Security Protocols */
    unsigned is_visible:1;  /* is the device visible in sysfs */ // 在sysfs可见
    unsigned wce_default_on:1;  /* Cache is ON by default */
    unsigned no_dif:1;  /* T10 PI (DIF) should be disabled */
    unsigned broken_fua:1;      /* Don't set FUA bit */
    unsigned lun_in_cdb:1;      /* Store LUN bits in CDB[1] */
    unsigned unmap_limit_for_ws:1;  /* Use the UNMAP limit for WRITE SAME */
    unsigned rpm_autosuspend:1; /* Enable runtime autosuspend at device
                     * creation time */
    unsigned ignore_media_change:1; /* Ignore MEDIA CHANGE on resume */
    unsigned silence_suspend:1; /* Do not print runtime PM related messages */
    unsigned no_vpd_size:1;     /* No VPD size reported in header */

    unsigned cdl_supported:1;   /* Command duration limits supported */
    unsigned cdl_enable:1;      /* Enable/disable Command duration limits */

    unsigned int queue_stopped; /* request queue is quiesced */
    bool offline_already;       /* Device offline message logged */

    atomic_t disk_events_disable_depth; /* disable depth for disk events */

    DECLARE_BITMAP(supported_events, SDEV_EVT_MAXBITS); /* supported events */
    DECLARE_BITMAP(pending_events, SDEV_EVT_MAXBITS); /* pending events */
    struct list_head event_list;    /* asserted events */
    struct work_struct event_work;

    unsigned int max_device_blocked; /* what device_blocked counts down from  */
#define SCSI_DEFAULT_DEVICE_BLOCKED 3

    atomic_t iorequest_cnt; // 已经请求的scsi命令的数目
    atomic_t iodone_cnt; // 已经完成的scsi命令的数目
    atomic_t ioerr_cnt; // 已经出错的scsi命令的数目
    atomic_t iotmo_cnt;

    struct device       sdev_gendev, // 链入scsi总线类型(scsi_bus_type)的设备链表
                sdev_dev; // 链入scsi设备类(sdev_class)的设备链表

    struct work_struct  requeue_work;

    struct scsi_device_handler *handler;
    void            *handler_data;

    size_t          dma_drain_len;
    void            *dma_drain_buf;

    unsigned int        sg_timeout;
    unsigned int        sg_reserved_size;

    struct bsg_device   *bsg_dev;
    unsigned char       access_state;
    struct mutex        state_mutex;
    enum scsi_device_state sdev_state;
    struct task_struct  *quiesced_by;
    unsigned long       sdev_data[]; // 用于传输层
} __attribute__((aligned(sizeof(unsigned long))));

// https://elixir.bootlin.com/linux/v6.6.14/source/drivers/scsi/sd.h#L84
struct scsi_disk {
    struct scsi_device *device;

    /*
     * disk_dev is used to show attributes in /sys/class/scsi_disk/,
     * but otherwise not really needed.  Do not use for refcounting.
     */
    struct device   disk_dev;
    struct gendisk  *disk; // 指向表示scsi磁盘的通用磁盘信息的gendisk
    struct opal_dev *opal_dev;
#ifdef CONFIG_BLK_DEV_ZONED
    /* Updated during revalidation before the gendisk capacity is known. */
    struct zoned_disk_info  early_zone_info;
    /* Updated during revalidation after the gendisk capacity is known. */
    struct zoned_disk_info  zone_info;
    u32     zones_optimal_open;
    u32     zones_optimal_nonseq;
    u32     zones_max_open;
    /*
     * Either zero or a power of two. If not zero it means that the offset
     * between zone starting LBAs is constant.
     */
    u32     zone_starting_lba_gran;
    u32     *zones_wp_offset;
    spinlock_t  zones_wp_offset_lock;
    u32     *rev_wp_offset;
    struct mutex    rev_mutex;
    struct work_struct zone_wp_offset_work;
    char        *zone_wp_update_buf;
#endif
    atomic_t    openers; // scsi设备的打开计数器
    sector_t    capacity;   /* size in logical blocks */ // scsi磁盘的容量(以512为单位), 通过READ CAPACITY命令获取
    int     max_retries;
    u32     min_xfer_blocks;
    u32     max_xfer_blocks;
    u32     opt_xfer_blocks;
    u32     max_ws_blocks;
    u32     max_unmap_blocks;
    u32     unmap_granularity;
    u32     unmap_alignment;
    u32     index; // scsi磁盘的索引, 系统内唯一, 用来确定对应通用磁盘的设备号和设备名
    unsigned int    physical_block_size;
    unsigned int    max_medium_access_timeouts;
    unsigned int    medium_access_timed_out;
    u8      media_present; // 1, 这个磁盘由介质存在
    u8      write_prot; // 是否被写保护. 使用Mode Sense命令的page 0x3f获得
    u8      protection_type;/* Data Integrity Field */ // 支持的保护类型, 通过READ CAPACITY 16命令获取
    u8      provisioning_mode;
    u8      zeroing_mode;
    u8      nr_actuators;       /* Number of actuators */
    bool        suspended;  /* Disk is suspended (stopped) */
    unsigned    ATO : 1;    /* state of disk ATO bit */ // ATO(Application Tag Own)位状态. 1, os可以访问DIF三元组中的应用程序标签. 通过Mode Sense命令的page 0x0a获取
    unsigned    cache_override : 1; /* temp override of WCE,RCD */
    unsigned    WCE : 1;    /* state of disk WCE bit */ // WCE(writeback cache enable)位状态. 1, 在写操作时可以使用cache, 只要写入cache就可返回. 0, 必须等待真正写到介质上才返回. 通过Mode Sense命令的page 0x08获取
    unsigned    RCD : 1;    /* state of disk RCD bit, unused */ // RCD(Read Cache Disable)位状态. 1, 读数据时必须从介质中读取. 0, 可以从缓存读取. 通过Mode Sense命令的page 0x08获取
    unsigned    DPOFUA : 1; /* state of disk DPOFUA bit */ // DPO(diable page out)FUA(Force Unit Access)位状态. DPO允许启动器通知目标器写入的数据不会很快被读出. 因此不需要保存在目标器的数据缓存; FUA允许启动器通知目标器立即将写入的数据保存到介质上, 不要保留在缓存中. 1, 直接写到介质, 不要保留在缓存
    unsigned    first_scan : 1; // 1, 第一次扫描. 在探测到scsi磁盘后, 需要在添加磁盘到系统前后均调用sd_revalidate_disk, 用这个标记区分, 防止重复打印冗余信息
    unsigned    lbpme : 1;
    unsigned    lbprz : 1;
    unsigned    lbpu : 1;
    unsigned    lbpws : 1;
    unsigned    lbpws10 : 1;
    unsigned    lbpvpd : 1;
    unsigned    ws10 : 1;
    unsigned    ws16 : 1;
    unsigned    rc_basis: 2;
    unsigned    zoned: 2;
    unsigned    urswrz : 1;
    unsigned    security : 1;
    unsigned    ignore_medium_access_errors : 1;
};
#define to_scsi_disk(obj) container_of(obj, struct scsi_disk, disk_dev)

// https://elixir.bootlin.com/linux/v6.6.14/source/include/scsi/scsi_driver.h#L12
// scsi_driver指的是scsi高层的scsi设备驱动, 比如sd,st,sr等, 而不是在scsi底层是scsi host驱动
struct scsi_driver {
    struct device_driver    gendrv;

    void (*rescan)(struct device *); // 用于重新扫描的回调函数
    blk_status_t (*init_command)(struct scsi_cmnd *);
    void (*uninit_command)(struct scsi_cmnd *);
    int (*done)(struct scsi_cmnd *); // 在底层驱动已经完成一个scsi命令时调用, 用于计算已完成的字节数
    int (*eh_action)(struct scsi_cmnd *, int);
    void (*eh_reset)(struct scsi_cmnd *);
};
```

linux用3个struct共同表示一个scsi磁盘, 每个结构反映其某方面属性:
1. scsi_device: 作为scsi设备方面的属性
1. scsi_disk: 作为scsi磁盘方面的属性
1. gendisk: 作为通用磁盘方面的属性

scsi子系统工作模式: scsi中间层注册scsi总线类型, scsi高层注册scsi设备驱动, scsi底层(scsi hba)负责扫描总线得到所有的scsi设备. scsi设备和scsi设备驱动的匹配是根据SCSI INQUIRY响应中的scsi设备类型类型决定.

对于scsi子系统, scsi_bus_type没有定义probe回调, 而是调用驱动的probe回到.

linux驱动子系统, 一般包含下面几个内容:
1. 子系统初始化：驱动bus的建立，子设备驱动的挂载

    - [init_scsi](https://elixir.bootlin.com/linux/v6.6.14/source/drivers/scsi/scsi.c#L963)

        ```c
        // https://elixir.bootlin.com/linux/v6.6.14/source/drivers/scsi/scsi.c#L963
        static int __init init_scsi(void)
        {
            int error;

            error = scsi_init_procfs(); // 创建一个/proc/scsi/scsi的文件节点. 这个节点会显示当前系统注册了哪些scsi设备，包括这些设备的channel编号,id编号 lun编号等信息. 这些信息都是实时变化的；如果有写入动作，也会触发子系统的scan动作
            if (error)
                goto cleanup_queue;
            error = scsi_init_devinfo(); // 创建/proc/scsi/device_info节点
            if (error)
                goto cleanup_procfs;
            error = scsi_init_hosts(); // 注册shost_class类, 在/sys/class创建scsi_host目录
            if (error)
                goto cleanup_devlist;
            error = scsi_init_sysctl(); // 注册scsi系统控制表, 比如创建一个/proc/sys/dev/scsi/logging_level节点，这个节点控制着scsi子系统debug打印的log等级，值越小，打印越少
            if (error)
                goto cleanup_hosts;
            error = scsi_sysfs_register(); // scsi_init_hosts和scsi_sysfs_register创建了scsi子系统最关键的bus和class（scsi, scsi_host和scsi_device）
            if (error)
                goto cleanup_sysctl;

            scsi_netlink_init(); // 初始化scsi传输netlink接口

            printk(KERN_NOTICE "SCSI subsystem initialized\n");
            return 0;

        cleanup_sysctl:
            scsi_exit_sysctl();
        cleanup_hosts:
            scsi_exit_hosts();
        cleanup_devlist:
            scsi_exit_devinfo();
        cleanup_procfs:
            scsi_exit_procfs();
        cleanup_queue:
            scsi_exit_queue();
            printk(KERN_ERR "SCSI subsystem failed to initialize, error = %d\n",
                   -error);
            return error;
        }
        ```

        子设备驱动加载一般比较简单，而且单独以module形式，耦合性很小. 它们一般在module初始化时注册到`scsi` bus总线上，然后一直等待有对应的子设备sdev_devgen挂到`scsi` bus上来, 比如[init_sd](https://elixir.bootlin.com/linux/v6.6.12/source/drivers/scsi/sd.c#L4022)
1. 外设扫描：对于scsi而言就是把device侧的所有lun扫描出来

    Scsi扫描过程定义： 是识别每个host，每个targe和每个lun，给其创建对应的device结构，并将device挂载到相应的bus或class上.

    设备扫描的方式很多：
    - 以host为单位进行scan。它会把host对应的device下面所有的target和lun全扫描出来

        由于host控制器各个芯片平台不一样，它的扫描过程是host device的父设备所在驱动完成的，它的父设备驱动可以是platform总线，也可以是pcie设备对应的pci_driver.

        比如通过`lspci -v -s 00:13.0`查到是ahci, `00:13.0`是来自`/sys/bus/scsi/devices/`的路径片段

        无论哪种当上一级驱动找到host后，会通过:
        1. scsi_host_alloc：创建shost_gendev和shost_dev
        1. scsi_add_host: 把shost_gendev和shost_dev挂靠到各自的bus或class上

    - 以target为单位触发scsi进行scan。它会把target下面所有的lun全扫出来
    - 以lun为单位触发scsi进行scan。它会扫描特定lun
    - 通过/proc/scsi/scsi触发特定的target或lun的scan
    - 通过host对应user空间设备的属性”scan”节点触发特定的target或lun的scan

    各种scan入口:
    - 以host为单位进行scan:scsi_scan_host

        扫描由scsi_scan_type决定

        向各个`<channel, id, lun>`发送INQUIRY命令
    - 以target为单位触发scsi进行scan:scsi_scan_target
    - 以lun为单位触发scsi进行scan:scsi_add_device或者__scsi_add_device
    - 通过/proc/scsi/scsi:scsi_scan_host_selected
    - 通过host对应user空间设备的属性`scan`节点:scsi_scan_host_selected


    不同的host实现可能采用不同的拓扑发现和设备添加机制, 常见的有2种:
    1. 由HBA固件完成拓扑发现, 驱动从固件获得发现的所有设备, 比如某些的带RAID的host, 具体例子是LSI的MPT驱动
    1. 由HBA Driver负责拓扑发现

        有2种:
        1. 不需要调用scsi中间层提供的服务, 比如SAS, FC, iscsi. iscsi没有拓扑发现过程, 直接由用户手动添加scsi设备
        2. 需要调用scsi中间层提供的服务, 比如SPI(SCSI并行接口)

            scsi中间层依次以可能的ID和LUN构造INQUIRY命令, 再将其交给块I/O子系统, 后者调用scsi中间层的策略例程, 提取到scsi命令后, 调用scsi底层驱动的queuecommand回调函数实现

1. 通路建立：建立子设备驱动和device之间的连接，对于scsi而言就是公版外设驱动和lundevice之间的通路。Scsi子系统是借助block通用块设备层完成这部分工作

    Scsi注册block层有两个方式，一种是single q，另一种是multi q方式，这里介绍multi q的方式.

    注册multi q，需要做两件事情:
    1. 通过blk_mq_alloc_tag_set注册一个blk_mq_tag_set。注册时我们要提供一堆钩子函数给通用块设备层，处理block发下来的request请求。
    1. 通过blk_mq_init_queue并以blk_mq_tag_set为参数为每个能独立处理block请求的实体申请一个request_queue。这样所有的request_queue都和tag_set关联起来了。

    通过上述操作后，所有发送到request_queue中的request都会汇集到tag_set中做处理.

    通过ioctl对sda或者sg设备的命令request都会进入到其对应lun的request_queue, 最终都会走到tag_set的queue_rq钩子函数, 也就是走到了scsi_queue_rq->scsi_dispatch_cmd->host->hostt->queuecommand函数，其中queuecommand是底层驱动注册上来的钩子函数，scsi子系统把request请求发送到这一步之后，剩下的工作就交给底层, 比如sata驱动去处理了.

    > scsi_dispatch_cmd提交scsi命令时, 提供一个scsi_done回调, 该回调由硬盘驱动的中断所调用, 同时也是I/O返回路径上的第一个函数.
1. 休眠唤醒：对于scsi而言，休眠过程是lun->target->host，唤醒过程是反过来。这个决定了host是爷爷辈设备，targe是父设备，lun是子设备，所有的公版驱动都是子设备驱动

    休眠唤醒是驱动的一部分，包括PM(suspendresume)，runtime PM，也有shutdown，remove等。以休眠为例：在”scsi” bus上那些公版driver实现了子设备的休眠唤醒操作. 这个级别的驱动操作的都是lun设备，因此这个级别的驱动是基于scsi命令对设备进行操作。那些更底层的操作例如断开link，给外设断电等是更底层的父设备们去完成的

    - sd_suspend_common: 硬盘驱动sd.c在休眠的时候，给lun发送了scsiSYNCHRONIZE_CACHE命令，要求lun把缓存数据回写到硬盘防止断电丢失，并发送了start_stop命令要求lun进入低功耗状态

    Linux设备驱动模型会保证子设备suspend之后，才会是父设备的suspend，向底层一级一级父辈驱动的suspend调用.

    Scsi里面的父设备target是有channel和id虚拟出来的，没有任何休眠唤醒动作.

scsi低层驱动是面向主机适配器的，低层驱动被加载时，需要添加主机适配器. 主机适配器添加有两种方式：1.在PCI子系统扫描挂载驱动时添加；2.手动方式添加. 所有基于硬件PCI接口的主机适配器都采用第一种方式. 添加主机适配器包括两个步骤：
1. 分别主机适配器数据结构scsi_host_alloc

    scsi_host包含两部分，一部分供SCSI中间层使用，另一部分供SCSI低层驱动专用，两部分一同分配
2. 将主机适配器添加到系统scsi_add_host

可参考aha1542适配器的代码[aha1542_hw_init](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/scsi/aha1542.c#L729).

#### 驱动代码
- [sd.c](https://elixir.bootlin.com/linux/v6.6.12/source/drivers/scsi/sd.c) : 操作的是硬盘，ssd等以sect为单位进行读取写入的存储设备

    “sd”会针对每个匹配上的sdev_gendev，做blk_alloc_disk/blk_mq_alloc_disk和device_add_disk操作, 也就是说在user空间创建对应的块设备节点，例如sda，sdb这些节点.
    “sd”也会在”scsi_disk”class上创建和sdev_gendev同名的device，会有对应group attr和其对应做一些操作.

    Sd设备驱动本身是块设备驱动，它需要使用block相关的request_queue来发送块设备相关请求给lun，而lun和host之间的沟通是通过block层来完成的，每个lun有自己独有的request_queue，因此sd驱动直接把这个request_queue拿来用之，把这个request_queue和本地申请的gendisk进行绑定。sda，sdb这些块设备就可以直接通过request_queue给lun发送请求
- sg.c

    sg.c比较特殊，不是对某个类型的设备驱动. 它不管三七二一，对所有挂到“scsi_device”class上的device，都创建一个char类型的设备节点到user空间. 由于所有被扫描出来的lun会有一个sdev_dev在”scsi_device”上，因此sg实际上是给每个lun创建了char设备节点.

    它也会创建一个同名的sg device挂在自定义的”scsi_generic” class上（没有什么特别作用）.

    sg作用：
    1. Sg存在的唯一目的，是使用ioctl命令，例如rpmb的操作，FFU固件升级等操作，都是通过ioctl方式完成
    1. 由于无论sg还是sd，还是别的什么scsi外设驱动创建出来用户态设备节点，最终都是通过lun对应的request_queue来完成发送scsi命令，所以sg能做的事情，其它节点也能做，因此有的平台没有打开sg编译开关

    ![scsi子系统 ioctl调用关系图](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9kNGhvWUpseE9qUGE4TEsxR1RhUXpWQnJRWGpiaWJROXRpY0JLMWYyS254VHdUY0ZHRGpmUWJnZlcxMmZ1NzNUdzZmdVpkaWNYNTh5QVUyTzQ5dUllTmpwdy82NDA?x-oss-process=image/format,png)

#### queuecommand
scsi_host_template中最重要的当属queuecommand域, queuecommand()是SCSI子系统LLDD的上边界，HLDD准备好SCSI命令的封装之后，通过中间层的接口发送命令，而中间层最终就是通过queuecommand将命令传递到LLDD，由它将SCSI命令由”封装”转变为底层操作下发到SCSI设备，比如，在PCIe RAID卡中，queuecommand()最终转换为对PCIe空间的写，RAID卡固件识别到这些数据后，从中取出SCSI命令的部分，下发到位于其上的SCSI适配器芯片中.

#### scsi_bus_match
scsi_bus_match()提供总线匹配方法，按照match的约定，可以匹配成功时返回非0，失败返回0，其实质是`(sdp->inq_periph_qual == SCSI_INQ_PQ_CON)? 1: 0`，可见SCSI设备对象与SCSI驱动的匹配不依靠name，id等机制，只要设备可用且没有被其他驱动匹配，即可被当前调用的驱动匹配.

SCSI子系统在首先通过适配器发送INQUARY命令探测总线上的设备，设备如果可用即以SCSI_INQ_PQ_CON应答之，SCSI子系统即在其设备对象中将该应答存入scsi_device->inq_periph_qual, 总线即根据响应是否为SCSI_INQ_PQ_CON来决定匹配结果. 此外，注意到，这里并没有初始化probe，即是说，SCSI子系统不提供总线级的初始化，而当”驱动”或”设备”注册到内核的时候，一旦匹配，最终都会调用really_probe. 即如果没有总线级的初始化，really_probe()就会执行driver中的probe.



#### 底层驱动注册
没有纯粹的scsi控制器, 实际的控制器可能是sata，它把scsi封装在自定义的通讯结构中的控制器. 因此linux scsi提供一套用于scsi和各种实际控制器驱动交互的钩子函数模板scsi_host_template, 比如[aha1542的scsi_host_template](https://elixir.bootlin.com/linux/v6.6.12/source/drivers/scsi/aha1542.c#L740).

这些钩子函数由scsi主动调用，scsi并不关注这些钩子的实现，例如aha1542_queuecommand，用于接收scsi发下来的请求，并把scsi命令封装到upiu中并发送给硬件host控制。scsi不关心aha1542驱动如何封装scsi命令，如何触发硬件发送命令.

Host驱动在申请scsi_host时会定义该驱动支持多少个channel和每个channel支持多少个id, 比如sata驱动[ata_scsi_add_hosts](https://elixir.bootlin.com/linux/v6.6.12/source/drivers/ata/libata-scsi.c#L4374)支持2个channel，16个id，每个id下面就1个lun.

在driver/scsi目录下搜索max_channel和channel，可以看到各种各样的用法，这些在scsi这层没有规定，完全取决于host驱动根据自身的情况来选择合适的用法.

### SCSI 设备扫描
以mptspi驱动为例的SCSI设备(主机适配器注册在/sys/devices/pci0000:00/0000:00:10.0)的初始化:
```c
mptspi_init(void)
    return pci_register_driver(&mptspi_driver);
        return __pci_register_driver(driver, THIS_MODULE);
            //drv类型是pci_driver
            drv->driver.name = drv->name;
            drv->driver.bus = &pci_bus_type;
            drv->driver.probe = pci_device_probe;
            drv->driver.remove = pci_device_remove;
            drv->driver.owner = drv->owner;
            drv->driver.kobj.ktype = &pci_driver_kobj_type;
            pci_init_dynids(&drv->dynids);
            driver_register(&drv->driver);
                return bus_add_driver(drv);
                    struct bus_type * bus = get_bus(drv->bus);
                    kobject_set_name(&drv->kobj, "%s", drv->name);
                    drv->kobj.kset = &bus->drivers;
                    // mptspi驱动注册到/sys/bus/pci/drivers
                    kobject_register(&drv->kobj)
                    driver_attach(drv);
                        struct bus_type * bus = drv->bus;
                        //遍历pci总线上的设备
                        list_for_each(entry, &bus->devices.list)
                            struct device * dev = container_of(entry, struct device, bus_list);
                            //调用驱动的probe函数，判断是否能和当前设备匹配
                            driver_probe_device(drv, dev);
                                dev->driver = drv;
                                // 对于mptspi驱动来说就是mptspi_probe
                                error = drv->probe(dev);
                                if (error) 
                                    return error;
                                // 将设备和驱动关联起来
                                device_bind_driver(dev);
                                    list_add_tail(&dev->driver_list, &dev->driver->devices);
                                    // 在/sys/bus/pci/drivers/mptspi目录下创建0000:00:10.0链接文件
                                    // 链接到/sys/devices/pci0000:00/0000:00:10.0
                                    sysfs_create_link(&dev->driver->kobj, &dev->kobj, kobject_name(&dev->kobj));
                                    // 在/sys/devices/pci0000:00/0000:00:10.0目录下创建driver链接文件
                                    // 链接到/sys/bus/pci/drivers/mptspi                               
                                    sysfs_create_link(&dev->kobj, &dev->driver->kobj, "driver");
                    module_add_driver(drv->owner, drv);
                    driver_add_attrs(bus, drv);
            pci_populate_driver_dir(drv);           


// 注册SCSI最底层的主机适配器设备
scsi_add_host(shost, &pdev->dev);
    struct scsi_host_template *sht = shost->hostt;
    /* 设置父设备 /sys/devices/pci0000:00/0000:00:10.0*/
    if (!shost->shost_gendev.parent)
        shost->shost_gendev.parent = dev ? dev : &platform_bus;
    /* 将内嵌设备添加到系统中  名字host0，也就是/sys/devices/pci0000:00/0000:00:10.0/host0*/
    device_add(&shost->shost_gendev);   
    set_bit(SHOST_ADD, &shost->shost_state);
    class_device_add(&shost->shost_classdev);
    scsi_sysfs_add_host(shost);
    scsi_proc_host_add(shost);
        struct scsi_host_template *sht = shost->hostt;
        sprintf(name,"%d", shost->host_no);
        create_proc_read_entry(name, S_IFREG | S_IRUGO | S_IWUSR, sht->proc_dir, proc_scsi_read, shost);

// 扫描SCSI主机适配器上的逻辑设备        
scsi_scan_host(shost);  
    scsi_scan_host_selected(shost, SCAN_WILD_CARD, SCAN_WILD_CARD, SCAN_WILD_CARD, 0);  
        for (channel = 0; channel <= shost->max_channel; channel++)
            scsi_scan_channel(shost, channel, id, lun, rescan);/* 扫描该通道 */
                for (id = 0; id < shost->max_id; ++id) 
                    // id代表了target的序号
                    scsi_scan_target(shost, channel, order_id, lun, rescan);
                        /* 先探测LUN0，目标节点必须响应对LUN0的扫描 */
                        res = scsi_probe_and_add_lun(shost, channel, id, 0, &bflags, &sdev, rescan, NULL);
                            struct scsi_device *sdev = scsi_alloc_sdev(host, channel, id, lun, hostdata);
                                struct scsi_device *sdev = kmalloc(sizeof(*sdev) + shost->transportt->device_size, GFP_ATOMIC);
                                sdev->vendor = scsi_null_device_strs;
                                sdev->model = scsi_null_device_strs;
                                sdev->rev = scsi_null_device_strs;
                                sdev->host = shost;
                                sdev->id = id;
                                sdev->lun = lun;
                                sdev->channel = channel;
                                sdev->sdev_state = SDEV_CREATED;
                                // 给scsi_device分配请求队列
                                sdev->request_queue = scsi_alloc_queue(sdev);
                                    struct Scsi_Host *shost = sdev->host;
                                    // 这里会指定IO调度程序里的请求处理函数为scsi_request_fn
                                    q = blk_init_queue(scsi_request_fn, &sdev->sdev_lock);
                                    ...
                                sdev->request_queue->queuedata = sdev;
                                scsi_sysfs_device_initialize(sdev);
                                    // 初始化内嵌的通用设备
                                    device_initialize(&sdev->sdev_gendev);
                                        kobj_set_kset_s(dev, devices_subsys);
                                    // sdev_gendev会注册到scsi_bus_type下
                                    sdev->sdev_gendev.bus = &scsi_bus_type;
                                    sdev->sdev_gendev.release = scsi_device_dev_release;
                                    // 比如0:0:0：0
                                    sprintf(sdev->sdev_gendev.bus_id,"%d:%d:%d:%d", sdev->host->host_no, sdev->channel, sdev->id, sdev->lun);
                                    class_device_initialize(&sdev->sdev_classdev);
                                    sdev->sdev_classdev.dev = &sdev->sdev_gendev;
                                    sdev->sdev_classdev.class = &sdev_class;
                                    snprintf(sdev->sdev_classdev.class_id, BUS_ID_SIZE,
                                         "%d:%d:%d:%d", sdev->host->host_no,
                                         sdev->channel, sdev->id, sdev->lun);   
                                    scsi_sysfs_target_initialize(sdev);
                                        struct scsi_target *starget = NULL;
                                        struct Scsi_Host *shost = sdev->host;
                                        // 每个目标节点的LUN0会加入到shost->_devices链表
                                        list_for_each_entry(device, &shost->__devices, siblings) 
                                            if (device->id == sdev->id && device->channel == sdev->channel)
                                                //  如果存在相同目标节点的逻辑设备，则将此逻辑设备加入到LUN0的same_target_siblings链表
                                                list_add_tail(&sdev->same_target_siblings, &device->same_target_siblings);
                                                sdev->scsi_level = device->scsi_level;
                                                starget = device->sdev_target;
                                                break;
                                            
                                        // scsi_target（中间设备）没有被注册，开始初始化
                                        if (!starget)
                                            starget = kmalloc(size, GFP_ATOMIC);
                                            dev = &starget->dev;
                                            device_initialize(dev);
                                                kobj_set_kset_s(dev, devices_subsys);
                                            dev->parent = get_device(&shost->shost_gendev);
                                            sprintf(dev->bus_id, "target%d:%d:%d", shost->host_no, sdev->channel, sdev->id);
                                            starget->id = sdev->id;
                                            starget->channel = sdev->channel;
                                            create = starget->create = 1;
                                            sdev->scsi_level = SCSI_2;
                                        sdev->sdev_gendev.parent = &starget->dev;
                                        sdev->sdev_target = starget;
                                        list_add_tail(&sdev->siblings, &shost->__devices);
                                    return sdev;
                            sreq = scsi_allocate_request(sdev, GFP_ATOMIC);
                            /* 发送INQUIRY命令探测逻辑单元 */
                            scsi_probe_lun(sreq, result, &bflags); // 发送SCSI INQUIRY探测逻辑单元
                            /* 根据规范，这个结果表示目标单元存在，但是没有物理设备。 */
                            if ((result[0] >> 5) == 3)
                                goto out_free_result;
                            /* 将逻辑设备添加到系统中 */
                            scsi_add_lun(sdev, result, &bflags);
                                /* 根据INQUIRY响应数据来设置SCSI设备描述符各个域 */
                                sdev->inquiry = kmalloc(sdev->inquiry_len, GFP_ATOMIC);
                                sprintf(sdev->devfs_name, "scsi/host%d/bus%d/target%d/lun%d",sdev->host->host_no, sdev->channel,sdev->id, sdev->lun);
                                scsi_device_set_state(sdev, SDEV_RUNNING);
                                /* 将scsi设备及对应的目标节点添加到sysfs文件系统，并创建对应的属性文件 */
                                scsi_sysfs_add_sdev(sdev); // 将scsi设备以及对于的target添加到sysfs
                                    struct scsi_target *starget = sdev->sdev_target;
                                    struct Scsi_Host *shost = sdev->host;
                                    create = starget->create;
                                    starget->create = 0;
                                    // 注册目标节点，只需要在探测LUN0的时候注册
                                    if (create)
                                        // 添加到/sys/devices/pci0000:00/0000:00:10.0/host0，名字是target0:0:0
                                        device_add(&starget->dev);
                                    scsi_device_set_state(sdev, SDEV_RUNNING)
                                    // 注册逻辑设备
                                    //添加到/sys/devices/pci0000:00/0000:00:10.0/host0/target0.0.0 ，名字是0:0:0
                                    device_add(&sdev->sdev_gendev);
                                    class_device_add(&sdev->sdev_classdev);
                                    for (i = 0; scsi_sysfs_sdev_attrs[i]; i++)
                                        struct device_attribute * attr = 
                                            attr_changed_internally(sdev->host, 
                                                scsi_sysfs_sdev_attrs[i]);
                                        device_create_file(&sdev->sdev_gendev, attr);
                        if (res == SCSI_SCAN_LUN_PRESENT) {/* LUN0有逻辑单元 */
                            /* 通过REPORT LUN命令探测逻辑单元数量，并对每个逻辑单元进行探测 */
                            if (scsi_report_lun_scan(sdev, bflags, rescan) != 0)
                                /* 探测失败，从1到最大编号进行依次探测 */
                                scsi_sequential_lun_scan(shost, channel, id, bflags,
                                            res, sdev->scsi_level, rescan);
                        } else if (res == SCSI_SCAN_TARGET_PRESENT) {
                            scsi_sequential_lun_scan(shost, channel, id, BLIST_SPARSELUN,
                                    SCSI_SCAN_TARGET_PRESENT, SCSI_2, rescan);
                        }
```

mptspi_init主要内容:
1. 将mptspi驱动注册到/sys/bus/pci/drivers。
2. 调用驱动探测函数（mptspi_probe）去匹配pci总线上的设备，如果探测到匹配的设备，就调用device_bind_driver把设备和驱动对应起来

mptspi_probe匹配SCSI主机适配器之后，就会依次调用scsi_host_alloc， scsi_add_host ，scsi_scan_host来初始化SCSI相关设备.

scsi_add_host的主要内容就是注册主机适配器，这里就是/sys/devices/pci0000:00/0000:00:10.0/host0.


scsi_scan_host扫描主机适配器的每一个通道里的每一个taregt目标设备，对每一个目标设备，先调用scsi_probe_and_add_lun探测LUN0设备，如果目标节点必须相应对LUN0探测的响应，scsi_probe_and_add_lun的主要内容：
1. 给scsi_device分配请求队列，后面通用磁盘设备的请求队列都是这个请求队列
2. 每个目标节点的LUN0都会加入到`shost->_devices`链表，同时会初始化scsi_target
3. 发送INQUIRY命令探测逻辑单元，如果LUN0不存在就退出
4. 注册目标设备，探测LUN0的时候就是target0，这里会注册到/sys/devices/pci0000:00/0000:00:10.0/host0/target0.0.0
5. 注册LUN0逻辑设备，注册到/sys/devices/pci0000:00/0000:00:10.0/host0/target0.0.0/0.0.0.0
6. 如果LUN0有逻辑单元，通过REPORT LUN命令探测逻辑单元数量，并对每个逻辑单元进行探测

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

## 其他数据结构/函数
### scsi_cmnd
scsi命令其实有2方面的含义:
1. scsi规范定义的scsi命令即SCSI CDB(command descriptor block)
1. scsi_cmnd

    在SCSI总线子系统中，这种抽象由scsi_cmnd提供. scsi_cmnd源于SCSI子系统中间层，传到SCSI低层. 传递到SCSI子系统的每个IO请求都会被转换为一个scsi_cmnd对象，而一个scsi_cmnd最终又会被转换为一个SCSI命令. 除了SCSI CDB本身之外, scsi_cmnd还封装了与之相关的上下文信息，比如数据缓冲区，完成回调函数以及关联的块设备请求(request)等.

    scsi_cmnd不一定都是I/O请求.

```c
// https://elixir.bootlin.com/linux/v6.6.14/source/include/scsi/scsi_cmnd.h#L74
struct scsi_cmnd {
    struct scsi_device *device; // 所属scsi_device
    struct list_head eh_entry; /* entry for the host eh_abort_list/eh_cmd_q */ // 链入所属scsi_device的错误恢复链表eh_cmd_q的连接件
    struct delayed_work abort_work;

    struct rcu_head rcu;

    int eh_eflags;      /* Used by error handlr */ // 用于错误恢复的标志, 超时是SCSI_EH_CANCEL_CMD

    int budget_token;

    /*
     * This is set to jiffies as it was when the command was first
     * allocated.  It is used to time how long the command has
     * been outstanding
     */
    unsigned long jiffies_at_alloc; // 命令首次分配的滴答数， 用来计算命令已经过去了多少时间

    int retries; // 已重试的次数
    int allowed; // 允许的重试次数

    unsigned char prot_op; // 保护操作(取决于DIF和DIX保护设置)
    unsigned char prot_type; // DIF 保护类型(取决于host保护能力和scsi磁盘保护设置)
    unsigned char prot_flags;
    enum scsi_cmnd_submitter submitter;

    unsigned short cmd_len; // 命令长度
    enum dma_data_direction sc_data_direction; // 命令的数据传输方向

    unsigned char cmnd[32]; /* SCSI CDB */ // 指向scsi规范格式的命令字符串

    /* These elements define the operation we ultimately want to perform */
    struct scsi_data_buffer sdb; // 数据缓冲区. 对于读操作, 由scsi底层驱动填入; 写操作时scsi中间层填入
    struct scsi_data_buffer *prot_sdb; // 保护信息缓冲区

    unsigned underflow; /* Return error if less than
                   this amount is transferred */ // 如果传输的数据小于这个量则返回错误

    unsigned transfersize;  /* How much we are guaranteed to
                   transfer with each SCSI transfer
                   (ie, between disconnect / 
                   reconnects.   Probably == sector
                   size */ // 传输单位(等于硬件扇区长度)
    unsigned resid_len; /* residual count */
    unsigned sense_len;
    unsigned char *sense_buffer; // 感测数据缓冲区
                /* obtained by REQUEST SENSE when
                 * CHECK CONDITION is received on original
                 * command (auto-sense). Length must be
                 * SCSI_SENSE_BUFFERSIZE bytes. */

    int flags;      /* Command flags */
    unsigned long state;    /* Command completion state */

    unsigned int extra_len; /* length of alignment and padding */

    /*
     * The fields below can be modified by the LLD but the fields above
     * must not be modified.
     */

    unsigned char *host_scribble;   /* The host adapter is allowed to
                     * call scsi_malloc and get some memory
                     * and hang it here.  The host adapter
                     * is also expected to call scsi_free
                     * to release this memory.  (The memory
                     * obtained by scsi_malloc is guaranteed
                     * to be at an address < 16Mb). */ // 被底层驱动使用

    int result;     /* Status code from lower level driver */ // 从底层驱动返回的状态码
};
```

### sd_probe
sd_probe是scsi磁盘驱动的实际入口. 它在驱动初始化期间, 以及有新的scsi设备被挂载到系统时被调用. 对于每个出现的scsi设备(不仅仅是磁盘)都会调用一次. 这个函数是从scsi中间层调用, 它将给定的`host,channel,id,lun`和新的设备名(比如/dev/sda)之间建立映射, 即选定了块设备的主次设备编号.

### sd_revalidate_disk
在scsi磁盘探测过程中, 会在add_disk前后2次调用sd_revalidate_disk, 是在于向块I/O子系统确定注册的方式.

### sd_spinup_disk
向scsi磁盘发送命令让磁盘转起来.

### scsi_execute_req
执行scsi命令

一旦底层设备驱动拿到一个scsi命令, 它要么调用中间层在scsi_host_template的queuecommand回调函数时传入的scsi_done来完成命令, 或者scsi中间层使其超时.

使用scsi_done完成一个scsi命令时, 会删除超时定时器, 将scsi命令通过对应块设备驱动层请求的连接件链入每个cpu的blk_cpu_done链表, 然后触发SCSI_SOFTIRQ.

SCSI_SOFTIRQ的处理句柄是scsi_softirq_done. 它调用scsi_decide_dispositon()来确定如何处理这个命令.

超时处理句柄是scsi_times_out(在scsi_alloc_queue中设置, 定时器由块I/O子系统维护), 之后调用scsi_eh_scmd_add进行错误恢复.

scsi命令执行错误或超时, 都会被添加到host的故障命令列表. 一旦由故障命令, 将不会有新命令被提交. 最终所有命令都会结束.

## scsi底层驱动编程模式
ref:
- <<存储技术原理分析 - 4.9 SCSI 底层驱动编程模式>>