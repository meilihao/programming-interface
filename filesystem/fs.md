# filesystem
类Unix文件系统是目录和文件的一种层次结构, 像一颗倒置的树, 起点是`/`(root目录).

[linux有FHS(Filesystem Hierarchy Standard,文件系统层次结构)标准](http://refspecs.linuxfoundation.org/fhs.shtml):
![](/images/fs/052049040017593.png)

**目录(directory)**是一种逻辑上包含若干文件的文件. 每个文件都会包含一个文件名(filename)和若干文件属性(文件类型, 大小, 所有者, 权限, 最后访问时间/修改时间等).

创建新目录时会自动创建两个文件名： `.`和`..`, 分别表示当前目
录和父目录; 仅在最高层次的根目录中， 两者指向相同, 都表示当前目录.

**路径名(pathname)**是由`/`分隔的若干文件名序列. 以`/`开头的路径名称为绝对路径(absolute pathname); 否则称为相对路径(relative pathname), 它是指向相对于某个文件的文件. **根目录是特殊的绝对路径, 它不包含文件名**.

## 文件类型
- 普通文件(regular file)
- 目录文件(directory file)
- 块特殊文件(block special file) : 提供对设备(比如磁盘)带缓冲的访问, 每次访问以固定长度为单位进行.
- 字符特殊文件(character special file) :
- FIFO : 管道, 用于进程间通信. 与socket类似, 以由socket取代. 
- 套接字(socket) : 用于进程间的网络通信
- 符号链接(symbolic link) : 指向另一个文件

可通过`ls -ld xxx`查看.

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

![](/images/fs/7b670449a3faa6a2cdab45c2298b66dd.png)

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

## 系统调用
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