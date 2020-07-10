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

设备做完了事情会通过中断来通知操作系统，而os有一个统一的流程来处理中断，使得不同设备的中断使用统一的流程. 一般的流程是，一个设备驱动程序初始化的时候，要先注册一个该设备的中断处理函数. 而中断的时候，触发的函数是 [handle_irq](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/irq.c#L226), 这个函数是中断处理的统一入口. 在这个函数里面，可以找到设备驱动程序注册的中断处理函数 Handler，然后执行它进行中断处理.

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

罗技鼠标, 驱动代码在 [drivers/input/mouse/logibm.c](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/input/mouse/logibm.c).
打印机，驱动代码在 [drivers/char/lp.c](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/char/lp.c).

logibm.c 里面定义了 logibm_open, logibm_close 用于处理打开和关闭的，定义了 logibm_interrupt 用来响应中断的.

lp.c 里面定义了 [struct file_operations lp_fops](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/char/lp.c#L785)用于操作设备文件. 而在 logibm.c 里面，找不到这样的结构，是因为鼠标属于众多输入设备的一种，而输入设备的操作被统一定义在 [drivers/input/input.c](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/input/input.c) 里面就, 是[input_devices_proc_ops](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/input/input.c#L1220)，logibm.c 只是定义了一些自己独有的操作.

> drivers/input/input.c#input_devices_fileops deleted on 97a32539b9568bb653683349e5a76d02ff3c3e2c for `"proc: convert everything to "struct proc_ops"`

## 打开字符设备
![](/misc/img/io/2e29767e84b299324ea7fc524a3dcee6.jpeg)

要使用一个字符设备，首先要把它的内核模块，通过 insmod 加载进内核. 这个时候，先调用的就是 module_init 调用的初始化函数.

[lp_init_module](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/char/lp.c#L1080) -> [lp_init](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/char/lp.c#L1019) -> [register_chrdev](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/fs.h#L2690) -> [__register_chrdev](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/char_dev.c#L268)->[cdev_add](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/char_dev.c#L479)

字符设备驱动的内核模块加载的时候，最重要的一件事情就是，注册这个字符设备. 注册的方式是调用 __register_chrdev_region，注册字符设备的主次设备号和名称，然后分配一个 struct cdev 结构，将 cdev 的 ops 成员变量指向这个模块声明的 file_operations. 然后，cdev_add 会将这个字符设备添加到内核中一个叫作 struct kobj_map *cdev_map 的结构，来统一管理所有字符设备. 

其中，MKDEV(cd->major, baseminor) 表示将主设备号和次设备号生成一个 dev_t 的整数，然后将这个整数 dev_t 和 cdev 关联起来.

在 logibm.c 的 logibm_init 找不到注册字符设备，这是因为logibm.c是通过 input.c 注册的, 这就相当于 input.c 对多个输入字符设备进行统一的管理. 调用链是: 在 logibm_init 中调用 [input_register_device](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/input/input.c#L2153) 加入到input.c的[input_dev_list](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/input/input.c#L37)->[input_attach_handler](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/input/input.c#L1022)-> `connect()` 即[evdev_connect](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/input/evdev.c#L1337) -> [input_register_handle](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/input/input.c#L2378).
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/input/input.c#L37
	list_for_each_entry(handler, &input_handler_list, node)
		input_attach_handler(dev, handler);
```

在内核启动的时候，input handler是已经注册了的，然后logibm.c的input_dev注册进来后遍历input_handler_list by list_for_each_entry()查找有没有一个合适的handler.

> input_dev和input_handler是一个多对一的关系.

内核模块加载完毕后，接下来要通过 mknod 在 /dev 下面创建一个设备文件，只有有了这个设备文件，才能通过文件系统的接口，对这个设备文件进行操作.
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/namei.c#L3611
SYSCALL_DEFINE4(mknodat, int, dfd, const char __user *, filename, umode_t, mode,
		unsigned int, dev)
{
	return do_mknodat(dfd, filename, mode, dev);
}

SYSCALL_DEFINE3(mknod, const char __user *, filename, umode_t, mode, unsigned, dev)
{
	return do_mknodat(AT_FDCWD, filename, mode, dev);
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/namei.c#L3567
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

// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/namei.c#L3520
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
// https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/base/devtmpfs.c#L66
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

## 中断通知
鼠标就是通过中断，将自己的位置和按键信息，传递给设备驱动程序.
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/input/mouse/logibm.c#L61
static irqreturn_t logibm_interrupt(int irq, void *dev_id)
{
	char dx, dy;
	unsigned char buttons;

	outb(LOGIBM_READ_X_LOW, LOGIBM_CONTROL_PORT);
	dx = (inb(LOGIBM_DATA_PORT) & 0xf);
	outb(LOGIBM_READ_X_HIGH, LOGIBM_CONTROL_PORT);
	dx |= (inb(LOGIBM_DATA_PORT) & 0xf) << 4;
	outb(LOGIBM_READ_Y_LOW, LOGIBM_CONTROL_PORT);
	dy = (inb(LOGIBM_DATA_PORT) & 0xf);
	outb(LOGIBM_READ_Y_HIGH, LOGIBM_CONTROL_PORT);
	buttons = inb(LOGIBM_DATA_PORT);
	dy |= (buttons & 0xf) << 4;
	buttons = ~buttons >> 5;

	input_report_rel(logibm_dev, REL_X, dx);
	input_report_rel(logibm_dev, REL_Y, dy);
	input_report_key(logibm_dev, BTN_RIGHT,  buttons & 1);
	input_report_key(logibm_dev, BTN_MIDDLE, buttons & 2);
	input_report_key(logibm_dev, BTN_LEFT,   buttons & 4);
	input_sync(logibm_dev);

	outb(LOGIBM_ENABLE_IRQ, LOGIBM_CONTROL_PORT);
	return IRQ_HANDLED;
}

static int logibm_open(struct input_dev *dev)
{
	if (request_irq(logibm_irq, logibm_interrupt, 0, "logibm", NULL)) {
		printk(KERN_ERR "logibm.c: Can't allocate irq %d\n", logibm_irq);
		return -EBUSY;
	}
	outb(LOGIBM_ENABLE_IRQ, LOGIBM_CONTROL_PORT);
	return 0;
}
```

要处理中断，需要有一个中断处理函数. 定义如下：
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/interrupt.h#L92
typedef irqreturn_t (*irq_handler_t)(int, void *);

// https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/irqreturn.h#L9:4
/**
 * enum irqreturn
 * @IRQ_NONE		interrupt was not from this device or was not handled
 * @IRQ_HANDLED		interrupt was handled by this device
 * @IRQ_WAKE_THREAD	handler requests to wake the handler thread
 */
enum irqreturn {
	IRQ_NONE		= (0 << 0),
	IRQ_HANDLED		= (1 << 0),
	IRQ_WAKE_THREAD		= (1 << 1),
};
```

其中，irq 是一个整数，是中断信号. dev_id 是一个 void * 的通用指针，主要用于区分同一个中断处理函数对于不同设备的处理.

这里的返回值有三种：
- IRQ_NONE 表示不是我的中断，不归我管
- IRQ_HANDLED 表示处理完了的中断
- IRQ_WAKE_THREAD 表示有一个进程正在等待这个中断，中断处理完了，应该唤醒它

上面的例子中，logibm_interrupt 这个中断处理函数，先是获取了 x 和 y 的移动坐标，以及左中右的按键，上报上去，然后返回 IRQ_HANDLED，这表示处理完毕.

其实，写一个真正生产用的中断处理程序还是很复杂的. 当一个中断信号 A 触发后，正在处理的过程中，这个中断信号 A 是应该暂时关闭的，这样是为了防止再来一个中断信号 A，在当前的中断信号 A 的处理过程中插一杠子. 但是，这个暂时关闭的时间应该多长呢？

如果太短了，应该原子化处理完毕的没有处理完毕，又被另一个中断信号 A 中断了，很多操作就不正确了；如果太长了，一直关闭着，新的中断信号 A 进不来，系统就显得很慢. 所以，很多中断处理程序将整个中断要做的事情分成两部分，称为上半部和下半部，或者成为关键处理部分和延迟处理部分. 在中断处理函数中，仅仅处理关键部分，完成了就将中断信号打开，使得新的中断可以进来，需要比较长时间处理的部分，也即延迟部分，往往通过工作队列等方式慢慢处理. 可参考《Linux Device Drivers》这本书. 有了中断处理函数，接下来要调用 request_irq 来注册这个中断处理函数. [request_irq](https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/interrupt.h#L157) 有这样几个参数：
- unsigned int irq 是中断信号
- irq_handler_t handler 是中断处理函数
- unsigned long flags 是一些标识位
- const char *name 是设备名称
- void *dev 这个通用指针应该和中断处理函数的 void *dev 相对应.

[request_irq](https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/interrupt.h#L157) -> [request_threaded_irq](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/manage.c#L1969)

对于每一个中断，都有一个对中断的描述结构 [struct irq_desc](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/irqdesc.h#L56). 它有一个重要的成员变量是 [struct irqaction](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/interrupt.h#L110)，用于表示处理这个中断的动作. 如果仔细看这个结构，会发现，它里面有 next 指针，也就是说，这是一个链表，对于这个中断的所有处理动作，都串在这个链表上.

每一个中断处理动作的结构 struct irqaction，都有以下成员：
1. 中断处理函数 handler
1. void *dev_id 为设备 id
1. irq 为中断信号
1. 如果中断处理函数在单独的线程运行，则有 thread_fn 是线程的执行函数，thread 是线程的 task_struct.

在 request_threaded_irq 函数中，irq_to_desc 根据中断信号查找中断描述结构. 一般情况下，所有的 [struct irq_desc](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/irqdesc.h#L56) 都放在一个[数组](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/irqdesc.c#L550)里面，直接按下标查找就可以了. 如果配置了 CONFIG_SPARSE_IRQ，那中断号是不连续的，就不适合用数组保存了，此时可以放在[一棵基数树](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/irqdesc.c#L344)上. 这种结构对于从某个整型 key 找到 value 速度很快，中断信号 irq 是这个整数. 通过它，很快就能定位到对应的 struct irq_desc.

为什么中断信号会有稀疏，也就是不连续的情况呢？这里需要说明一下，这里的 irq 并不是真正的、物理的中断信号，而是一个抽象的、虚拟的中断信号. 因为物理的中断信号和硬件关联比较大，中断控制器也是各种各样的. 作为内核，不可能写程序的时候，适配各种各样的硬件中断控制器，因而就需要有一层中断抽象层. 这里虚拟中断信号到中断描述结构的映射，就是抽象中断层的主要逻辑.

下面介绍真正中断响应的时候，会涉及物理中断信号. 可以想象，如果只有一个 CPU，一个中断控制器，则基本能够保证从物理中断信号到虚拟中断信号的映射是线性的，这样用数组表示就没啥问题，但是如果有多个 CPU，多个中断控制器，每个中断控制器各有各的物理中断信号，就没办法保证虚拟中断信号是连续的，所以就要用到基数树了.

接下来，request_threaded_irq 函数分配了一个 struct irqaction，并且初始化它，接着调用 [__setup_irq](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/manage.c#L1326). 在这个函数里面，如果 struct irq_desc 里面已经有 struct irqaction 了，就将新的 struct irqaction 挂在链表的末端. 如果设定了以单独的线程运行中断处理函数，[setup_irq_thread](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/manage.c#L1271) 就会创建这个内核线程，[wake_up_process](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/manage.c#L1645) 会唤醒它.

至此为止，request_irq 完成了它的使命. 总结来说，它就是根据中断信号 irq，找到基数树上对应的 irq_desc，然后将新的 irqaction 挂在链表上.

真正中断的发生还是要从硬件开始, 这里面有四个层次:
1. 第一个层次是外部设备给中断控制器发送物理中断信号
1. 第二个层次是中断控制器将物理中断信号转换成为中断向量 interrupt vector，发给各个 CPU
1. 第三个层次是每个 CPU 都会有一个中断向量表，根据 interrupt vector 调用一个 IRQ 处理函数. 注意这里的 IRQ 处理函数还不是上面指定的 irq_handler_t，到这一层还是 CPU 硬件的要求
1. 第四个层次是在 IRQ 处理函数中，将 interrupt vector 转化为抽象中断层的中断信号 irq，调用中断信号 irq 对应的中断描述结构里面的 irq_handler_t

![](/misc/img/io/dd492efdcf956cb22ce3d51592cdc113.png)

从 CPU 收到中断向量开始分析, x86 CPU 收到的中断向量定义在文件 [arch/x86/include/asm/irq_vectors.h](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/irq_vectors.h)中.

通过这些注释，可以看出，CPU 能够处理的中断总共 256 个，用宏 NR_VECTOR 或者 FIRST_SYSTEM_VECTOR 表示.

为了处理中断，CPU 硬件要求每一个 CPU 都有一个中断向量表，通过 [load_idt](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/desc.h#L111) 加载，里面记录着每一个中断对应的处理方法，这个中断向量表定义在文件 [arch/x86/kernel/idt.c](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/idt.c#L161) 中.

对于一个 CPU 可以处理的中断被分为几个部分，[第一部分 0 到 31 的前 32 位](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/irq_vectors.h#L18)是系统陷入或者系统异常，这些错误无法屏蔽，一定要处理.

这些中断的处理函数在系统初始化的时候，在 start_kernel 函数中调用 [trap_init()](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/traps.c#L1079)来设置.

在 trap_init 中 会通过[idt_setup_traps](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/idt.c#L241) 最终调用[idt_setup_from_table](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/idt.c#L196) 其实就是将每个中断都设置了中断处理函数，放在中断向量表 idt_table 中.

> idt_table的 前 32 个中断定义在 [arch/x86/include/asm/trapnr.h](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/trapnr.h) 文件中.
> trap_init() 原先会调用大量的 set_intr_gate, 后被移入[idt_setup_traps](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/idt.c#L241) on b70543a0b2b680f8953b6917a83b9203b20d7abd for "x86/idt: Move regular trap init to tables"

trap_init 结束后，中断向量表中已经填好了前 32 位(不一定每位都设置)，外加一位 32 位系统调用IA32_SYSCALL_VECTOR，其他的都是用于设备中断.

在 start_kernel 调用完毕 trap_init 之后，还会调用 [init_IRQ()的`x86_init.irqs.intr_init()`](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/irqinit.c#L70) 来初始化其他的设备中断，最终会调用到 [native_init_IRQ](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/irqinit.c#L90).

这样任何一个中断向量到达任何一个 CPU，最终都会走到 [common_interrupt](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/irq.c#L239).

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/idtentry.h#L196
#define DEFINE_IDTENTRY_IRQ(func)					\
static __always_inline void __##func(struct pt_regs *regs, u8 vector);	\
									\
__visible noinstr void func(struct pt_regs *regs,			\
			    unsigned long error_code)			\
{									\
	bool rcu_exit = idtentry_enter_cond_rcu(regs);			\
									\
	instrumentation_begin();					\
	irq_enter_rcu();						\
	kvm_set_cpu_l1tf_flush_l1d();					\
	__##func (regs, (u8)error_code);				\
	irq_exit_rcu();							\
	instrumentation_end();						\
	idtentry_exit_cond_rcu(regs, rcu_exit);				\
}									\
									\
static __always_inline void __##func(struct pt_regs *regs, u8 vector)

// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/idtentry.h#L420
/* Entries for common/spurious (device) interrupts */
#define DECLARE_IDTENTRY_IRQ(vector, func)				\
	idtentry_irq vector func

// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/idtentry.h#L420
/* Device interrupts common/spurious */
DECLARE_IDTENTRY_IRQ(X86_TRAP_OTHER,	common_interrupt);

// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/irq.c#L239
/*
 * common_interrupt() handles all normal device IRQ's (the special SMP
 * cross-CPU interrupts have their own entry points).
 */
DEFINE_IDTENTRY_IRQ(common_interrupt)
{
	struct pt_regs *old_regs = set_irq_regs(regs);
	struct irq_desc *desc;

	/* entry code tells RCU that we're not quiescent.  Check it. */
	RCU_LOCKDEP_WARN(!rcu_is_watching(), "IRQ failed to wake up RCU");

	desc = __this_cpu_read(vector_irq[vector]); // ???vector is where from
	if (likely(!IS_ERR_OR_NULL(desc))) {
		handle_irq(desc, regs);
	} else {
		ack_APIC_irq();

		if (desc == VECTOR_UNUSED) {
			pr_emerg_ratelimited("%s: %d.%u No irq handler for vector\n",
					     __func__, smp_processor_id(),
					     vector);
		} else {
			__this_cpu_write(vector_irq[vector], VECTOR_UNUSED);
		}
	}

	set_irq_regs(old_regs);
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/entry/entry_64.S
/*
 * Interrupt entry/exit.
 *
 + The interrupt stubs push (vector) onto the stack, which is the error_code
 * position of idtentry exceptions, and jump to one of the two idtentry points
 * (common/spurious).
 *
 * common_interrupt is a hotpath, align it to a cache line
 */
.macro idtentry_irq vector cfunc
	.p2align CONFIG_X86_L1_CACHE_SHIFT
	idtentry \vector asm_\cfunc \cfunc has_error_code=1
.endm

// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/entry/entry_64.S#L348
/**
 * idtentry - Macro to generate entry stubs for simple IDT entries
 * @vector:		Vector number
 * @asmsym:		ASM symbol for the entry point
 * @cfunc:		C function to be called
 * @has_error_code:	Hardware pushed error code on stack
 *
 * The macro emits code to set up the kernel context for straight forward
 * and simple IDT entries. No IST stack, no paranoid entry checks.
 */
.macro idtentry vector asmsym cfunc has_error_code:req
SYM_CODE_START(\asmsym)
	UNWIND_HINT_IRET_REGS offset=\has_error_code*8
	ASM_CLAC

	.if \has_error_code == 0
		pushq	$-1			/* ORIG_RAX: no syscall to restart */
	.endif

	.if \vector == X86_TRAP_BP
		/*
		 * If coming from kernel space, create a 6-word gap to allow the
		 * int3 handler to emulate a call instruction.
		 */
		testb	$3, CS-ORIG_RAX(%rsp)
		jnz	.Lfrom_usermode_no_gap_\@
		.rept	6
		pushq	5*8(%rsp)
		.endr
		UNWIND_HINT_IRET_REGS offset=8
.Lfrom_usermode_no_gap_\@:
	.endif

	idtentry_body \cfunc \has_error_code

_ASM_NOKPROBE(\asmsym)
SYM_CODE_END(\asmsym)
.endm

// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/entry/entry_64.S#L321
/**
 * idtentry_body - Macro to emit code calling the C function
 * @cfunc:		C function to be called
 * @has_error_code:	Hardware pushed error code on stack
 */
.macro idtentry_body cfunc has_error_code:req

	call	error_entry
	UNWIND_HINT_REGS

	movq	%rsp, %rdi			/* pt_regs pointer into 1st argument*/

	.if \has_error_code == 1
		movq	ORIG_RAX(%rsp), %rsi	/* get error code into 2nd argument*/
		movq	$-1, ORIG_RAX(%rsp)	/* no syscall to restart */
	.endif

	call	\cfunc

	jmp	error_return
.endm
```

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/apic/vector.c#L732
static struct irq_desc *__setup_vector_irq(int vector)
{
	int isairq = vector - ISA_IRQ_VECTOR(0);

	/* Check whether the irq is in the legacy space */
	if (isairq < 0 || isairq >= nr_legacy_irqs())
		return VECTOR_UNUSED;
	/* Check whether the irq is handled by the IOAPIC */
	if (test_bit(isairq, &io_apic_irqs))
		return VECTOR_UNUSED;
	return irq_to_desc(isairq);
}

/* Online the local APIC infrastructure and initialize the vectors */
void lapic_online(void)
{
	unsigned int vector;

	lockdep_assert_held(&vector_lock);

	/* Online the vector matrix array for this CPU */
	irq_matrix_online(vector_matrix);

	/*
	 * The interrupt affinity logic never targets interrupts to offline
	 * CPUs. The exception are the legacy PIC interrupts. In general
	 * they are only targeted to CPU0, but depending on the platform
	 * they can be distributed to any online CPU in hardware. The
	 * kernel has no influence on that. So all active legacy vectors
	 * must be installed on all CPUs. All non legacy interrupts can be
	 * cleared.
	 */
	for (vector = 0; vector < NR_VECTORS; vector++)
		this_cpu_write(vector_irq[vector], __setup_vector_irq(vector));
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/hw_irq.h
typedef struct irq_desc* vector_irq_t[NR_VECTORS];
DECLARE_PER_CPU(vector_irq_t, vector_irq);

// https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/apic/io_apic.c#L2014
static void lapic_register_intr(int irq)
{
	irq_clear_status_flags(irq, IRQ_LEVEL);
	irq_set_chip_and_handler_name(irq, &lapic_chip, handle_edge_irq,
				      "edge");
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/chip.c#L984
static void
__irq_do_set_handler(struct irq_desc *desc, irq_flow_handler_t handle,
		     int is_chained, const char *name)
{
	if (!handle) {
		handle = handle_bad_irq;
	} else {
		struct irq_data *irq_data = &desc->irq_data;
#ifdef CONFIG_IRQ_DOMAIN_HIERARCHY
		/*
		 * With hierarchical domains we might run into a
		 * situation where the outermost chip is not yet set
		 * up, but the inner chips are there.  Instead of
		 * bailing we install the handler, but obviously we
		 * cannot enable/startup the interrupt at this point.
		 */
		while (irq_data) {
			if (irq_data->chip != &no_irq_chip)
				break;
			/*
			 * Bail out if the outer chip is not set up
			 * and the interrupt supposed to be started
			 * right away.
			 */
			if (WARN_ON(is_chained))
				return;
			/* Try the parent */
			irq_data = irq_data->parent_data;
		}
#endif
		if (WARN_ON(!irq_data || irq_data->chip == &no_irq_chip))
			return;
	}

	/* Uninstall? */
	if (handle == handle_bad_irq) {
		if (desc->irq_data.chip != &no_irq_chip)
			mask_ack_irq(desc);
		irq_state_set_disabled(desc);
		if (is_chained)
			desc->action = NULL;
		desc->depth = 1;
	}
	desc->handle_irq = handle;
	desc->name = name;

	if (handle != handle_bad_irq && is_chained) {
		unsigned int type = irqd_get_trigger_type(&desc->irq_data);

		/*
		 * We're about to start this interrupt immediately,
		 * hence the need to set the trigger configuration.
		 * But the .set_type callback may have overridden the
		 * flow handler, ignoring that we're dealing with a
		 * chained interrupt. Reset it immediately because we
		 * do know better.
		 */
		if (type != IRQ_TYPE_NONE) {
			__irq_set_trigger(desc, type);
			desc->handle_irq = handle;
		}

		irq_settings_set_noprobe(desc);
		irq_settings_set_norequest(desc);
		irq_settings_set_nothread(desc);
		desc->action = &chained_action;
		irq_activate_and_startup(desc, IRQ_RESEND);
	}
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/chip.c#L1086
void
irq_set_chip_and_handler_name(unsigned int irq, struct irq_chip *chip,
			      irq_flow_handler_t handle, const char *name)
{
	irq_set_chip(irq, chip);
	__irq_set_handler(irq, handle, 0, name);
}

// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/chip.c#L1054
void
__irq_set_handler(unsigned int irq, irq_flow_handler_t handle, int is_chained,
		  const char *name)
{
	unsigned long flags;
	struct irq_desc *desc = irq_get_desc_buslock(irq, &flags, 0);

	if (!desc)
		return;

	__irq_do_set_handler(desc, handle, is_chained, name);
	irq_put_desc_busunlock(desc, flags);
}
EXPORT_SYMBOL_GPL(__irq_set_handler);
```

中断控制器发送给每个 CPU 的中断向量都是每个 CPU 局部的，而抽象中断处理层的虚拟中断信号 irq 以及它对应的中断描述结构 irq_desc 是全局的，也即这个 CPU 的 200 号的中断向量和另一个 CPU 的 200 号中断向量对应的虚拟中断信号 irq 和中断描述结构 irq_desc 可能不一样，这就需要一个映射关系. 这个映射关系放在 PerCPU 变量 vector_irq 里面.

在系统初始化的时候，我们会调用 lapic_online，将虚拟中断信号 irq 分配到某个 CPU 上的中断向量.

> do_IRQ已被[common_interrupt](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/irq.c#L239)替换

[handle_irq](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kernel/irq.c#L226)->`run_on_irqstack_cond/__handle_irq`-> `desc->handle_irq(desc)` in `__irq_do_set_handler` -> [handle_edge_irq](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/chip.c#L786) -> [handle_irq_event](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/handle.c#L205) -> [handle_irq_event_percpu](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/handle.c#L191) -> [__handle_irq_event_percpu](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/irq/handle.c#L137).

__handle_irq_event_percpu 里面调用了 irq_desc 里每个 hander，这些 hander 是在所有 action 列表中注册的，这才是设置的那个中断处理函数. 如果返回值是 IRQ_HANDLED，就说明处理完毕；如果返回值是 IRQ_WAKE_THREAD 就唤醒线程. 至此，中断的整个过程就结束了.

![](/msic/img/io/26bde4fa2279f66098856c5b2b6d308f.png)

通用中断子系统实现了以下这些标准流控回调函数，这些函数都定义在kernel/irq/chip.c中：
- handle_simple_irq 用于简易流控处理
- handle_level_irq 用于电平触发中断的流控处理
- handle_edge_irq 用于边沿触发中断的流控处理
- handle_fasteoi_irq 用于需要响应eoi的中断控制器
- handle_percpu_irq 用于只在单一cpu响应的中断
- handle_nested_irq 用于处理使用线程的嵌套中断

## 块设备
块设备涉及三种文件系统.

当插入一个usb盘时, mknod 还是会创建在 /dev 路径下面，这一点和字符设备一样. /dev 路径下面是 devtmpfs 文件系统. 这是块设备遇到的第一个文件系统. kernel会为这个块设备文件，分配一个特殊的 inode，这一点和字符设备也是一样的. 只不过字符设备走 S_ISCHR 这个分支 by [init_special_inode](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/inode.c#L2110)，对应 inode 的 file_operations 是 def_chr_fops；而块设备走 S_ISBLK 这个分支，对应的 inode 的 file_operations 是 def_blk_fops, 且inode 里面的 i_rdev 被设置成了块设备的设备号 dev_t.

特殊 inode 的默认 file_operations 是 [def_blk_fops](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/block_dev.c#L2150)，就像字符设备一样，有打开、读写这个块设备文件，但是常规操作不会这样做, 而会将这个块设备文件 mount 到一个文件夹下面.

> 打开这个块设备的操作 blkdev_open, 它里面调用的是 blkdev_get 打开这个块设备.

接下来，要调用 mount，将这个块设备文件挂载到一个文件夹下面. 如果这个块设备原来被格式化为一种文件系统的格式，例如 ext4，那调用的就是 ext4 `struct file_system_type [ext4_fs_type](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/ext4/super.c#L6262)`相应的 mount 操作. 这是块设备遇到的第二个文件系统，也是向这个块设备读写文件，需要基于文件系统.

在将一个硬盘的块设备 mount 成为 ext4 的时候，会调用 [ext4_mount](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/ext4/super.c#L6200)->[mount_bdev](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/super.c#L1363).

mount_bdev 主要做了两件大事情:
1. blkdev_get_by_path 根据 /dev/xxx 这个名字，找到相应的设备并打开它
1. sget 根据打开的设备文件，填充 ext4 文件系统的 super_block，从而以此为基础，建立一整套文件系统的体系.

一旦这套体系建立起来以后，对于文件的读写都是通过 ext4 文件系统这个体系进行的，创建的 inode 结构也是指向 ext4 文件系统的.

mount_bdev 做的第一件大事情，通过 [blkdev_get_by_path](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/block_dev.c#L1780)，根据设备名 /dev/xxx，得到 struct block_device *bdev.


```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/block_dev.c#L1780
/**
 * blkdev_get_by_path - open a block device by name
 * @path: path to the block device to open
 * @mode: FMODE_* mask
 * @holder: exclusive holder identifier
 *
 * Open the blockdevice described by the device file at @path.  @mode
 * and @holder are identical to blkdev_get().
 *
 * On success, the returned block_device has reference count of one.
 *
 * CONTEXT:
 * Might sleep.
 *
 * RETURNS:
 * Pointer to block_device on success, ERR_PTR(-errno) on failure.
 */
struct block_device *blkdev_get_by_path(const char *path, fmode_t mode,
					void *holder)
{
	struct block_device *bdev;
	int err;

	bdev = lookup_bdev(path);
	if (IS_ERR(bdev))
		return bdev;

	err = blkdev_get(bdev, mode, holder);
	if (err)
		return ERR_PTR(err);

	if ((mode & FMODE_WRITE) && bdev_read_only(bdev)) {
		blkdev_put(bdev, mode);
		return ERR_PTR(-EACCES);
	}

	return bdev;
}
EXPORT_SYMBOL(blkdev_get_by_path);

// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/block_dev.c#L2176
/**
 * lookup_bdev  - lookup a struct block_device by name
 * @pathname:	special file representing the block device
 *
 * Get a reference to the blockdevice at @pathname in the current
 * namespace if possible and return it.  Return ERR_PTR(error)
 * otherwise.
 */
struct block_device *lookup_bdev(const char *pathname)
{
	struct block_device *bdev;
	struct inode *inode;
	struct path path;
	int error;

	if (!pathname || !*pathname)
		return ERR_PTR(-EINVAL);

	error = kern_path(pathname, LOOKUP_FOLLOW, &path);
	if (error)
		return ERR_PTR(error);

	inode = d_backing_inode(path.dentry);
	error = -ENOTBLK;
	if (!S_ISBLK(inode->i_mode))
		goto fail;
	error = -EACCES;
	if (!may_open_dev(&path))
		goto fail;
	error = -ENOMEM;
	bdev = bd_acquire(inode);
	if (!bdev)
		goto fail;
out:
	path_put(&path);
	return bdev;
fail:
	bdev = ERR_PTR(error);
	goto out;
}
EXPORT_SYMBOL(lookup_bdev);
```

blkdev_get_by_path 干了两件事情:
1. lookup_bdev 根据设备路径 /dev/xxx 得到 block_device
1, 打开这个设备，调用 blkdev_get. 上面分析过 def_blk_fops 的默认打开设备函数 blkdev_open，它也是调用 blkdev_get 的. 块设备的打开往往不是直接调用设备文件的打开函数，而是调用 mount 来打开的.

lookup_bdev 这里的 pathname 是设备的文件名，例如 /dev/xxx. 这个文件是在 devtmpfs 文件系统中的，kern_path 可以在这个文件系统里面，一直找到它对应的 dentry. 接下来，d_backing_inode 会获得 inode. 这个 inode 就是那个 init_special_inode 生成的特殊 inode.

接下来，bd_acquire 通过这个特殊的 inode，找到 struct block_device.

[bd_acquire](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/block_dev.c#L946) 中最主要的就是调用 [bdget](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/block_dev.c#L881)，它的参数是特殊 inode 的 i_rdev. 这里面在 mknod 的时候，放的是设备号 dev_t.

在 bdget 中，就遇到了第三个文件系统，bdev 伪文件系统. bdget 函数根据传进来的 dev_t，在 [blockdev_superblock](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/block_dev.c#L837) 这个文件系统里面找到 inode. 这里注意，这个 inode 已经不是 devtmpfs 文件系统的 inode 了. blockdev_superblock 的初始化在整个系统初始化的时候，会调用 [bdev_cache_init](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/block_dev.c#L837) 进行初始化.

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/block_dev.c#L837
static struct file_system_type bd_type = {
	.name		= "bdev",
	.init_fs_context = bd_init_fs_context,
	.kill_sb	= kill_anon_super,
};

struct super_block *blockdev_superblock __read_mostly;
EXPORT_SYMBOL_GPL(blockdev_superblock);

void __init bdev_cache_init(void)
{
	int err;
	static struct vfsmount *bd_mnt;

	bdev_cachep = kmem_cache_create("bdev_cache", sizeof(struct bdev_inode),
			0, (SLAB_HWCACHE_ALIGN|SLAB_RECLAIM_ACCOUNT|
				SLAB_MEM_SPREAD|SLAB_ACCOUNT|SLAB_PANIC),
			init_once);
	err = register_filesystem(&bd_type);
	if (err)
		panic("Cannot register bdev pseudo-fs");
	bd_mnt = kern_mount(&bd_type);
	if (IS_ERR(bd_mnt))
		panic("Cannot create bdev pseudo-fs");
	blockdev_superblock = bd_mnt->mnt_sb;   /* For writeback */
}
```

所有表示块设备的 inode 都保存在伪文件系统 bdev 中，这些对用户层不可见，主要为了方便块设备的管理. Linux 将块设备的 block_device 和 bdev 文件系统的块设备的 inode，通过 [struct bdev_inode](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/block_dev.c#L837) 进行关联. 所以，在 bdget 中，BDEV_I 就是通过 bdev 文件系统的 inode，获得整个 struct bdev_inode 结构的地址，然后取成员 bdev，得到 block_device.