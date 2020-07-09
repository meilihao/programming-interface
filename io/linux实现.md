# io
![io抽象层次](/misc/img/io/80e152fe768e3cb4c84be62ad8d6d07f.jpg)

## 用设备控制器屏蔽设备差异
计算机系统里, CPU 并不直接和设备打交道，而是通过设备控制器（Device Control Unit）的组件中转.

输入输出设备大致可以分为两类:
1. 块设备（Block Device） : 块设备将信息存储在固定大小的块中，每个块都有自己的地址. 硬盘就是常见的块设备.

    由于块设备传输的数据量比较大，控制器里往往会有缓冲区.

1. 字符设备（Character Device）: 字符设备发送或接收的是字节流, 而不用考虑任何块结构，没有办法寻址. 鼠标就是常见的字符设备.

CPU 同控制器的寄存器和数据缓冲区通信的方式: 
1. 每个控制寄存器被分配一个 I/O 端口，之后可以通过特殊的汇编指令（例如 in/out 类似的指令）操作这些寄存器
1. 数据缓冲区，可内存映射 I/O，可以分配一段内存空间给它，就像读写内存一样读写数据缓冲区. 如果去看内存空间的话，有一个区域 ioremap，就是做这个的.

对于 CPU 来讲，这些外部设备都有自己的大脑，可以自行处理一些事情，但是有个问题是，当cpu给设备发了一个指令，让它读取一些数据，它读完的时候，怎么通知cpu呢？控制器的寄存器一般会有状态标志位，可以通过检测状态标志位，来确定输入或者输出操作是否完成. 第一种方式就是轮询等待，就是一直查，一直查，直到完成. 当然这种方式很不好，于是有了第二种方式，就是可以通过中断的方式，通知操作系统输入输出操作已经完成.

为了响应中断，一般会有一个硬件的中断控制器，当设备完成任务后触发中断到中断控制器，中断控制器就通知 CPU，一个中断产生了，CPU 需要停下当前手里的事情来处理中断. 中断有两种，一种软中断，例如代码调用 INT 指令触发，一种是硬件中断，就是硬件通过中断控制器触发的.

有的设备需要读取或者写入大量数据. 如果所有过程都让 CPU 协调的话，就需要占用 CPU 大量的时间，比方说，磁盘就是这样的. 这种类型的设备需要支持 DMA 功能，也就是说，允许设备在 CPU 不参与的情况下，能够自行完成对内存的读写. 实现 DMA 机制需要有个 DMA 控制器帮 CPU 来做协调，就像下面这个图中显示的一样.

CPU 只需要对 DMA 控制器下指令，说它想读取多少数据，放在内存的某个地方就可以了，接下来 DMA 控制器会发指令给磁盘控制器，读取磁盘上的数据到指定的内存位置，传输完毕之后，DMA 控制器发中断通知 CPU 指令完成，CPU 就可以直接用内存里面现成的数据了. 内存的 DMA 区域，就是这个作用.

![](/misc/img/io/1ef05750bc9ff87a3330104802965335.jpeg)

## 用驱动程序屏蔽设备控制器差异
os用设备驱动程序来对接各个设备控制器.

设备控制器不属于操作系统的一部分，但是设备驱动程序属于操作系统的一部分. 操作系统的内核代码可以像调用本地代码一样调用驱动程序的代码，而驱动程序的代码需要发出特殊的面向设备控制器的指令，才能操作设备控制器. 设备驱动程序中是一些面向特殊设备控制器的代码, 不同的设备不同. 但是对于操作系统其它部分的代码而言，设备驱动程序应该有统一的接口.

设备做完了事情会通过中断来通知操作系统，而os有一个统一的流程来处理中断，使得不同设备的中断使用统一的流程. 一般的流程是，一个设备驱动程序初始化的时候，要先注册一个该设备的中断处理函数. 而中断的时候，触发的函数是 [handle_irq](https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/kernel/irq.c#L226), 这个函数是中断处理的统一入口. 在这个函数里面，可以找到设备驱动程序注册的中断处理函数 Handler，然后执行它进行中断处理.

![](/misc/img/io/aa9d074d9819f0eb513e11014a5772c0.jpg)

> 设备的中断处理函数在驱动中.

> do_IRQ deleted on 7c1d7cdcef1b54f4a78892b6b99d19f12c4f398e for "x86: unify do_IRQ() "

另外，对于块设备来讲，在驱动程序之上，文件系统之下，还需要一层通用设备层, 它里面的逻辑和磁盘设备没有什么关系，可以说是通用的逻辑. 在写文件的最底层，可看到了 BIO 字眼的函数，但是好像和设备驱动也没有什么关系. 是的，因为块设备类型非常多，而 Linux 操作系统里面一切是文件, 要不想文件系统以下，就直接对接各种各样的块设备驱动程序，这样会使得文件系统的复杂度非常高
, 通过在中间加了一层通用块层，将与块设备相关的通用逻辑放在这一层，维护与设备无关的块的大小，然后通用块层下面对接各种各样的驱动程序.

## 用文件系统接口屏蔽驱动程序的差异
上面从硬件设备到设备控制器，到驱动程序，到通用块层，到文件系统，层层屏蔽不同的设备的差别，最终到涉及对用户使用接口，也要统一. 虽然操作设备，都是基于文件系统的接口，也要有一个统一的标准.

首先要统一的是设备名称. 所有设备都在 /dev/ 文件夹下面创建一个特殊的设备文件. 这个设备特殊文件也有 inode，但是它不关联到硬盘或任何其他存储介质上的数据，而是建立了与某个设备驱动程序的连接. /dev/sdb 是一个设备文件, 这个文件本身和硬盘上的文件系统没有任何关系. 这个设备本身也不对应硬盘上的任何一个文件，/dev/sdb 其实是在一个特殊的文件系统 devtmpfs 中. 但是将 /dev/sdb 格式化成一个文件系统 ext4 的时候，就会将它 mount 到一个路径下面, 例如在 /mnt/sdb 下面. 这个时候 /dev/sdb 还是一个设备文件在特殊文件系统 devtmpfs 中，而 /mnt/sdb 下面的文件才是在 ext4 文件系统中，只不过这个文件系统是在 /dev/sdb 设备上的.

`ls -al /dev`, 如果是字符设备文件，则以 c 开头，如果是块设备文件，则以 b 开头. 其次是里面的两个号，一个是主设备号，一个是次设备号. 主设备号定位设备驱动程序，次设备号作为参数传给启动程序，选择相应的单元.

从ls列表可以看出来，mem、null、random、urandom、zero 都是用同样的主设备号 1，也就是它们使用同样的字符设备驱动，而 vda、vda1、vdb、vdc 也是同样的主设备号，也就是它们使用同样的块设备驱动. 有了设备文件，就可以使用对于文件的操作命令和 API 来操作设备了. 例如，`cat /dev/urandom | od -x`.

在 Linux 上面，如果一个新的设备从来没有加载过驱动，也需要安装驱动. Linux 的驱动程序已经被写成和操作系统有标准接口的代码，可以看成一个标准的内核模块. 在 Linux 里面，安装驱动程序，其实就是加载一个内核模块. 可以用命令 lsmod，查看有没有加载过相应的内核模块.

如果没有安装过相应的驱动，可以通过 insmod/modprobe 安装内核模块. 内核模块的后缀一般是 ko.

一旦有了驱动，就可以通过命令 `mknod filename type major minor`在 /dev 文件夹下面创建设备文件，其中 filename 就是 /dev 下面的设备名称，type 就是 c 为字符设备，b 为块设备，major 就是主设备号，minor 就是次设备号. 一旦执行了这个命令，新创建的设备文件就和上面加载过的驱动关联起来，这个时候就可以通过操作设备文件来操作驱动程序，从而操作设备.

自动识别设备需要另一个管理设备的文件系统，也就是 /sys 路径下面的 sysfs 文件系统, 它把实际连接到系统上的设备和总线组成了一个分层的文件系统. 这个文件系统是当前系统上实际的设备数的真实反映.

在 /sys 路径下有下列的文件夹：
- /sys/devices 是内核对系统中所有设备的分层次的表示
- /sys/dev 目录下一个 char 文件夹，一个 block 文件夹，分别维护一个按字符设备和块设备的主次号码 (major:minor) 链接到真实的设备 (/sys/devices 下) 的符号链接文件
- /sys/block 是系统中当前所有的块设备
- /sys/module 有系统中所有模块的信息

有了 sysfs 以后，还需要一个守护进程 udev. 当一个设备新插入系统的时候，内核会检测到这个设备，并会创建一个内核对象 kobject, 这个对象通过 sysfs 文件系统展现到用户层，同时内核还向用户空间发送一个热插拔消息. udevd 会监听这些消息，在 /dev 中创建对应的文件.

![](/misc/img/io/6234738aac8d5897449e1a541d557090.jpg)

有了文件系统接口之后，不但可以通过文件系统的命令行操作设备，也可以通过程序，调用 read、write 函数，像读写文件一样操作设备. 但是有些任务只使用读写很难完成，例如检查特定于设备的功能和属性，超出了通用文件系统的限制. 所以，对于设备来讲，还有一种接口称为 ioctl，表示输入输出控制接口，是用于配置和修改特定设备属性的通用接口.

# 字符设备
![](/misc/img/io/fba61fe95e0d2746235b1070eb4c18cd.jpeg)

罗技鼠标, 驱动代码在 [drivers/input/mouse/logibm.c](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/input/mouse/logibm.c).
打印机，驱动代码在 [drivers/char/lp.c](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/char/lp.c).

logibm.c 里面定义了 logibm_open, logibm_close 用于处理打开和关闭的，定义了 logibm_interrupt 用来响应中断的.

lp.c 里面定义了 [struct file_operations lp_fops](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/char/lp.c#L785)用于操作设备文件. 而在 logibm.c 里面，找不到这样的结构，是因为鼠标属于众多输入设备的一种，而输入设备的操作被统一定义在 [drivers/input/input.c](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/input/input.c) 里面就, 是[input_devices_proc_ops](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/input/input.c#L1220)，logibm.c 只是定义了一些自己独有的操作.

> drivers/input/input.c#input_devices_fileops deleted on 97a32539b9568bb653683349e5a76d02ff3c3e2c for `"proc: convert everything to "struct proc_ops"`

## 打开字符设备
![](/misc/img/io/2e29767e84b299324ea7fc524a3dcee6.jpeg)

要使用一个字符设备，首先要把它的内核模块，通过 insmod 加载进内核. 这个时候，先调用的就是 module_init 调用的初始化函数.

[lp_init_module](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/char/lp.c#L1080) -> [lp_init](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/char/lp.c#L1019) -> [register_chrdev](https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/fs.h#L2690) -> [__register_chrdev](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/char_dev.c#L268)->[cdev_add](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/char_dev.c#L479)

字符设备驱动的内核模块加载的时候，最重要的一件事情就是，注册这个字符设备. 注册的方式是调用 __register_chrdev_region，注册字符设备的主次设备号和名称，然后分配一个 struct cdev 结构，将 cdev 的 ops 成员变量指向这个模块声明的 file_operations. 然后，cdev_add 会将这个字符设备添加到内核中一个叫作 struct kobj_map *cdev_map 的结构，来统一管理所有字符设备. 

其中，MKDEV(cd->major, baseminor) 表示将主设备号和次设备号生成一个 dev_t 的整数，然后将这个整数 dev_t 和 cdev 关联起来.

在 logibm.c 的 logibm_init 找不到注册字符设备，这是因为logibm.c是通过 input.c 注册的, 这就相当于 input.c 对多个输入字符设备进行统一的管理. 调用链是: 在 logibm_init 中调用 [input_register_device](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/input/input.c#L2153) 加入到input.c的[input_dev_list](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/input/input.c#L37)->[input_attach_handler](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/input/input.c#L1022)-> `connect()` 即[evdev_connect](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/input/evdev.c#L1337) -> [input_register_handle](https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/input/input.c#L2378).
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/input/input.c#L37
	list_for_each_entry(handler, &input_handler_list, node)
		input_attach_handler(dev, handler);
```

在内核启动的时候，input handler是已经注册了的，然后logibm.c的input_dev注册进来后遍历input_handler_list by list_for_each_entry()查找有没有一个合适的handler.

> input_dev和input_handler是一个多对一的关系.

内核模块加载完毕后，接下来要通过 mknod 在 /dev 下面创建一个设备文件，只有有了这个设备文件，才能通过文件系统的接口，对这个设备文件进行操作.
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3611
SYSCALL_DEFINE4(mknodat, int, dfd, const char __user *, filename, umode_t, mode,
		unsigned int, dev)
{
	return do_mknodat(dfd, filename, mode, dev);
}

SYSCALL_DEFINE3(mknod, const char __user *, filename, umode_t, mode, unsigned, dev)
{
	return do_mknodat(AT_FDCWD, filename, mode, dev);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3567
long do_mknodat(int dfd, const char __user *filename, umode_t mode,
		unsigned int dev)
{
	struct dentry *dentry;
	struct path path;
	int error;
	unsigned int lookup_flags = 0;

	error = may_mknod(mode);
	if (error)
		return error;
retry:
	dentry = user_path_create(dfd, filename, &path, lookup_flags);
	if (IS_ERR(dentry))
		return PTR_ERR(dentry);

	if (!IS_POSIXACL(path.dentry->d_inode))
		mode &= ~current_umask();
	error = security_path_mknod(&path, dentry, mode, dev);
	if (error)
		goto out;
	switch (mode & S_IFMT) {
		case 0: case S_IFREG:
			error = vfs_create(path.dentry->d_inode,dentry,mode,true);
			if (!error)
				ima_post_path_mknod(dentry);
			break;
		case S_IFCHR: case S_IFBLK:
			error = vfs_mknod(path.dentry->d_inode,dentry,mode,
					new_decode_dev(dev));
			break;
		case S_IFIFO: case S_IFSOCK:
			error = vfs_mknod(path.dentry->d_inode,dentry,mode,0);
			break;
	}
out:
	done_path_create(&path, dentry);
	if (retry_estale(error, lookup_flags)) {
		lookup_flags |= LOOKUP_REVAL;
		goto retry;
	}
	return error;
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3520
int vfs_mknod(struct inode *dir, struct dentry *dentry, umode_t mode, dev_t dev)
{
	bool is_whiteout = S_ISCHR(mode) && dev == WHITEOUT_DEV;
	int error = may_create(dir, dentry);

	if (error)
		return error;

	if ((S_ISCHR(mode) || S_ISBLK(mode)) && !is_whiteout &&
	    !capable(CAP_MKNOD))
		return -EPERM;

	if (!dir->i_op->mknod)
		return -EPERM;

	error = devcgroup_inode_mknod(mode, dev);
	if (error)
		return error;

	error = security_inode_mknod(dir, dentry, mode, dev);
	if (error)
		return error;

	error = dir->i_op->mknod(dir, dentry, mode, dev);
	if (!error)
		fsnotify_create(dir, dentry);
	return error;
}
EXPORT_SYMBOL(vfs_mknod);
```

在do_mkdirat里看到，在文件系统上，顺着路径找到 /dev/xxx 所在的文件夹，然后为这个新创建的设备文件创建一个 dentry. 这是维护文件和 inode 之间的关联关系的结构.

接下来，如果是字符文件 S_IFCHR 或者设备文件 S_IFBLK，就调用 vfs_mkno -> 调用对应的文件系统的 inode_operations.

通过`sudo mount |grep "/dev"`发现`/dev`挂载的是devtmpfs.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/drivers/base/devtmpfs.c#L66
static struct file_system_type internal_fs_type = {
	.name = "devtmpfs",
#ifdef CONFIG_TMPFS
	.init_fs_context = shmem_init_fs_context,
	.parameters	= shmem_fs_parameters,
#else
	.init_fs_context = ramfs_init_fs_context,
	.parameters	= ramfs_fs_parameters,
#endif
	.kill_sb = kill_litter_super,
};

// https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/shmem.c#L3774
static const struct inode_operations shmem_dir_inode_operations = {
#ifdef CONFIG_TMPFS
	.create		= shmem_create,
	.lookup		= simple_lookup,
	.link		= shmem_link,
	.unlink		= shmem_unlink,
	.symlink	= shmem_symlink,
	.mkdir		= shmem_mkdir,
	.rmdir		= shmem_rmdir,
	.mknod		= shmem_mknod,
	.rename		= shmem_rename2,
	.tmpfile	= shmem_tmpfile,
#endif
#ifdef CONFIG_TMPFS_XATTR
	.listxattr	= shmem_listxattr,
#endif
#ifdef CONFIG_TMPFS_POSIX_ACL
	.setattr	= shmem_setattr,
	.set_acl	= simple_set_acl,
#endif
};

// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/ramfs/inode.c#L150

static const struct inode_operations ramfs_dir_inode_operations = {
	.create		= ramfs_create,
	.lookup		= simple_lookup,
	.link		= simple_link,
	.unlink		= simple_unlink,
	.symlink	= ramfs_symlink,
	.mkdir		= ramfs_mkdir,
	.rmdir		= simple_rmdir,
	.mknod		= ramfs_mknod,
	.rename		= simple_rename,
};
```

这里可以看出，devtmpfs 在挂载的时候，有两种模式，一种是 ramfs，一种是 shmem 都是基于内存的文件系统.

> ramfs是Linux下一种基于RAM做存储的文件系统. 由于ramfs的实现就相当于把RAM作为最后一层的存储，所以在ramfs中不会使用swap.
> shmem也是Linux下的一个文件系统，它将所有的文件都保存在虚拟内存中，umount tmpfs后所有的数据也会丢失，tmpfs就是ramfs的衍生品. tmpfs使用了虚拟内存的机制，它会进行swap，但是它有一个相比ramfs的好处：mount时指定的size参数是起作用的，这样就能保证系统的安全，而不是像ramfs那样，一不留心因为写入数据太大吃光系统所有内存导致系统被hang住.

这两个 mknod 虽然实现不同，但是都会调用到同一个函数 [init_special_inode](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/inode.c#L2110).

显然这个文件是个特殊文件，inode 也是特殊的. 这里这个 inode 可以关联字符设备、块设备、FIFO 文件、Socket.

这里的 inode 的 file_operations 指向一个 def_chr_fops.

另外，inode 的 i_rdev 指向这个设备的 dev_t. 通过这个 dev_t，可以找到之前刚加载的字符设备 cdev.

到目前为止，只是创建了 /dev 下面的一个文件，并且和相应的设备号关联起来. 但是，还没有打开这个 /dev 下面的设备文件.

打开文件的进程的 task_struct 里，有一个数组代表它打开的文件，下标就是文件描述符 fd，每一个打开的文件都有一个 struct file 结构，会指向一个 dentry 项. dentry 可以用来关联 inode. 这个 dentry 就是上面 mknod 的时候创建的. 在进程里面调用 open 函数，最终会调用到这个特殊的 inode 的 open 函数，也就是 [chrdev_open](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/char_dev.c#L373).

在chrdev_open里面，首先看这个 inode 的 i_cdev，是否已经关联到 cdev. 如果第一次打开，当然没有. 没有没关系，inode 里面有 i_rdev 呀，也就是有 dev_t. 可以通过它在 cdev_map 中找 cdev. 找到后就将 inode 的 i_cdev，关联到找到的 cdev new.

找到 cdev 就好办了. cdev 里面有 file_operations，这是设备驱动程序自己定义的, 可以通过它来操作设备驱动程序，把它付给 struct file 里面的 file_operations. 这样以后操作文件描述符，就是直接操作设备了. 最后，需要调用设备驱动程序的 file_operations 的 open 函数，真正打开设备. 对于打印机，调用的是 lp_open. 对于鼠标调用的是 input_proc_devices_open，最终会调用到 logibm_open.

## 写入设备
![打印机驱动写入的过程](/misc/img/fs/9bd3cd8a8705dbf69f889ba3b2b5c2e2.jpeg)

写入一个字符设备，就是用文件系统的标准接口 write，参数文件描述符 fd，在内核里面调用的 sys_write，在 sys_write 里面根据文件描述符 fd 得到 struct file 结构, 接下来再调用 vfs_write.

在 __vfs_write 里面，会调用 struct file 结构里的 file_operations 的 write 函数. 之前打开字符设备的时候，已经将 struct file 结构里面的 file_operations 指向了设备驱动程序的 file_operations 结构，所以这里的 write 函数最终会调用到 lp_write.

这个设备驱动程序的写入函数的实现还是比较典型的, 先是调用 copy_from_user 将数据从用户态拷贝到内核态的缓存中，然后调用 parport_write 写入外部设备. 这里还有一个 schedule 函数，也即写入的过程中，给其他线程抢占 CPU 的机会. 然后，如果 count 还是大于 0，也就是数据还没有写完，那就接着 copy_from_user，接着 parport_write，直到写完为止.

## 使用 IOCTL 控制设备
调用 ioctl 可执行一些特殊的 I/O 操作.
![](/misc/img/fs/c3498dad4f15712529354e0fa123c31d.jpeg)

[ioctl](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/ioctl.c#L760)也是一个系统调用.

其中，fd 是这个设备的文件描述符，cmd 是传给这个设备的命令，arg 是命令的参数. 其中，对于命令和命令的参数，使用 ioctl 系统调用的用户和驱动程序的开发人员约定好行为即可.

其实 cmd 看起来是一个 int，其实他的组成比较复杂，它由几部分组成：
1. 最低八位为 NR，是命令号
1. 然后八位是 TYPE，是类型
1. 然后十四位是参数的大小；
1. 最高两位是 DIR，是方向，表示写入、读出，还是读写

由于组成比较复杂，[有一些宏](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/uapi/asm-generic/ioctl.h#L85)是专门用于组成这个 cmd 值的.

在用户程序中，可以通过[上面的“Used to create numbers”这些宏](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/uapi/asm-generic/ioctl.h#L85)，根据参数生成 cmd，在驱动程序中，可以通过[下面的“used to decode ioctl numbers”这些宏](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/uapi/asm-generic/ioctl.h#L94)，解析 cmd 后，执行指令.

ioctl 中会调用 [do_vfs_ioctl](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/ioctl.c#L667)，这里面对于已经定义好的 cmd，进行相应的处理. 如果不是默认定义好的 cmd，则执行默认操作. 对于普通文件，调用 file_ioctl；对于其他文件调用 vfs_ioctl.

由于这里是设备驱动程序，所以调用的是 vfs_ioctl.

这里面调用的是 struct file 里 file_operations 的 unlocked_ioctl 函数. 之前初始化设备驱动的时候，已经将 file_operations 指向设备驱动的 file_operations 了, 这里调用的是设备驱动的 unlocked_ioctl. 对于打印机程序来讲，调用的是 lp_ioctl. 可以看出来，它里面就是 switch 语句，它会根据不同的 cmd，做不同的操作.