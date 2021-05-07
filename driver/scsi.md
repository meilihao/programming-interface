# scsi
参考:
- [Linux SCSI子系统的分层架构](/misc/img/driver/v2-2889c8d1151b30f7a2afb4781cbc3d94_720w.jpg)
- [+SCSI开发基础](https://www.daimajiaoliu.com/daima/4ed12fa929003e4)
- [+scsi总线驱动的初始化](https://blog.csdn.net/yunsongice/article/details/6171286)
- [Scsi host driver编写总结](http://chinaunix.net/uid-15724196-id-128154.html)

上层，它代表各种具体的scsi设备类型的驱动，如scsi磁盘驱动(`driver/scsi/sd.c`)，scsi磁带驱动等. 该层向上表现为块设备、对上提供块设备接口，因此，具有块设备的接口及一切属性， 向下表现scsi设备，因为scsi disk基于scsi总线进行数据通信. top level驱动与具体的scsi设备相关，所以该类驱动往往由设备开发者提供，但是如果scsi设备为标准类设备，那么驱动可以通用. 高层驱动认领低层驱动发现的scsi设备，为这些设备分配名称，将对设备的IO转换为scsi命令，交由低层驱动处理.

中间层，用于实现SCSI的公共服务函数，它实际上就是scsi总线层驱动，按照scsi协议进行设备枚举、数据传输、出错处理. middle level层的驱动与scsi specification相关，在一类操作系统平台上只需实现一次，所以该类驱动往往由操作系统开发者提供. 高层和低层通过调用中层的函数完成其功能，而中层在执行过程中，也需要调用高层和低层注册的回调函数做一些个性化处理.

下层，它代表与SCSI的物理接口的实际驱动器，主要为各个厂商为其特定的主机适配器(Host Bus Adapter, HBA)驱动，例如： FC卡驱动、SAS卡驱动和iSCSI（iSCSI可以使硬件HBA卡或者基于普通网卡的软件实现）等. 其需要与scsi middle level层进行接口，所以往往由提供适配器的硬件厂商完成驱动开发，只有硬件厂商才对自己定义的register file（寄存器堆）最清楚. 当然，在lower level层可以做虚拟的scsi host，所以该层的驱动也不一定对硬件进行操作。低层驱动主要作用是发现连接到主机适配器的scsi设备，在内存中构建scsi子系统所需的数据结构，并提供消息传递接口，将scsi命令的接受与发送解释为主机适配器的操作.

总结就是:
- SCSI device driver(上层)是SCSI设备驱动，也可以称之为功能驱动（Function driver）
- SCSI middle level driver为SCSI中间层驱动，抽象了SCSI的总线逻辑
- SCSI host driver(下层)控制SCSI总线控制器，实现SCSI数据的物理层传输

> 通过`/boot/System.map-5.xx.xx-amd64-desktop`查找`init_sd`发现`driver/scsi/sd.c`是直接编入kernel而不是以sd.ko的形式存在.

# scsi disk(sd)
参考:
- [Linux那些事儿 之 我是SCSI硬盘](https://blog.csdn.net/fudan_abc/category_351156.html)

先看[driver/scsi/Kconfig#BLK_DEV_SD](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/Kconfig#L68), 根据`depends on SCSI`发现BLK_DEV_SD依赖`SCSI`(其实就是scsi core).

在看[driver/scsi/Makefile#`obj-$(CONFIG_BLK_DEV_SD)`](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/Makefile#L147)会发现, scsi disk驱动在sd.c里.

[sd.c](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L3796):
```c
// https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L3710
/**
 *  init_sd - entry point for this driver (both when built in or when
 *  a module).
 *
 *  Note: this function registers this driver with the scsi mid-level.
 **/
static int __init init_sd(void)
{
    int majors = 0, i, err;

    SCSI_LOG_HLQUEUE(3, printk("init_sd: sd driver entry point\n"));

    for (i = 0; i < SD_MAJORS; i++) {
        if (__register_blkdev(sd_major(i), "sd", sd_default_probe))
            continue;
        majors++;
    }

    if (!majors)
        return -ENODEV;

    err = class_register(&sd_disk_class);
    if (err)
        goto err_out;

    sd_cdb_cache = kmem_cache_create("sd_ext_cdb", SD_EXT_CDB_SIZE,
                     0, 0, NULL);
    if (!sd_cdb_cache) {
        printk(KERN_ERR "sd: can't init extended cdb cache\n");
        err = -ENOMEM;
        goto err_out_class;
    }

    sd_cdb_pool = mempool_create_slab_pool(SD_MEMPOOL_SIZE, sd_cdb_cache);
    if (!sd_cdb_pool) {
        printk(KERN_ERR "sd: can't init extended cdb pool\n");
        err = -ENOMEM;
        goto err_out_cache;
    }

    sd_page_pool = mempool_create_page_pool(SD_MEMPOOL_SIZE, 0);
    if (!sd_page_pool) {
        printk(KERN_ERR "sd: can't init discard page pool\n");
        err = -ENOMEM;
        goto err_out_ppool;
    }

    err = scsi_register_driver(&sd_template.gendrv);
    if (err)
        goto err_out_driver;

    return 0;

err_out_driver:
    mempool_destroy(sd_page_pool);

err_out_ppool:
    mempool_destroy(sd_cdb_pool);

err_out_cache:
    kmem_cache_destroy(sd_cdb_cache);

err_out_class:
    class_unregister(&sd_disk_class);
err_out:
    for (i = 0; i < SD_MAJORS; i++)
        unregister_blkdev(sd_major(i), "sd");
    return err;
}

/**
 *  exit_sd - exit point for this driver (when it is a module).
 *
 *  Note: this function unregisters this driver from the scsi mid-level.
 **/
static void __exit exit_sd(void)
{
    int i;

    SCSI_LOG_HLQUEUE(3, printk("exit_sd: exiting sd driver\n"));

    scsi_unregister_driver(&sd_template.gendrv);
    mempool_destroy(sd_cdb_pool);
    mempool_destroy(sd_page_pool);
    kmem_cache_destroy(sd_cdb_cache);

    class_unregister(&sd_disk_class);

    for (i = 0; i < SD_MAJORS; i++)
        unregister_blkdev(sd_major(i), "sd");
}

module_init(init_sd);
module_exit(exit_sd);
...
```

首先,__register_blkdev注册一个块设备. 因为SD_MAJORS实际上被定义为16, 因此经过16次循环之后, `/proc/devices`里看到叫sd的有16个. `sd_major(i)`返回的就是Linux中的主设备号, 其中,8,65-71,136-143这16个号码已被scsi disk所用.

class_register的作用是`/sys/class`下出现`scsi_disk`目录.

scsi_register_driver则是注册一个scsi设备驱动, 其直观效果就是`/sys/bus/scsi/drivers`下出现`sd`目录. linux设备模型对于每个设备驱动,有一个与之对应的struct device_driver结构体,而为了体现各类设备驱动自身的特点,各个子系统可以定义自己的结构体,然后把struct device_driver包含进来, 就如C++中基类和扩展类一样. 对于scsi子系统,这个基类就是struct scsi_driver,这个结构体本身定义于[include/scsi/scsi_driver.h](https://elixir.bootlin.com/linux/latest/source/include/scsi/scsi_driver.h#L13). 而scsi驱动自然定义了一个scsi_driver的结构体实例[sd_template](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L614), 其中, gendrv就是struct device_driver的结构体变量.

而与以上三个函数相反的就是exit_sd()中的另外三函数: `scsi_unregister_driver,class_unregister,unregister_blkdev`.

## sd_probe
对驱动而已, 我们最关心的还是probe函数即sd_template.gendrv.probe也即是[sd_probe](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L3366).

首先,驱动为scsi device准备一个指针,struct scsi_device *sdp,为scsi disk准备一个指针,struct scsi_disk *sdkp,此外,甭管是scsi硬盘还是ide硬盘,都少不了一个结构体struct gendisk,这里准备了一个指针struct gendisk *gd.

sd_probe将会由scsi核心层调用,或者也叫scsi mid-level来调用. scsi mid-level在调用sd_probe之前,已经为这个scsi设备准备好了struct device,struct scsi_device, 并已经为它们做好了初始化,所以这里struct device *dev作为参数传递进来就可以直接引用它的成员了.

通过判断sdp->type,这是struct scsi_device结构体中的成员char type,它用来表征这个scsi设备是哪种类型的,scsi设备五花八门,而只有这里列出来的这四种是sd_mod所支持的: 这其中我们最熟悉的当属TYPE_DISK, 它就是普通的scsi磁盘; 而TYPE_MOD表示的是磁光盘(Magneto-Optical disk),一种采用激光和磁场共同作用的磁光方式存储技术实现的介质, 外观和3.5英寸软盘相似; TYPE_RBC中的RBC表示Reduced Block Commands, 即支持RBC指令集的设备; 另外TYPE_ZBC中ZBC表示Zoned Block Command, 与TYPE_RBC类似.

`kzalloc(sizeof(*sdkp), GFP_KERNEL);`为sdkp申请内存, sdkp的定义是[scsi_disk](https://elixir.bootlin.com/linux/latest/source/include/scsi/scsi_device.h#L101).

`gd = alloc_disk(SD_MINORS);`分配一个gendisk.

[`ida_alloc`](https://www.kernel.org/doc/html/latest/core-api/idr.html), 为disk分配index, 表示一块唯一的盘, 来源可参考[New IDA API](https://lwn.net/Articles/750154/).

`gd->major = sd_major((index & 0xf0) >> 4); gd->first_minor = ((index & 0xf) << 4) | (index & 0xfff00);`, [sd_major](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L654). 前面说过scsi disk的主设备号是已经固定好了的,它就是瓜分了8,65-71,128-135这几个号,这里SCSI_DISK0_MAJOR就是8,SCSI_DISK1_MAJOR就是65,SCSI_DISK8_MAJOR就是128.sd_major()接受的参数就是index的bit4到bit7,而它取值范围自然就是0到15,这也正是sd_major()中switch/case语句判断的范围,即实际上major_idx就是主设备号的一个索引,就说是在这个16个主设备号中它算老几.而first_minor就是对应于本index的第一个次设备号,我们可以用代入法得到,当index为0,则first_minor为0,当index为1,则first_minor为16,当index为2,则first_minor为32.

`device_initialize ~ device_add`用于在`/sys/class/scsi_device/`下生成`SCSI Address`.

get_device是访问一个struct device的第一步,增加引用计数.以后不用的时候自然会有一个相对的函数put_device被调用.

dev_set_drvdata,就是设置dev->driver_data等于sdkp,即让struct device的指针dev和struct scsi_disk的指针sdkp给联系起来.

### sd_revalidate_disk

#### sd_spinup_disk

[sd_revalidate_disk](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L3170)中的[sd_spinup_disk](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L2138)就是让磁盘转起来. sd_spinup_disk的核心就是`scsi_execute_req`用于执行scsi命令:
```c
// https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L2158
            the_result = scsi_execute_req(sdkp->device, cmd,
                              DMA_NONE, NULL, 0,
                              &sshdr, SD_TIMEOUT,
                              sdkp->max_retries, NULL);
```
cmd=TEST_UNIT_READY.Test Unit Ready, 是一个很基本的SCSI命令. DMA_NONE代表传输方向,buffer和bufflen用不上,因为这个命令就是测试设备准备好了没有,不需要传递什么数据.

执行这个Test Unit Ready命令,返回的结果基本上都是好的,除非设备真的有问题. 其实`sg_turs /dev/sdb`命令就是用来手工发送Test Unit Ready用的.

[scsi_status_is_good](https://elixir.bootlin.com/linux/latest/source/include/scsi/scsi.h#L41)用于判断scsi_execute_req返回的结果. scsi_status_is_good用到的那些status宏被称为状态码, scsi_execute_req()的返回值就是这些状态码中的一个. 而其中可以被认为是good的状态就是scsi_status_is_good函数中列出来的这五种,当然理论上来说最理想的就是SAM_STAT_GOOD,而另外这几种也勉强算是可以接受.

注意: the_result和状态码还是有区别的,毕竟状态码只有那么多,用8位来表示足矣,而the_result我们看到是unsigned int,显然它不只是8位,于是我们就充分利用资源,因此就有了下面这些宏.
```c
// https://elixir.bootlin.com/linux/latest/source/include/scsi/scsi.h#L212
/*
 *  Use these to separate status msg and our bytes
 *
 *  These are set by:
 *
 *      status byte = set from target device
 *      msg_byte    = return status from host adapter itself.
 *      host_byte   = set by low-level driver to indicate status.
 *      driver_byte = set by mid-level.
 */
#define status_byte(result) (((result) >> 1) & 0x7f)
#define msg_byte(result)    (((result) >> 8) & 0xff)
#define host_byte(result)   (((result) >> 16) & 0xff)
#define driver_byte(result) (((result) >> 24) & 0xff)
```

也就是说除了最低的那个byte是作为status byte用,剩下的byte也没浪费,它们都被用来承载信息,其中driver_byte,即bit23到bit31,这8位被用来承载mid-level设置的信息.而这里用它和DRIVER_SENSE相与,则判断的是是否有sense data. scsi世界里的sense data就是错误信息. scsi_execute_req外层的do-while循环就是如果不成功就最多重复三次,循环结束了之后,用`if (driver_byte(the_result) != DRIVER_SENSE)`再次判断有没有sense data,如果没有,则说明也许成功了.

Scsi子系统最麻烦的地方就在于错误判断的代码特别的多.而针对sense data的处理则是错误判断的一部分.

[scsi_sense_hdr](https://elixir.bootlin.com/linux/latest/source/include/scsi/scsi_common.h#L50)被用来描述一个sense data."hdr"就是header的意思,因为sense data可能长度比较长,但是其前8个bytes是最重要的,所以这部分被叫做header, 大多数情况下只要处理头部就足以. scsi_execute_req()中第六个参数就是`struct scsi_sense_hdr *sshdr`. 如果命令执行出错了,那么sense data就会通过这个参数返回. 可通过判断sshdr和它的各个成员,来决定下一步.
```c
// https://elixir.bootlin.com/linux/latest/source/include/scsi/scsi_common.h#L50
/*
 * This is a slightly modified SCSI sense "descriptor" format header.
 * The addition is to allow the 0x70 and 0x71 response codes. The idea
 * is to place the salient data from either "fixed" or "descriptor" sense
 * format into one structure to ease application processing.
 *
 * The original sense buffer should be kept around for those cases
 * in which more information is required (e.g. the LBA of a MEDIUM ERROR).
 */
struct scsi_sense_hdr {     /* See SPC-3 section 4.5 */
    u8 response_code;   /* permit: 0x0, 0x70, 0x71, 0x72, 0x73 */
    u8 sense_key;
    u8 asc;
    u8 ascq;
    u8 byte4;
    u8 byte5;
    u8 byte6;
    u8 additional_length;   /* always 0 for fixed sense format */
};

static inline bool scsi_sense_valid(const struct scsi_sense_hdr *sshdr)
{
    if (!sshdr)
        return false;

    return (sshdr->response_code & 0x70) == 0x70;
}
```

而sense data中,最基本的一个元素叫做response_code,它相当于为一个sense data定了性,即它属于哪一个类别,因为sense data毕竟有很多种.response code总共就是8个bits,目前使用的值只有70h,71h,72h,73h,其它的像00h到6Fh以及74h到7Eh这些都是保留的,以备将来之用.所以这里判断的就是response code得是0x70,0x71,0x72,0x73才是valid,否则就是invalid.这就是scsi_sense_valid()做的事情.

关于sense data, 可参考SCSI Primary Commands(SPC)的`Sense data`章节. Sense data中最有意义的东西叫做sense key和sense code.这两个概念基本上确定了这个错误究竟是什么错误.

`sshdr.sense_key == UNIT_ATTENTION`, 这个信息表示这个设备可能被重置了或者可移动的介质发生了变化,或者更通俗一点说,只要设备发生了一些变化,然后它希望引起主机控制器的关注,比如说设备原本是on-line的,突然变成了off-line,或者反过来,设备从off-line回到了on-line. 在正式读写设备之前,如果有UNIT_ATTENTION条件,必须把它给清除掉.而这(清除UNIT ATTENTION)也正是Test Unit Ready的工作之一.

而如果sense key等于NOT_READY,则表明这个logical unit不能被访问. 而如果sense key等于NOT READY,而asc等于04h,ascq等于03h,这表明”Logical Unit Not Ready,Manual Intervention required”, 这说明需要人工干预.

当然大多数情况下,应该执行的是`if (sense_valid && sshdr.sense_key == NOT_READY)`所包含的代码, 即磁盘确实应该是NOT_READY,于是我们需要发送下一个命令,即START STOP,在SCSI Block Commands-2(SBC-2)的书中, 有章节介绍了START STOP UNIT这个命令. 这个命令简而言之,就相当于电源开关,SBC-2中Table 48给出了这个命令的格式.

spintime_expire为100秒,即这个时间为软件忍耐极限,磁盘只要在100秒之内动起来,驱动就既往不咎,倘若还是不行,那就没办法了,while循环自然结束.

#### sd_read_write_protect_flag
sd_revalidate_disk的[sd_read_write_protect_flag](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L2643)也很重要. 它调用set_disk_ro()从而确定本磁盘是否是写保护的.

set_disk_ro就是设置磁盘只读,为0就是可读可写,为1才是设置为只读.但是这只是软件意义上的作个记录而已, 硬件上还得听磁盘自己的. 所以通过后续的一大段代码最终得到这一信息,最终会再次设置.

那么如何得知写保护是否设置了呢? 自然是发送命令给设备,这个命令就是MODE SENSE.MODE SENSE这个命令的目的在于获得设备内部很多潜在的信息,这其中包括设备是否设置了写保护,当然还有更多SCSI特有的信息. 只不过此时此刻只关注写保护设了没有. 对于SCSI设备来说,很多特性可以改变,但是有些特性就不可以改变了,比如medium type,即它属于哪种类型的设备,对于SCSI Block设备,其内部保存MEDIUM TYPE的这个byte一定是00h.

在驱动中为了设置写保护, 先调用[sd_do_mode_sense()](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L2629). 而sd_do_mode_sense来自scsi核心层统一提供的scsi_mode_sense(), 总之执行之后,结果就是保存在了data中,而data是[struct scsi_mode_data](https://elixir.bootlin.com/linux/latest/source/include/scsi/scsi_device.h#L22)结构体变量. 它的每一个成员都在scsi协议中能够找到对应物.就比如刚才说得medium_type,对于SCSI磁盘,它一定是00h.这是没得商量的.

`sdkp->write_prot = ((data.device_specific & 0x80) != 0)`, 判断device_specific和0x80相与. 参考SBC-2中的Device Specific, 其里bit7叫做WP,即Write Protect,写保护位. 如果这一位为1就说明设置了写保护,反之则是没有设置. 如果没有设置写保护,那么在日志文件里就能看到类似下面这行的一句话: `... sdb: Write Protect is off`.

`use_192_bytes_for_3f`是flag, 这是因为实践表明,很多磁盘只能接受MODE SENSE在page=0x3f时传输长度为192bytes,所以在定义struct scsi_device的时候为这些设备准备了这么一个flag,在scsi总线扫描设备初始化的时候就可以设置这么一个flag. 相应的我们发送命令的时候就设置好192.

skip_ms_page_3f,这也是一个类似的flag,MODE SENSE命令有一个参数page,同样是实践表明,某些愚蠢的设备在page=0x3f的时候会出错. 所以写代码的做出让步,又准备了一个flag.

如果你还不是很明白这个page是啥意思,那么看一下SPC-4中MODE SENSE命令的格式是如何的.

这其中,PAGE CODE就是我们上面说的page,很明显,它一共占6个bits.因此它的取值范围就是00h到3Fh(即11 1111).而我们上面说到3fh,就是说当发送MODE SENSE命令的时候,设置PAGE CODE为3fh的时候,因为3fh是最后一个page,很多设备都会有一些莫名其妙的错误,于是我们需要设置种种flag来处理这些情况.

scsi信息被定义成类似一本书,而要阅读这本书,就必须发送MODE SENSE命令. 但是就像读别的书一样,必须一页一页的读,所以需要给定一个PAGE CODE,或者说页码,同时看到命令的第3byte叫做SUBPAGE CODE,这就是子页号码, 理解为一页中某一个段落好了,即设备允许一页一页的读,也允许一段一段的读.很显然,由于SUBPAGE CODE是8个bits,因此其最大值就是255.即一个page可以有最多255个subpage.

#### sd_read_cache_type
sd_revalidate_disk的[sd_read_cache_type](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L2702)函数最主要的工作还是调用sd_do_mode_sense,即还是发送MODE SENSE命令.我们前面说过,SCSI设备写真集最多就是64页(64=0x3f+1).而这里我们给modepage赋值为8,或者对于RBC,赋值为6. 本函数的目的是读取设备中关于Cache的信息,事实上每个SCSI磁盘,或者更有专业的说法, 每一个Direct-access block device,都可以实现caches,通过使用cache可以提高设备的性能,比如可以减少访问时间,比如可以增加数据吞吐量.而在SBC-2中,为SCSI磁盘定义了一个Mode Page专门用来描述和cache相关的信息.

08h这个Page,被叫做caching mode page,这一个Page就是当前需要的.这也就是为什么我们赋值modepage为8.而对于遵循RBC协议的设备这个值会是6.

SPC-4中的Table238定义了MODE SENSE命令的返回值的格式一共有三部分,即Mode Parameter Header,Block Descriptor,Mode Page(s). 而Mode Page出现在第三部分.比如我们这里点名要Mode Page 8,那么它就出现在这里的第三部分. 首先我们所有的返回值都保存在buffer[]数组中,如果我们要访问Mode Pages这一部分,我们就必须知道前面两个部分的长度.假设前面两个部分的长度为offset,那么我们要访问第三部分就可以使用buffer[offset],这样我们就知道这个offset的含义了: `int offset = data.header_length + data.block_descriptor_length;`.

如果深入scsi_mode_sense()函数, 会发现,其实data.header_length恰恰就是这个Mode Parameter Header的长度,而data.block_descriptor_length恰恰就是第二部分的长度,即Block Descriptor的长度.

于是我们用buffer[offset]就定位到了Mode Page这一部分, Caching Mode Page定义在SBC-2的Table 101. 看到这个Page一共有19个bytes,而我们知道buffer[offset]就应该对应它的Byte0.而这里我们看到Byte0的bit0到bit5代表的就是PAGE CODE,即对于caching mode page来说它应该是08h, 可用于校验是否正确的response.

而接下来,再次根据modepage是8还是6来做不同的赋值,我们还是只看主流的情况,即考虑modepage为8的情况. buffer[offset+2]就是这里的Byte2.很明显对照Table 101来看,我们要的是这里的WCE和RCD这两个bits,看看它们是1还是0.

WCE为1说明我们在写操作的时候可以启用cache,即只要写入数据到了cache中就先返回成功,而不用等到真正写到介质中以后再返回.RCD为1则说明我们读数据的时候必须从介质中读,而不是从cache中读.

最后,一个叫做DPOFUA的bit也是需要牢记心中的.看到这个东西来自data.device_specific,这个东西就是前面那幅Mode Parameter Header中的DEVICE SPECIFIC PARAMETER,对于遵循SBC-2的设备,这一项的格式也是专门有定义的: Table 97.

其实这幅图我们似曾相识.之前我们就是通过这个WP位的了解设备是否设置了写保护的.而这里bit4叫做DPOFUA,这一个Bit如果为1.说明设备支持DPO和FUA bits,如果为0,说明设备并不支持DPO和FUA bits.DPO是disable page out的缩写,FUA是force unit access的缩写.

SBC-2中专门介绍Cache的一节有DPO和FUA的相关信息. 总结就是: 如果DPO和FUA bits被设置成了1,那么读写操作都不会使用cache.因为DPOFUA是这里的bit4. data.device_specific就是和0x10相与, 这样得到的就是bit4.

不过,和MODE SENSE命令一样,READ/WRITE命令也有6字节和10字节的,对于READ/WRITE操作,默认情况下咱们会先尝试使用10字节的.但是咱们也允许你违反游戏规则.struct scsi_device结构体中unsigned use_10_for_rw,就是你可以设置的.默认情况下,咱们会在设备初始化的时候,确切的说,在scsi总线扫描的时候,scsi_add_lun函数中会把这个flag设置为1.但如果你偏要特立独行,那也随便你.但是实际上6字节的READ/Write命令中没有定义FUA,DPO,所以这里我们需要设置DPOFUA为0.

最后,带着sdkp->WCE,sdkp->RCD,sdkp->DPOFUA,sd_read_cache_type()函数满载而归.

## ioctl
向scsi磁盘发送命令的时候,ioctl函数多半会被调用.

执行`sg_inq /dev/sdg`, 用kdb追踪即可看到, 系统调用ioctl,或者说从sys_ioctl进来的, 会走到了[sd_ioctl](https://elixir.bootlin.com/linux/latest/source/drivers/scsi/sd.c#L1757), 最终走到[scsi_cmd_ioctl](https://elixir.bootlin.com/linux/latest/source/block/scsi_ioctl.c#L767).

通过kdb的跟踪, 会发现传递进来的是cmd是SG_IO.换言之,switch-case这一大段最终会定格在`case SG_IO`里. 这里涉及到一个结构体就很重要了: [sg_io_hdr](https://elixir.bootlin.com/linux/latest/source/include/scsi/sg.h#L44), 其中,cmdp指针指向的正是命令本身.

sg_io_hdr的[get_sg_io_hdr](https://elixir.bootlin.com/linux/latest/source/block/scsi_ioctl.c#L591)使用了copy_from_user.

对copy_from_user和copy_to_user这两个骨灰级的函数应该不陌生, 内核空间和用户空间传递数据的两个经典函数, 而这里具体来说,先是把arg里的玩意儿copy到hdr中,然后调用sg_io完成实质性的工作,然后再把hdr里的内容copy回到arg中.

之后调用[sg_io](https://elixir.bootlin.com/linux/latest/source/block/scsi_ioctl.c#L282).

不难看出这里hdr(sg_io_hdr)实际上扮演了两个角色,一个是输入,一个是输出.而我们为了得到信息所采取的手段依然是blk_execute_rq,即仍然是以提交request的方式,在blk_execute_rq之后,实际上信息已经保存到了rq的各个成员里边,而这之后的代码就是把信息从rq的成员中转移到hdr的成员中.

我们再来关注一些细节.首先结合struct sg_io_hdr这个结构体来看代码.

interface_id,必须得是大"S".表示Scsi Generic. 从历史渊源来说表示当年那个叫做sg的模块.而与之对应的是另一个叫做pg的模块(parallel port generic driver,并行端口通用驱动),也会有interface_id这么一个变量,它的这个变量则被设置为"P".

dxfer_direction,这个就表示数据传输方向. 比如对于写命令这个变量可以取值为SG_DXFER_TO_DEV,对于读命令这个变量可以取值为SG_DXFER_FROM_DEV.

cmd_len就不用说了,表征该scsi命令的长度. 它必须小于等于16.因为scsi命令最多就是16个字节. 这就是为什么`if (hdr->cmd_len > BLK_MAX_CDB) `判断是否大于16.(BLK_MAX_CDB被定义为16.)

dxfer_len就是数据传输阶段到底会传输多少字节.queue_max_hw_sectors表示单个request最大能传输多少个sectors,这个是硬件限制.一个sector是512个字节,所以要左移9位,即乘上512.

blk_get_request基本上可以理解为申请一个struct request结构体rq.

[blk_fill_sghdr_rq](https://elixir.bootlin.com/linux/latest/source/block/scsi_ioctl.c#L220)用于填充rq.

iovec_count和dxferp是分不开的.如果iovec_count为0,dxferp表示用户空间内存地址,如果iovec_count大于0,那么dxferp实际指向了一个scatter-gather数组.

而blk_rq_map_user和blk_rq_map_user_iov的作用是一样的,建立用户数据和request之间的映射. 如今几乎每个人都听说过Linux中所谓的零拷贝特性,而传说中神奇的零拷贝这一刻离我们竟是如此之近. 调用这两个函数的目的就是为了使用零拷贝技术.零拷贝的作用自不必说,改善性能,提高效率.

在执行完blk_execute_rq之后就简单了.

hdr->duration表示从scsi命令被发送到完成之间的时间,单位是毫秒.

hdr->resid(in blk_complete_sghdr_rq)表示还剩下多少字节没有传输.

hdr->sbp指向用于写Scsi sense buffer的user memory.

hdr->sb_len_wr表示实际写了多少个bytes到sbp指向的user memory中.命令如果成功了基本上就不会写sense buffer,这样的话sb_len_wr就是0.

hdr->mx_sb_len则表征能往sbp中写的最大的size,sb_len_wr<=mx_sb_len.

那么现在可以大致了解了从应用层来讲是如何给scsi设备发送命令的.sg_inq实际上触发的是ioctl的系统调用,经过几次辗转反侧,最终sd_ioctl会被调用.而sd_ioctl会调用scsi核心层提供的函数,sg_io,最终走的路线依然是blk_execute_rq.