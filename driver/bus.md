# bus
Linux 用 BUS（总线）组织设备和驱动, 在 `/sys/bus` 目录下输入 `tree` 就可以看到所有总线下的所有设备了, 组织方式是总线目录下有设备目录，设备目录下是设备文件.

从tree输出来看Linux 的驱动模型至少有三个核心数据结构，分别是总线、设备和驱动，但是要像输出那样有层次化地组织它们, 还得有两个数据结构来组织它们，那就是 kobject 和 kset.

kobject 和 kset 是构成 /sys 目录下的目录节点和文件节点的核心，也是层次化组织总线、设备、驱动的核心数据结构，kobject、kset 数据结构都能表示一个目录或者文件节点.

[int __init platform_bus_init(void)](https://elixir.bootlin.com/linux/v5.12.9/source/drivers/base/platform.c#L1547)是总线注册入口. 其中的`device_register(&platform_bus)`是设备注册函数. `bus_register(&platform_bus_type)`是总线注册函数, 它的作用是把总线对象注册到kernel.


```c
// https://elixir.bootlin.com/linux/v5.12.9/source/drivers/base/bus.c#L785
/**
 * bus_register - register a driver-core subsystem
 * @bus: bus to register
 *
 * Once we have that, we register the bus with the kobject
 * infrastructure, then register the children subsystems it has:
 * the devices and drivers that belong to the subsystem.
 */
int bus_register(struct bus_type *bus)
{
    int retval;
    struct subsys_private *priv;
    struct lock_class_key *key = &bus->lock_key;

    priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->bus = bus;
    bus->p = priv;

    BLOCKING_INIT_NOTIFIER_HEAD(&priv->bus_notifier);

    retval = kobject_set_name(&priv->subsys.kobj, "%s", bus->name); // 给kobject命名
    if (retval)
        goto out;

    priv->subsys.kobj.kset = bus_kset;
    priv->subsys.kobj.ktype = &bus_ktype;
    priv->drivers_autoprobe = 1;

    retval = kset_register(&priv->subsys); //子系统注册. 注册过程是往sysfs写入一个目录
    if (retval)
        goto out;

    retval = bus_create_file(bus, &bus_attr_uevent);
    if (retval)
        goto bus_uevent_fail;

    priv->devices_kset = kset_create_and_add("devices", NULL,
                         &priv->subsys.kobj); // devices是一个[kset](https://elixir.bootlin.com/linux/v5.12.9/source/include/linux/kobject.h#L192)对象, 给devices内含的kobject对象命名, 然后注册
    if (!priv->devices_kset) {
        retval = -ENOMEM;
        goto bus_devices_fail;
    }

    priv->drivers_kset = kset_create_and_add("drivers", NULL,
                         &priv->subsys.kobj); // drivers与devices类似
    if (!priv->drivers_kset) {
        retval = -ENOMEM;
        goto bus_drivers_fail;
    }

    INIT_LIST_HEAD(&priv->interfaces);
    __mutex_init(&priv->mutex, "subsys mutex", key);
    klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
    klist_init(&priv->klist_drivers, NULL, NULL);

    retval = add_probe_files(bus);
    if (retval)
        goto bus_probe_files_fail;

    retval = bus_add_groups(bus, bus->bus_groups);
    if (retval)
        goto bus_groups_fail;

    pr_debug("bus: '%s': registered\n", bus->name);
    return 0;

bus_groups_fail:
    remove_probe_files(bus);
bus_probe_files_fail:
    kset_unregister(bus->p->drivers_kset);
bus_drivers_fail:
    kset_unregister(bus->p->devices_kset);
bus_devices_fail:
    bus_remove_file(bus, &bus_attr_uevent);
bus_uevent_fail:
    kset_unregister(&bus->p->subsys);
out:
    kfree(bus->p);
    bus->p = NULL;
    return retval;
}
EXPORT_SYMBOL_GPL(bus_register);
```

bus对象包含两个kset, 一个devices_kset代表总线包含的设备对象, 一个drivers_kset代表总线包含的驱动对象.

bus_register总结为3个注册:
- bus对象自身
- devices_kset
- drivers_kset

## kobject与kset的关系
kset里封装了一个kobject, 同时包含一个链表头list, 属于该kset的所有kobject都要链接到list.

```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/kobject.h#L64
struct kobject {
    const char      *name; //名称，反映在sysfs中
    struct list_head    entry; //挂入kset结构的链表
    struct kobject      *parent; //指向父结构
    struct kset     *kset; //指向所属的kset
    const struct kobj_type  *ktype;
    struct kernfs_node  *sd; /* sysfs directory entry */ //指向sysfs目录项
    struct kref     kref; // 引用计数器结构
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
    struct delayed_work release;
#endif
    unsigned int state_initialized:1; //初始化状态
    unsigned int state_in_sysfs:1; //是否在sysfs中
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;
};


//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/kobject.h#L166
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
    struct list_head list; //挂载kobject的链表
    spinlock_t list_lock;
    struct kobject kobj; //自身包含一个kobject结构
    const struct kset_uevent_ops *uevent_ops;
} __randomize_layout;
```

每一个 kobject，都对应着 /sys 目录下（其实是 sysfs 文件系统挂载在 /sys 目录下） 的一个目录或者文件，目录或者文件的名字就是 kobject 结构中的 name.

kset 结构中本身又包含一个 kobject 结构，所以它既是 kobject 的容器，同时本身还是一个 kobject.

在 Linux 内核中，至少有两个顶层 kset:
```c
//https://elixir.bootlin.com/linux/v6.5.1/source/drivers/base/core.c#L4063
int __init devices_init(void)
{
    devices_kset = kset_create_and_add("devices", &device_uevent_ops, NULL); //管理所有设备
    if (!devices_kset)
        return -ENOMEM;
    dev_kobj = kobject_create_and_add("dev", NULL);
    if (!dev_kobj)
        goto dev_kobj_err;
    sysfs_dev_block_kobj = kobject_create_and_add("block", dev_kobj);
    if (!sysfs_dev_block_kobj)
        goto block_kobj_err;
    sysfs_dev_char_kobj = kobject_create_and_add("char", dev_kobj);
    if (!sysfs_dev_char_kobj)
        goto char_kobj_err;

    return 0;

 char_kobj_err:
    kobject_put(sysfs_dev_block_kobj);
 block_kobj_err:
    kobject_put(dev_kobj);
 dev_kobj_err:
    kset_unregister(devices_kset);
    return -ENOMEM;
}

//https://elixir.bootlin.com/linux/v6.5.1/source/drivers/base/bus.c#L1374
int __init buses_init(void)
{
    bus_kset = kset_create_and_add("bus", &bus_uevent_ops, NULL); //管理所有总线
    if (!bus_kset)
        return -ENOMEM;

    system_kset = kset_create_and_add("system", NULL, &devices_kset->kobj); //在设备kset之下建立system的kset
    if (!system_kset)
        return -ENOMEM;

    return 0;
}
```

kset 与 kobject 结构只是基础数据结构, 只能实现层次结构，其它的什么也不能干, 因此它们肯定是需要嵌入到更高级的数据结构之中使用.

Linux 把总线抽象成 bus_type 结构:
```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/device/bus.h#L80
/**
 * struct bus_type - The bus type of the device
 *
 * @name:   The name of the bus.
 * @dev_name:   Used for subsystems to enumerate devices like ("foo%u", dev->id).
 * @bus_groups: Default attributes of the bus.
 * @dev_groups: Default attributes of the devices on the bus.
 * @drv_groups: Default attributes of the device drivers on the bus.
 * @match:  Called, perhaps multiple times, whenever a new device or driver
 *      is added for this bus. It should return a positive value if the
 *      given device can be handled by the given driver and zero
 *      otherwise. It may also return error code if determining that
 *      the driver supports the device is not possible. In case of
 *      -EPROBE_DEFER it will queue the device for deferred probing.
 * @uevent: Called when a device is added, removed, or a few other things
 *      that generate uevents to add the environment variables.
 * @probe:  Called when a new device or driver add to this bus, and callback
 *      the specific driver's probe to initial the matched device.
 * @sync_state: Called to sync device state to software state after all the
 *      state tracking consumers linked to this device (present at
 *      the time of late_initcall) have successfully bound to a
 *      driver. If the device has no consumers, this function will
 *      be called at late_initcall_sync level. If the device has
 *      consumers that are never bound to a driver, this function
 *      will never get called until they do.
 * @remove: Called when a device removed from this bus.
 * @shutdown:   Called at shut-down time to quiesce the device.
 *
 * @online: Called to put the device back online (after offlining it).
 * @offline:    Called to put the device offline for hot-removal. May fail.
 *
 * @suspend:    Called when a device on this bus wants to go to sleep mode.
 * @resume: Called to bring a device on this bus out of sleep mode.
 * @num_vf: Called to find out how many virtual functions a device on this
 *      bus supports.
 * @dma_configure:  Called to setup DMA configuration on a device on
 *          this bus.
 * @dma_cleanup:    Called to cleanup DMA configuration on a device on
 *          this bus.
 * @pm:     Power management operations of this bus, callback the specific
 *      device driver's pm-ops.
 * @iommu_ops:  IOMMU specific operations for this bus, used to attach IOMMU
 *              driver implementations to a bus and allow the driver to do
 *              bus-specific setup
 * @need_parent_lock:   When probing or removing a device on this bus, the
 *          device core should lock the device's parent.
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
    const char      *name; // 总线名称
    const char      *dev_name; //用于枚举设备
    const struct attribute_group **bus_groups; //总线的默认属性
    const struct attribute_group **dev_groups; //总线上设备的默认属性
    const struct attribute_group **drv_groups; //总线上驱动的默认属性

    int (*match)(struct device *dev, struct device_driver *drv); // 每当有新的设备或驱动程序被添加到这个总线上时调用
    int (*uevent)(const struct device *dev, struct kobj_uevent_env *env); //当一个设备被添加、移除或其他一些事情时被调用产生uevent来添加环境变量
    int (*probe)(struct device *dev); //当一个新的设备或驱动程序添加到这个总线时被调用，并回调特定驱动程序探查函数，以初始化匹配的设备
    void (*sync_state)(struct device *dev); //将设备状态同步到软件状态时调用
    void (*remove)(struct device *dev); //当一个设备从这个总线上删除时被调用
    void (*shutdown)(struct device *dev); //当系统关闭时被调用

    int (*online)(struct device *dev); //调用以使设备重新上线（在下线后）
    int (*offline)(struct device *dev); //调用以使设备离线，以便热移除。可能会失败

    int (*suspend)(struct device *dev, pm_message_t state); //当这个总线上的设备想进入睡眠模式时调用
    int (*resume)(struct device *dev); //调用以使该总线上的一个设备脱离睡眠模式

    int (*num_vf)(struct device *dev); //调用以找出该总线上的一个设备支持多少个虚拟设备功能

    int (*dma_configure)(struct device *dev); //调用以在该总线上的设备配置DMA
    void (*dma_cleanup)(struct device *dev);

    const struct dev_pm_ops *pm; //该总线的电源管理操作，回调特定的设备驱动的pm-ops

    const struct iommu_ops *iommu_ops; //此总线的IOMMU具体操作，用于将IOMMU驱动程序实现到总线上

    bool need_parent_lock; //当探测或移除该总线上的一个设备时，设备驱动核心应该锁定该设备
};
```

总线不仅仅是组织设备和驱动的容器，还是同类设备的共有功能的抽象层.

```c
//https://elixir.bootlin.com/linux/v6.5.1/source/drivers/base/base.h#L42
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
 * @lock_key:   Lock class key for use by the lock validator
 *
 * This structure is the one that is the actual kobject allowing struct
 * bus_type/class to be statically allocated safely.  Nothing outside of the
 * driver core should ever touch these fields.
 */
struct subsys_private {
    struct kset subsys; //定义这个子系统结构的kset
    struct kset *devices_kset; //该总线的"设备"目录，包含所有的设备
    struct list_head interfaces; //总线相关接口的列表
    struct mutex mutex; //保护设备和接口列表

    struct kset *drivers_kset; //该总线的"驱动"目录，包含所有的驱动
    struct klist klist_devices; //挂载总线上所有设备的可迭代链表
    struct klist klist_drivers; //挂载总线上所有驱动的可迭代链表
    struct blocking_notifier_head bus_notifier;
    unsigned int drivers_autoprobe:1;
    const struct bus_type *bus; //指向所属总线
    struct device *dev_root;

    struct kset glue_dirs;
    const struct class *class; //指向这个结构所关联类结构的指针

    struct lock_class_key lock_key;
};
#define to_subsys_private(obj) container_of_const(obj, struct subsys_private, subsys.kobj) //通过kobject找到对应的subsys_private
```

subsys_private，它是总线的驱动核心的私有数据. 通过 bus_kset 可以找到所有的 kset，通过 kset 又能找到 subsys_private，再通过 subsys_private 就可以找到总线了，也可以找到该总线上所有的设备与驱动.

Linux 系统中设备用device表示:
```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/device.h#L676
/**
 * struct device - The basic device structure
 * @parent: The device's "parent" device, the device to which it is attached.
 *      In most cases, a parent device is some sort of bus or host
 *      controller. If parent is NULL, the device, is a top-level device,
 *      which is not usually what you want.
 * @p:      Holds the private data of the driver core portions of the device.
 *      See the comment of the struct device_private for detail.
 * @kobj:   A top-level, abstract class from which other classes are derived.
 * @init_name:  Initial name of the device.
 * @type:   The type of device.
 *      This identifies the device type and carries type-specific
 *      information.
 * @mutex:  Mutex to synchronize calls to its driver.
 * @bus:    Type of bus device is on.
 * @driver: Which driver has allocated this
 * @platform_data: Platform data specific to the device.
 *      Example: For devices on custom boards, as typical of embedded
 *      and SOC based hardware, Linux often uses platform_data to point
 *      to board-specific structures describing devices and how they
 *      are wired.  That can include what ports are available, chip
 *      variants, which GPIO pins act in what additional roles, and so
 *      on.  This shrinks the "Board Support Packages" (BSPs) and
 *      minimizes board-specific #ifdefs in drivers.
 * @driver_data: Private pointer for driver specific info.
 * @links:  Links to suppliers and consumers of this device.
 * @power:  For device power management.
 *      See Documentation/driver-api/pm/devices.rst for details.
 * @pm_domain:  Provide callbacks that are executed during system suspend,
 *      hibernation, system resume and during runtime PM transitions
 *      along with subsystem-level and driver-level callbacks.
 * @em_pd:  device's energy model performance domain
 * @pins:   For device pin management.
 *      See Documentation/driver-api/pin-control.rst for details.
 * @msi:    MSI related data
 * @numa_node:  NUMA node this device is close to.
 * @dma_ops:    DMA mapping operations for this device.
 * @dma_mask:   Dma mask (if dma'ble device).
 * @coherent_dma_mask: Like dma_mask, but for alloc_coherent mapping as not all
 *      hardware supports 64-bit addresses for consistent allocations
 *      such descriptors.
 * @bus_dma_limit: Limit of an upstream bridge or bus which imposes a smaller
 *      DMA limit than the device itself supports.
 * @dma_range_map: map for DMA memory ranges relative to that of RAM
 * @dma_parms:  A low level driver may set these to teach IOMMU code about
 *      segment limitations.
 * @dma_pools:  Dma pools (if dma'ble device).
 * @dma_mem:    Internal for coherent mem override.
 * @cma_area:   Contiguous memory area for dma allocations
 * @dma_io_tlb_mem: Pointer to the swiotlb pool used.  Not for driver use.
 * @archdata:   For arch-specific additions.
 * @of_node:    Associated device tree node.
 * @fwnode: Associated device node supplied by platform firmware.
 * @devt:   For creating the sysfs "dev".
 * @id:     device instance
 * @devres_lock: Spinlock to protect the resource of the device.
 * @devres_head: The resources list of the device.
 * @knode_class: The node used to add the device to the class list.
 * @class:  The class of the device.
 * @groups: Optional attribute groups.
 * @release:    Callback to free the device after all references have
 *      gone away. This should be set by the allocator of the
 *      device (i.e. the bus driver that discovered the device).
 * @iommu_group: IOMMU group the device belongs to.
 * @iommu:  Per device generic IOMMU runtime data
 * @physical_location: Describes physical location of the device connection
 *      point in the system housing.
 * @removable:  Whether the device can be removed from the system. This
 *              should be set by the subsystem / bus driver that discovered
 *              the device.
 *
 * @offline_disabled: If set, the device is permanently online.
 * @offline:    Set after successful invocation of bus type's .offline().
 * @of_node_reused: Set if the device-tree node is shared with an ancestor
 *              device.
 * @state_synced: The hardware state of this device has been synced to match
 *        the software state of this device by calling the driver/bus
 *        sync_state() callback.
 * @can_match:  The device has matched with a driver at least once or it is in
 *      a bus (like AMBA) which can't check for matching drivers until
 *      other devices probe successfully.
 * @dma_coherent: this particular device is dma coherent, even if the
 *      architecture supports non-coherent devices.
 * @dma_ops_bypass: If set to %true then the dma_ops are bypassed for the
 *      streaming DMA operations (->map_* / ->unmap_* / ->sync_*),
 *      and optionall (if the coherent mask is large enough) also
 *      for dma allocations.  This flag is managed by the dma ops
 *      instance from ->dma_supported.
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
    struct device       *parent; //指向父设备

    struct device_private   *p; //设备的私有数据

    const char      *init_name; /* initial name of the device */
    const struct device_type *type; // 设备类型

    const struct bus_type   *bus;   /* type of bus device is on */ //指向设备所属总线
    struct device_driver *driver;   /* which driver has allocated this
                       device */ // 指向设备的驱动
    void        *platform_data; /* Platform specific data, device
                       core doesn't touch it */ // 设备平台数据
    void        *driver_data;   /* Driver data, set and get with
                       dev_set_drvdata/dev_get_drvdata */ //设备驱动的私有数据
    struct mutex        mutex;  /* mutex to synchronize calls to
                     * its driver.
                     */

    struct dev_links_info   links; // 设备供应商链接
    struct dev_pm_info  power; // 用于设备的电源管理
    struct dev_pm_domain    *pm_domain; // 提供在系统暂停时执行调用

#ifdef CONFIG_ENERGY_MODEL
    struct em_perf_domain   *em_pd;
#endif

#ifdef CONFIG_PINCTRL
    struct dev_pin_info *pins;
#endif
    struct dev_msi_info msi;
#ifdef CONFIG_DMA_OPS
    const struct dma_map_ops *dma_ops;
#endif
    u64     *dma_mask;  /* dma mask (if dma'able device) */
    u64     coherent_dma_mask;/* Like dma_mask, but for
                         alloc_coherent mappings as
                         not all hardware supports
                         64 bit addresses for consistent
                         allocations such descriptors. */
    u64     bus_dma_limit;  /* upstream dma constraint */
    const struct bus_dma_region *dma_range_map;

    struct device_dma_parameters *dma_parms;

    struct list_head    dma_pools;  /* dma pools (if dma'ble) */

#ifdef CONFIG_DMA_DECLARE_COHERENT
    struct dma_coherent_mem *dma_mem; /* internal for coherent mem
                         override */
#endif
#ifdef CONFIG_DMA_CMA
    struct cma *cma_area;       /* contiguous memory area for dma
                       allocations */
#endif
#ifdef CONFIG_SWIOTLB
    struct io_tlb_mem *dma_io_tlb_mem;
#endif
    /* arch specific additions */
    struct dev_archdata archdata;

    struct device_node  *of_node; /* associated device tree node */ //用于访问设备树节点
    struct fwnode_handle    *fwnode; /* firmware device node */ // 设备固件节点

#ifdef CONFIG_NUMA
    int     numa_node;  /* NUMA node this device is close to */
#endif
    dev_t           devt;   /* dev_t, creates the sysfs "dev" */ // 用于创建sysfs "dev"
    u32         id; /* device instance */ //设备实例id

    spinlock_t      devres_lock; //设备资源链表锁
    struct list_head    devres_head; //设备资源链表

    const struct class  *class; //设备的class
    const struct attribute_group **groups;  /* optional groups */ //可选的属性组

    void    (*release)(struct device *dev); //在所有引用结束后释放设备
    struct iommu_group  *iommu_group; //该设备属于的iommu组
    struct dev_iommu    *iommu; //每个设备的通用iommu运行时数据

    struct device_physical_location *physical_location;

    enum device_removable   removable;

    bool            offline_disabled:1;
    bool            offline:1;
    bool            of_node_reused:1;
    bool            state_synced:1;
    bool            can_match:1;
#if defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_DEVICE) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU_ALL)
    bool            dma_coherent:1;
#endif
#ifdef CONFIG_DMA_OPS_BYPASS
    bool            dma_ops_bypass : 1;
#endif
};
```

device 结构中同样包含了 kobject 结构，这使得设备可以加入 kset 和 kobject 组建的层次结构中。device 结构中有总线和驱动指针，这能帮助设备找到自己的驱动程序和总线.

linux驱动用device_driver表示:
```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/device/driver.h#L96
/**
 * struct device_driver - The basic device driver structure
 * @name:   Name of the device driver.
 * @bus:    The bus which the device of this driver belongs to.
 * @owner:  The module owner.
 * @mod_name:   Used for built-in modules.
 * @suppress_bind_attrs: Disables bind/unbind via sysfs.
 * @probe_type: Type of the probe (synchronous or asynchronous) to use.
 * @of_match_table: The open firmware table.
 * @acpi_match_table: The ACPI match table.
 * @probe:  Called to query the existence of a specific device,
 *      whether this driver can work with it, and bind the driver
 *      to a specific device.
 * @sync_state: Called to sync device state to software state after all the
 *      state tracking consumers linked to this device (present at
 *      the time of late_initcall) have successfully bound to a
 *      driver. If the device has no consumers, this function will
 *      be called at late_initcall_sync level. If the device has
 *      consumers that are never bound to a driver, this function
 *      will never get called until they do.
 * @remove: Called when the device is removed from the system to
 *      unbind a device from this driver.
 * @shutdown:   Called at shut-down time to quiesce the device.
 * @suspend:    Called to put the device to sleep mode. Usually to a
 *      low power state.
 * @resume: Called to bring a device from sleep mode.
 * @groups: Default attributes that get created by the driver core
 *      automatically.
 * @dev_groups: Additional attributes attached to device instance once
 *      it is bound to the driver.
 * @pm:     Power management operations of the device which matched
 *      this driver.
 * @coredump:   Called when sysfs entry is written to. The device driver
 *      is expected to call the dev_coredump API resulting in a
 *      uevent.
 * @p:      Driver core's private data, no one other than the driver
 *      core can touch this.
 *
 * The device driver-model tracks all of the drivers known to the system.
 * The main reason for this tracking is to enable the driver core to match
 * up drivers with new devices. Once drivers are known objects within the
 * system, however, a number of other things become possible. Device drivers
 * can export information and configuration variables that are independent
 * of any specific device.
 */
struct device_driver {
    const char      *name; //驱动名称
    const struct bus_type   *bus; //指向总线

    struct module       *owner; //模块持有者
    const char      *mod_name;  /* used for built-in modules */ //用于内置模块

    bool suppress_bind_attrs;   /* disables bind/unbind via sysfs */ //禁用通过sysfs的绑定/解绑
    enum probe_type probe_type; //要使用的探查类型（同步或异步）

    const struct of_device_id   *of_match_table; //开放固件表
    const struct acpi_device_id *acpi_match_table; //ACPI匹配表

    int (*probe) (struct device *dev); //被调用来查询一个特定设备的存在
    void (*sync_state)(struct device *dev); //将设备状态同步到软件状态时调用
    int (*remove) (struct device *dev); //当设备被从系统中移除时被调用，以便解除设备与该驱动的绑定
    void (*shutdown) (struct device *dev); //关机时调用，使设备停止
    int (*suspend) (struct device *dev, pm_message_t state); //调用以使设备进入睡眠模式，通常是进入一个低功率状态
    int (*resume) (struct device *dev); //调用以使设备从睡眠模式中恢复
    const struct attribute_group **groups; //默认属性
    const struct attribute_group **dev_groups; //绑定设备的属性

    const struct dev_pm_ops *pm; //设备电源操作
    void (*coredump) (struct device *dev); //当sysfs目录被写入时被调用

    struct driver_private *p; //驱动程序私有数据
};

//https://elixir.bootlin.com/linux/v6.5.1/source/drivers/base/base.h#L78
struct driver_private {
    struct kobject kobj;
    struct klist klist_devices; //驱动管理的所有设备的链表
    struct klist_node knode_bus; //加入bus链表的节点
    struct module_kobject *mkobj; //指向用kobject管理模块节点
    struct device_driver *driver; //指向驱动本身
};
#define to_driver(obj) container_of(obj, struct driver_private, kobj)
```


在 device_driver 结构中，包含了驱动程序的名字、驱动程序所在模块、设备探查和电源相关的回调函数的指针。在 driver_private 结构中同样包含了 kobject 结构，用于组织所有的驱动，还指向了驱动本身，它与bus_type 中的 subsys_private 结构的机制如出一辙.


在 Linux 系统中提供了更为高级的封装，Linux 将设备分成几类分别是：字符设备、块设备、网络设备以及杂项设备.

这些类型的设备的数据结构，都会直接或者间接包含基础的 device 结构. 以杂项设备为例，Linux 用 miscdevice 结构表示一个杂项设备:
```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/miscdevice.h#L79
struct miscdevice  {
    int minor; //设备号
    const char *name; //设备名称
    const struct file_operations *fops; //文件操作函数结构
    struct list_head list; //链表
    struct device *parent; //执行父设备的device
    struct device *this_device; //执行本设备的device
    const struct attribute_group **groups;
    const char *nodename; // 节点名称
    umode_t mode; //访问权限
};


//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/fs.h#L1774
struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
    int (*iopoll)(struct kiocb *kiocb, struct io_comp_batch *,
            unsigned int flags);
    int (*iterate_shared) (struct file *, struct dir_context *);
    __poll_t (*poll) (struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
    int (*mmap) (struct file *, struct vm_area_struct *);
    unsigned long mmap_supported_flags;
    int (*open) (struct inode *, struct file *);
    int (*flush) (struct file *, fl_owner_t id);
    int (*release) (struct inode *, struct file *);
    int (*fsync) (struct file *, loff_t, loff_t, int datasync);
    int (*fasync) (int, struct file *, int);
    int (*lock) (struct file *, int, struct file_lock *);
    unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
    int (*check_flags)(int);
    int (*flock) (struct file *, int, struct file_lock *);
    ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
    ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
    void (*splice_eof)(struct file *file);
    int (*setlease)(struct file *, long, struct file_lock **, void **);
    long (*fallocate)(struct file *file, int mode, loff_t offset,
              loff_t len);
    void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
    unsigned (*mmap_capabilities)(struct file *);
#endif
    ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
            loff_t, size_t, unsigned int);
    loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
                   struct file *file_out, loff_t pos_out,
                   loff_t len, unsigned int remap_flags);
    int (*fadvise)(struct file *, loff_t, loff_t, int);
    int (*uring_cmd)(struct io_uring_cmd *ioucmd, unsigned int issue_flags);
    int (*uring_cmd_iopoll)(struct io_uring_cmd *, struct io_comp_batch *,
                unsigned int poll_flags);
} __randomize_layout;
```

miscdevice 结构就是一个杂项设备，它一般在驱动程序代码文件中静态定义。它有个 this_device 指针，它指向下层的、属于这个杂项设备的 device 结构. file_operations 结构，是设备一经注册，就会在 sys 相关的目录下建立设备对应的文件结点，对这个文件结点打开、读写等操作，最终会调用到驱动程序对应的函数，而对应的函数指针就保存在 file_operations 结构中.

file_operations 结构的地址存在一个文件的 inode 结构中. 在 Linux 系统中，都是用 inode 结构表示一个文件，不管它是数据文件还是设备文件.

Linux 调用file_operations 结构中的函数, 可参考Linux 的打开系统调用接口时调用 filp_open 函数.


Linux 系统也提供了很多专用接口函数，用来建立总线和设备.
Linux 系统提供了一个 bus_register 函数向内核注册一个总线，相当于建立了一个总线. bus_register 函数会在系统中注册一个总线，所需参数就是总线结构的地址 (&devicesinfo_bus)，返回非 0 表示注册失败.

添加设备驱动见[miscdrv demo](https://gitee.com/lmos/cosmos/tree/master/lesson31/misc).

正常情况下，是不能获取 bus_kset 地址的，它是所有总线的根，包含了所有总线的 kobject，Linux 为了保护 bus_kset，并没有在 bus_type 结构中直接包含 kobject，而是让总线指向一个 subsys_private 结构，在其中包含了 kobject 结构. 所以，要注册一个总线，这样就能拔出萝卜带出泥，得到 bus_kset，根据它又能找到所有 subsys_private 结构中的 kobject，接着找到 subsys_private 结构，反向查询到 bus_type 结构的地址。然后调用 Linux 提供的 bus_for_each_dev 函数，就可以遍历一个总线上的所有设备，它每遍历到一个设备，就调用一个函数，这个函数是用参数的方式传给它的，在miscdrv.c就是 misc_find_match 函数.

```c
//https://elixir.bootlin.com/linux/v6.5.2/source/drivers/base/bus.c#L844
/**
 * bus_register - register a driver-core subsystem
 * @bus: bus to register
 *
 * Once we have that, we register the bus with the kobject
 * infrastructure, then register the children subsystems it has:
 * the devices and drivers that belong to the subsystem.
 */
int bus_register(const struct bus_type *bus)
{
    int retval;
    struct subsys_private *priv;
    struct kobject *bus_kobj;
    struct lock_class_key *key;

    priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL); // 分配一个subsys_private
    if (!priv)
        return -ENOMEM;

    priv->bus = bus;

    BLOCKING_INIT_NOTIFIER_HEAD(&priv->bus_notifier);

    bus_kobj = &priv->subsys.kobj;
    retval = kobject_set_name(bus_kobj, "%s", bus->name); //把总线的名称加入subsys_private的kobject中
    if (retval)
        goto out;

    bus_kobj->kset = bus_kset;
    bus_kobj->ktype = &bus_ktype;
    priv->drivers_autoprobe = 1;

    retval = kset_register(&priv->subsys); //把subsys_private中的kset注册到系统中
    if (retval)
        goto out;

    retval = bus_create_file(bus, &bus_attr_uevent);  //建立总线的文件结构在sysfs中
    if (retval)
        goto bus_uevent_fail;

    //建立subsys_private中的devices和drivers的kset
    priv->devices_kset = kset_create_and_add("devices", NULL, bus_kobj);
    if (!priv->devices_kset) {
        retval = -ENOMEM;
        goto bus_devices_fail;
    }

    priv->drivers_kset = kset_create_and_add("drivers", NULL, bus_kobj);
    if (!priv->drivers_kset) {
        retval = -ENOMEM;
        goto bus_drivers_fail;
    }

    INIT_LIST_HEAD(&priv->interfaces);
    key = &priv->lock_key;
    lockdep_register_key(key);
    __mutex_init(&priv->mutex, "subsys mutex", key);
    //建立subsys_private中的devices和drivers链表，用于属于总线的设备和驱动
    klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
    klist_init(&priv->klist_drivers, NULL, NULL);

    retval = add_probe_files(bus);
    if (retval)
        goto bus_probe_files_fail;

    retval = sysfs_create_groups(bus_kobj, bus->bus_groups);
    if (retval)
        goto bus_groups_fail;

    pr_debug("bus: '%s': registered\n", bus->name);
    return 0;

bus_groups_fail:
    remove_probe_files(bus);
bus_probe_files_fail:
    kset_unregister(priv->drivers_kset);
bus_drivers_fail:
    kset_unregister(priv->devices_kset);
bus_devices_fail:
    bus_remove_file(bus, &bus_attr_uevent);
bus_uevent_fail:
    kset_unregister(&priv->subsys);
out:
    kfree(priv);
    return retval;
}
EXPORT_SYMBOL_GPL(bus_register);
```

```c
//https://elixir.bootlin.com/linux/v6.5.2/source/drivers/char/misc.c#L211
/**
 *  misc_register   -   register a miscellaneous device
 *  @misc: device structure
 *
 *  Register a miscellaneous device with the kernel. If the minor
 *  number is set to %MISC_DYNAMIC_MINOR a minor number is assigned
 *  and placed in the minor field of the structure. For other cases
 *  the minor number requested is used.
 *
 *  The structure passed is linked into the kernel and may not be
 *  destroyed until it has been unregistered. By default, an open()
 *  syscall to the device sets file->private_data to point to the
 *  structure. Drivers don't need open in fops for this.
 *
 *  A zero is returned on success and a negative errno code for
 *  failure.
 */

int misc_register(struct miscdevice *misc)
{
    dev_t dev;
    int err = 0;
    bool is_dynamic = (misc->minor == MISC_DYNAMIC_MINOR);

    INIT_LIST_HEAD(&misc->list);

    mutex_lock(&misc_mtx);

    if (is_dynamic) {//minor次设备号如果等于255就自动分配次设备
        int i = misc_minor_alloc();

        if (i < 0) {
            err = -EBUSY;
            goto out;
        }
        misc->minor = i;
    } else {//否则检查次设备号是否已经被占有
        struct miscdevice *c;

        list_for_each_entry(c, &misc_list, list) {
            if (c->minor == misc->minor) {
                err = -EBUSY;
                goto out;
            }
        }
    }

    dev = MKDEV(MISC_MAJOR, misc->minor); //合并主、次设备号

    misc->this_device =
        device_create_with_groups(&misc_class, misc->parent, dev,
                      misc, misc->groups, "%s", misc->name); //建立设备
    if (IS_ERR(misc->this_device)) {
        if (is_dynamic) {
            misc_minor_free(misc->minor);
            misc->minor = MISC_DYNAMIC_MINOR;
        }
        err = PTR_ERR(misc->this_device);
        goto out;
    }

    /*
     * Add it to the front, so that later devices can "override"
     * earlier defaults
     */
    list_add(&misc->list, &misc_list); //把这个misc加入到全局misc_list链表
 out:
    mutex_unlock(&misc_mtx);
    return err;
}
EXPORT_SYMBOL(misc_register);

//https://elixir.bootlin.com/linux/v6.5.2/source/drivers/base/core.c#L4372
/**
 * device_create_with_groups - creates a device and registers it with sysfs
 * @class: pointer to the struct class that this device should be registered to
 * @parent: pointer to the parent struct device of this new device, if any
 * @devt: the dev_t for the char device to be added
 * @drvdata: the data to be added to the device for callbacks
 * @groups: NULL-terminated list of attribute groups to be created
 * @fmt: string for the device's name
 *
 * This function can be used by char device classes.  A struct device
 * will be created in sysfs, registered to the specified class.
 * Additional attributes specified in the groups parameter will also
 * be created automatically.
 *
 * A "dev" file will be created, showing the dev_t for the device, if
 * the dev_t is not 0,0.
 * If a pointer to a parent struct device is passed in, the newly created
 * struct device will be a child of that device in sysfs.
 * The pointer to the struct device will be returned from the call.
 * Any further sysfs files that might be required can be created using this
 * pointer.
 *
 * Returns &struct device pointer on success, or ERR_PTR() on error.
 */
struct device *device_create_with_groups(const struct class *class,
                     struct device *parent, dev_t devt,
                     void *drvdata,
                     const struct attribute_group **groups,
                     const char *fmt, ...)
{
    va_list vargs;
    struct device *dev;

    va_start(vargs, fmt);
    dev = device_create_groups_vargs(class, parent, devt, drvdata, groups,
                     fmt, vargs);
    va_end(vargs);
    return dev;
}
EXPORT_SYMBOL_GPL(device_create_with_groups);

//https://elixir.bootlin.com/linux/v6.5.2/source/drivers/base/core.c#L4372
static __printf(6, 0) struct device *
device_create_groups_vargs(const struct class *class, struct device *parent,
               dev_t devt, void *drvdata,
               const struct attribute_group **groups,
               const char *fmt, va_list args)
{
    struct device *dev = NULL;
    int retval = -ENODEV;

    if (IS_ERR_OR_NULL(class))
        goto error;

    dev = kzalloc(sizeof(*dev), GFP_KERNEL);
    if (!dev) {
        retval = -ENOMEM;
        goto error;
    }

    device_initialize(dev); // 初始化sturct device
    dev->devt = devt; // 设置设备号
    dev->class = class; // 设置设备类
    dev->parent = parent; // 设置设备的父设备
    dev->groups = groups; // 设置设备属性
    dev->release = device_create_release;
    dev_set_drvdata(dev, drvdata); // 设置设备地址到device

    retval = kobject_set_name_vargs(&dev->kobj, fmt, args); //把名称设置到设备的kobject中去
    if (retval)
        goto error;

    retval = device_add(dev); //把设备加入到系统中
    if (retval)
        goto error;

    return dev;

error:
    put_device(dev);
    return ERR_PTR(retval);
}
```

misc_register 函数只是负责分配设备号，以及把 miscdev 加入链表，真正的核心工作由 device_create_with_groups 函数来完成.