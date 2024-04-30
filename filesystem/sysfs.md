# sysfs
sysfs是为设备服务的内存文件系统, 主要作用是在用户态展示设备信息.

> sys下不能创建和删除文件, 因为它本身不提供这些功能.

> `/dev`目录下的设备文件只是一个代表符号, 不包括设备相关的信息.

从3.10到5.05版内核,sysfs的使用变化不大,但内部实现却已经面目全非. 在3.10版内核中,sysfs的逻辑基本由其自行实现. 5.05版内核中,sysfs使用kernfs模块实现.

kernfs是一个通用的模块,协助其他模块实现文件系统,除了sysfs之外,resctrl和cgroup2文件系统也是借助它实现的.

sysfs mount调用kernfs_mount_ns实现. sysfs的文件支持的操作: kernfs_get_inode调用iget_locked获取新的inode,然后调用kernfs_init_inode为其赋值.

kernfs 将 它 的 文 件 分 为 三 类 : KERNFS_DIR 、KERNFS_FILE和KERNFS_LINK。请注意,这三类只是kernfs内部的划分,与文件系统的文件类型是不同的,但它们有对应关系,KERNFS_DIR对应目录,KERNFS_LINK对应symlink,KERNFS_FILE对应普通文件.

```c
// https://elixir.bootlin.com/linux/v6.6.28/source/fs/kernfs/inode.c#L199
static void kernfs_init_inode(struct kernfs_node *kn, struct inode *inode)
{
	kernfs_get(kn);
	inode->i_private = kn;
	inode->i_mapping->a_ops = &ram_aops;
	inode->i_op = &kernfs_iops;
	inode->i_generation = kernfs_gen(kn);

	set_default_inode_attr(inode, kn->mode);
	kernfs_refresh_inode(kn, inode);

	/* initialize inode according to type */
	switch (kernfs_type(kn)) {
	case KERNFS_DIR:
		inode->i_op = &kernfs_dir_iops;
		inode->i_fop = &kernfs_dir_fops;
		if (kn->flags & KERNFS_EMPTY_DIR)
			make_empty_dir_inode(inode);
		break;
	case KERNFS_FILE:
		inode->i_size = kn->attr.size;
		inode->i_fop = &kernfs_file_fops;
		break;
	case KERNFS_LINK:
		inode->i_op = &kernfs_symlink_iops;
		break;
	default:
		BUG();
	}

	unlock_new_inode(inode);
}
```

目录的inode的i_op操作为kernfs_dir_iops,它定义了lookup、mkdir和 rmdir等操作,没有定义create,mkdir操作需要文件系统定义kernfs_syscall_ops.mkdir,sysfs没有定义它,所以实际上mkdir也是不支持的。这说明sysfs文件系统是不支持用户空间直接创建文件的.

sysfs的文件层级结构是靠kernfs_node维护的 ,每一个文件,无论是否已经有inode与之对应,都对应一个kernfs_node对象kn,父与子的关系用红黑树表示,父目录是红黑树的根,子目录和文件是红黑树的结点.

fs角度看看sysfs:
- kobject: 目录
- kset: 若干kobject的集合, 也是一个目录
- attribute: KERNFS_FILE类型的文件
- bin_attribute: KERNFS_FILE类型的文件, binary

sysfs文件系统的每一个文件都对应一个kernfs_node对象,所以上面四个数据结构实际上都直接或间接地关联]kernfs_node.

> 定义两套数据结构的原因: 为kernfs_node只在文件系统内部使用,用来维护文件层级结构等,但它无法表示四类文件的特性,分
别定义结构体来表示它们,方便其他模块使用.

sysfs与Linux的设备驱动模型(LinuxDeviceDriver,LDD)完美地契合。LDD定义了device和device_attribute结构体,它们分别内嵌了kobject和attribute结构体;它同时还封装了sysfs,定义了一套额外的函数供驱动使用.



sysfs入口是[int __init sysfs_init(void)](https://elixir.bootlin.com/linux/v5.12.9/source/fs/sysfs/mount.c#L97).

sysfs使用[sysfs_create_dir](https://elixir.bootlin.com/linux/v5.12.9/source/fs/sysfs/dir.c#L40)创建目录文件. 它的入参包括一个[kobject](https://elixir.bootlin.com/linux/v5.12.9/source/include/linux/kobject.h#L64)指针, kobject与sysfs是紧密结合的.

## 创建文件
创建sysfs文件函数:
- kset_create_and_add : 在parent目录下创建name目录

	[`devices_kset = kset_create_and_add("devices", &device_uevent_ops, NULL)`](https://elixir.bootlin.com/linux/v6.6.29/source/drivers/base/core.c#L4096): /sys/devices, parent=NULL, devices目录直接出现在/sys下
- kset_unregister: 注销kset, 回收资源
- kobject_create_and_add(name, parent) : 在parent目录下创建name目录

	- [`dev_kobj = kobject_create_and_add("dev", NULL);`](https://elixir.bootlin.com/linux/v6.6.29/source/drivers/base/core.c#L4099): /sys/dev, parent=NULL, dev直接出现在/sys下
	- [`sysfs_dev_block_kobj = kobject_create_and_add("block", dev_kobj);`](https://elixir.bootlin.com/linux/v6.6.29/source/drivers/base/core.c#L4099): /sys/dev/block, parent=dev_kobj
- kobject_init(kobj, ktype)/kobject_add(kobj, parent, fmt, ...) : 在parent目录下创建name目录, 一般配对使用
- kobject_put: 注销kobject, 回收资源
- sysfs_create_file(kobj, attr): 在kobj目录下创建文件
- sysfs_remove_file(kobj, attr): 删除文件
- sysfs_create_bin_file(kobj, bin_attr): 在kobj目录下创建文件
- sysfs_remove_bin_file(kobj, bin_attr): 删除文件
- sysfs_create_link(kobj, target, name): 在kobj目录下创建符号链接文件, 将name链接到target
- sysfs_remove_link(kobj, name): 删除文件


以上函数创建文件无非是根据父目录、文件名和文件类型初始化一 个 kernfs_node对象的过程。但创建普通文件时
[sysfs_create_file->sysfs_create_file_ns->sysfs_add_file_mode_ns](https://elixir.bootlin.com/linux/v6.6.29/source/include/linux/sysfs.h#L620)和[sysfs_create_bin_file->sysfs_add_bin_file_mode_ns](https://elixir.bootlin.com/linux/v6.6.29/source/fs/sysfs/file.c#L560),这关系到文件的IO.

不同的kernfs_ops会有不同的文件IO过程.

> 在5.05上, sysfs_create_file和sysfs_create_bin_file最终都调用sysfs_add_file_mode_ns函数完成,主要区别在于传递给函数的is_bin参
数不同,前者等于false,后者等于true.

## 文件io
kernfs为普通文件提供的文件操作为kernfs_file_fops , 包括open、read、write和mmap等操作.

以write为例,写文件首先调用的是kernfs_file_fops.write,也就是kernfs_fop_write函数.

```c
// https://elixir.bootlin.com/linux/v6.6.29/source/fs/kernfs/file.c#L332
	ops = kernfs_ops(of->kn);
	if (ops->write)
		len = ops->write(of, buf, len, iocb->ki_pos);
```

ops->write有sysfs_kf_write和sysfs_kf_bin_write两种取值,分别对应attribute和bin_attribute.

对bin_attribute而言, 最终调用的就是为它定制的write.

对attribute而言,sysfs_ops(函数中的ops)的store回调函数触发了写文件操作. store的第二个参数是指向attribute对象的指针,也就是
kn->priv. 对KERNFS_DIR文件而言,kn->priv是指向kobject的指针; 对KERNFS_FILE文件而言,kn->priv指向attribute. store定义了写目录
所属的文件的方法(sysfs_ops是属于目录的).

```c
// https://elixir.bootlin.com/linux/v6.6.29/source/fs/sysfs/file.c#L202
/* kernfs write callback for regular sysfs files */
static ssize_t sysfs_kf_write(struct kernfs_open_file *of, char *buf,
			      size_t count, loff_t pos)
{
	const struct sysfs_ops *ops = sysfs_file_ops(of->kn);
	struct kobject *kobj = of->kn->parent->priv;

	if (!count)
		return 0;

	return ops->store(kobj, of->kn->priv, buf, count);
}

// https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/sysfs.c#L446
static ssize_t ext4_attr_store(struct kobject *kobj,
			       struct attribute *attr,
			       const char *buf, size_t len)
{
	...
}
```

如果为 bin_attribute定义了mmap操作,文件的kernfs_ops会被赋值为sysfs_bin_kfops_mmap,也就是说它是支持mmap
操作的, 而直接使用attribute创建的sysfs文件是没有mmap操作的.

设备驱动probe设备后, 会在sysfs创建对应的目录和文件, 也可以通过判断这些文件的状态来调试驱动.

## FAQ
### sysfs与ioctl
首先, 从实现的角度看, sysfs是一个文件系统,用户空间都是通过文件来与内核沟通的,添加一个功能, 需要新建一个文件; ioctl通过设备文件的回调函数实现,添加一个功能, 需要函数中多加一个分支(switch case). 从这个角度比较,二者各有优劣, ioctl在不断添加新功能的过程中, 可能导致函数复杂度过高而难以维护. sysfs将一个个功能分割开来, 彼此相对独立, 但如果添加的功能过多, 文件会变多, 也会对用户造成困扰.

其次, 文件是所见即所得的,ioctl则需要编写程序才能操作,比如需要查看设备的状态,如果该功能由sysfs实现,直接cat读文件就可以完成;如果选择的是ioctl,就需要编写程序了. 所以sysfs
在调试程序方面更加方便.

最后,因为sysfs的功能最终都要通过读写文件来完成,使用每个功能都需要查找文件,即执行open和close等操作; 而ioctl将功能统一到一个文件中,效率可能会更高.

最后,二者可以共存,可以根据某个功能的使用频率、访问方式来选择使用哪种方式.