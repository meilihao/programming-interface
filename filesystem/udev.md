# udev
ref:
- [udev](https://blog.zybz.fun/posts/linux/subsystem/udev/)
- [udev (简体中文)](https://wiki.archlinux.org/title/Udev_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

> hotplug(热插拔)狠推了一把udev.

> **当前libudev代码在systemd里**.

配置文件: /etc/udev/udev.conf
默认的udev规则在/etc/udev/rules.d下. udev规则文件的文件名通常以两个数字开头, 表明系统应用该规则的顺序.

udev规则文件的一行就是一条规则. 规则以kv形式表示, 由多个kv组成并以`,`分隔.

## /run/udev/data
`/run/udev/data`保存了device的udev info.


相关属性:
- ID_PART_ENTRY_SIZE : 磁盘分区的块数, 比如`/sys/block/sda/sda1/size`
- ID_PART_ENTRY_NUMBER: 分区序号, 比如sda1中的`1`. 非disk partition没有该属性
- ID_PART_ENTRY_DISK: parent device的`MAJ:MIN`


推测udev info由`systemd-250/src/udev/udevadm-info.c`输出

> `udevadm info -a /dev/sda`会输出sda设备在`sys`里的相关属性(会“遍历”父级设备链并整合信息)

## libudev
- udev_device_new_from_subsystem_sysname

	udev_device_new_from_subsystem_sysname(&device, "block", "sdc1")

	```c
	...
	name = strdupa_safe(sysname);
	for (size_t i = 0; name[i]; i++)
	      if (name[i] == '/')
	              name[i] = '!';

	FOREACH_STRING(s, "/sys/subsystem/", "/sys/bus/") {
	      r = device_strjoin_new(s, subsystem, "/devices/", name, ret);
	      if (r < 0)
	             return r;
	     if (r > 0)
	             return 0;
	}

	r = device_strjoin_new("/sys/class/", subsystem, "/", name, ret);
	if (r < 0)
	     return r;
	if (r > 0)
	     return 0;
	...
	```

	查找顺序(by strace):
	1. `/sys/subsystem/block/devices/sdc1`
	1. `/sys/bus/block/devices/sdc1`
	1. `/sys/class/block/sdc1`, 找打

		1. readlink /sys/class/block/sdc1得到`../../devices/pci0000:00/0000:00:13.0/ata2/host1/target1:0:0/1:0:0:0/block/sda/sdc1/`
		1. 读取uevent, 获取major=8, minor=33
		1. 解析`/run/udev/data/b8:33`