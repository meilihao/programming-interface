# driver
参考:
- [+**Linux驱动之设备驱动模型**](https://carlyleliu.github.io/2020/Linux%E9%A9%B1%E5%8A%A8%E4%B9%8B%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8%E6%A8%A1%E5%9E%8B/)
- [Linux Platform驱动模型(二) _驱动方法](https://www.cnblogs.com/xiaojiang1025/p/6367910.html)
- [**Linux设备驱动模型**](https://mshrimp.github.io/2020/05/10/Linux%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8%E6%A8%A1%E5%9E%8B/)

	有demo代码
- [Linux的设备模型](https://doc.embedfire.com/linux/stm32mp1/driver/zh/latest/linux_driver/base_linux_device_model.html)
- [Linux设备驱动模型简述](https://hughesxu.github.io/posts/Linux_device_and_driver_model/)

	有注册后的数据结构展示
- [<<Linux设备驱动开发详解>>]()

	**由很多编写驱动的模板**

设备的创建就是在Linux内核中维护一些数据结构来对硬件设备进行描述.

![](/misc/img/driver/1771657-20201229232507268-1330517277.png)

linux驱动模型: kernel基于kobject将系统中的总线, 设备和驱动用`bus_type`, `device`和`device_driver`等对象将其组织成一个层次结构的系统, 统一管理各种类别(class)的设备及其接口(class_interface), 同时借助sysfs将内核所见的设备系统展示给用户空间.

总线是处理器与设备之间的通道, 在设备模型中，所有的设备都通过总线相连；总线作为Linux设备驱动模型的核心架构，系统中的设备和驱动都挂接在相应的总线上, 由总线将设备和驱动绑定，来完成各自的工作.

每个子系统有自己的总线类型, 它有一条驱动链表和一条设备链表, 用来链接已加载的驱动和已发现的设备, 驱动加载和设备发现的顺序是任意的. 每个设备至多绑定一个驱动

> 每个设备至多绑定一个驱动: 这个限制在某些情况下, 过于严苛. linux引入类和接口, 每个设备还可以唯一属于某个类, 设备被链入到类的设备链表中. 在设备被发现, 或接口被添加时, 无论何种顺序, 只要它们属于同一个类, 那么接口注册的回调函数都会被作用在设备之上.

> kset是具有相同类型的kobject构成的对象集.

> buses_init()函数在sysfs文件系统的根目录下建立一个bus目录，即/sys/bus，是系统中后续注册总线的连接点. Linux系统在启动时的初始化阶段，通过在driver_init()中调用[buses_init()](https://elixir.bootlin.com/linux/v6.6.13/source/drivers/base/bus.c#L1374)函数，完成所有总线的最初操作，创建出bus的祖先.


```c
// https://elixir.bootlin.com/linux/v6.6.13/source/include/linux/kobject.h#L64
struct kobject {
	const char		*name; // 名称
	struct list_head	entry; // 将kobject链接到kset的连接件
	struct kobject		*parent; //指向包含它的kset的内嵌kobject/其他kobject/NULl
	struct kset		*kset; // 如果kobject已经链接到kset, 则指向它
	const struct kobj_type	*ktype; // 类型
	struct kernfs_node	*sd; /* sysfs directory entry */
	struct kref		kref; // kobject的引用计数

	unsigned int state_initialized:1; // 1: 已被初始化过
	unsigned int state_in_sysfs:1; // 已被添加到内核关系树中
	unsigned int state_add_uevent_sent:1; // 已发送过添加事件到用户空间
	unsigned int state_remove_uevent_sent:1; // 已发送删除事件到用户空间
	unsigned int uevent_suppress:1; // 抑制发送事件到用户空间

#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
	struct delayed_work	release;
#endif
};


// https://elixir.bootlin.com/linux/v6.6.13/source/include/linux/kobject.h#L116
struct kobj_type {
	void (*release)(struct kobject *kobj); // 销毁方法. 不同类型的对象的release方法不同, 同一类型的则相同
	const struct sysfs_ops *sysfs_ops;
	const struct attribute_group **default_groups;
	const struct kobj_ns_type_operations *(*child_ns_type)(const struct kobject *kobj);
	const void *(*namespace)(const struct kobject *kobj);
	void (*get_ownership)(const struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};

// https://elixir.bootlin.com/linux/v6.6.13/source/include/linux/kobject.h#L168
/**
 * struct kset - a set of kobjects of a specific type, belonging to a specific subsystem.
 *
 * A kset defines a group of kobjects.  They can be individually
 * different "types" but overall these kobjects all want to be grouped
 * together and operated on in the same manner.  ksets are used to
 * define the attribute callbacks and other common events that happen to
 * a kobject.
 *
 * @list: the list of all kobjects for this kset
 * @list_lock: a lock for iterating over the kobjects
 * @kobj: the embedded kobject for this kset (recursion, isn't it fun...)
 * @uevent_ops: the set of uevent operations for this kset.  These are
 * called whenever a kobject has something happen to it so that the kset
 * can add new environment variables, or filter out the uevents if so
 * desired.
 */
struct kset {
	struct list_head list; // 保存在该kset里的所有kobject的链表
	spinlock_t list_lock; // 用于遍历这个kset的所有kobject的自旋锁
	struct kobject kobj; // 内嵌kobject
	const struct kset_uevent_ops *uevent_ops; // uevent操作集合
} __randomize_layout;

// https://elixir.bootlin.com/linux/v6.6.13/source/include/linux/kobject.h#L133
struct kset_uevent_ops {
	int (* const filter)(const struct kobject *kobj); // 过滤回调函数, 返回0表示不需要向用户空间报告该事件
	const char *(* const name)(const struct kobject *kobj); // 名称回调函数, 返回子系统名称, 即环境变量SUBSYSTEM的值
	int (* const uevent)(const struct kobject *kobj, struct kobj_uevent_env *env); // uevent回调函数, 添加子系统特定的环境变量
};
```

> kernel会以环境数据的格式(kobj_uevent_env, k=v+`\0`分隔)向用户空间(systemd-udevd.service by netlink机制/user_helper机制)报告内核对象变化. 环境数据是支持设备热插拔的关键组件, 因为该操作需要kernel和用户空间配合完成.

kset作用:
1. 作为包含一组对象的容器, 可用来追踪资源, 比如所有的块设备或者所有pci设备驱动
1. 作为目录级别的"粘合剂", 将设备模型中的kobject和sysfs粘在一起, 构建设备模型层次
1. 支持kobject的热插拔, 影响热插拔事件被报告给用户空间的方式

热插拔事件是一个从内核空间发送到用户空间的通知, 表示系统配置已改变, 无论kobject被创建或删除, 都会产生这种事件. 热插拔事件会导致对/sbin/hotplug的调用, 它通过加载驱动, 创建设备节点, 挂载分区或者其他响应动作. 热插拔事件通常是由总线驱动程序产生的.

kobject的创建和初始化分别是kobject_create和kobject_init. kobject_add是向sysfs注册它.

<table>
<thead>
<tr>
<th>名称</th>
<th>类型</th>
<th>对应的数据结构</th>
<th>代码文件</th>
</tr>
</thead>
<tbody><tr>
<td>总线</td>
<td>bus</td>
<td>struct bus_type+struct subsys_private</td>
<td>drivers/base/bus.c</td>
</tr>
<tr>
<td>设备</td>
<td>device</td>
<td>struct device+device_private</td>
<td>drivers/base/core.c</td>
</tr>
<tr>
<td>驱动</td>
<td>driver</td>
<td>struct device_driver(include/linux/device.h)+driver_private</td>
<td>drivers/base/driver.c</td>
</tr>
<tr>
<td>类/接口</td>
<td></td>
<td>class+class_interface(https://elixir.bootlin.com/linux/v6.6.13/source/include/linux/device/class.h#L52)</td>
<td></td>
</tr>
</tbody></table>

platform总线作为Linux的基础总线，在内核启动阶段便完成了注册，注册的入口函数为platform_bus_init(), 它对platform总线的注册主要分为两步：
1. device_register(&platform_bus)

	1. device_initialize()：对struct device中基本成员进行初始化，包括kobject、struct device_private、struct mutex等。
	1. device_add(dev)：将platform总线也作为一个设备platform_bus注册到驱动模型中，重要的函数包括device_create_file()、device_add_class_symlinks()、bus_add_device()、bus_probe_device()等. device_add(&platform_bus)主要功能是完成/sys/devices/platform目录的建立
1. bus_register(&platform_bus_type)

	bus_register(&platform_bus_type)将总线platform注册到Linux的总线系统中，主要完成了subsystem的注册，对struct subsys_private结构进行了初始化，具体包括：

	1. platform_bus_type->p->drivers_autoprobe = 1
	1. 对struct kset类型成员subsys进行初始化，作为子系统中kobject对象的parent。kset本身也包含kobject对象，在sysfs中也表现为一个目录，即/sys/bus/platform。
	1. 建立struct kset类型的drivers_kset和devices_kset，作为总线下挂载的所有驱动和设备的集合，sysfs中表现为/sys/bus/platform/drivers和/sys/bus/platform/devices。
	1. 初始化链表klist_drivers和klist_devices，将总线下的驱动和设备分别链接在一起。
	1. 增加probe文件，对应/sys/bus/platform目录的文件drivers_autoprobe和drivers_probe

Linux内核中对依赖于platform总线的驱动定义了platform_driver结构体，内部封装了struct device_driver, 以驱动globalfifo_driver为实例. globalfifo_driver注册的入口函数为platform_driver_register(&globalfifo_driver)，具体实现为__platform_driver_register(&globalfifo_driver, THIS_MODULE)。
该函数会对struct device_driver的bus、probe、remove等回调函数进行初始化，紧接着调用driver_register(&globalfifo_driver->driver).

总线用struct subsys_private结构体来管理总线中设备和驱动的关系:
```c
// https://elixir.bootlin.com/linux/v6.6.13/source/drivers/base/base.h#L42
/**
 * struct subsys_private - structure to hold the private to the driver core portions of the bus_type/class structure.
 *
 * @subsys - the struct kset that defines this subsystem
 * @devices_kset - the subsystem's 'devices' directory
 * @interfaces - list of subsystem interfaces associated
 * @mutex - protect the devices, and interfaces lists.
 *
 * @drivers_kset - the list of drivers associated
 * @klist_devices - the klist to iterate over the @devices_kset
 * @klist_drivers - the klist to iterate over the @drivers_kset
 * @bus_notifier - the bus notifier list for anything that cares about things
 *                 on this bus.
 * @bus - pointer back to the struct bus_type that this structure is associated
 *        with.
 * @dev_root: Default device to use as the parent.
 *
 * @glue_dirs - "glue" directory to put in-between the parent device to
 *              avoid namespace conflicts
 * @class - pointer back to the struct class that this structure is associated
 *          with.
 * @lock_key:	Lock class key for use by the lock validator
 *
 * This structure is the one that is the actual kobject allowing struct
 * bus_type/class to be statically allocated safely.  Nothing outside of the
 * driver core should ever touch these fields.
 */
struct subsys_private {
	struct kset subsys; // bus所在的子系统
	struct kset *devices_kset; // bus上所有设备的集合
	struct list_head interfaces;
	struct mutex mutex;

	struct kset *drivers_kset; // bus上所有驱动的集合
	struct klist klist_devices; // 该总线上所有设备的链表
	struct klist klist_drivers; // 该总线上所有驱动的链表
	struct blocking_notifier_head bus_notifier; // 总线类型变化通知链表的表头. 在总线类型某些变化时将通知这个链表中的元素, 各个元素注册的回调函数将会被调用以便对变化进行相应的处理, 比如添加/删除/绑定/解绑设备
	unsigned int drivers_autoprobe:1; // 向系统总线中注册设备或驱动时，是否进行设备和驱动的绑定操作
	const struct bus_type *bus; // 指向相关联的bus_type的指针
	struct device *dev_root;

	struct kset glue_dirs;
	const struct class *class;

	struct lock_class_key lock_key;
};

// https://elixir.bootlin.com/linux/v6.6.13/source/include/linux/device/bus.h#L80
/**
 * struct bus_type - The bus type of the device
 *
 * @name:	The name of the bus.
 * @dev_name:	Used for subsystems to enumerate devices like ("foo%u", dev->id).
 * @bus_groups:	Default attributes of the bus.
 * @dev_groups:	Default attributes of the devices on the bus.
 * @drv_groups: Default attributes of the device drivers on the bus.
 * @match:	Called, perhaps multiple times, whenever a new device or driver
 *		is added for this bus. It should return a positive value if the
 *		given device can be handled by the given driver and zero
 *		otherwise. It may also return error code if determining that
 *		the driver supports the device is not possible. In case of
 *		-EPROBE_DEFER it will queue the device for deferred probing.
 * @uevent:	Called when a device is added, removed, or a few other things
 *		that generate uevents to add the environment variables.
 * @probe:	Called when a new device or driver add to this bus, and callback
 *		the specific driver's probe to initial the matched device.
 * @sync_state:	Called to sync device state to software state after all the
 *		state tracking consumers linked to this device (present at
 *		the time of late_initcall) have successfully bound to a
 *		driver. If the device has no consumers, this function will
 *		be called at late_initcall_sync level. If the device has
 *		consumers that are never bound to a driver, this function
 *		will never get called until they do.
 * @remove:	Called when a device removed from this bus.
 * @shutdown:	Called at shut-down time to quiesce the device.
 *
 * @online:	Called to put the device back online (after offlining it).
 * @offline:	Called to put the device offline for hot-removal. May fail.
 *
 * @suspend:	Called when a device on this bus wants to go to sleep mode.
 * @resume:	Called to bring a device on this bus out of sleep mode.
 * @num_vf:	Called to find out how many virtual functions a device on this
 *		bus supports.
 * @dma_configure:	Called to setup DMA configuration on a device on
 *			this bus.
 * @dma_cleanup:	Called to cleanup DMA configuration on a device on
 *			this bus.
 * @pm:		Power management operations of this bus, callback the specific
 *		device driver's pm-ops.
 * @iommu_ops:  IOMMU specific operations for this bus, used to attach IOMMU
 *              driver implementations to a bus and allow the driver to do
 *              bus-specific setup
 * @need_parent_lock:	When probing or removing a device on this bus, the
 *			device core should lock the device's parent.
 *
 * A bus is a channel between the processor and one or more devices. For the
 * purposes of the device model, all devices are connected via a bus, even if
 * it is an internal, virtual, "platform" bus. Buses can plug into each other.
 * A USB controller is usually a PCI device, for example. The device model
 * represents the actual connections between buses and the devices they control.
 * A bus is represented by the bus_type structure. It contains the name, the
 * default attributes, the bus' methods, PM operations, and the driver core's
 * private data.
 */
struct bus_type {
	const char		*name;
	const char		*dev_name;
	const struct attribute_group **bus_groups;
	const struct attribute_group **dev_groups;
	const struct attribute_group **drv_groups;

	int (*match)(struct device *dev, struct device_driver *drv); // 检查给定的驱动是否可以和设备绑定
	int (*uevent)(const struct device *dev, struct kobj_uevent_env *env); // 添加总线类型特定的环境变量
	int (*probe)(struct device *dev); // 如果设备可以被绑定到驱动, 则用来对设备进行初始化
	void (*sync_state)(struct device *dev);
	void (*remove)(struct device *dev); // 将设备从驱动解绑时调用的回调函数
	void (*shutdown)(struct device *dev); // 在设备被断电时调用的回调函数

	int (*online)(struct device *dev);
	int (*offline)(struct device *dev);

	int (*suspend)(struct device *dev, pm_message_t state); // 在该总线类型的设备进入到节能状态时调用的方法, 保存硬件上下文状态, 并改变设备的电影级别
	int (*resume)(struct device *dev); // 在该总线类型的设备恢复到正常状态时调用的方法, 改变设备的电源级别, 并回复硬件上下文状态

	int (*num_vf)(struct device *dev);

	int (*dma_configure)(struct device *dev);
	void (*dma_cleanup)(struct device *dev);

	const struct dev_pm_ops *pm; // 指向总线类型的电源管理操作表的指针

	const struct iommu_ops *iommu_ops;

	bool need_parent_lock;
};

// https://elixir.bootlin.com/linux/v6.6.13/source/include/linux/device.h#L705
/**
 * struct device - The basic device structure
 * @parent:	The device's "parent" device, the device to which it is attached.
 * 		In most cases, a parent device is some sort of bus or host
 * 		controller. If parent is NULL, the device, is a top-level device,
 * 		which is not usually what you want.
 * @p:		Holds the private data of the driver core portions of the device.
 * 		See the comment of the struct device_private for detail.
 * @kobj:	A top-level, abstract class from which other classes are derived.
 * @init_name:	Initial name of the device.
 * @type:	The type of device.
 * 		This identifies the device type and carries type-specific
 * 		information.
 * @mutex:	Mutex to synchronize calls to its driver.
 * @bus:	Type of bus device is on.
 * @driver:	Which driver has allocated this
 * @platform_data: Platform data specific to the device.
 * 		Example: For devices on custom boards, as typical of embedded
 * 		and SOC based hardware, Linux often uses platform_data to point
 * 		to board-specific structures describing devices and how they
 * 		are wired.  That can include what ports are available, chip
 * 		variants, which GPIO pins act in what additional roles, and so
 * 		on.  This shrinks the "Board Support Packages" (BSPs) and
 * 		minimizes board-specific #ifdefs in drivers.
 * @driver_data: Private pointer for driver specific info.
 * @links:	Links to suppliers and consumers of this device.
 * @power:	For device power management.
 *		See Documentation/driver-api/pm/devices.rst for details.
 * @pm_domain:	Provide callbacks that are executed during system suspend,
 * 		hibernation, system resume and during runtime PM transitions
 * 		along with subsystem-level and driver-level callbacks.
 * @em_pd:	device's energy model performance domain
 * @pins:	For device pin management.
 *		See Documentation/driver-api/pin-control.rst for details.
 * @msi:	MSI related data
 * @numa_node:	NUMA node this device is close to.
 * @dma_ops:    DMA mapping operations for this device.
 * @dma_mask:	Dma mask (if dma'ble device).
 * @coherent_dma_mask: Like dma_mask, but for alloc_coherent mapping as not all
 * 		hardware supports 64-bit addresses for consistent allocations
 * 		such descriptors.
 * @bus_dma_limit: Limit of an upstream bridge or bus which imposes a smaller
 *		DMA limit than the device itself supports.
 * @dma_range_map: map for DMA memory ranges relative to that of RAM
 * @dma_parms:	A low level driver may set these to teach IOMMU code about
 * 		segment limitations.
 * @dma_pools:	Dma pools (if dma'ble device).
 * @dma_mem:	Internal for coherent mem override.
 * @cma_area:	Contiguous memory area for dma allocations
 * @dma_io_tlb_mem: Software IO TLB allocator.  Not for driver use.
 * @dma_io_tlb_pools:	List of transient swiotlb memory pools.
 * @dma_io_tlb_lock:	Protects changes to the list of active pools.
 * @dma_uses_io_tlb: %true if device has used the software IO TLB.
 * @archdata:	For arch-specific additions.
 * @of_node:	Associated device tree node.
 * @fwnode:	Associated device node supplied by platform firmware.
 * @devt:	For creating the sysfs "dev".
 * @id:		device instance
 * @devres_lock: Spinlock to protect the resource of the device.
 * @devres_head: The resources list of the device.
 * @knode_class: The node used to add the device to the class list.
 * @class:	The class of the device.
 * @groups:	Optional attribute groups.
 * @release:	Callback to free the device after all references have
 * 		gone away. This should be set by the allocator of the
 * 		device (i.e. the bus driver that discovered the device).
 * @iommu_group: IOMMU group the device belongs to.
 * @iommu:	Per device generic IOMMU runtime data
 * @physical_location: Describes physical location of the device connection
 *		point in the system housing.
 * @removable:  Whether the device can be removed from the system. This
 *              should be set by the subsystem / bus driver that discovered
 *              the device.
 *
 * @offline_disabled: If set, the device is permanently online.
 * @offline:	Set after successful invocation of bus type's .offline().
 * @of_node_reused: Set if the device-tree node is shared with an ancestor
 *              device.
 * @state_synced: The hardware state of this device has been synced to match
 *		  the software state of this device by calling the driver/bus
 *		  sync_state() callback.
 * @can_match:	The device has matched with a driver at least once or it is in
 *		a bus (like AMBA) which can't check for matching drivers until
 *		other devices probe successfully.
 * @dma_coherent: this particular device is dma coherent, even if the
 *		architecture supports non-coherent devices.
 * @dma_ops_bypass: If set to %true then the dma_ops are bypassed for the
 *		streaming DMA operations (->map_* / ->unmap_* / ->sync_*),
 *		and optionall (if the coherent mask is large enough) also
 *		for dma allocations.  This flag is managed by the dma ops
 *		instance from ->dma_supported.
 *
 * At the lowest level, every device in a Linux system is represented by an
 * instance of struct device. The device structure contains the information
 * that the device model core needs to model the system. Most subsystems,
 * however, track additional information about the devices they host. As a
 * result, it is rare for devices to be represented by bare device structures;
 * instead, that structure, like kobject structures, is usually embedded within
 * a higher-level representation of the device.
 */
struct device {
	struct kobject kobj;
	struct device		*parent; // 指向父设备

	struct device_private	*p; // 执行设备私有数据

	const char		*init_name; /* initial name of the device */
	const struct device_type *type; // 指向所属设备类型的指针, 其中包含该类型设备公共的方法和成员

	const struct bus_type	*bus;	/* type of bus device is on */ // 指向所属总线类型
	struct device_driver *driver;	/* which driver has allocated this // 指向所绑定的驱动
					   device */
	void		*platform_data;	/* Platform specific data, device
					   core doesn't touch it */ // 指向平台专有数据
	void		*driver_data;	/* Driver data, set and get with
					   dev_set_drvdata/dev_get_drvdata */
	struct mutex		mutex;	/* mutex to synchronize calls to
					 * its driver.
					 */

	struct dev_links_info	links;
	struct dev_pm_info	power; // 设备的电源管理信息
	struct dev_pm_domain	*pm_domain;

#ifdef CONFIG_ENERGY_MODEL
	struct em_perf_domain	*em_pd;
#endif

#ifdef CONFIG_PINCTRL
	struct dev_pin_info	*pins;
#endif
	struct dev_msi_info	msi;
#ifdef CONFIG_DMA_OPS
	const struct dma_map_ops *dma_ops;
#endif
	u64		*dma_mask;	/* dma mask (if dma'able device) */ // dma掩码指针
	u64		coherent_dma_mask;/* Like dma_mask, but for
					     alloc_coherent mappings as
					     not all hardware supports
					     64 bit addresses for consistent
					     allocations such descriptors. */ // 同dma_mask, 但用于coherent DMA映射
	u64		bus_dma_limit;	/* upstream dma constraint */
	const struct bus_dma_region *dma_range_map;

	struct device_dma_parameters *dma_parms; // 设备DMA参数

	struct list_head	dma_pools;	/* dma pools (if dma'ble) */ // 这个设备为进行DMA而创建的consistent memory block链表的表头

#ifdef CONFIG_DMA_DECLARE_COHERENT
	struct dma_coherent_mem	*dma_mem; /* internal for coherent mem
					     override */ // 用于相干内存覆写
#endif
#ifdef CONFIG_DMA_CMA
	struct cma *cma_area;		/* contiguous memory area for dma
					   allocations */
#endif
#ifdef CONFIG_SWIOTLB
	struct io_tlb_mem *dma_io_tlb_mem;
#endif
#ifdef CONFIG_SWIOTLB_DYNAMIC
	struct list_head dma_io_tlb_pools;
	spinlock_t dma_io_tlb_lock;
	bool dma_uses_io_tlb;
#endif
	/* arch specific additions */
	struct dev_archdata	archdata; // 架构特定数据

	struct device_node	*of_node; /* associated device tree node */
	struct fwnode_handle	*fwnode; /* firmware device node */

#ifdef CONFIG_NUMA
	int		numa_node;	/* NUMA node this device is close to */
#endif
	dev_t			devt;	/* dev_t, creates the sysfs "dev" */ // 该设备的设备编号
	u32			id;	/* device instance */

	spinlock_t		devres_lock; // 用于保护设备资源链表的自旋锁
	struct list_head	devres_head; // 本设备的设备资源链表的表头. 为简化程序开发, 便于驱动卸载或探测失败路径处理, 将设备申请过的资源保存在此链表中, 以便最后统一释放

	const struct class	*class; // 指向所属类
	const struct attribute_group **groups;	/* optional groups */ // 这个设备独有的属性组

	void	(*release)(struct device *dev);
	struct iommu_group	*iommu_group;
	struct dev_iommu	*iommu;

	struct device_physical_location *physical_location;

	enum device_removable	removable;

	bool			offline_disabled:1;
	bool			offline:1;
	bool			of_node_reused:1;
	bool			state_synced:1;
	bool			can_match:1;
#if defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_DEVICE) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU_ALL)
	bool			dma_coherent:1;
#endif
#ifdef CONFIG_DMA_OPS_BYPASS
	bool			dma_ops_bypass : 1;
#endif
};

// https://elixir.bootlin.com/linux/v6.6.13/source/drivers/base/base.h#L108
/**
 * struct device_private - structure to hold the private to the driver core portions of the device structure.
 *
 * @klist_children - klist containing all children of this device
 * @knode_parent - node in sibling list
 * @knode_driver - node in driver list
 * @knode_bus - node in bus list
 * @knode_class - node in class list
 * @deferred_probe - entry in deferred_probe_list which is used to retry the
 *	binding of drivers which were unable to get all the resources needed by
 *	the device; typically because it depends on another driver getting
 *	probed first.
 * @async_driver - pointer to device driver awaiting probe via async_probe
 * @device - pointer back to the struct device that this structure is
 * associated with.
 * @dead - This device is currently either in the process of or has been
 *	removed from the system. Any asynchronous events scheduled for this
 *	device should exit without taking any action.
 *
 * Nothing outside of the driver core should ever touch these fields.
 */
struct device_private {
	struct klist klist_children; // 本设备的孩子链表的表头
	struct klist_node knode_parent; // 链接到所属父设备的孩子链表的连接件
	struct klist_node knode_driver; // 链接到所绑定驱动的设备链表的连接件
	struct klist_node knode_bus; // 链接到所属父总线类型的设备链表的连接件
	struct klist_node knode_class;
	struct list_head deferred_probe;
	struct device_driver *async_driver;
	char *deferred_probe_reason;
	struct device *device; // 指向相关联的device
	u8 dead:1;
};

// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/device.h#L89
/*
 * The type of device, "struct device" is embedded in. A class
 * or bus can contain devices of different types
 * like "partitions" and "disks", "mouse" and "event".
 * This identifies the device type and carries type-specific
 * information, equivalent to the kobj_type of a kobject.
 * If "name" is specified, the uevent will contain it in
 * the DEVTYPE variable.
 */
struct device_type {
	const char *name;
	const struct attribute_group **groups; // 这一类型的属性公有的属性组
	int (*uevent)(const struct device *dev, struct kobj_uevent_env *env); //添加设备类型特定的环境变量
	char *(*devnode)(const struct device *dev, umode_t *mode,
			 kuid_t *uid, kgid_t *gid); // 在为这种设备创建设备节点时, 提供名字线索
	void (*release)(struct device *dev); // 用于释放属于该类型的设备的回调函数

	const struct dev_pm_ops *pm; // 指向设备类型的电源管理操作表
};

// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/device/driver.h#L51
/**
 * struct device_driver - The basic device driver structure
 * @name:	Name of the device driver.
 * @bus:	The bus which the device of this driver belongs to.
 * @owner:	The module owner.
 * @mod_name:	Used for built-in modules.
 * @suppress_bind_attrs: Disables bind/unbind via sysfs.
 * @probe_type:	Type of the probe (synchronous or asynchronous) to use.
 * @of_match_table: The open firmware table.
 * @acpi_match_table: The ACPI match table.
 * @probe:	Called to query the existence of a specific device,
 *		whether this driver can work with it, and bind the driver
 *		to a specific device.
 * @sync_state:	Called to sync device state to software state after all the
 *		state tracking consumers linked to this device (present at
 *		the time of late_initcall) have successfully bound to a
 *		driver. If the device has no consumers, this function will
 *		be called at late_initcall_sync level. If the device has
 *		consumers that are never bound to a driver, this function
 *		will never get called until they do.
 * @remove:	Called when the device is removed from the system to
 *		unbind a device from this driver.
 * @shutdown:	Called at shut-down time to quiesce the device.
 * @suspend:	Called to put the device to sleep mode. Usually to a
 *		low power state.
 * @resume:	Called to bring a device from sleep mode.
 * @groups:	Default attributes that get created by the driver core
 *		automatically.
 * @dev_groups:	Additional attributes attached to device instance once
 *		it is bound to the driver.
 * @pm:		Power management operations of the device which matched
 *		this driver.
 * @coredump:	Called when sysfs entry is written to. The device driver
 *		is expected to call the dev_coredump API resulting in a
 *		uevent.
 * @p:		Driver core's private data, no one other than the driver
 *		core can touch this.
 *
 * The device driver-model tracks all of the drivers known to the system.
 * The main reason for this tracking is to enable the driver core to match
 * up drivers with new devices. Once drivers are known objects within the
 * system, however, a number of other things become possible. Device drivers
 * can export information and configuration variables that are independent
 * of any specific device.
 */
struct device_driver {
	const char		*name; // 驱动名称
	const struct bus_type	*bus; // 指向所属总线类型

	struct module		*owner; // 指向实现了这个设备驱动的模块
	const char		*mod_name;	/* used for built-in modules */ // 编译进内核的模块需要根据模块名找对对应的module_kobject, 不能使用上面的`struct module`

	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */ //1: 禁止通过sysfs进行绑定/解绑
	enum probe_type probe_type;

	const struct of_device_id	*of_match_table;
	const struct acpi_device_id	*acpi_match_table;

	int (*probe) (struct device *dev); // 设备可用绑定到驱动, 则用来对设备进行初始化
	void (*sync_state)(struct device *dev);
	int (*remove) (struct device *dev); // 将设备从驱动解绑时被调用的回调函数
	void (*shutdown) (struct device *dev); // 设备被断电时调用的回调函数
	int (*suspend) (struct device *dev, pm_message_t state); // 设备进入到节能状态时调用的方法
	int (*resume) (struct device *dev); // 设备恢复到正常状态时调用的方法
	const struct attribute_group **groups; // 驱动独有的属性组
	const struct attribute_group **dev_groups;

	const struct dev_pm_ops *pm; // 指向设备的电源管理操作表
	void (*coredump) (struct device *dev);

	struct driver_private *p; // 指向驱动私有数据
};

// https://elixir.bootlin.com/linux/v6.6.14/source/drivers/base/base.h#L78
struct driver_private {
	struct kobject kobj;
	struct klist klist_devices; // 这个驱动支持的设备链表的表头
	struct klist_node knode_bus; // 链接到所属总线类型的驱动链表的连接件
	struct module_kobject *mkobj; // 指向这个驱动对应的module_kobject
	struct device_driver *driver; // 指向相关联的device_driver
};
#define to_driver(obj) container_of(obj, struct driver_private, kobj)

// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/device/class.h#L52
/**
 * struct class - device classes
 * @name:	Name of the class.
 * @class_groups: Default attributes of this class.
 * @dev_groups:	Default attributes of the devices that belong to the class.
 * @dev_uevent:	Called when a device is added, removed from this class, or a
 *		few other things that generate uevents to add the environment
 *		variables.
 * @devnode:	Callback to provide the devtmpfs.
 * @class_release: Called to release this class.
 * @dev_release: Called to release the device.
 * @shutdown_pre: Called at shut-down time before driver shutdown.
 * @ns_type:	Callbacks so sysfs can detemine namespaces.
 * @namespace:	Namespace of the device belongs to this class.
 * @get_ownership: Allows class to specify uid/gid of the sysfs directories
 *		for the devices belonging to the class. Usually tied to
 *		device's namespace.
 * @pm:		The default device power management operations of this class.
 * @p:		The private data of the driver core, no one other than the
 *		driver core can touch this.
 *
 * A class is a higher-level view of a device that abstracts out low-level
 * implementation details. Drivers may see a SCSI disk or an ATA disk, but,
 * at the class level, they are all simply disks. Classes allow user space
 * to work with devices based on what they do, rather than how they are
 * connected or how they work.
 */
struct class {
	const char		*name;

	const struct attribute_group	**class_groups;
	const struct attribute_group	**dev_groups;

	int (*dev_uevent)(const struct device *dev, struct kobj_uevent_env *env); // 添加类特定的环境变量
	char *(*devnode)(const struct device *dev, umode_t *mode); // 可以在为属于该类的设备的创建设备节点时, 提供名字线索

	void (*class_release)(const struct class *class); // 释放该类的回调函数
	void (*dev_release)(struct device *dev); // 释放属于该类的设备的回调函数

	int (*shutdown_pre)(struct device *dev);

	const struct kobj_ns_type_operations *ns_type;
	const void *(*namespace)(const struct device *dev);

	void (*get_ownership)(const struct device *dev, kuid_t *uid, kgid_t *gid);

	const struct dev_pm_ops *pm; // 指向类的电源管理操作表
};

// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/device/class.h#L219
struct class_interface {
	struct list_head	node; // 链入所属类的接口链表的连接件
	const struct class	*class; // 指向所属类

	int (*add_dev)		(struct device *dev); // 在接口被注册到类, 或者设备被添加到接口所在的类是, 调用该函数向接口添加设备
	void (*remove_dev)	(struct device *dev); // 在接口被销毁, 或者从设备所在类删除该设备时, 调用该函数从接口删除设备
};
```
> 几乎所有的类都显示在/sys/class中, 但/sys/block因为历史原因除外.

bus_register()函数用来注册一个bus总线子系统，可能会失败，必须检查返回值；注册成功后，可以在/sys/bus/目录下看到该总线；之后就可以向总线中添加设备了.

bus_type的match方法：当总线上添加新设备或新驱动程序时，会多次调用match方法，将device和device_driver进行匹配，如果匹配成功，说明指定的驱动程序能够处理指定的设备，match方法返回非零值

device_register()函数用来注册设备，可能会失败，必须检查返回值；注册成功后，可以在/sys/devices目录下看到该设备；以后添加到改总线上的任何设备都可以在/sys/devices/ldd目录下显示.

> device_register->device_add->bus_probe_device

driver_register()函数用来注册驱动. driver_register(&(globalfifo_driver.driver))主要的工作包括：
1. 确认驱动依附的总线platform_bus已经被注册并初始化（必要条件）。
1. 对probe、remove、shutdown等回调函数初始化进行判断，保证总线和驱动上相应的函数只能存在一个。
1. driver_find()查找总线上是否已存在当前驱动的同名驱动。
1. bus_add_driver(&(globalfifo_driver.driver))，将驱动注册到总线上
1. 发起KOBJ_ADD类型uevent，指示驱动已经添加完成

> driver_register->bus_add_driver->driver_attach(尝试让驱动去绑定所有可能的设备)-> `__driver_attach`

class_register: 类注册

<table>
<thead>
<tr>
<th>类型</th>
<th>属性数据结构</th>
<th>设置属性方法</th>
</tr>
</thead>
<tbody><tr>
<td>bus</td>
<td>bus_attribute(include/linux/device.h)</td>
<td>BUS_ATTR()</td>
</tr>
<tr>
<td>device</td>
<td>device_attribute(include/linux/device.h)</td>
<td>DEVICE_ATTR()</td>
</tr>
<tr>
<td>driver</td>
<td>driver_attribute( include/linux/device.h)</td>
<td>手动设置driver_attribute</td>
</tr>
</tbody></table>

BUS_ATTR()宏，定义一个以bus_attr_开头的总线属性，生成总线属性文件需要使用bus_create_file()函数来完成.

驱动本质上只做了两件事：向上提供接口，向下控制硬件:

开发一个驱动的流程：
1. 确定驱动架构：根据硬件连接方式结合分层/分离思想设计驱动的基本结构
1. 确定驱动对象：内核中的一个驱动/设备就是一个对象，1.定义，2.初始化，3.注册，4.注销
1. 向上提供接口：根据业务需要确定提供cdev/proc/sysfs哪种接口
1. 向下控制硬件：1.查看原理图确定引脚和控制逻辑，2.查看芯片手册确定寄存器配置方式，3.进行内存映射，4.实现控制逻辑

## 引用计数
[kref](https://elixir.bootlin.com/linux/v6.6.13/source/include/linux/kref.h#L19)

## 地址
物理地址: CPU地址总线使用的地址, 由硬件电路控制其具体含义. 物理地址中很大一部分是留给内存的, 但也常被映射到其他存储器上（如显存、BIOS等）. 在程序指令中的虚拟地址经过段映射和页面映射后, 就生成了物理地址.

总线地址: 总线使用的地址

外设使用的是总线地址, CPU使用的是物理地址.

物理地址与总线地址之间的关系由系统的设计决定的. 在x86平台上物理地址就是总线地址, 这是因为它们共享相同的地址空间.

虚拟地址: 现代操作系统普遍采用虚拟内存管理（Virtual Memory Management）机制, 这需要MMU（Memory Management Unit）的支持. MMU通常是CPU的一部分, 如果处理器没有MMU或者有MMU但没有启用, CPU执行单元发出的内存地址将直接传到芯片引脚上, 被内存芯片（物理内存）接收，这称为物理地址（Physical Address）. 如果处理器启用了MMU, CPU执行单元发出的内存地址将被MMU截获, 从CPU到MMU的地址称为虚拟地址（Virtual Address）, 而MMU将这个地址翻译成另一个地址发到CPU芯片的外部地址引脚上, 也就是将虚拟地址映射成物理地址.

## 和cpu交互的数据传输方式
1. 无条件传送方式

    用此方式的数据源设备一定是随时准备好了数据, CPU 随时取随时拿都没问题, 如寄存器、内存就是类似这样的设备, CPU 取数据时不用提前打招呼
2. 查询传送方式
    
    也称为程序 I/O或PIO (Programming Input/Output Model ）, 是指传输之前, 由程序先去检测设备的状态. 数据源设备在一定的条件下才能传送数据, 这类设备通常是低速设备, 比CPU 慢很多. CPU 需要数据时, 先检查该设备的状态, 如果状态为“准备好了可以发送飞, CPU 再去获取数据. 硬盘有 status 寄存器, 里面保存了工作状态, 所以对硬盘可以用此方式来获取数据.
3. 中断传送方式

    也称为中断驱动I/O. “查询传送方式”有这样的缺陷, 由于 CPU需要不断查询设备状态, 所以意味着只有最后一刻的查询才是有意义的, 之前的查询都是发生在数据尚未准备好的时间段里，所以说效率不高，仅对于不要求速度的系统可以采用. 可以改进的地方是如果数据源设备将数据准备好后再通知 CPU 来取，这样效率就高了. 通知 CPU 可以采用中断的方式，当数据源设备准备好数据后，它通过发中断来通知 CPU 来拿数据，这样避免了 CPU花在查询上的时间，效率较高.

    bios设置的中断号从0x08开始, 但由于中断号 0x00--0x1F 属于 Intel 公司专门保留给 CPU 使用的, 为解决这个冲突Linux 操作系统不会直接使用这些 BIOS 默认设置好的中断号。在上电启动时， Linux 操作系统会在内核初始化期间又重新对 8259A 进行了设置，把所有系统硬件中断请求号全部都映射到了 0X20 及以上中断号上.
4. 直接存储器存取方式(DMA)

	> 基于DMA的硬件使用的是总线地址, 而不是物理地址. 总线地址是从设备角度上看到的内存地址, 物理地址从cpu mmu角度看的内存地址. 在pc上对ISA和PCI, 总线地址就是物理地址, 但不是每个平台都这样, 比如PRep(PowerPC Reference Platform), 此时需要virt_to_bus/bus_to_virt的转换.

    在中断传送方式中，虽然极大地提高了 CPU 的利用率，但通过中断方式来通知 CPU, CPU 就要通过压找来保护现场，还要执行传输指令，最后还要恢复现场. 其实还可一点都不要浪费 CPU 资源，不让 CPU参与传输，完全由数据源设备和内存直接传输. CPU 直接到内存中拿数据就好了. 这就是此方式中“直接”的意思. 不过 DMA 是由硬件实现的, 所以需要 DMA 控制器才行.

    DMA的发起者可以是处理器, 也可以是I/O设备. 以处理器发起DMA为例, 设备驱动首先在内存中分配一块DMA缓冲区, 随后发起DMA请求, 设备收到请求后通过DMA机制将数据传输至DMA缓冲区. DMA操作完成后, 设备触发中断通知处理器对DMA缓冲区中的数据进行处理.

    在一些总线中, DMA的发起还需要DMA控制器的参与, 此时DMA控制器相当于收发双方（处理器和设备）的第三方, 因此这种机制被称为第三方DMA（third-party DMA）, 也被称为标准DMA.

    在标准DMA场景下由处理器发起DMA的具体步骤:
	1. 处理器向DMA控制器发送DMA缓冲区的位置和长度, 以及数据传输的方向, 随后放弃对总线的控制
	2. DMA控制器获得总线控制权, 可以直接与内存和设备进行通信
	3. DMA控制器根据从处理器获得的指令, 将设备的数据拷贝至内存, 在这期间处理器可以执行其他任务
	4. DMA控制器完成DMA后向处理器发送中断, 通知处理器DMA已经完成. 此时, 处理器会重新得到总线的控制权

	另一种情况是由设备发起DMA, 设备首先通知DMA控制器, 并由DMA控制器向总线控制器申请占用总线, 此后的流程与处理器发起DMA的情况相似.

	不少总线允许设备直接获取总线控制权并进行DMA操作, 而无须借助DMA控制器, 这种机制被称为第一方DMA（first-party DMA）, 也称为总线控制（bus mastering）. 此时可以理解为将DMA控制器的角色和设备的角色合二为一, 其执行过程和上述2种情况类似. 此时, 如果多个设备希望同时进行DMA, 总线控制器需要进行仲裁, 决定优先次序, 同一时间只允许一个设备进行DMA.

	设备不一定能在所有的内存地址上指向DMA操作, 此时需要dma_set_mask(), 其本质是修改device结构体中的dma_mask成员.

	dma_alloc_coherent()申请的DMA缓冲区可保证该缓冲区的cache一致性.

	驱动申请的DMA缓冲区时, 用一致性DMA缓冲区最方便. 但有时是内核上层通过`kmalloc, __get_free_pages`申请的(网卡驱动中的网络报文, 块设备驱动中要写入设备的数据), 此时应使用流式DMA映射(使用dma_map_single()). 其操作本质上多是进行cache的使能无效或清除操作, 以解决cache一致性问题.

	linux内核推荐使用dmaengine的驱动框架来编写DMA控制器的驱动, 同时外设的驱动使用标准的dmaengine API进行DMA的准备, 发起和完成时的回调工作.

5. I/O 处理机传送方式

    DMA中数据输入之后或输出之前还是有一部分工作要由 CPU 来完成的.

    I/O处理机是专门用于处理 IO, 并且它其实是一种处理器, 只不过用的是另一套擅长 IO 的指令系统，随时可以处理数据, 彻底解放CPU, 让CPU 甚至可以不知道有传输这回事.

## cpu访问外设的方法
ref:
- [由于与IO地址空间相关的一些限制和不良影响, IO地址空间很快就失去了软件和硬件供应商的青睐](https://blog.csdn.net/u013253075/article/details/119491100)
- [x86 I/O端口分配表](https://bochs.sourceforge.io/techspec/PORTS.LST)

CPU通常以读写设备寄存器的方式与设备进行通信. 一个设备通常有多个寄存器, 可以在内存地址空间或I/O地址空间中被访问. 设备寄存器可以分为如下几种类型:
1. 控制寄存器:用于接收来自驱动程序的命令
1. 状态寄存器:用于反馈当前设备的工作状态
1. 输入/输出寄存器(即数据寄存器):用于驱动和设备之间的数据交互

设备寄存器也称为`I/O端口`, 而IO端口有两种编址方式：独立编址和统一编址. 而具体采用哪一种则取决于CPU的体系结构.

- 内存映射(memory-mapped, 统一编址, mmio): 此类cpu通常只实现一个物理地址空间, 会将外设 I/O端口的物理地址映射到CPU的单一物理地址空间中, 成为存储空间的一部分，通过直接读写内存地址的方式来访问外设.

	RISC指令系统的CPU（如ARM、PowerPC等）通常支持内存映射模式.

	优点: 访问速度较快, 编程比较简单
	缺点: 需要占用CPU的内存地址空间
	适用: 需要频繁访问外设的应用

	IO内存：当寄存器或内存位于内存空间时的称呼, 可通过`cat /proc/iomem`查看.

	IO内存使用流程: request_mem_region(申请IO内存) -> ioremap(映射地址)-> readb, readl, writeb, writel等(使用) ->iounmap(取消地址映射)->release_mem_region(释放IO内存)
	- request_mem_region+ioremap: 在驱动模块加载或open()中进行
	- 使用: 在设备驱动初始化, read(), write(), ioctl等中进行
	- iounmap+release_mem_region: 在驱动模块卸载或release()中进行


- io映射(i/o-mapped, 独立编址): 为外设专门实现了一个单独地地址空间, 称为`I/O地址空间或I/O端口空间`(32位X86有64K的IO空间, `cat /proc/ioport`). IO地址与内存地址分开独立编址, I/O端口地址不占用存储空间的地址范围. 此时在系统中就存在了另一种与存储地址无关的IO地址. 此时, 需要使用专用的CPU指令来访问某种特定外设, 比如x86的in/out等指令.

	接口的作用是连接处理器和外部设备, 处理速度不匹配, 格式转换(模拟转数字, cpu只能处理数字信号)等情况.

	> IA32 体系系统中，因为用于存储端口号的寄存器是 16 位的，所以最大有 65536 个端口，即 0～ 65535.

	x86体系的CPU均支持io映射模式.

	优点: 不占用CPU的内存地址
	缺点: 编程比较复杂, 也就是CPU访问外设时相对复杂
	适用: 需要同时访问多个外设的应用

	IO端口：当寄存器或内存位于IO空间时的称呼, 可通过`cat /proc/ioports`查看.

	**不过x86平台也支持内存映射（MMIO）, 该技术是PCI规范的一部分, 如果硬件支持MMIO, 就可将其IO设备端口映射到内存空间. 为了兼容一些之前开发的软件, PCIe仍然支持IO地址空间, 只是建议在新开发的软件中采用MMIO**.

	IO端口使用流程: request_region(申请)-> inb/outb, ...(使用) ->release_region(释放)

> 控制显卡用的是内存映射+io映射.

对于Linux内核而言, 它适用于不同的CPU, 所以它必须都要考虑这两种方式, 于是它采用一种新的方法, 将基于I/O映射方式(对应于独立编址)的或内存映射方式(对应于统一编址)的I/O端口通称为`I/O区域(I/O region, include/linux/ioport.h)`, 不论采用哪种方式, 都要先申请IO区域: request_resource(), 结束时释放它: release_resource().

## 中断
单核cpu使用pic模型, 只有一张IDT; 多核使用APIC.

APIC由2部分组成:
- LAPIC

	每个core一个LAPIC, 且有一个唯一id, 该id用作区分多核cpu不同核心的标志

	每个LAPIC有自己的定时器, 能处理自己的内部中断.

	x86 LAPIC的属性在一些列的寄存器中, 它们通过mmio映射, 因此设置LAPIC定时器的中断时间就可通过mmio实现.

	LVT(Local Vector Table, 本地向量表)是联系中断信号和IDT的纽带, 是程序调度实现的关键硬件基础.

	LVT的Vector保存IDT的中断号

	获取LAPIC路径: RSDP->XSDT->MADT->LAPIC
- IO APIC

	只有一个io apic, 也有表, 可设置哪个中断由哪个cpu处理.

	处理硬盘, 网卡等外部设备产生的中断

ps: bsp(随机指定by IPI) cpu core通过IPI(Inter-Processor Interrupt, 是硬件中断)中断来唤醒其他cpu core.

## IOMMU
在大多数情况下, 处理器上执行的操作系统代码在进行内存访问时使用的是虚拟地址, 并通过MMU翻译成物理地址. 设备进行DMA时访问的内存地址是总线地址（bus address）, 它不同于虚拟地址和物理地址. 当操作系统闷DMA控制器注册DMA内存缓冲区时, 需要填写的是总线地址. 设备和内存之间的输入输出内存管理单元（Input-Output Memory Management Unit,  IOMMU）会负责将总线地址翻译成物理地址.


IOMMU作用:
1. 由于总线或者一些设备本身的限制, 设备可以访问到的内存范围远远小于物理内存的总大小. 对于这种设备, os在分配DMA内存缓冲区时, 内存地址受限. 它可以有效地解决这—问题

	某些设备只能使用24位长度的地址, 因此只能访问到16MB范围内的物理内存
1. 由于DMA在进行过程中会向连续的地址写入数据. 因此操作系统需要分配连续的物理内存作为DMA缓冲区. IOMMU的存在免除了DMA缓冲区各内存页必须物理连续的限制, 同时IOMMU还限定了设备可以访问到的物理内存范围
1. 为os提供内存访问保护机制, 用于防止恶意设备以及驱动对物理内存的非法访问, 尤其是虚拟化环境下.

## 设备识别
计算机系统需要提供某种的机制, 帮助os识别识别计算机已经连接的设备. 常见的设备识别机制有设备树和ACPI.

ACPI和设备树均使用树状结构对设备信息和层次关系进行描述. 类似于设备树的compatible属性, ACPI则通过命名空间（namespace）和特定的ACPI/PNPID来完成设备和驱动代码的匹配. ACPI的命名空间用于对平台上设备的拓扑关系进行建模, 设备以命名空间内的对象（object）的形式存在, 类似于设备树的节点（node）.

设备树描述的仅仅是一种of的关系, 即当前计算机系统拥有哪些设备. 而ACPI除了描述当前计算机的设备信息之外, 还能通过AML代码直接控制设备的行为, 这是设备树所不具备的.

### 设备树
> 设备树源于OpenFirmware(OF), 在linux 3.x开始应用, 比如`arch/arm/boot/dts`

ARM设备种类数不胜数, 配置信息各不相同, 把硬件配置硬编码到系统代码中的方式非常不灵活, 会极大增加os的维护成本, 所以ARM架构普遍使用设备树来编码硬件信息.

设备树（device tree）是描述计算机硬件信息的数据结构, 或理解为os可理解的硬件描述语言.

设备树中包含了CPU的名称、内存、总线、设备等硬件信息, bootloader会将它传递给kernel.

设备树并不对所有设备信息都进行描述, 通常只有不能被**动态探测**的设备才会出现在设备树中.

设备树源码（DeviceTreeSource, DTS）文件以文本形式描述了计算机系统设备的一个树状结构. 每种设备都存在一个节点描述, 并且挂载在根节点`/`之下, 这种树状格式被称为设备树块（Device Tree Blob, DTB）或者展开的设备树（Flattened Device Tree, FDT）.

设备树源码文件的扩展名为.dts. 在Linux内核项目源码里, 可以在平台相关代码目录中找到大量设备树源码文件. `.dts`源码文件可以编译生成二进制文件, 其扩展名为`.dtb`. 在启动过程中, os内核读取dtb二进制文件以获取设备信息.

> DTC(Device Tree Compiler)将.dts编译为.dtb, 其代码在`scripts/dtc`. 使能设备树后, 编译kernel时, 也会编译DTC; 单独安装时, 用`device-tree-compiler`. DTC也支持反编译. `make dtbs`可单独编译设备树文件.

> .dtb通常和zImage绑定一起做成镜像文件

> Documentation/devicetree/bindings下有很多txt文件用来描述设备的硬件细节, 便于构建设备树.

由于一个Soc可能对应多个设备, 此时把Soc公用的部分或者多个设备共同的部分提炼为`.dtsi`, 其他设备对应的`.dts`导入该`.dtsi`.

Linux内核根据设备树注册各节点所表示的设备, 根据compatible属性后接的字符串内容识别设备并匹配对应的驱动代码, 进而实现对设备的管理.

PCIe Host的设备树内容:
```
pcie: pcie@fd0e0000 {
	compatible = "xlnx,nwl-pcie-2.11";
	status = "disabled";
	#address-cells = <3>;
	#size-cells = <2>;
	#interrupt-cells = <1>;
	msi-controller;
	device_type = "pci";
    
	interrupt-parent = <&gic>;
	interrupts = <0 118 4>,
		     <0 117 4>,
		     <0 116 4>,
		     <0 115 4>,	/* MSI_1 [63...32] */
		     <0 114 4>;	/* MSI_0 [31...0] */
	interrupt-names = "misc", "dummy", "intx", "msi1", "msi0";
	msi-parent = <&pcie>;
    
	reg = <0x0 0xfd0e0000 0x0 0x1000>,
	      <0x0 0xfd480000 0x0 0x1000>,
	      <0x80 0x00000000 0x0 0x1000000>;
	reg-names = "breg", "pcireg", "cfg";
	ranges = <0x02000000 0x00000000 0xe0000000 0x00000000 0xe0000000 0x00000000 0x10000000	/* non-prefetchable memory */
		  0x43000000 0x00000006 0x00000000 0x00000006 0x00000000 0x00000002 0x00000000>;/* prefetchable memory */
	bus-range = <0x00 0xff>;
    
	interrupt-map-mask = <0x0 0x0 0x0 0x7>;
	interrupt-map =     <0x0 0x0 0x0 0x1 &pcie_intc 0x1>,
			    <0x0 0x0 0x0 0x2 &pcie_intc 0x2>,
			    <0x0 0x0 0x0 0x3 &pcie_intc 0x3>,
			    <0x0 0x0 0x0 0x4 &pcie_intc 0x4>;
    
	pcie_intc: legacy-interrupt-controller {
		interrupt-controller;
		#address-cells = <0>;
		#interrupt-cells = <1>;
	};
};
```
字段说明:
- compatible：用于匹配PCIe Host驱动；
- msi-controller：表示是一个MSI（Message Signaled Interrupt）控制器节点，这里需要注意的是，有的SoC中断控制器使用的是GICv2版本，而GICv2并不支持MSI，所以会导致该功能的缺失；
- device-type：必须是"pci"；
- interrupts：包含NWL PCIe控制器的中断号；
- interrupts-name：msi1, msi0用于MSI中断，intx用于旧式中断，与interrupts中的中断号对应；
- reg：包含用于访问PCIe控制器操作的寄存器物理地址和大小；
- reg-name：分别表示Bridge registers，PCIe Controller registers， Configuration space region，与reg中的值对应；
- ranges：PCIe地址空间转换到CPU的地址空间中的范围；
- bus-range：PCIe总线的起始范围；
- interrupt-map-mask和interrupt-map：标准PCI属性，用于定义PCI接口到中断号的映射；
- legacy-interrupt-controller：旧式的中断控制器

probe流程
1. 系统会根据dtb文件创建对应的platform_device并进行注册；
1. 当驱动与设备通过compatible字段匹配上后，会调用probe函数，也就是nwl_pcie_probe

### ACPI
这是x86架构计算机上的标准设备识别方案. ACPI可以理解为设备和操作系统之间的一层抽象, 该层抽象统一地向os汇报硬件设备的情况, 同时提供管理设备的接口和方法.

ACPI分为两部分:
1. ACPI表: ACPI提供了大量的表结构信息, 这些信息描述了当前计算机系统的各种状态, 包括多处理器信息、NUMA的配置信息等
1. ACPI运行时: ACPI提供了ACPI机器语言（ACPI Machine Language, 简称AML）. os可以解释执行AML代码和底层的固件进行交互,进而完成对设备的识别和相应配置功能

ACPI还负责计算机整体系统、各个处理器以及不同设备的电源和温度的管理.

考虑到ACPI在服务器领域的广泛使用, ARMv8服务器也支持了ACPI.

> windows不支持设备树. 基于兼容性的考量. Linux系统同时支持设备树和ACPI两种设备识别机制.

## 中断控制器
ARM架构在CPU与设备之间引入了中断控制器（interrupt controller）来管理中断.

ARMv8的中断控制器称为通用中断控制器（Generic Interrupt Controller, GIC）, 负责对设备的中断信号进行处理, 并将其发送至CPU.

GIC的中断类型可以分为三类:
1. 软件生成中断（Software Generated Interrupt, SGI）: 由软件通过写GICD_SGIR系统寄存器触发, 常用于发送核间中断
1. 私有设备中断（Private Peripheral Interrupt, PPI）:由每个处理器核上私有的设备触发, 如通用定时器（Generic Time）
1. 共享设备中断（Shared Peripheral Interrupt, SPI）:由所有CPU核心共同连接的设备触发, 可以发送给任意核心

	通过irq_set_affinity设定中断触发的cpu core, arm linux 默认情况下, 中断都是在cpu0上产生的.

根据中断事件的重要程度和紧迫程度, GIC允许os为中断配置不同的中断优先级. 通常优先级的数字越小表示优先级越高. 在中断仲裁时, 高优先级的中断会先于低优先级的中断被发送给CPU处理. 当CPU在响应低优先级中断时, 如果此时有高优先级中断到来, 那么高优先级中断会抢占低优先级中断, 被CPU优先响应.

GIC中断类型以及对应的INTID(Interrupt ID)之间的关系:
- SGI, 0～15 : 由核间通信等产生的中断, 需交由指定核进行处理, 每个核各自维护中断队列
- PPI, 16～31, 1056～1119: 由设备产生的中断, 需交由指定核进行处理, 每个核各自维护中断队列
- SPI, 32～1019, 4096 ~ 5119 : 由设备产生的中断, 可交由指定核/非指定的任意一个或多个核进行处理, 所有核维护一个共享的中断队列

GIC中的每个中断都存在如下四种状态:
1. Inactive:中断处于无效状态, 此时没有中断到来
1. Pending:中断处于有效状态, 此时中断已经发生, 但CPU没有响应中断
1. Active:CPU处于响应并处理中断的过程中
1. Active & Pending: 在CPU响应并处理中断的过程中, 同时又有相同中断号的中断发生

GIC的中断响应过程通常包括如下四个步骡:
1. Generate:某个中断源产生中断, 传递给GIC, 中断从Inactive变为Pending
2. Deliver: GIC将中断转发给CPU,中断仍处于Pending状态
3. Active: CPU调用中断处理函数响应并处理该中断, 中断处于Active状态
4. Deactivate: CPU处理完中断, 通知GIC中断处理完毕, GIC将中断状态更新为Inactive

其中的第四步又称为中断完成（End Of Interrupt, EOI）. EOI在中断处理中是非常重要的步骤, 只有在CPU确认了EOI后, 对应的中断（此时的状态为Inactive）才能重新被响应.

为了提高中断响应的实时性, GIC将中断完成分解为两个阶段:
1. 优先级重置（priority drop）:将当前中断屏蔽的优先级进行重置, 从而能够响应较低???优先级的中断
1. 中断无效（interrupt deactivation）:将对应中断的状态设置为Inactive, 从而可以重新响应该优先级的中断

为了及时响应中断, Linux操作系统将中断处理过程分为两个阶段:
1. 上半部（top half）:完成一些必要但轻量级的操作, 比如向中断控制器确认中断
1. 下半部（bottom half）:完成剩余的、复杂且时延要求相对较低的操作

当出现中断时, 会首先执行上半部, 在执行上半部期间关闭中断. 上半部执行完成后, 立即向中断控制器声明该中断事件已处理完毕, 从而允许CPU继续响应该中断. 而下半部的执行时间将由系统调度来确定. Linux提供了多种下半部的实现方式, 下半部属于具有较高优先级的内核任务.

设备驱动代码通过调用request_irq()来注册相应设备的硬中断处理函数. 硬中断处理函数又称为中断服务例程（Interrup Service Routine, ISR）. 硬中断处理函数实质上是Linux中断处理的上半部. 上半部在执行过程中, 会向系统注册新的处理任务, 将其作为下半部的执行函数. linux有多种注册下半部任务的方式, 常用的三种下半部机制是软中断, tasklet和工作队列.

软中断（softirq）是最基本的下半部处理方式, 可以看作一种用普通内核任务模拟硬中断处理函数的方法. 软中断和硬中断都用于处理来自I/O设备的中断请求, 只不过软中断不使用硬中断上下文, 而是拥有独立的上下文环境. 软中断不会中断当前CPU的任务, 而是等待调度器的主动调度.

软中断与硬中断的另一点区别是, 软中断的处理函数必须是可重人的. 硬中断之所以无须可重入是因为在硬中断响应过程中, 处理器已经关闭了中断, 从而屏蔽了新到来的硬中断. 而软中断的执行允许被硬中断抢占, 同时Linux还允许软中断在多个CPU核上并行执行以提升软中断的处理效率.

Linux内核会选择恰当的时机来处理软中断. 常见的处理时机是在硬中断得到处理后, 此时内核会随即处理软中断. linux限制了软中断连续处理的最长时间和最大次数, 分别对应于MAXSOFT工RQ_T工ME和MAX_SOFTIRQ_RESTART两个常量, 一旦超过这个限制就将软中断任务追加到名为ksoftirqd的内核线程. 在Linux内核中, 每个CPU上都运行着`ksoftirqd/<n>`内核线程（n是线程所在的处理器编号）. ksoftirqd线程的优先级被设置为最低, 因此不会和系统上的其他重要任务争夺资源, 从而避免发生用户任务饿死的情况.

Linux的软中断只能在代码编译时静态分配, 而无法根据需要在运行时进行动态创建. 为了解决这个问题, Linux基于软中断实现了tasklet机制, 其也运行在软中断上下文中.

Linux内核使用ne×t指针将需要处理的tasklet以链表的形式串联起来. 每个tasklet都有两种状态, 其中TASKLET_STATE_SCHED表示当前tasklet巳经在内核中被注册, 并处于可调度的状态, 而TASKLET_STATE_RUN则表示当前tasklet正在运行. count变量指明当前tasklet是否被禁用,大于0表示被禁用, 等于0表示启用. func指针指向将被执行的具体函数, date是func回调函数的参数.

Linux内核用tasklet_schedule()或tasklet_hi_shedule()来调度tasklet, 它们分别对应TASKLET_SOFTIRQ和HI_SOFTIRQ两个不同的优先级.

除了允许在运行时动态创建tasklet外, Linux不允许在多个CPU上并发执行tasklet(使用state), 因此避免了同步. Linux还保证了tasklet的执行的原子性, 使其不能被其他下半部机制所抢占, 因此无须考虑可重入问题, 开发起来也更直接简单.

同一时刻一个tasklet只能在一个cpu执行, 不同的tasklet可以在不同的cpu上执行. 软中断同一时刻可以在不同的cpu上并发执行, 因此软中断需要考虑重入问题.

通过对比软中断和tasklet, 可以发现这两种下半部机制在设计上有不同的侧重: 软中断侧重于中断的处理效率, 如在多核情况下软中断可并行执行; tasklet则更侧重于编程开发的友好性, tasklet使用者无须考虑同步加锁和代码可重入等问题.

因为tasklet本身依托于中断上下文, 执行期间不能睡眠, 外加设计上的不可抢占性, 导致tasklet可能引起难以预测的系统延迟, 严重的话甚至可能影响系统的整体实时性.

工作队列, 它允许睡眠, 降低了下半部的开发难度, 同时也关注效率.

工作队列（workqueue）把需要推迟执行的函数, 即中断下半部, 交由内核线程来执行. 工作队列的特点是借助线程上下文来执行下半部操作, 从而允许下半部的睡眠和重新调度, 解决了因为软中断和tasklet单个实例执行时间过长且无法睡眠而导致的系统实时性下降的问题.

和ksoftirqd类似, 在每个CPU上都运行着一个`kworker/<n>`工作线程. 内核工作线程会串行地执行加人队列中的所有work. 如果某个队列中没有work要做, 那么该内核工作线程就会变成idle态.

工作队列并非没有缺点:
1. 本身原因:
	1. 因为工作队列本身是串行执行的, 所以一旦有某个work被阻塞, 将导致该队列上其他work无法执行
1. 驱动开发原因: 任意创建工作队列, 会导致大量工作队列的存在
	1. 创建工作队列会消耗PID全局资源（PID的默认最大值为32768）
	1. 内核线程导致的资源抢占也给调度器带来压力, 造成频繁调度的开销, 反而使系统整体性能下降
	1. Linux工作队列中的工作任务是不能进行跨CPU迁移的, 如果同一队列中的工作任务之间存在相互依赖会进一步导致死锁

因为以上原因, Linux内核引入了并发可管理工作队列（Concurrency Managed Work Queue, CMWQ）机制. CMWQ本质上是将工作队列的管理从驱动开发者手里交还给内核, 由内核来决定工作线程的创建时机. CMWQ不同于传统工作队列的主要特点是同一个工作队列里的两个work不再遵循严格的串行性执行原则, 而是可以在两个不同CPU上并发执行, 不必等待其中一个完成后才调度另一个.

和传统的Linux工作队列相比, CMWQ在设计上具有如下优点:
1. 兼容性:CMWQ在接口上保持对传统工作队列接口的兼容, 方便已有设备驱动的移植
1. 灵活性: CMWQ提出了线程池（worker-pool）的概念, 不同工作队列之间可以共享线程池, 从而有效地减少了资源浪费和频繁调度
1. 并发性:CMWQ会根据情况动态创建新的工作线程来处理被阻塞的任务, 同时自动调节线程池大小与并发级别. 使用CMWQ的驱动开发者不必关心并发的具体细节, 这也正是CMWQ得名的原因

linux不同中断处理方式的特点比较
|属 性|硬中断|软中断|tasklet|工作队列|CMWQ|
|是否屏蔽相同中断|是 是 否 否 否|
|是否可以睡眠（拥有线程上下文）|否 否 否 是 是|
|是否可以被高优先级的任务所抢占|否 否 否 是 是|
|是否能在多个处理器核心上同时运行相同实例| 是 是 否 是 是|
|分配后是否能在不同处理器核心上迁移|否 否 否 否 是|

## 设备驱动与设备驱动模型
设备驱动（device driver）是操作系统中负责控制设备的定制化(根据设备的具体型号和相应参数进行特定配置)程序.

设备驱动模型是为了解决设备多样性的问题.

Linux系统里的三种基本设备抽象: 字符设备, 块设备, 网络设备.

字符设备（char device）的主要特点是将设备上的信息抽象为连续的字节流, 应用程序通常以顺序方式对字符设备进行字节粒度的读写. 字符设备包含了标准的文件操作接口,包括open, read, write, close等. 在Linux中, 可以通过查看`/proc/devices`来获取字符设备列表. 终端（tty）、显存（Ib）、声卡（alsa）都属于Linux的字符设备.

对于每个字符设备, Linux系统都会分配一个主设备号（major number）, 用于识别操作该设备的驱动实体. 一个驱动可能同时操作同类型的多个设备, 因此还会再分配一个次设备号（minor number）.

块设备（BlockDevjce）是以块的粒度对设备上的信息进行随机读写访问. 块是指设备寻址的最小单元, 通常为512字节、1KB或4KB. 设备一般用于存储设备的抽象. 由于块设备要求能访问存储设备的任意位置, 提供随机读写能力, 为了降低块设备操作的复杂度, Linux内核专门设计了BlockI/O（简称BIO）子系统, 向上服务于存储栈的文件系统, 向下沟通于存储设备的驱动程序. Linux的块设备包括虚拟磁盘（ramdisk）, 存储卡（SD卡和MMC卡）, 磁盘（HDD和SSD）等.

通常, 应用程序对字符设备的读写会直接触发驱动对设备的I/O操作; 而块设备的访问则通过添加一层页缓存来降低系统和设备频繁I/O所导致的性能开销. 

网络设备（net device）处理的数据单位是网络包（packet）. 网络设备使用独立的接抽象--套接字（socket）. 应用通过套接字接口与网络设备进行通信, 通常使用send, receive等网络独有的接口完成对网络包数据的收发请求. linux在内核空间维护着复杂的协议栈, 负责对网络包进行封装, 解析, 寻址等处理.

Ljnux的设备驱动模型包括一套统一的数据结构和一套用户空间接口. 设备驱动模型的数据结构对象对应于系统拓扑结构的各个部分, 各对象的关系也体现了不同设备之间的依赖关系, 从而可以帮助内核记录并追踪系统连接的设备.

Linux设备驱动模型定义了四种基本的数据结构:
1. 设备（device）:用于抽象系统中所有的硬件, 包括CPU和内存
1. 驱动（driver）:用于抽象系统中用于控制设备的驱动程序. Linux内核驱动的开发基本围绕该抽象进行, 即实现规定好的接口函数
1. 总线（bus）:用于抽象I／O设备和CPU之间的通信. Linux规定, 所有的设备都要至少连接一条总线（USB或PCI）
1. 类（class）:具有相似功能或属性的设备集合. 其思想来自面向对象程序设计中的类（class）的概念, 旨在抽象出一套可以在多个设备间共享的数据结构和接口. 属于相同类的驱动程序, 就可以直接继承父类定义好的公共资源.

在Linux设备驱动模型里,总线和类可以看成在两个不同的层面上对设备的组织和管理. 总线是在拓扑结构上对设备进行组织, 而类则是右逻辑结构上对设备进行组织, 这两种组织彼此正交.

Linux设备驱动模型的本质是围绕没备, 驱动, 总线和类这四个核心抽象, 将不问的外部设备和驱动方法以树状的形式, 交给os统一管理.

为了实现统一管理, Linux内核设计了专门的内核抽象kobject, 是所有设备驱动模型抽象的统一基类. kernel将所有kobject以层次结构进行管理同时引入引用计数机制对其释放进行管理, 还通过sysfs, 将每个kobject的特性、当前状态和参数暴露给用户空间.

kset表示为一组kobject的集合, 通常被理解为kobject的容器抽象或sysfs里的子系统. `/sys/devices`目录就是一个kset对象, 它包含系统中的设备所对应的目录. ktype则表示为kobjec属性及其操作的集合, 其数扰结构包含sysfs_ops, 用于通过sysfs来和驱动进行交互的应用程序读写函数.

Linux将设备驱动抽象为device_driver结构体, 驱动开发人员只需要关注结构体内要求的搜口实现. device_driver结构体中包含多个函数指针, 要求开发者柱册相应的回调函数, 以响应设备驱动需要处理的系统事件, 这些系统事覆盖了驱动的整个生命周期, 比如:
- probe: 需要实现驱动与设备的匹配检测的功能, 并将驱动对象与设备对象相关联
- remove: 函数用于在移除驱动时释放设备申请的系统资源
- shutdown、suspend和resume函数分别对应于os的关机、休眠和唤醒事件, 需要重新配置设备的工作状态.

Linux设备驱动模型提供了设备资源管理（device resource management）框架, 驱动开发者只需要申请资源, 而资源的回收和释放则交给设备资源管理框架自动完成. 使用设备资源管理框架提供的以`devm_`开
头的内核接口, 可以在驱动初始化失败时由内核自动回滚对资源的申请.

设备资源管理框架将设备所用到的系统资源以链表的形式进行管理, 在驱动模块被`remove()`移除的时候自动释放资源.

设备驱动环境（Device Driver Environment, DDE）的主要用意是复用其他系统上的设备驱动环境, 以便其他操作系统（如Linux和FreeBSD）的现成设备驱动程序可以在L4上重用.

由于Ljnux的快速迭代, 以及设备类型和数量的持续增多, 基于DDE模拟Linux设备驱动模型环境的维护成本愈来愈高. 后来L4采取了借助虚拟化复用二进制驱动程序的路线, 将自己化身为虚拟机监控器, 用L4微内核的接口将Linux运行在用户态上, L4Linux作为L4在用户态的服务器, 为L4系统上的其他应用提供驱动服务.

## sysfs
通过[sysfs_init](https://elixir.bootlin.com/linux/v6.6.13/source/fs/sysfs/mount.c#L97)初始化. 它由vfs初始化代码调用.

对sysfs文件的读写会转换为对属性的[show和store](https://elixir.bootlin.com/linux/v6.6.13/source/fs/sysfs/file.c#L219), 见sysfs_add_file_mode_ns.

## Linux的用户态驱动框架—用户空间I/O和虚拟空间I/O
UIO有一个明显的缺点, 它缺乏在用户态空间动态创建DMA区域的能力, 这也限制了一些驱动在UIO上的移植.

VFIO借助IOMMU限制了用户空间驱动代码的内存访问能力, 从而允许非特权用户驱动进程进行DMA操作.

和UIO类似, VFIO也在devfs下暴露了`/dev/vfio/<Group>`接口, 用于和用户态驱动进行交互. 这里的group是VFIO使用的基本隔离粒度, 表示一组和系统内其他设备相隔离的设备. group是确保安全访问的基本隔离粒度, 但不一定是VFIO控制设备的基本粒度. 出于性能考虑, VFIO允许在不同group之间共享同一虚拟地址空间. 为此, VFIO又引人了container类型, 容纳多个彼此间共享页表的group, 可以将container视为VFIO的基本操作单位.

打开`/dev/vfio/<vfio>字符设备`就代表创建了一个container, 但是container本身提供的接口很少, 用户需要将和设备相关的group加入container中才能进一步操作设备.

## 固件
固件子系统使用 sysfs 和热插拔机制工作. 当调用 request_firmware时, 函数将在 /sys/class/firmware 下创建一个以设备名为目录名的新目录，其中包含 3 个属性:
- loading ：这个属性应当被加载固件的用户空间进程设置为 1。当加载完毕, 它将被设为 0。被设为 -1 时，将中止固件加载。
- data ：一个用来接收固件数据的二进制属性。在设置 loading 为1后, 用户空间进程将固件写入这个属性。
- device ：一个链接到 /sys/devices 下相关入口项的符号链接。

一旦创建了 sysfs 入口, 内核将为设备产生一个热插拔事件，并传递包括变量 FIRMWARE 的环境变量给处理热插拔的用户空间程序. FIRMWARE 被设置为提供给 request_firmware 的固件文件名.
用户空间程序定位固件文件, 并将其拷贝到内核提供的二进制属性；若无法定位文件, 用户空间程序设置 loading 属性为 -1. 若固件请求在 10 秒内没有被服务, 内核就放弃并返回一个失败状态给驱动, 超时时间可通过 sysfs 属性 /sys/class/firmware/timeout 属性修改.

request_firmware 接口允许使用驱动发布设备固件. 当正确地集成进热插拔机制后, 固件加载子系统允许设备不受干扰地工作.

## 等待队列
在linux驱动中, 可以使用等待队列来实现阻塞进程的唤醒. 信号量也是基于等待队列实现的. 唤醒时, 是唤醒工作队列中的所有进程.

wake_up()应和wait_event()或wait_event_timeout()成对使用, 而wake_up_interruptible()则应与wake_event_interruptible()或wait_event_interruptible_timeout()成对使用. wake_up()可唤醒处于TASK_INTERRUPTIBLE和TASK_UNINTERRUPTIBLE的进程, 而wake_up_interruptible()只能唤醒TASK_INTERRUPTIBLE的进程.

sleep_on()是将进程的状态设置为TASK_UNINTERRUPTIBLE, 并定义一个等待队列成员, 然后把它挂到指定等待队列中, 直到资源可获得, 就唤醒该进程.

interruptible_sleep_on()与sleep_on()类似, 区别是进程设置为TASK_UNINTERRUPTIBLE, 唤醒进程或进程收到信号.

sleep_on()和wake_up成对使用; interruptible_sleep_on()和wake_up_interruptible()成对使用.

## 信号
为了使设备支持异步通知机制, 驱动程序涉及3项工作:
1. 支持F_SETOWN命令, 能在这个控制命令处理中设置filp->f_owner为对应进程id. 该项操作已由kernel完成, 设备驱动无需处理
1. 支持F_SETFL命令, 每当FASYNC标志改变时, 驱动程序中的fasync()函数将得以执行. 因此驱动需要实现fasync()
1. 在设备资源可获得时, 调用kill_fasync()激发相应的信号

## soc
soc不同于pc硬件环境, 外设控制器通常是集成在soc中. 因此它使用虚拟的总线即platform总线, 相应的设备(I2C, RTC, LCD, 看门狗等)称为platform_device, 驱动是platform_driver. platform_device并不是与字符/块/网络设备并列的概念. soc中的I2C, RTC, LCD, 看门狗本身是字符设备.

```c
// https://elixir.bootlin.com/linux/v6.6.22/source/include/linux/platform_device.h#L23
struct platform_device {
	const char	*name;
	int		id;
	bool		id_auto;
	struct device	dev;
	u64		platform_dma_mask;
	struct device_dma_parameters dma_parms;
	u32		num_resources;
	struct resource	*resource;

	const struct platform_device_id	*id_entry;
	/*
	 * Driver name to force a match.  Do not set directly, because core
	 * frees it.  Use driver_set_override() to set or clear it.
	 */
	const char *driver_override;

	/* MFD cell pointer */
	struct mfd_cell *mfd_cell;

	/* arch specific additions */
	struct pdev_archdata	archdata;
};

// https://elixir.bootlin.com/linux/v6.6.22/source/include/linux/platform_device.h#L236
struct platform_driver {
	int (*probe)(struct platform_device *);

	/*
	 * Traditionally the remove callback returned an int which however is
	 * ignored by the driver core. This led to wrong expectations by driver
	 * authors who thought returning an error code was a valid error
	 * handling strategy. To convert to a callback returning void, new
	 * drivers should implement .remove_new() until the conversion it done
	 * that eventually makes .remove() return void.
	 */
	int (*remove)(struct platform_device *);
	void (*remove_new)(struct platform_device *);

	void (*shutdown)(struct platform_device *);
	int (*suspend)(struct platform_device *, pm_message_t state);
	int (*resume)(struct platform_device *);
	struct device_driver driver;
	const struct platform_device_id *id_table;
	bool prevent_deferred_probe;
	/*
	 * For most device drivers, no need to care about this flag as long as
	 * all DMAs are handled through the kernel DMA API. For some special
	 * ones, for example VFIO drivers, they know how to manage the DMA
	 * themselves and set this flag so that the IOMMU layer will allow them
	 * to setup and manage their own I/O address space.
	 */
	bool driver_managed_dma;
};

// https://elixir.bootlin.com/linux/v6.6.22/source/drivers/base/platform.c#L1482
struct bus_type platform_bus_type = {
	.name		= "platform",
	.dev_groups	= platform_dev_groups,
	.match		= platform_match,
	.uevent		= platform_uevent,
	.probe		= platform_probe,
	.remove		= platform_remove,
	.shutdown	= platform_shutdown,
	.dma_configure	= platform_dma_configure,
	.dma_cleanup	= platform_dma_cleanup,
	.pm		= &platform_dev_pm_ops,
};
EXPORT_SYMBOL_GPL(platform_bus_type);
```

直接使用platform_driver的suspend, resume做电源关了回调的做法已经过时, 推荐使用platform_driver.driver中的dev_pm_ops().

与platform_driver地位对等的i2c_driver, spi_driver, usb_driver, pci_driver都包含了device_driver. device_driver描述了各种xxx_driver(xxx是总线名)在驱动上的一些共性.

系统为platform总线定义了一个bus_type实例platform_bus_type. 其match决定了platform_device和platform_driver如何进行匹配, 里面定义了5中可能性:
1. driver_override
1. 基于设备数风格
1. 基于ACPI分割
1. 匹配ID表(platform_device设备名是否出现在platform_driver的ID表中)
1. platform_device设备名与驱动名称

对于ARM, 内核2.6时, platform_device的定义通常在BSP的板文件中定义, 将platform_device归纳为一个数组, 最终通过platform_add_devices()注册到系统中; 3.x开始倾向于根据设备树中的内容自动展开platform_device.

## 输入设备驱动
input_allocate_device/input_free_device: 分配/释放一个输入设备. 分配返回一个input_dev

input_register_device/input_unregister_device: 注册/销毁输入设备

input_event(): 报告指定type, code的输入事件
input_report_key(): 报告键值
input_report_rel(): 报告相对坐标
input_report_abs(): 报告绝对坐标
input_sync(): 报告同步事件

input_event: 对于所有的输入事件, 内核统一用它描述.

[drivers/input/keyboard/gpio_keys.c](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/input/keyboard/gpio_keys.c)基于input架构实现了一个通用的GPIO按键驱动, 它基于platform_driver架构.

GPIO按键驱动通过input_event()和input_sync()来汇报按键事件和同步事件. 该驱动没有file_operations的动作, 也没有各种I/O模型, 这是由于与VFS相关的全部在drivers/input/evdev.c中实现了.

## RTC(实时钟)设备驱动
RTC借助电池供电, 在系统下电的情况下依旧能正常计时. 它通常还具有产生周期性中断以及闹钟(alarm)中断的能力, 是一种字符设备.

drivers/rtc/rtc-dev.c实现了RTC驱动通用的字符设备驱动层. 它导出rtc_class_ops描述底层的RTC硬件操作. 该通用层使得底层的RTC驱动不再关心RTC作为字符设备驱动的具体实现, 也无需关心一些通用的RTC控制逻辑. 例子见drivers/rtc/rtc-s3c.c

## framebuffer设备驱动
framebuffer(帧缓冲)是linux为显示设备提供的一个接口, 将显示缓冲区抽象, 屏蔽图形硬件的底层差异, 允许上层应用程序直接对显示缓冲区进行读写操作.

framebuffer设备驱动提供给用户空间的file_operations由drivers/video/fbdev/core/fbmem.c中的file_operations提供. 特定帧缓冲区设备fb_info的注册, 销毁以及成员的维护, 尤其是fb_ops中成员函数的实现由对应的xxxfb.c实现. fb_ops的成员函数最终会操作LCD的硬件寄存器.

多数显存的操作都是规范的, 按照像素点格式写帧缓冲区即可. 但少数LCD特殊, 此时drivers/video/fbdev/core/fbmem.c实现的fb_write()允许底层提供一个重写自身的机会.

## 终端设备驱动
终端是一种字符设备, 有很多类型, 通常使用tty来简称各种类型的终端设备. 在嵌入式中, 最常见的是UART(Universal Asynchronous Receiver/Transmitter)串口设备.

linux tty层次:
1. 核心的tty_io.c

	标准的字符设备驱动, 对上有字符设备的职责, 实现file_operations成员函数
1. tty线路规程n_tty.c(实现N_TTY线路规程):

	tty线路规程的工作是以特殊的方式格式化一个用户或者硬件收到的数据, 这种格式化通常是一种协议转换
1. tty驱动实例xxx_tty.c 

	填充tty_driver的成员, 实现其中的tty_operations

tty设备工作流程：
- 发送数据: tty核心从一个用户获取将要发送给一个tty设备的数据, tty核心将数据传递给tty线路规程驱动, 接着数据被传递给tty驱动, tty驱动将数据转换为可以发送给硬件的格式
- 接收数据: 从tty硬件接收到的数据向上给tty驱动, 接着进入tty线路规程驱动, 在进入tty核心, 再被一个用户获取

鉴于串口的共性, linux在drivers/tty/serial/serial_core.c实现了UART设备的通用tty驱动层. 因此UART驱动的主要任务演变为实现serial-core.c定义的uart_xxx接口而不是tty_xxx接口.

串口核心层定义了uart_driver和uart_ops, 由uart_register_driver()注册uart_driver. uart_driver本质是派生自tty_driver, 因为它内嵌了tty_driver.

> tty_driver是字符设备的泛化, serial-core是tty_driver的泛化, 具体串口驱动是serial-core的泛化.

## misc设备驱动
部分没法具体划分归属类型的设备可使用miscdevice框架. miscdevice本质是字符设备.

## SPI
spi_master用于描述SPI主机控制器驱动

arm 3.x启用设备树后, 倾向于在SPI控制器节点下填写子节点, 比如[`arch/arm/boot/dts/ti/omap/omap3-overo-common-lcd43.dtsi`](https://elixir.bootlin.com/linux/v6.6.22/source/arch/arm/boot/dts/ti/omap/omap3-overo-common-lcd43.dtsi#L151)的ads7846节点.

## 驱动核心层
思想:
1. 对上提供接口

	file_operations的读,写,ioctl都被中间层搞定, 各种I/O模型也被处理好了
1. 中间层实现通用逻辑

	共享代码, 避免重复实现
1. 对下定义框架

	底层驱动不用关系VFS接口和各种肯能的I/O模型, 只需处理与具体硬件相关的访问

## 上下文
内核包括:
1. 进程上下文

	进程上下文，就是一个进程在执行的时候，CPU的所有寄存器中的值、进程的状态以及堆栈上的内容，当内核需要切换到另一个进程时，它 需要保存当前进程的所有状态，即保存当前进程的进程上下文，以便再次执行该进程时，能够恢复切换时的状态，继续执行

1. 中断上下文

	硬件通过触发信号，向CPU发送中断信号，导致内核调用中断处理程序，进入内核空间。这个过程中，硬件的一些变量和参数也要传递给内核， 内核通过这些参数进行中断处理。

	所以，“中断上下文”就可以理解为硬件传递过来的这些参数和内核需要保存的一些环境，主要是被中断的进程的环境

进程上下文和中断上下文不可能同时发生.