# sensor
常见sensor:
1. 环境光传感器(Ambient Light Sensor,ALS),可以用来测量环境光的亮度等。自动调节背光亮度的依据就是它产生的数据,在屋内外穿梭时的光线变化被它捕捉到汇报给系统。
1. 加速度传感器(Accelerometer,ACC),测量设备的加速度,数据为X/Y/Z三个维度。设备静止时,合加速度等于9.8(重力加速度),所以它可以用来计算设备和水平面的夹角,可以辅助横竖屏切换
1. 磁力传感器(Magnetic),测试设备周围的磁场,分为X/Y/Z三个维度。手机上的APP使用的指南针的数据就是由它提供的。
1. 陀螺仪传感器(Gyroscope),测试设备旋转的角速度,一个好的游戏手机必备
1. 图像传感器,即摄像头(Camera),它的应用场景极为广泛,拍照,监控,行车记录仪,自动驾驶系统中的标志识别、全景泊车等。它是智能设备的眼睛,是它们识别事物的前提, 市场的空间和潜力巨大。
1. 雷达(Radar),主要应用于无人驾驶领域,可以识别障碍物和周围的车辆信息

多数传感器可以使用input子系统,特殊的传感器有专门的机制. 从软件角度,input子系统也是不能满足需求的,摄像头使用的是V4L架构.

手机的传感器数据量较小,很多使用i2c总线即可,但是摄像头和激光雷达等并不是i2c可以满足的. 摄像头硬件上数据传输使用mipi, 激光雷达是一类使用激光进行探测和测距的设备,它能够每秒钟向环境发送百万级光脉冲,内部是一种旋转的结构,使得激光雷达能够实时地建立起周围环境的三维地图,可以使用USB进行数据传输.

## camera
Camera硬件架构:
- cpu控制isp, 也通过i2c控制camera
- camera通过mipi向isp传输信息

这是一个比较常用的组合,由i2c传递控制信息(格式、帧率等)到Camera,Camera产生数据后,经由ISP(Image Signal Processor)处理,生成系统支持的格式的图像. ISP也需要控制(自动曝光、自动对焦、自动白平衡等),一般通过MMIO读写它的寄存器即可完成.

Camera一般作为一个外部设备使用,isp则多为平台的一部分,一个系统中可以有多个Camera,比如智能手机上一般至少存在前后两个摄像头.

### V4L2架构
V4L2的全称是Video For Linux Two, 它在2002年融入内核主干,十几年的验证和完善让它成了很多设备驱动架构的不二之选,Camera就是其中之一.

4L2支持的设备类型广泛,比如视频输入输出设备、VBI设备和Radio设备等,Camera属于第一种类型。用户空间通过操作文件来控制它们,它们对应的文件的名字不同,分别为/dev/videoX、/dev/vbiX
和/dev/radioX,其中X是一个数字,值与它们在同类设备中注册的顺序有关.

这些文件与video_device结构体有关,对文件的操作由它传递至最终设备,它是V4L2给用户空间开的“一扇窗”.

```c
// https://elixir.bootlin.com/linux/v6.6.30/source/include/media/v4l2-dev.h#L263
struct video_device {
#if defined(CONFIG_MEDIA_CONTROLLER)
	struct media_entity entity;
	struct media_intf_devnode *intf_devnode;
	struct media_pipeline pipe;
#endif
	const struct v4l2_file_operations *fops;

	u32 device_caps;

	/* sysfs */
	struct device dev; // 设备相关
	struct cdev *cdev; // 设备相关. dev和cdev用于创建/sys和/dev文件 

	struct v4l2_device *v4l2_dev; // 关联的v4l2_device
	struct device *dev_parent; // 父设备

	struct v4l2_ctrl_handler *ctrl_handler;

	struct vb2_queue *queue;

	struct v4l2_prio_state *prio;

	/* device info */
	char name[32];
	enum vfl_devnode_type vfl_type; // 设备的类型
	enum vfl_devnode_direction vfl_dir;
	int minor;
	u16 num; // 同类设备中的序号
	unsigned long flags;
	int index;

	/* V4L2 file handles */
	spinlock_t		fh_lock;
	struct list_head	fh_list; // v4l2_fh组成的链表的头

	int dev_debug;

	v4l2_std_id tvnorms;

	/* callbacks */
	void (*release)(struct video_device *vdev); // 释放video_device
	const struct v4l2_ioctl_ops *ioctl_ops; // v4l2 ioctl操作
	DECLARE_BITMAP(valid_ioctls, BASE_VIDIOC_PRIVATE);

	struct mutex *lock;
};

// https://elixir.bootlin.com/linux/v6.6.30/source/include/media/v4l2-device.h#L47
/**
 * struct v4l2_device - main struct to for V4L2 device drivers
 *
 * @dev: pointer to struct device.
 * @mdev: pointer to struct media_device, may be NULL.
 * @subdevs: used to keep track of the registered subdevs
 * @lock: lock this struct; can be used by the driver as well
 *	if this struct is embedded into a larger struct.
 * @name: unique device name, by default the driver name + bus ID
 * @notify: notify operation called by some sub-devices.
 * @ctrl_handler: The control handler. May be %NULL.
 * @prio: Device's priority state
 * @ref: Keep track of the references to this struct.
 * @release: Release function that is called when the ref count
 *	goes to 0.
 *
 * Each instance of a V4L2 device should create the v4l2_device struct,
 * either stand-alone or embedded in a larger struct.
 *
 * It allows easy access to sub-devices (see v4l2-subdev.h) and provides
 * basic V4L2 device-level support.
 *
 * .. note::
 *
 *    #) @dev->driver_data points to this struct.
 *    #) @dev might be %NULL if there is no parent device
 */
struct v4l2_device {
	struct device *dev;
	struct media_device *mdev;
	struct list_head subdevs;
	spinlock_t lock;
	char name[V4L2_DEVICE_NAME_SIZE];
	void (*notify)(struct v4l2_subdev *sd,
			unsigned int notification, void *arg);
	struct v4l2_ctrl_handler *ctrl_handler;
	struct v4l2_prio_state prio;
	struct kref ref;
	void (*release)(struct v4l2_device *v4l2_dev);
};

// https://elixir.bootlin.com/linux/v6.6.30/source/include/media/v4l2-subdev.h#L1050
/**
 * struct v4l2_subdev - describes a V4L2 sub-device
 *
 * @entity: pointer to &struct media_entity
 * @list: List of sub-devices
 * @owner: The owner is the same as the driver's &struct device owner.
 * @owner_v4l2_dev: true if the &sd->owner matches the owner of @v4l2_dev->dev
 *	owner. Initialized by v4l2_device_register_subdev().
 * @flags: subdev flags. Can be:
 *   %V4L2_SUBDEV_FL_IS_I2C - Set this flag if this subdev is a i2c device;
 *   %V4L2_SUBDEV_FL_IS_SPI - Set this flag if this subdev is a spi device;
 *   %V4L2_SUBDEV_FL_HAS_DEVNODE - Set this flag if this subdev needs a
 *   device node;
 *   %V4L2_SUBDEV_FL_HAS_EVENTS -  Set this flag if this subdev generates
 *   events.
 *
 * @v4l2_dev: pointer to struct &v4l2_device
 * @ops: pointer to struct &v4l2_subdev_ops
 * @internal_ops: pointer to struct &v4l2_subdev_internal_ops.
 *	Never call these internal ops from within a driver!
 * @ctrl_handler: The control handler of this subdev. May be NULL.
 * @name: Name of the sub-device. Please notice that the name must be unique.
 * @grp_id: can be used to group similar subdevs. Value is driver-specific
 * @dev_priv: pointer to private data
 * @host_priv: pointer to private data used by the device where the subdev
 *	is attached.
 * @devnode: subdev device node
 * @dev: pointer to the physical device, if any
 * @fwnode: The fwnode_handle of the subdev, usually the same as
 *	    either dev->of_node->fwnode or dev->fwnode (whichever is non-NULL).
 * @async_list: Links this subdev to a global subdev_list or
 *		@notifier->done_list list.
 * @async_subdev_endpoint_list: List entry in async_subdev_endpoint_entry of
 *				&struct v4l2_async_subdev_endpoint.
 * @subdev_notifier: A sub-device notifier implicitly registered for the sub-
 *		     device using v4l2_async_register_subdev_sensor().
 * @asc_list: Async connection list, of &struct
 *	      v4l2_async_connection.subdev_entry.
 * @pdata: common part of subdevice platform data
 * @state_lock: A pointer to a lock used for all the subdev's states, set by the
 *		driver. This is	optional. If NULL, each state instance will get
 *		a lock of its own.
 * @privacy_led: Optional pointer to a LED classdev for the privacy LED for sensors.
 * @active_state: Active state for the subdev (NULL for subdevs tracking the
 *		  state internally). Initialized by calling
 *		  v4l2_subdev_init_finalize().
 * @enabled_streams: Bitmask of enabled streams used by
 *		     v4l2_subdev_enable_streams() and
 *		     v4l2_subdev_disable_streams() helper functions for fallback
 *		     cases.
 *
 * Each instance of a subdev driver should create this struct, either
 * stand-alone or embedded in a larger struct.
 *
 * This structure should be initialized by v4l2_subdev_init() or one of
 * its variants: v4l2_spi_subdev_init(), v4l2_i2c_subdev_init().
 */
struct v4l2_subdev {
#if defined(CONFIG_MEDIA_CONTROLLER)
	struct media_entity entity;
#endif
	struct list_head list; // 将其链接到v4l2_device的subdevs的链表中
	struct module *owner;
	bool owner_v4l2_dev;
	u32 flags;
	struct v4l2_device *v4l2_dev; // 关联的v4l2_device
	const struct v4l2_subdev_ops *ops; // 专有操作
	const struct v4l2_subdev_internal_ops *internal_ops; // 通用操作
	struct v4l2_ctrl_handler *ctrl_handler;
	char name[V4L2_SUBDEV_NAME_SIZE];
	u32 grp_id;
	void *dev_priv; // 私有数据
	void *host_priv;
	struct video_device *devnode;
	struct device *dev; // 关联的device
	struct fwnode_handle *fwnode;
	struct list_head async_list;
	struct list_head async_subdev_endpoint_list;
	struct v4l2_async_notifier *subdev_notifier;
	struct list_head asc_list;
	struct v4l2_subdev_platform_data *pdata;
	struct mutex *state_lock;

	/*
	 * The fields below are private, and should only be accessed via
	 * appropriate functions.
	 */

	struct led_classdev *privacy_led;

	/*
	 * TODO: active_state should most likely be changed from a pointer to an
	 * embedded field. For the time being it's kept as a pointer to more
	 * easily catch uses of active_state in the cases where the driver
	 * doesn't support it.
	 */
	struct v4l2_subdev_state *active_state;
	u64 enabled_streams;
};
```

video_device的字段多与设备和文件操作有关,这是它的主要使命, 也可以定义一个更加复杂的结构体(下文以my_video_device为例)将它内嵌其中. 调用video_register_device函数可以注册一个video_device,调用video_unregister_device函数可以取消注册.

video_register_device的第二个参数type表示设备的类型,会被赋值给video_device的vfl_type字段。第三个参数表示期望的同类设备中的序号,该参数不等于-1时,函数从nr开始查找空闲序号,失败则从0开始重新查找;该参数等于-1时直接从0开始查找,最终的序号被赋值给num字段.

函数执行成功后会生成用户空间使用的文件,文件的名字由设备的类型和序号决定,序号决定了名字中的数字,比如/dev/video0就是序号为0的VFL_TYPE_GRABBER设备.

设备类型和名字的关系:
- VFL_TYPE_GRABBER, video, 视频输入输出设备
- VFL_TYPE_VBI, vbi, vbi设备
- VFL_TYPE_RADIO, radio, radi tuners
- VFL_TYPE_SUBDEV, v4l-subdev, V4L2子设备
- VFL_TYPE_SDR, swradio, Software Defined Radio tuners
- VFL_TYPE_TOUCH, v4l-touch, 触摸屏

video_register_device会为video_device查找一个空闲的次设备号赋值给它的minor字段,该次设备号在所有video_device中是唯一的(并不局限于视频输入输出设备,上表中所有的设备类型均包括在内).

得到次设备号后,video_register_device会将它作为数组下标,保存video_device指针。该数组是一个video_device指针类型的数组,名为video_devices,这样我们在文件的操作中使用次设备号就可以得到video_device指针,内核定义了video_devdata函数实现该功能.

video_device的fh_list字段是v4l2_fh组成的链表的头,v4l2_fh是V4L2文件的句柄(file handle),使用句柄的V4L2文件打开的时候, 需要调用v4l2_fh_init和v4l2_fh_add或者调用v4l2_fh_open初始化并添加一个句柄到链表中。句柄在某些文件操作中会作为参数,我们可以定义一个更大的结构体将其内嵌其中,使用container_of得到整个结构体。

video_device与设备实体并不一定是一对一的关系,设备实体可以按照物理模块或者逻辑模块划分为多个部分,每一个部分都可以注册为一个video_device,比如一个isp,可以注册preview、capture和control等多个video_device.

与物理设备对应的结构体是v4l2_device,它是V4L2设备驱动的核心结构体.

v4l2_device一般会内嵌在一个更大的结构体中(下文以my_v4l2_device为例)使用,dev字段是device指针类型,指向设备实体,比如一个platform_device的dev。内核定义了v4l2_device_register函数,可以用来注册并初始化v4l2_device,调用v4l2_device_unregister用来取消注册.

v4l2_device_register会为dev字段赋值,如果还未取名字,根据驱动名和设备名为name字段赋值。如果dev_get_drvdata为空,调用dev_set_drvdata (dev, v4l2_dev),此后调用dev_get_drvdata(dev)即可得到v4l2_dev。

video_device的v4l2_dev字段指向v4l2_device,这样我们就可以在文件操作中通过video_device得到v4l2_device,除此之外,我们还可以调用video_set_drvdata建立video_device和my_v4l2_device的联系,然后可以通过video_get_drvdata直接得到my_v4l2_device.

在 Camera 硬件结构中 ,isp与Camera 是一对多的关系,此处的v4l2_device与isp对应,与Camera(更确切地说是i2cclient)对应的结构体是v4l2_subdev(代码中多简称sd),显然v4l2_device与它也是一对多的关系.

可把Camera当作一个采光的传感器。实际上,控制Camera的是i2c,CPU通过i2c读写它的寄存器控制它的行为,所以实际上与v4l2_subdev对应的是Camera对应的i2c client,而不是Camera本身.

v4l2_subdev的dev字段,与v4l2_device的dev一样,都是device指针类型,这意味着它们实际上都不是device,只是关联device。在例子
中,v4l2_device关联的是isp的platform_device.dev,v4l2_subdev关联的是Camera对应的i2c_client.dev.

v4l2_subdev_internal_ops定义了针对v4l2_subdev的通用操作,共registered、unregistered、open和close四种。

v4l2_subdev_ops定义了适合不同类型的v4l2_subdev的操作,包括core、tuner、audio、video、vbi、ir、sensor和pad八种,某个v4l2_subdev只需要定义它适用的操作即可。

一个v4l2_subdev设备的驱动中,需要通过设备找到v4l2_subdev,也需要通过v4l2_subdev找到设备。本例中的设备指的就是i2c_client对象(以下简称client),v4l2_i2c_subdev_init函数可以帮助我们完成该任务。它初始化v4l2_subdev的字段,并调用v4l2_set_subdevdata(sd,client)和i2c_set_clientdata(client,sd)建立需要的联系,成功后在驱动中
调用v4l2_get_subdevdata(sd)和i2c_get_clientdata(client)即可得到client和sd。

v4l2_subdev可以内嵌在一个更大的结构体内使用.

在Camera结构中,v4l2_device对应isp,video_device对应它的几个部分,v4l2_subdev对应Camera,在驱动中已经确立了v4l2_device和video_device的关系(my_v4l2_probe)。实际使用过程中,用户空间对V4L2文件的操作也需要v4l2_subdev的参与,所以剩下的问题就是v4l2_subdev是如何与它们关联的。

一 般 情 况 下 , 与 v4l2_subdev 关 联 的 是 v4l2_device , 而 不 是video_device,这从逻辑上也是合理的,video_device的任务主要是
V4L2文件操作,所以问题就变成了v4l2_device与v4l2_subdev的关系是如何建立的。实际上,它们建立关系的方式并不是一成不变的,甚至
是平台(isp一般集成在平台中)可以自由发挥,宗旨只有一个,就是在文件操作中,通过v4l2_device可以找到v4l2_subdev。

这些方式可以分为两类,一类要求注册v4l2_device时v4l2_subdev是已知的,另一类没有该要求,或者称为异步的(asynchronously)。

第一类方式的实现又可以分为两种,第一种是使用内核定义的函数,由v4l2_device的驱动注册v4l2_subdev,v4l2_device_register_subdev可以胜任,该函数成功返回后,sd->v4l2_dev会指向v4l2_dev,sd会链接到v4l2_dev->subdevs的链表上,二者关系建立完毕.

第二种实现比较自由,比如可以在my_v4l2_device中添加一个字段来表示关联的sd, 在结构中有两个Camera.

```c
struct my_v4l2_device {
    struct v4l2_device v4l2_dev;
    struct v4l2_subdev *sd_array;
    int curr_input;
}
```

两 个 Camera , 一 个 前 摄 , 一 个 后 摄 。 我 们 可 以 在 驱 动 中 指 定sd_array[0]为前摄,sd_array[1]为后摄,curr_input表示当前使用的是哪
个Camera,以此应对用户空间的要求。

第一类方式的两种实现都有一个前提,就是v4l2_subdev是已知的,第二种要求更加苛刻,加载v4l2_device驱动前,v4l2_subdev必须已经注册,也就是说Camera的驱动要在此之前加载.

第二类异步方式动态匹配v4l2_device和v4l2_subdev,对它们的顺序并没有要求。注册v4l2_device时,调用v4l2_async_notifier_register,注册v4l2_subdev时,调用v4l2_async_register_subdev即可.

内核分别为notifier和sd维护了notifier_list和subdev_list两个全局链表,以上函数的本质就是访问链表,匹配链表上的元素,匹配成功则调用v4l2_device_register_subdev建立它们的关系。

至此,从video_device到v4l2_device,再到v4l2_subdev的通路建立完毕.

### ioctl
video_device共有3个与文件操作相关的字段,除了以上介绍的fops和ioctl_ops外,还有一个间接的cdev->ops字段,三者的类型分别为`v4l2_file_operations* 、 v4l2_ioctl_ops* 和 file_operations*`。 按 照 类型“对号入座”,V4L2文件首先是文件(确切地说是字符文件),cdev->ops表示它的文件操作,其次它又是V4L2文件,有V4L2相关的特殊操作,由fops表示,至于ioctl_ops,它表示V4L2文件的ioctl操作,将它单独作为一个字段,足见ioctl操作对V4L2架构的重要性.

与用户空间关系最紧密的是cdev->ops,它被赋值为v4l2_fops,对V4L2文件的操作由它实现。我们知道,文件操作(file_operations)定义的回调函数都有一个file*参数(不妨称之为filp),结合上节对
video_device 的 讨 论 , 调 用 video_devdata(filp) 就 可 以 得 到video_device(vdev),进而可以调用vdev->fops的对应操作.

类似的,cdev->ops->unlocked_ioctl定义为v4l2_ioctl,它调用的是vdev->fops->unlocked_ioctl 。 驱 动 可 以 自 行 为 vdev 定 义 fops-
>unlocked_ioctl,不过一般情况下,使用内核提供的video_ioctl2函数即可.

video_ioctl2由video_usercopy实现,后者获取用户空间传递的ioctl操作的参数(arg),然后调用__video_do_ioctl。根据用户空间传递的
ioctl操作,__video_do_ioctl会做出几种不同的处理。

V4L2定义了一个v4l2_ioctl_info(简称info)类型的数组,名为v4l2_ioctls,包含了几十种ioctl操作的信息,它们是V4L2系统预定义的ioctl操作。根据info的func回调函数的实现方式不同,可用将它们分为两大类,一类是stub,直接调用vdev->ioctl_ops实现;另一类由V4L2定义的特定函数实现.

一个video_device支持的ioctls操作可用分为两部分,一部分包含在v4l2_ioctls 内 , 另 一 部 分 由 vdev->ioctl_ops->vidioc_default 实 现 。video_device支持的在v4l2_ioctls内的操作由它的valid_ioctls字段表示,video_device注册的时候会根据ioctl_ops字段的定义设置该字段。

不考虑特殊情况,可以将问题分为以下三种情形。

第一种,用户空间传递的ioctl操作在v4l2_ioctls定义的范围内,且video_device支持该操作,这是最常见的情况,调用info->func回调函数。
第二种,用户空间传递的ioctl操作在v4l2_ioctls定义的范围内,但video_device不支持该操作,可能是用户空间与驱动的匹配出现问题。一般情况下,用户空间的程序需要根据驱动做相应调整,除非出于特殊考虑。
第三种,用户空间传递的ioctl操作不在v4l2_ioctls定义的范围内,调用vdev->ioctl_ops->vidioc_default回调函数,如果函数没有定义,返回错误(-ENOTTY).

举几个具体的例子。

如 果 用 户 空 间 传 递 的 ioctl 操 作 是 VIDIOC_G_INPUT , 它 在v4l2_ioctls 定 义 的 范 围 内 , 调 用 的 是 VIDIOC_G_INPUT 对 应 的 info-
>func,它属于stub类函数,直接回调vdev->ioctl_ops->vidioc_g_input函数;如果是VIDIOC_S_INPUT操作,同样在v4l2_ioctls定义的范围内,但 实 现 函 数 不 是 stub 类 , 调 用 的 是 v4l_s_input 函 数 ; 如 果 是 不 在v4l2_ioctls 定 义 的 范 围 内 的 其 他 的 自 定 义 操 作 , 调 用 的 是 vdev->ioctl_ops->vidioc_default函数.

非stub类info调用的是V4L2定义的函数,与vdev->ioctl_ops没有直接调用关系,但是这些函数多数也是通过调用vdev->ioctl_ops实现的, 比如v4l_s_input,它最终还是调用vdev->ioctl_ops->vidioc_s_input函数实现.

至此,cdev->ops->unlocked_ioctl到vdev->fops->unlocked_ioctl,再到vdev->ioctl_ops的过程分析完毕。该过程中,出现了video_device,但v4l2_device和v4l2_subdev仍未“出场”,它们的作用体现在哪里呢? 答案在vdev->ioctl_ops的实现中.

### Camera的核心ioctl操作
V4L2有很强的通用性,同一类设备,一般使用同一套ioctl操作即可,Camera常用的ioctl操作.
```c
// https://elixir.bootlin.com/linux/v6.6.30/source/include/uapi/linux/videodev2.h#L2623
/*
 *	I O C T L   C O D E S   F O R   V I D E O   D E V I C E S
 *
 */
#define VIDIOC_QUERYCAP		 _IOR('V',  0, struct v4l2_capability) // 查询设备的功能, query capability
#define VIDIOC_ENUM_FMT         _IOWR('V',  2, struct v4l2_fmtdesc) // 查询设备支持的视频格式
#define VIDIOC_G_FMT		_IOWR('V',  4, struct v4l2_format) // 查询当前设备使用的格式
#define VIDIOC_S_FMT		_IOWR('V',  5, struct v4l2_format) // 设置当前设备使用的格式
#define VIDIOC_REQBUFS		_IOWR('V',  8, struct v4l2_requestbuffers) // 申请多个缓冲区
#define VIDIOC_QUERYBUF		_IOWR('V',  9, struct v4l2_buffer) // 查询某个缓冲区
#define VIDIOC_G_FBUF		 _IOR('V', 10, struct v4l2_framebuffer)
#define VIDIOC_S_FBUF		 _IOW('V', 11, struct v4l2_framebuffer)
#define VIDIOC_OVERLAY		 _IOW('V', 14, int)
#define VIDIOC_QBUF		_IOWR('V', 15, struct v4l2_buffer) // 将缓冲区放入队列
#define VIDIOC_EXPBUF		_IOWR('V', 16, struct v4l2_exportbuffer)
#define VIDIOC_DQBUF		_IOWR('V', 17, struct v4l2_buffer) // 将缓冲区从队列中删除, 获取数据
#define VIDIOC_STREAMON		 _IOW('V', 18, int) // stream on
#define VIDIOC_STREAMOFF	 _IOW('V', 19, int) // stream off
#define VIDIOC_G_PARM		_IOWR('V', 21, struct v4l2_streamparm)
#define VIDIOC_S_PARM		_IOWR('V', 22, struct v4l2_streamparm)
#define VIDIOC_G_STD		 _IOR('V', 23, v4l2_std_id)
#define VIDIOC_S_STD		 _IOW('V', 24, v4l2_std_id)
#define VIDIOC_ENUMSTD		_IOWR('V', 25, struct v4l2_standard)
#define VIDIOC_ENUMINPUT	_IOWR('V', 26, struct v4l2_input) // 查询某个设备的信息
#define VIDIOC_G_CTRL		_IOWR('V', 27, struct v4l2_control) // 查询指定control的值
#define VIDIOC_S_CTRL		_IOWR('V', 28, struct v4l2_control) // 设置指定control的值
#define VIDIOC_G_TUNER		_IOWR('V', 29, struct v4l2_tuner)
#define VIDIOC_S_TUNER		 _IOW('V', 30, struct v4l2_tuner)
#define VIDIOC_G_AUDIO		 _IOR('V', 33, struct v4l2_audio)
#define VIDIOC_S_AUDIO		 _IOW('V', 34, struct v4l2_audio)
#define VIDIOC_QUERYCTRL	_IOWR('V', 36, struct v4l2_queryctrl)
#define VIDIOC_QUERYMENU	_IOWR('V', 37, struct v4l2_querymenu)
#define VIDIOC_G_INPUT		 _IOR('V', 38, int) // 查询当前使用的设备的信息
#define VIDIOC_S_INPUT		_IOWR('V', 39, int) // 选择一个设备
#define VIDIOC_G_EDID		_IOWR('V', 40, struct v4l2_edid)
#define VIDIOC_S_EDID		_IOWR('V', 41, struct v4l2_edid)
#define VIDIOC_G_OUTPUT		 _IOR('V', 46, int)
#define VIDIOC_S_OUTPUT		_IOWR('V', 47, int)
#define VIDIOC_ENUMOUTPUT	_IOWR('V', 48, struct v4l2_output)
#define VIDIOC_G_AUDOUT		 _IOR('V', 49, struct v4l2_audioout)
#define VIDIOC_S_AUDOUT		 _IOW('V', 50, struct v4l2_audioout)
#define VIDIOC_G_MODULATOR	_IOWR('V', 54, struct v4l2_modulator)
#define VIDIOC_S_MODULATOR	 _IOW('V', 55, struct v4l2_modulator)
#define VIDIOC_G_FREQUENCY	_IOWR('V', 56, struct v4l2_frequency)
#define VIDIOC_S_FREQUENCY	 _IOW('V', 57, struct v4l2_frequency)
#define VIDIOC_CROPCAP		_IOWR('V', 58, struct v4l2_cropcap)
#define VIDIOC_G_CROP		_IOWR('V', 59, struct v4l2_crop)
#define VIDIOC_S_CROP		 _IOW('V', 60, struct v4l2_crop)
#define VIDIOC_G_JPEGCOMP	 _IOR('V', 61, struct v4l2_jpegcompression)
#define VIDIOC_S_JPEGCOMP	 _IOW('V', 62, struct v4l2_jpegcompression)
#define VIDIOC_QUERYSTD		 _IOR('V', 63, v4l2_std_id)
#define VIDIOC_TRY_FMT		_IOWR('V', 64, struct v4l2_format)
#define VIDIOC_ENUMAUDIO	_IOWR('V', 65, struct v4l2_audio)
#define VIDIOC_ENUMAUDOUT	_IOWR('V', 66, struct v4l2_audioout)
#define VIDIOC_G_PRIORITY	 _IOR('V', 67, __u32) /* enum v4l2_priority */
#define VIDIOC_S_PRIORITY	 _IOW('V', 68, __u32) /* enum v4l2_priority */
#define VIDIOC_G_SLICED_VBI_CAP _IOWR('V', 69, struct v4l2_sliced_vbi_cap)
#define VIDIOC_LOG_STATUS         _IO('V', 70)
#define VIDIOC_G_EXT_CTRLS	_IOWR('V', 71, struct v4l2_ext_controls) // 查询多个control的值
#define VIDIOC_S_EXT_CTRLS	_IOWR('V', 72, struct v4l2_ext_controls) // 设置多个control的值
#define VIDIOC_TRY_EXT_CTRLS	_IOWR('V', 73, struct v4l2_ext_controls)
#define VIDIOC_ENUM_FRAMESIZES	_IOWR('V', 74, struct v4l2_frmsizeenum)
#define VIDIOC_ENUM_FRAMEINTERVALS _IOWR('V', 75, struct v4l2_frmivalenum)
#define VIDIOC_G_ENC_INDEX       _IOR('V', 76, struct v4l2_enc_idx)
#define VIDIOC_ENCODER_CMD      _IOWR('V', 77, struct v4l2_encoder_cmd)
#define VIDIOC_TRY_ENCODER_CMD  _IOWR('V', 78, struct v4l2_encoder_cmd)

/*
 * Experimental, meant for debugging, testing and internal use.
 * Only implemented if CONFIG_VIDEO_ADV_DEBUG is defined.
 * You must be root to use these ioctls. Never use these in applications!
 */
#define	VIDIOC_DBG_S_REGISTER	 _IOW('V', 79, struct v4l2_dbg_register)
#define	VIDIOC_DBG_G_REGISTER	_IOWR('V', 80, struct v4l2_dbg_register)

#define VIDIOC_S_HW_FREQ_SEEK	 _IOW('V', 82, struct v4l2_hw_freq_seek)
#define	VIDIOC_S_DV_TIMINGS	_IOWR('V', 87, struct v4l2_dv_timings)
#define	VIDIOC_G_DV_TIMINGS	_IOWR('V', 88, struct v4l2_dv_timings)
#define	VIDIOC_DQEVENT		 _IOR('V', 89, struct v4l2_event)
#define	VIDIOC_SUBSCRIBE_EVENT	 _IOW('V', 90, struct v4l2_event_subscription)
#define	VIDIOC_UNSUBSCRIBE_EVENT _IOW('V', 91, struct v4l2_event_subscription)
#define VIDIOC_CREATE_BUFS	_IOWR('V', 92, struct v4l2_create_buffers)
#define VIDIOC_PREPARE_BUF	_IOWR('V', 93, struct v4l2_buffer)
#define VIDIOC_G_SELECTION	_IOWR('V', 94, struct v4l2_selection)
#define VIDIOC_S_SELECTION	_IOWR('V', 95, struct v4l2_selection)
#define VIDIOC_DECODER_CMD	_IOWR('V', 96, struct v4l2_decoder_cmd)
#define VIDIOC_TRY_DECODER_CMD	_IOWR('V', 97, struct v4l2_decoder_cmd)
#define VIDIOC_ENUM_DV_TIMINGS  _IOWR('V', 98, struct v4l2_enum_dv_timings)
#define VIDIOC_QUERY_DV_TIMINGS  _IOR('V', 99, struct v4l2_dv_timings)
#define VIDIOC_DV_TIMINGS_CAP   _IOWR('V', 100, struct v4l2_dv_timings_cap)
#define VIDIOC_ENUM_FREQ_BANDS	_IOWR('V', 101, struct v4l2_frequency_band)

/*
 * Experimental, meant for debugging, testing and internal use.
 * Never use this in applications!
 */
#define VIDIOC_DBG_G_CHIP_INFO  _IOWR('V', 102, struct v4l2_dbg_chip_info)

#define VIDIOC_QUERY_EXT_CTRL	_IOWR('V', 103, struct v4l2_query_ext_ctrl)
```

VIDIOC_G_EXT_CTRLS、VIDIOC_S_EXT_CTRLS和VIDIOC_G_CTRL、VIDIOC_S_CTRL除了数量上的区别外,还有一个区 别 就 是 VIDIOC_G_CTRL/VIDIOC_S_CTRL 只 能 操 作 User
Control(可通过用户界面更改的设置),不过很多设备并不区分它们,统一由后两个操作处理.

缓冲区(video buffer)有关的操作,驱动有两套架构可以选择,分别是videobuf和videobuf2。它们使用的结构体也不同,videobuf使用videobuf_queue表示缓冲区队列,使用videobuf_buffer表示一个缓冲区,videobuf2则使用vb2_queue和vb2_buffer。不过,用户空间并不关心它们的差异,也不需要关心缓冲区是如何管理的.

V4L2 的 缓 冲 区 的 访 问 有 V4L2_MEMORY_MMAP 、V4L2_MEMORY_USERPTR 、 V4L2_MEMORY_OVERLAY 和V4L2_MEMORY_DMABUF四种类型,前两种比较常见,videobuf2并
不支持第三种,videobuf不支持第四种.

V4L2_MEMORY_MMAP方式采用内存映射机制,将设备映射进内存,不需要数据拷贝。V4L2_MEMORY_USERPTR方式,内存由用户空间分配,设备访问用户空间指定的内存。需要说明的是,并不是
只有CPU才可以访问内存,有些设备也可以直接访问内存,一般是硬件实现的.

### 3A
Camera的控制是通过ioctl实现的,当更改Camera的设置时,最终也由ioctl实现,3A的设置也是通过它们实现的.

3A 就 是 AF 、 AE 和 AWB , 全 称 分 别 为 Auto Focus 、 AutoEXPOSURE和Auto White Balance,也就是我们常说的自动对焦、自动
曝光和自动白平衡。

它们之中有些是Camera可以实现的,有些是isp实现的,不同的平台上的方案可能不同.