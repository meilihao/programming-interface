# filesystem
类Unix文件系统是目录和文件的一种层次结构, 像一颗倒置的树, 起点是`/`(root目录).

[linux有FHS(Filesystem Hierarchy Standard,文件系统层次结构)标准](http://refspecs.linuxfoundation.org/fhs.shtml):
![](/misc/img/fs/052049040017593.png)

**目录(directory)**是一种逻辑上包含若干文件的文件. 每个文件都会包含一个文件名(filename)和若干文件属性(文件类型, 大小, 所有者, 权限, 最后访问时间/修改时间等).

创建新目录时会自动创建两个文件名： `.`和`..`, 分别表示当前目
录和父目录; 仅在最高层次的根目录中， 两者指向相同, 都表示当前目录.

**路径名(pathname)**是由`/`分隔的若干文件名序列. 以`/`开头的路径名称为绝对路径(absolute pathname); 否则称为相对路径(relative pathname), 它是指向相对于某个文件的文件. **根目录是特殊的绝对路径, 它不包含文件名**.

## 文件类型
定义在[`#include <sys/stat.h>`](https://en.wikibooks.org/wiki/C_Programming/POSIX_Reference/sys/stat.h)里, 可通过`os.FileMode`进行位操作来判断.

- 普通文件(regular file)
- 目录文件(directory file)
- 块特殊文件(block special file)
- 字符特殊文件(character special file)
- FIFO : 管道, 用于进程间通信. 与socket类似, 以由socket取代. 
- 套接字(socket) : 用于进程间的网络通信
- 符号链接(symbolic link) : 指向另一个文件

可通过`ls -ld xxx`查看.

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

### 设备文件
设备文件让程序能够同系统的硬件和外围设备进行通信. 在Linux中,所有的设备以文件的形式出现在目录 /dev 中, 由`systemd-udevd`管理(随着kernel报告的硬件变化创建或删除设备文件).

设备用主设备号和次设备号表示其特征, 主设备号和次设备号统称为设备号. 主设备号用来表示一个特定的驱动程序, 次设备号用来表示使用该驱动程序的各设备.

`crw-rw-rw-  1 root tty       4,     3 6月  26 21:34 tty3`, `4`是主设备号, `3`是次设备号.

实际上设备文件也分块设备文件(block special file) 和 字符设备文件(character special file).  块设备每次读取或写入一块数据(一组字节, 通常是512的倍数), 字符设备每次读取或写入一个字节.

还有一种有设备驱动但没有实际设备的虚拟设备称为伪设备, 比如pty(伪tty), /dev/null, /dev/random等. 

## inode
Linux 在生成文件的时候，内容会为每一个文件生成一个唯一的索引节点（Inode），文件的属性都会保存在这个Inode中.

## 链接
链接是一种为了共享文件和快速访问而建立起来的文件映射关系, 有**软链接(推荐)**和硬链接之分:
- 硬链接(类似于指针) : 指向目标文件的inode, 因此没有创建新文件.
	1. 只能对已存在的文件进行创建
	1. 硬链接和目标文件必须是同一文件系统(不同的文件系统有不同的inode table)
	1. 为同一个文件创建多个硬链接即多个别名（他们有共同的 inode）
	1. 只有root才能创建执行目录的硬链接(且需要文件系统支持), 这样做是为了避免遍历目录时出现循环.
- 软链接又叫符号链接(symbol links), 类似于Windows下面的快捷键
	1. 可对不存在的文件创建软链接; 允许跨文件系统
	1. 软链接是一个新文件(包含了它所链接的另一个文件的路径)
	1. 符号链接的大小是其链接文件的路径名的字节数

> `ln`命令创建硬链接会增加链接数，`rm`命令会减少链接数.一个文件除非链接数为0，否则不会从文件系统中被物理地删除.
> 可使用`readlink`命令读取链接
> 这两种链接的本质区别关键点在于inode
> 针对目录的软链接删除: `rm -rf symbol_name(删除软链接symbol_name)` 和 `rm -rf symbol_name/(仅删除symbol_name目录下的所有文件, 其他不变)`

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

## lost+found
fsck(文件系统一致性检查工具)找到一个无法确定其父目录的文件时就会将其放入其中.

## vfs
虚拟文件系统(Virtual Files System, VFS) 是驻留在用户进程和各种类型的linux 文件系统之间的一个抽象接口层.
vfs提供了访问文件系统对象的通用对象模型(比如索引节点, 文件对象, 分页缓存, 目录条目等等)和方法, 对用户进程隐藏了实现每个文件系统的差异.

脏页(dirty page): 磁盘上的数据与内存中的数据不一致, 此时内存中的数据就是脏页, 需要尽快同步到磁盘中, 以防止意外丢失.

[块层](https://zhuanlan.zhihu.com/p/25096747)处理所有与块设备操作相关的活动, 其关键数据结构是bio(block input output)结构. 它处理bio请求, 并放入`i/o`请求队列.
bio结构是在文件系统层和块层之间的一个接口.

## 特殊文件系统

### /sys
`/sys`是一种在内存中的虚拟文件系统sysfs的mount point, 提供了有关系统上可用设备, 设备的配置及其状态的信息. sysfs的指定原则之一是`/sys`里的每个文件都只表示下层设备的一个属性.

可通过`udevadm`查询设备信息, 触发事件, 控制vdevd守护进程, 以及监视udev和内核的事件, 比如查看ssd信息`udevadm info -a -n nvme0n1`.

`/sys`目录:
- block : 有关硬盘之类的块设备信息
- bus : 总线: pci-e, scsi, usb等
- class : 按设备的功能类型(比如声卡, 显卡, 输入设备, 网卡)组织的一颗树
- dev : 区分块设备和字符设备的设备信息
- devices : 正确表示所有找到的设备
- firmware : 特定于平台的子系统的接口, 如ACPI
- fs : 内核知道的一些但不是全部文件系统的目录
- hypervisor
- kernel : 内核的内部信息, 比如高速缓存和虚拟内存状态
- module : 内核动态加载的模块
- power : 系统电源状态的几种详细信息

### /proc
proc fs并不是真正的文件系统. 它不存储数据, 但提供了访问内核的接口, 内核产生的所有状态信息和统计信息均在里面.

大多数linux工具都依赖`/proc`提供的信息进行性能监控. 比如ps和top均从`/proc`读取进程状态信息, 再比如vmstat, cpuinfo等.

## /proc
`/proc`文件系统是一种虚拟文件系统,以文件系统目录和文件形式,提供一个指向内核数据结构的接口.

`/proc`的子目录:
- acpi : 大多数现代桌面和笔记本支持的高级配置和电源接口. acpi主要是pc技术, 服务器上通常被禁用.
- bus : 包含总线子系统的信息, 比如pci总线或各自系统的usb接口
- irq : 系统中的中断信息. 每个子目录是一个中断, 并可能是一个连接的设备, 比如一个网卡接口. 这里可修改一个特定中断的cpu亲和力.
- net : 网络接口的大量原始统计, 比如接收的组播数据包或每个接口的路由.
- scsi : scsi子系统信息, 比如连接的设备或驱动程序版本.
- sys : 可调整的内核参数, 比如虚拟内存管理器或网络协议栈的行为.

	- abi : 文件与应用程序的二进制信息
	- dev : 特定设备的信息
	- fs : 特定的文件系统
	- kernel : 可控制一系列的内核参数
	- net : 内核网络部分的调整
	- vm : 内存管理调优, buffer和cache管理
- tty : 各个虚拟终端和与它连接的物理设备信息.
- `/proc/${pid}`

	- cmdline : 进程的完整命令行(以null分隔)
	- cwd : 链接到进程当前工作目录的符号链接
	- environ : 进程的环境变量(以null分隔)
	- exe : 链接到进程可执行文件的符号链接
	- fd : 包含链接到进程每个打开文件的符号链接. 链接到的管道和socket没有关联的文件名, 但有具体id
	- fdinfo : 进程中文件描述符的相关信息
		- pos 字段 : 当前的文件偏移量
		- flags : 为一个八进制数,表征文件访问标志和已打开文件的状态标志
	- maps : 内存映射信息(共享段, lib等)
	- root : 链接到进程的根目录(由chroot设置)的符号链接
	- state : 进程的总状态信息(ps可解析该信息)
	- statm : 内存使用情况的信息

## /dev/fd
对于每个进程,内核都提供有一个特殊的虚拟目录/dev/fd. 该目录中包含`/dev/fd/${n}`形式的文件名,其中 n 是与进程中的打开文件描述符相对应的编号.

打开/dev/fd 目录中的一个文件等同于复制相应的文件描述符.

> /dev/fd 实际上是一个符号链接,链接到 Linux 所专有的/proc/self/fd 目录.
> `ls | diff /dev/fd/0 oldlist`取代`ls | diff - oldlist`, 以避免部分命令未实现`-`(表示stdin/stdout)或部分命令使用`-`作为命令行选项结束的分隔符.