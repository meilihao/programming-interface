# udev
参考:
- [udev概述](https://huataihuang.gitbooks.io/cloud-atlas/content/os/linux/device/udev/udev_infrastructure.html)
- [udev](https://wiki.archlinux.org/index.php/Udev_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

热插拔使用udev. 冷插拔设备在开机就存在(即udev启动前), 因此它使用sysfs的uevent节点, 来发送netlink消息, 让后udev收到并处理该消息.

udev当前已与systemd合并. 嵌入式系统中, 由udev的轻量级版本mdev, 它集成在busybox中. Android使用vold, 机制与udev相同.


> 冷插拔: 在系统启动前就已连接好设备; 热插拔: 在运行过程中往系统插入设备.

> /proc/sys/kernel/hotplug已废弃, 由udevd管理热插拔的全部工作.

热插拔过程:
1. 插入设备后, 内核产生一个uevent, 可用`udevmonitor --env`查看
1. udevd通过netlink接收uevent, 并根据内核给的alias调用modprobe
1. modprobe在`/lib/modules/<version>/modules.alias`寻找匹配的模块, 然后inmod

## cmd
- `udevadm test /dev`

## devfs
它使得设备驱动程序能自主管理自己的设备文件, 后被udev取代.