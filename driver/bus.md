# bus
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