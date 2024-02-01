# filesystem
类Unix文件系统是目录和文件的一种层次结构, 像一颗倒置的树, 起点是`/`(root目录).

[linux有FHS(Filesystem Hierarchy Standard,文件系统层次结构)标准](http://refspecs.linuxfoundation.org/fhs.shtml):
![](/misc/img/fs/052049040017593.png)

Linux 系统中常见的目录名称以及相应内容:
- /boot 开机所需文件—内核、开机菜单以及所需配置文件等
- /dev 以文件形式存放任何设备与接口
- /etc 配置文件
- /home 用户家目录
- /bin 存放单用户模式下还可以操作的命令
- /lib 开机时用到的函数库，以及/bin 与/sbin 下面的命令要调用的函数
- /sbin 开机过程中需要的命令
- /media 用于挂载设备文件的目录
- /opt 放置第三方的软件
- /root 系统管理员的家目录
- /srv 一些网络服务的数据文件目录
- /tmp 任何人均可使用的“共享”临时目录
- /proc 虚拟文件系统，例如系统内核、进程、外部设备及网络状态等
- /usr/local 用户自行安装的软件
- /usr/sbin Linux 系统开机时不会使用到的软件/命令/脚本
- /usr/share 帮助与说明文件，也可放置共享文件
- /var 主要存放经常变化的文件，如日志
- /lost+found 当文件系统发生错误时，将一些丢失的文件片段存放在这里

**目录(directory)**是一种逻辑上包含若干文件的文件. 每个文件都会包含一个文件名(filename)和若干文件属性(文件类型, 大小, 所有者, 权限, 最后访问时间/修改时间等).

创建新目录时会自动创建两个文件名： `.`和`..`, 分别表示当前目
录和父目录; 仅在最高层次的根目录中， 两者指向相同, 都表示当前目录.

**路径名(pathname)**是由`/`分隔的若干文件名序列. 以`/`开头的路径名称为绝对路径(absolute pathname); 否则称为相对路径(relative pathname), 它是指向相对于某个文件的文件. **根目录是特殊的绝对路径, 它不包含文件名**.

## 文件类型
定义在[`#include <sys/stat.h>`](https://en.wikibooks.org/wiki/C_Programming/POSIX_Reference/sys/stat.h)里, 可通过`os.FileMode`进行位操作来判断.

- - :普通文件(regular file)
- d : 目录文件(directory file)
- b : 块特殊文件(block special file)
- c : 字符特殊文件(character special file)
- p : 管道(FIFO), 用于进程间通信. 与socket类似, 以由socket取代. 
- 套接字(socket) : 用于进程间的网络通信
-l : 符号链接(symbolic link) : 指向另一个文件

可通过`ls -ld xxx`查看.

### 目录
在文件系统中,目录的存储方式类似于普通文件. 目录与普通文件的区别有二:
- 在其 i-node 条目中,会将目录标记为一种不同的文件类型
- 目录是经特殊组织而成的文件. 本质上说就是一个表格,包含文件名和 i-node 编号.

### 文件描述符和打开文件之间的关系
文件由内核的 3 个数据结构维护:
1. 进程级的文件描述符表

	针对每个进程,内核为其维护打开文件的描述符(open file descriptor)表, 该表的每一条目都记录了单个文件描述符的相关信息:
	- 控制文件描述符操作的一组标志(目前,此类标志仅定义了一个,即 close-on-exec 标志)
	- 对打开文件句柄的引用
1. 系统级的打开文件表

	内核对所有打开的文件维护有一个系统级的描述表格(open file description table), 有时也称之为打开文件表(open file table),并将表中各条目称为打开文件句柄(open file handle) . 一个打开文件句柄存储了与一个打开文件相关的全部信息:
	- 当前文件偏移量(调用 read()和 write()时更新,或使用 lseek()直接修改)
	- 打开文件时所使用的状态标志(即,open()的 flags 参数)
	- 文件访问模式(如调用 open()时所设置的只读模式、只写模式或读写模式)
	- 与信号驱动 I/O 相关的设置
	- 对该文件 i-node 对象的引用
1. 文件系统的 i-node 表

	每个文件系统都会为驻留其上的所有文件建立一个 i-node 表:
	- 文件类型(例如,常规文件、套接字或 FIFO)和访问权限
	- 一个指针,指向该文件所持有的锁的列表
	- 文件的各种属性,包括文件大小以及与不同类型操作相关的时间戳

同一进程的两个文件描述符指向同一个打开的文件句柄: 通过调用 dup()、dup2()或 fcntl()而形成的.
不同进程的两个文件描述符指向同一个打开的文件句柄: fork() 或 通过UNIX 域套接字将一个打开的文件描述符传递给另一进程.
不同/同一进程的不同打开文件句柄指向 i-node 表中的相同条目: 对同一文件发起了 多次 open()调用.

指向同一打开文件句柄,将共享同一文件偏移量. 因此,如果通过其中一个文件描述符来修改文件偏移量(由调用 read()、write()或 lseek()
所致),那么从另一文件描述符中也会观察到这一变化; 且打开的文件标志也共享(例如,O_APPEND、O_NONBLOCK 和 O_ASYNC, 可通过 fcntl()的 F_GETFL 和 F_SETFL 操作).

但文件描述符标志(亦即,close-on-exec 标志)为进程和文件描述符所私有, 对这一标志的修改将不会影响同一进程或不同进程中的其他文件描述符.

> i-node 在磁盘和内存中的差异: 内存i-node = 磁盘i-node + 记录了引用该 i-node 的打开文件句柄数量以及该 i-node 所在设备的主、从设
备号,还包括一些打开文件时与文件相关的临时属性,例如:文件锁. 

## [设备文件](/arch/device.md)

## inode
Linux 在生成文件的时候，内容会为每一个文件生成一个唯一的索引节点（Inode），文件的属性都会保存在这个Inode中.

## 链接
链接是一种为了共享文件和快速访问而建立起来的文件映射关系, 有**软链接(推荐)**和硬链接之分:
- 硬链接(类似于指针) : 指向目标文件的inode, 因此没有创建新文件.
	1. 只能对已存在的文件进行创建
	1. 硬链接和目标文件必须是同一文件系统(不同的文件系统有不同的inode table, i-node 编号的唯一性仅在一个文件系统之内才能得到保障)
	1. 为同一个文件创建多个硬链接即多个别名（他们有共同的 inode）
	1. 只有root才能创建执行目录的硬链接(且需要文件系统支持), 这样做是为了避免遍历目录时出现循环.
- 软链接又叫符号链接(symbol links), 类似于Windows下面的快捷键
	1. 可对不存在的文件创建软链接; 允许跨文件系统
	1. 软链接是一个新文件(包含了它所链接的另一个文件的路径)
	1. 符号链接的大小是其链接文件的路径名的字节数

> `ln`命令创建硬链接时会增加链接数，`rm`命令会减少链接数.一个文件除非硬链接数为0，否则不会从文件系统中被物理地删除.
> 可使用`readlink`命令读取链接
> 这两种链接的本质区别关键点在于inode
> 针对目录的软链接删除: `rm -rf symbol_name(删除软链接symbol_name)` 和 `rm -rf symbol_name/(仅删除symbol_name目录下的所有文件, 其他不变)`
> 系统调用对符号链接的解释可见[Linux/UNIX系统编程手册#表 18-1 各个函数对符号链接的解释]().

大部分操作会无视符号链接的所有权和权限(创建符号链接时会为其赋予所有权限). 
是否允许操作反而是由符号链接所指代文件的所有权和权限来决定. 仅当在带有粘性权限
位的目录中对符号链接进行移除或改名操作时,才会考虑符号链接自身的所有权.

![](/misc/img/fs/7b670449a3faa6a2cdab45c2298b66dd.png)

## example
1. 列出一个目录中所有的文件
```go
package main

import (
	"io/ioutil"
	"log"
	"os"
)

func main() {
	args := os.Args
	if len(args) != 2 {
		log.Println("usage: ls directory_name")
		os.Exit(1)
	}

	// 获取所有文件
	files, err := ioutil.ReadDir(args[1])
	CheckErr(err)

	for _, file := range files {
		if file.IsDir() {
			log.Printf("%s is dir\n", file.Name())
		} else {
			log.Printf("%s is file\n", file.Name())
		}
	}
}

func CheckErr(err error) {
	if err != nil {
		log.Fatal(err)
	}
}
```

## 有关系统统调用
对于文件的操作,下面这六个系统调用是最重要的:
1. 对于已经有的文件,可以使用open打开这个文件,close关闭这个文件;
1. 对于没有的文件,可以使用creat创建文件;
1. 打开文件以后,可以使用lseek跳到文件的某个位置;
1. 对文件的内容进行读写,读的系统调用是read,写是write

进程会为每个文件分配一个文件描述符(File Descriptor, 一个整数), 有了它我们就可以使用系统调用操作文件了

> Linux 里一切皆文件

## mount
将文件系统挂载到文件树上, 挂载的目录会映射为新加入的文件系统的根,而原目录内容就被隐藏,该目录也叫挂载点(mount point).

需永久挂载时可将信息保持到`/etcd/fstab`

umount用于卸载文件系统, 但不能卸载正处于"busy"状态的文件系统, 此时可以使用`fuser -c /xxx`查找哪个进程在使用文件.

> 通过 Linux 专有的虚拟文件/proc/mounts,可查看当前已挂载文件系统的列表, 包含了已挂载文件系统的精确信息.
> 随着引入mount namespace, 每个进程都拥有一个/proc/PID/mounts 文件, 其中会列出组成进程挂载空间的挂载点,而/proc/mounts只是指向/proc/self/mounts的符号链接.

/etc/fstab 的格式:
1. 已挂载设备名
2. 设备的挂载点
3. 文件系统类型
4. 挂载标志. 比如`rw`表示以可读写方式挂载文件系统. 默认是`default`表示`rw, suid, dev, exec, auto, nouser, async`.
5. 自检(一个数字) : dump(8)会使用其来控制对文件系统的备份操作. 只有/etc/fstab 文件才会用到该字段和第 6 个字段,在/proc/mounts 和/etc/mtab 中,该字段总是为 0(开机不自检). 1是开机进行磁盘自检.
6. 优先级(一个数字). 在系统引导时如果`自检`为1, 则可对多块磁盘进行自检优先级设置, 即用于控制 fsck(8)对文件系统的检查顺序.

## lost+found
fsck(文件系统一致性检查工具)找到一个无法确定其父目录的文件时就会将其放入其中.

## ramdisk(已淘汰)
将一部分固定大小的内存当作分区来使用.

以一块固定大小的内存作为一个block设备创建文件系统，其中的内容只存在于内存中，修改它的内容不会记录到磁盘中. 它用内存模拟了block设备，所以其中的内容还要先加载到page cache中，本身它是内存，创建的page cache也是内存，这产生了对很多内存的浪费.

am disk被弃用的另外一个原因是环回设备(loopback)引入. 环回设备提供了一种更加灵活、方便的从文件而不是从内存块中创建综合块设备的方法.

## ramfs
ramfs是空间规模动态变化的RAM文件系统，是用来实现Linux缓存机制(缓存page cache and dentry cache)的文件系统, 不是用swap.

## tmpfs
tmpfs是ramfs的衍生物，是驻留于内存中的虚拟文件系统, 可限制缓存大小、其可使用swap.

创建命令: `mount -t tmpfs ${source} ${target} [-o size=<n>m]`, 无需预先mkfs. source是要创建的tmpfs的名称.

tmpfs 文件系统还有以下两个特殊用途:
- 由内核内部挂载的隐形 tmpfs 文件系统,用于实现 System V 共享内存和共享匿名内存映射.
- 挂载于/dev/shm 的 tmpfs 文件系统, 被 glibc 用以实现 POSIX 共享内存和 POSIX 信号量.

## devfs
设备文件系统, 管理/dev下的所有设备

## sysfs
在/sys下管理设备, 是当前系统上实际设备树的一个直观反应, 是通过kobject子系统管理的, 用于取代devfs.

udev是在用户空间管理设备的工具. 它利用了sysfs提供的信息来实现所有devfs的功能, 能够根据系统中的硬件设备的状况动态更新设备文件，包括设备文件的创建，删除等. 它需要内核sysfs和tmpfs的支持，sysfs为udev提供设备入口和uevent通道，tmpfs为udev设备文件提供存放空间.

## initramfs
解决找init的问题

## 特殊文件系统

### /sys
ref:
- [sysfs、udev 和 它们背后的 Linux 统一设备模型](https://www.binss.me/blog/sysfs-udev-and-Linux-Unified-Device-Model/)

sysfs代码在`fs/sysfs`.

`/sys`是一种在内存中的虚拟文件系统sysfs的mount point, 提供了有关系统上可用设备, 设备的配置及其状态的信息. sysfS的目录层次严格对应于内核的抽象组织层次,其用户态的文件形式分别对应于
内核中相应的抽象.

sysfs 文件形式和内核抽象的对应关系:
- 目录, 内核对象, 目录的包含关系决定了内核对象的从属关系

	- sysfs_create_dir_ns
	- sysfs_remove_dir
- 文件, 对象属性, 属性说明了内核对象的能力, 如LED的闪烁

	sysfs的指定原则之一是`/sys`里的每个文件都只表示下层设备的一个属性.

	- sysfs_create_files
	- sysfs_remove_files
	- sysfs_create_group : 操作一系列属性
	- sysfs_remove_group
	- sysfs_create_bin_file: 操作二进制属性
	- sysfs_remove_bin_file
- 链接, 对象关系

	- sysfs_create_link
	- sysfs_create_remove

可通过`udevadm`查询设备信息, 触发事件, 控制vdevd守护进程, 以及监视udev和内核的事件, 比如查看ssd信息`udevadm info -a -n nvme0n1`.

`/sys`目录:
- block : 系统中当前所有的块设备所在，按照功能来说放置在 /sys/class 之下会更合适，但只是由于**历史遗留**因素而一直存在于/sys/block, 但从 2.6.22 开始就已标记为过时，只有在打开了 CONFIG_SYSFS_DEPRECATED配置下编译才会有这个目录的存在，并且**在 2.6.26 内核中已正式移到 /sys/class/block**, 旧的接口 /sys/block为了向后兼容保留存在，但其中的内容已经变为指向它们在 /sys/devices/ 中真实设备的符号链接文件

	- sda : scsi盘

		- device: 设备信息

			- state: 设备状态. `running`即在线

				`echo offline > /sys/block/sda/device/state`可将磁盘offline
			- delete: 用于从SCSI子系统删除磁盘

				`echo 1 > /sys/block/device-name/device/delete`
- bus : 内核设备按总线类型(pci-e, scsi, usb等)分类放置的目录结构，devices 中的所有设备都是连接于某种总线之下，在这里的每一种具体总线之下可以找到每一个具体设备指向`/sys/devices/`的符号链接，它也是构成 Linux 统一设备模型的一部分

	- scsi : scsi总线

		- drivers : scsi总线上的驱动程序
		- devices : scsi设备, 名称是SCSI设备在Linux系统中的逻辑地址映射
	- usb
		- devices
			- <device>
				- serial: usb序列号, 编码在usb EEPROM中

	> 对应 kernel 中的 struct bus_type
	> 某个总线目录之下的 drivers 目录包含了该总线所需的所有驱动的符号链接
- class : 按照设备功能分类的设备模型(比如声卡, 显卡, 输入设备, 网卡)组织的一颗树, 里面也是指向`/sys/devices/`下对应设备的符号链接

	> /sys/class目录下有三个主要文件夹跟 fibre channel相关的文件夹fc_transport、fc_remote_ports，fc_hosts.

	- input : 系统所有输入设备都会出现在这，而不论它们是以何种总线连接到系统。它也是构成 Linux 统一设备模型的一部分
	- fc_host : 主机HBA卡信息, 通常一张光纤卡有2个光纤口.

		- host<H> : 光纤口

			- port_id : HBA端口的 24位交换机端口ID
			- port_name : 存储端口的64位port name, wwpn
			- issues_lip : 重置HBA端口，重新尝试发现存储端口
			- symbolic_name : 保持光纤卡型号, 固件版本, 使用的qla2xxx驱动版本
	- fc_remote_ports : 主机到存储端口链路信息（包含未给主机分配存储的链路信息）

		- rport-H:B-R : (H代表主机，B代表bus号，T代表target，L代表lun id，R代表对端端口号)
			- port_id : 存储端口的 24位交换机端口ID
			- node_name : 存储端口(即target)的64位node name
			- port_name : 存储端口(即target)的64位port name, wwpn
			- dev_loss_tmo : 链路故障等待时间

				故障链路不再处理任何新的IO。默认dev_loss_tmo值视具体HBA卡而定，Qlogic默认是35秒，Emulex默认是30秒。HBA卡自带驱动可以覆盖这个 参数值。dev_loss_tmo最大值600秒，如果dev_loss_tmo值小于0或者大于600，HBA自带超时值生效。
			- fast_io_fail_tmo : IO故障等待时间. 链路波动情况，IO尝试多长时间
			- port_state: 链路状态
	- fc_transport : 已分配的target信息, 在fc client端显示.

		- targetH:B:T : (H代表主机，B代表bus号，T代表target，L代表lun id，R代表对端端口号)

			- port_id : 存储端口的 24位交换机端口ID
			- node_name : 存储端口的64位node name
			- port_name : 存储端口的64位port name
	- graphics: 图形设备
	- dmi
		- id
			- product_uuid : 主板uuid, 来自bios dmi, 仅适用于x86

	> 对应 kernel 中的 struct class

- dev : 设备驱动程序分类(块设备和字符设备)的设备信息. 它维护一个按字符设备和块设备的主次号码 (major:minor)链接到真实的设备(/sys/devices下)的符号链接文件，是在内核 2.6.26 首次引入
	> 对应 kernel 中的 struct device_driver
- devices : 包含所有被发现的注册在各种总线上的各种物理设备, 是内核对系统中所有设备的分层次表达模型，也是 /sys 文件系统管理设备的最重要的目录结构

	所有的物理设备都按其在总线上的拓扑结构来显示, 除了 platform devices 和 system devices:
	- platform devices 一般是挂在芯片内部高速或者低速总线上的各种控制器和外设, 能被 CPU 直接寻址
	- system devices 不是外设, 是芯片内部的核心结构, 比如 CPU，timer 等, 一般没有相关的driver, 但是会有一些体系结构相关的代码来配置它们.

	> 对应 kernel 中的 struct device
- firmware : 特定于平台的子系统的接口, 如ACPI. 是系统加载固件机制对用户空间的接口，关于固件有专用于固件加载的一套API
- fs : 内核知道的一些但不是全部文件系统的目录, 是用于描述系统中所有文件系统，包括文件系统本身和按文件系统分类存放的已挂载点，但目前只有 fuse,gfs2 等少数文件系统支持sysfs 接口，一些传统的虚拟文件系统(VFS)层次控制参数仍然在 sysctl (/proc/sys/fs) 接口中
- hypervisor: 如果开启了 Xen, 这个目录下会提供相关属性文件
- kernel : 内核所有可调整参数的位置及内核的内部信息, 比如高速缓存和虚拟内存状态
- module : kernel所有模块的信息，不论这些模块是以内联(inlined)方式编译到内核映像文件(vmlinuz)中还是编译为外部模块(ko文件)，都可能会出现在 /sys/module 中
- power : 电源选项, 可用于控制整个机器的电源状态, 如写入控制命令进行关机、重启等

## /proc
`/proc`文件系统是一种虚拟文件系统, 所有信息是**内存的映射**, 以文件系统目录和文件形式,提供一个指向内核数据结构的接口, 可系统运行中修改kernel参数, 内核产生的所有状态信息和统计信息均在里面.

> 大多数linux工具都依赖`/proc`提供的信息进行性能监控. 比如ps和top均从`/proc`读取进程状态信息, 再比如vmstat, cpuinfo等.

`/proc`: 各种系统信息
- `${pid}` : 进程信息

	- cmdline : 进程的完整命令行(以null即`\0`分隔)
	- cwd : 链接到进程当前工作目录的符号链接
	- environ : 进程的环境变量(以null分隔)
	- exe : 链接到进程可执行文件的符号链接, 指向可执行程序的绝对路径
	- fd : 包含链接到进程每个打开文件的符号链接. 链接到的管道和socket没有关联的文件名, 但有具体id
	- fdinfo : 进程中文件描述符的相关信息
		- pos 字段 : 当前的文件偏移量
		- flags : 为一个八进制数,表征文件访问标志和已打开文件的状态标志
	- maps : 内存映射信息(共享段, lib等)
	- men : 进程虚拟内存(在 I/O 操作之前必须调用 lseek()移至有效偏移量)
	- mounts : 进程的安装点
	- status : 各种信息(比如,进程 ID、凭证、内存使用量、信号)
	- stack : 主线程的内核堆栈的信息(线程内核模式堆栈位于`/proc/[PID]/task/[TID]/stack`下)
	- wchan : 表示导致进程睡眠或者等待的函数,可结合进程状态来查看
	- root : 链接到进程的根目录(由chroot设置)的符号链接
	- state : 进程的总状态信息(ps可解析该信息)
	- statm : 内存使用情况的信息
	- task : 为进程中的每个线程均包含一个子目录(始自 Linux 2.6)
	- oom_adj : OOM Killer分值, range is [-16, 15]和-17. -17表示禁止被OOM机制处理. 其他具体值是用2^N来体现的, 因此n是正数时容易被OOM Killer选定.
	- oom_score_adj : 用于替换oom_adj, range is [-1000, 1000], -1000即禁止被OOM机制处理
	- limits : limits
	- ioports : I/O地址空间
	- iomem: 物理地址的分配情况
- acpi : 大多数现代桌面和笔记本支持的高级配置和电源接口. acpi主要是pc技术, 服务器上通常被禁用
- asound : alsa声卡驱动接口
- buddyinfo : buddy内存分配信息
- bus : 包含总线子系统的信息, 比如pci总线或各自系统的usb接口

	- input : 输入设备

		- devices : 目前检测到的输入设备, 比如键盘

			设备的Handlers中的event<N>与`/dev/input/event<N>`对应.
- cgroups : 查看系统支持的cgruop subsystem(controller)
- cmdline : 内核启动参数
- cpuinfo : cpu的信息
- devices : kernel中的设备驱动程序列表
- diskstats : 磁盘i/o的统计信息
- driver : 驱动信息
- dma : 当前使用的dma通道
- execdomains : 执行区域列表
- fb : frame buffer信息
- filesystems : 查看当前为内核所支持的文件系统类型
- fs : 文件系统特别信息
- interrupt : 中断的使用情况, 记录中断的产生次数
- iomem : i/o内存映射信息
- ioports : i/o端口分配情况
- irq : 中断请求设置接口. 每个子目录是一个中断, 并可能是一个连接的设备, 比如一个网卡接口. 这里可修改一个特定中断的cpu亲和力.

	修改cpu亲缘性: <irq id>/smp_smp_affinity, 其内容是16进制数, 用位表示某cpu, 比如01表示cpu0.
- interrupts : 中断报告文件, 关于哪些中断正在使用和每个处理器各被中断了多少次的信息

	文件信息:
	1. 中断号
	1. cpu接到该中断的数量
	1. 最后列: 使用该中断的设备
- kallsyms : 保存了Linux内核符号表(也包含了已加载模块的符号表), 可用于检查kernel函数是否存在. 例如bcc使用`BPF.get_kprobe_functions(b'xxx')`函数校验kernel函数是否存在就是基于它.
- kcore : 内核核心映像, gdb可以利用它查看当前内核的所有数据结构状态
- key-users : 密钥保留服务文件
- kmsg : 内核消息
- loadavg : 负载均衡信息
- locks : 内核锁
- mdstat : 磁盘阵列信息
- meminfo : 内存信息, 包括虚拟内存
- misc : 杂项信息
- modules : 系统已加载的kernel模块信息

	列:
	1. 模块的名字
	1. 模块的内存大小，单位是bytes
	1. 被load的次数，0以为着没有被load过
	1. 是否依赖第三方moudle，列出这些module
	1. 模块的状态，有Live， Loading， Unloading三种状态
	1. 模块当前的内核内存偏移位置. 这些信息，debug的时候会非常有用, 例如一些诊断工具 oprofile
- mounts : 当前已挂载文件系统的列表
- net : 网络接口的大量原始统计, 比如接收的组播数据包或每个接口的路由
- partitions : 记录了系统中每个磁盘分区的主辅设备编号、大小和名称
- scsi : scsi子系统信息, 比如连接的设备或驱动程序版本
- self : 访问procfs的进程信息
	- sessionid: 标识特定 Linux 登录会话的 ID, 最好将其与 /proc/sys/kernel/random/boot_id 结合使用. 部分发行版不支持总是返回`-1`
- slabinfo : 内核缓存信息
- stat : 系统的各种状态信息
- swaps : 交换空间使用情况
- sys : 可调整的内核参数, 比如虚拟内存管理器或网络协议栈的行为.

	- abi : 文件与应用程序的二进制信息
	- dev : 特定设备的信息
	- fs : 特定的文件系统

		- file-max: 登入用户可打开的最大文件数
	- kernel : 可控制一系列的内核参数
			- printk : 调整内核printk的打印级别

			```bash
			# Avoid kernel message corrupt the dialog screen(openvt)
			# turn off printk to avoid "kernel tainted" message display on setup screen
			old_printk=`cat /proc/sys/kernel/printk`
			echo "0 0 0 0" > /proc/sys/kernel/printk
			```

			```c
			# printk有8个loglevel,定义在include/linux/kernel.h
			# /kernel/printk.c:
			#define DEFAULT_MESSAGE_LOGLEVEL 4
			#define MINIMUM_CONSOLE_LOGLEVEL 1
			#define DEFAULT_CONSOLE_LOGLEVEL 7

			int console_printk[4] = {
				DEFAULT_CONSOLE_LOGLEVEL,  # 终端级别, 小于该优先级的消息才会被输出到控制台
				DEFAULT_MESSAGE_LOGLEVEL,  # 默认级别
				MINIMUM_CONSOLE_LOGLEVEL,  # 让用户使用的最小级别
				DEFAULT_CONSOLE_LOGLEVEL,  # 默认终端级别
			};
			```
			- random : 随机数相关
				- boot_id : boot id, see `hostnamectl`
	- net : 网络和套接字的设置
	- vm : 内存管理设置, 包括buffer和cache管理

		- drop_caches : 强制释放memory的磁盘缓存, 不推荐使用, 内存不够时还是加内存的好
		
			写入1, 释放页面缓存; 写入2, 释放目录文件和inodes; 写入3, 释放页面缓存, 目录文件和inodes
- sysrq-trigger : 触发sysrq的触发器
- sysvipc : 有关 System V IPC 对象的信息
- tty : 各个虚拟终端和与它连接的物理设备信息
- uptime : 系统的总的运行时间和空闲时间
- version : kernel版本信息
- vmstat : 虚拟内存统计表
- zoneinfo : 内存管理区信息

> 针对进程中的每个线程,内核提供了以/proc/PID/task/TID 命名的子目录,其中 TID 是该线程的线程 ID. TID 子目录中都有一套类似于/proc/PID 目录内容的文件和目录.

## /dev/fd
对于每个进程,内核都提供有一个特殊的虚拟目录/dev/fd. 该目录中包含`/dev/fd/${n}`形式的文件名,其中 n 是与进程中的打开文件描述符相对应的编号.

打开/dev/fd 目录中的一个文件等同于复制相应的文件描述符.

> /dev/fd 实际上是一个符号链接,链接到 Linux 所专有的/proc/self/fd 目录.
> `ls | diff /dev/fd/0 oldlist`取代`ls | diff - oldlist`, 以避免部分命令未实现`-`(表示stdin/stdout)或部分命令使用`-`作为命令行选项结束的分隔符.

## 配额
rhel7安装有quota磁盘容量配额服务程序, 但存储设备默认没有启用.

启用quota: mount时添加uquota 参数.

quota命令还有软限制和硬限制的功能:
- 软限制：当达到软限制时会提示用户，但仍允许用户在限定的额度内继续使用
- 硬限制：当达到硬限制时会提示用户，且强制终止用户的操作

edquota 命令可用于编辑用户的 quota 配额限制.

## xfs
XFS是一种高性能的日志文件系统，而且是 RHEL 7 中默认的文件管理系统. 它的优势在发生意外宕机后尤其明显，即可以快速地恢复可能被破坏的文件，而且强大的日志功能只用花费极低的计算和存储性能, 并且它最大可支持的存储容量为 18EB.

查看xfs分区信息: `xfs_info ${分区}`
XFS管理 quota 磁盘容量配额的命令: `xfs_quota`
查看文件块状况: `xfs_bmap`
查看磁盘碎片: `xfs_db`
碎片整理: `xfs_fsr`

## ext4
参考:
- [理解inode](https://www.ruanyifeng.com/blog/2011/12/inode.html)

> Linux系统使用struct inode作为数据结构名称. BSD派生的系统，使用vnode名称，其中v表示`virtual file system`.

它支持的存储容量高达 1EB（1EB=1,073,741,824GB），且能够有无限多的子目录. 另外，Ext4 文件系统能够批量分配 block 块，从而极大地提高了读写效率. 它是最常见的fs.

文件系统由以下几部分组成:
- 引导块 : 总是作为文件系统的首块. 引导块不为文件系统所用,只是包含用来引导操作系统的信息. 操作系统虽然只需一个引导块,但所有文件系统都设有引导块(其中的绝大多数都未使用).
- 超级块:紧随引导块之后的一个独立块,包含与文件系统有关的参数信息,其中包括:
	- i 节点表容量
	- 文件系统中逻辑块的大小
	- 以逻辑块计,文件系统的大小
- inode table : 文件系统中的每个文件或目录在 i 节点表中都对应着唯一一条记录(inode). 这条记录登记了关乎文件的各种信息. 有时也将 i 节点表称为 i-list.
- 数据块 : 文件系统的大部分空间都用于存放数据,以构成驻留于文件系统之上的文件和目录

磁盘格式化时就划分了两个区域: 一个是数据区，存放文件数据；另一个是inode区（inode table），存放inode所包含的信息. 

在ext4上每个inode节点的大小为256B.  inode节点的总数和大小，在格式化时就给定了.
查看每个硬盘分区的inode总数和已经使用的数量，可以使用`df -i`命令. 查看每个inode节点的大小可用`dumpe2fs -h /dev/sda1 | grep "Inode size"`, 数据块的大小可用`dumpe2fs -h /dev/sda1 | grep "Block size"`查看

```
$ dumpe2fs -h /dev/sda1
...
Inode count:              3932160
Block count:              15728384 # 根据 block count 和 inode count，可算出 16k bytes-per-inode （15728384*4096/3932160)
Block size:               4096 # 每个 inode 大小为 256byte，block 大小为 4k byte
Inode size:	          256
Inodes per group:         8192
Inode blocks per group:   512
...
```

> dumpe2fs适用exit4, 不适用xfs.

驻留于同一物理设备上的不同文件系统,其类型、大小以及参数设置(比如,块大小)都可以有所不同. 这也是将一块磁盘划分为多个分区的原因之一.

针对驻留于文件系统上的每个文件,文件系统的 i 节点表会包含一个 i 节点(索引节点的简称). 对 i 节点的标识,采用的是 i节点表中的顺续位置,以数字表示. 文件的 i 节点号(或简称为 i 号)是`ls -li`命令所显示的第一列. i 节点所维护的信息如下所示, 该信息可以`stat ${文件名}`来查看:
- 文件类型(比如,常规文件、目录、符号链接,以及字符设备等)
- 文件属主(亦称用户 ID 或 UID) 和 属组(亦称为组 ID 或 GID)
- 文件的特殊权限（SUID、SGID、SBIT）
- 3类用户的访问权限:属主(有时也称为用户)、属组以及其他用户(属主和属组用户之外的用户)
- 3 个时间戳:对文件的最后访问时间(`ls -lu`所显示的时间)、对文件的最后修改时间(`ls -l`所显示的时间),以及文件状态的最后改变时间(`ls -lc`所显示的最后改变 i 节点信息的时间). 值得注意的是,与其他 UNIX 实现一样,大多数 Linux 文件系统不会记录文件的创建时间. 各种函数对文件时间戳的影响可见[Linux/UNIX系统编程手册#表 15-2 各种函数对文件时间戳的影响]().
- 指向文件的硬链接数量
- 文件的大小,以字节为单位
- 实际分配给文件的块数量,以 512 字节块为单位. 这一数字可能不会简单等同于文件的字节大小,因为考虑文件中包含空洞的情形,分配给文件的块数可能会低于根据文件正常大小(以字节为单位)所计算出的块数.
- 指向文件数据块的指针(pointer)

## 数据恢复
extundelete工具 from <<循序渐进Linux第2版>>.

## inotify
允许应用程序监控文件事件.

使用 inotify API 有以下几个关键步骤:
1. 使用 inotify_init()来创建一 inotify 实例
1. 使用 inotify_add_watch()向 inotify 实例的监控列表添加条目

	参数 mask 为一位掩码,针对 pathname 定义了意欲监控的事件
1. 为获得事件通知, 需针对 inotify 文件描述符执行 read()操作

	每次对 read()的成功调用,都会返回一个或多个 inotify_event 结构,其中各自记录了处于 inotify 实例监控之下的某一路径名所发生的事件.

	inotify 事件可见[Linux/UNIX系统编程手册#表 19-1 inotify 事件]().
1. 在结束监控时会关闭 inotify 文件描述符. 这会自动清除与 inotify 实例相关的所有监控项.

inotify 机制可用于监控文件或目录. 当监控目录时,与路径自身及其所含文件相关的事件都会通知给应用程序.

inotify 监控机制为**非递归**. 若应用程序有意监控整个目录子树内的事件,则需对该树中的每个目录发起 inotify_add_watch()调用.

可使用 epoll 以及由信号驱动的 I/O(自 Linux 2.6.25 起)来监控 inotify文件描述符.

`/proc/sys/fs/inotify/*`提供了对notify 机制的操作施以各种限制:
- max_queued_events

	inotify 实例队列中的事件数量设置上限. 一旦超出这一上限,系统将生成 IN_Q_OVERFLOW 事件,并丢弃多余的事件. 溢出事件的 wd 字段值
为−1.
- max_user_instances

	对由每个真实用户 ID 创建的 inotify 实例数的限制值
- max_user_watches

	对由每个真实用户 ID 创建的监控项数量的限制值

## fuse
fuse(filesystem in userspace, 用户空间文件系统)是在用户空间创建fs的框架. 一些文件系统如glusterfs和lustre使用FUSE实现.

在用户态实现文件系统必然会引入额外的内核态/用户态切换带来的开销，对性能会产生一定影响

## lock
参考:
- [被遗忘的桃源——flock 文件锁](https://zhuanlan.zhihu.com/p/25134841)

文件锁包括建议性锁和强制性锁:
- 建议锁：要求每个使用上锁文件的进程都要检查是否有锁存在，并且尊重已有的锁. 在一般情况下，内核和系统都不使用建议性锁，它们依靠程序员遵守这个规定
- 强制锁：是由内核执行的锁，当一个文件被上锁进行写入操作的时候，内核将阻止其他任何文件对其进行读写操作. 采用强制性锁对性能的影响很大，每次读写操作都必须检查是否有锁存在

Linux操作系统下，想要对一个文件加锁，一般有两种方式，一种是flock，另外一种则是 fcntl.

- flock:

	作用域: 整个文件
	锁的类型:	FL_FLOCK BSD版本
	支持: 建议锁

	flock主要三种操作类型：
	- LOCK_SH，共享锁，多个进程可以使用同一把锁，常被用作读共享锁
	- LOCK_EX，排他锁，同时只允许一个进程使用，常被用作写锁
	- LOCK_UN，释放锁

	在默认情况下，flock 提供是阻塞模式的文件锁（与多线程互斥的阻塞非阻塞类似）,  `flock(open_fd, LOCK_SH | LOCK_NB)`时是非阻塞的.

	> 在非阻塞模式下，加文件锁失败并不影响进程流程的执行，但要注意加入错误处理逻辑，在加锁失败时，不能对目标文件进行操作.

- fcntl:

	作用域: 记录锁可以锁定文件的部分区域甚至一个字节
	锁的类型:	FL_POSIX POSIX版本
	支持: 建议锁/强制锁

实际上，fcntl和flock在对文件加锁的时候，虽然都是在文件的inode上进行上锁，但是flock认为，锁的持有者是打开文件表(系统级)，而fcntl则认为锁的持有者是进程，这就造成了两者在实际使用中的不同.

文件锁适合多进程环境, 但不适合多进程多线程环境.

## FAQ
### 关闭文件系统日志功能
执行`tune2fs -O "^has_journal" /dev/mapper/VolGroup-lv_home`

> dumpe2fs 获取文件系统属性信息, tune2fs 调整文件系统属性, 之后e2fsck 检查文件系统(**几乎大部分都不推荐这样做, 因为可能会丢失数据**)