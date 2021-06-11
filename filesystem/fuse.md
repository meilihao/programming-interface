# fuse
参考:
- [FUSE协议解析](http://blog.mingforpc.me/2018/11/30/FUSE%E5%8D%8F%E8%AE%AE%E8%A7%A3%E6%9E%90/)
- [libfuse文档](https://libfuse.github.io/doxygen/index.html)

FUSE的全称是Filesystem in Userspace，即用户空间文件系统，是系统内核提供的一个功能，使得可以在用户态下实现一个自定义的文件系统. 比如CEPH和GlusterFS等都有使用到FUSE.

use包含包含一个内核模块和一个用户空间守护进程. fuse 两种开发模式：
1. high-level 模式，入口函数为 fuse_main，封装了一系列初始化操作，使用简单，但不灵活
1. 另一种是low-level模式，可以利用 fuse 提供的底层函数灵活开发应用程序

FUSE分为三大模块：
- FUSE内核模块(内核态)

    FUSE 内核模块实现 VFS 接口（fuse文件驱动注册、supper block、dentry、inode的维护），接收请求传递给LibFUSE，LibFUSE 再传递给用户程序的接口进行操作
- LibFUSE模块(用户态)

    LibFUSE 实现文件系统主要框架、对实现的文件系统操作进行封装、mount管理、通过设备/dev/fuse与内核模块通信
- 用户程序模块(用户态)

    FUSE需要把VFS层的请求传到用户态的fuseapp，在用户态处理，然后再返回到内核态，把结果返回给VFS层

## 原理
参考:
- [libfuse 源码分析](https://blog.csdn.net/zhonglinzhang/article/details/104262658)
- [用户空间文件系统FUSE源码解析](https://www.geek-share.com/detail/2499981020.html)

程序需要先打开/dev/fuse，然后通过mount()将/dev/fuse的fd，进程的用户id和组id传入，进行文件系统的挂载.

> libfuse通过自己写的fusermount程序(编译安装libfuse后会在/bin/下)，可以让我们实现的文件系统程序在非root权限下挂载.

主要流程：
fuse_fs_init(); //用于截获用户的文件系统调用，包括open/read/write等等
fuse_dev_init(); //创建设备文件/dev/fuse，这个文件主要是用来完成内核和用户空间的通信
fuse_sysfs_init(); //注册fuse的信息到sysfs
fuse_ctl_init(); //创建一个控制文件系统，用于查看某个连接的请求情况或者强制结束一个连接。

初始化函数给了我们一些基本的概念，下面介绍一下fuse实现的基本原理。

首先，我们用libfuse开发的用户空间文件实现，其实只是实现了一个文件系统调用的回调函数。（怎么使用libfuse开发用户空间文件系统可以参考其demo）
当我们运行我们的实现程序时，libfuse内部会进行mount操作，并且挂载的文件系统类型就是fuse，那么在挂载目录下的文件访问活动都会被fuse_fs_init函数中注册的fuse文件系统捕获并处理。

fuse文件系统是在内核空间捕获到这些文件调用的（事实上就是vfs根据挂载点信息找到并分派给它的）。它怎么把请求传递到用户空间，供我们的回调钩子函数处理呢？
这里fuse利用了fuse_dev_init这个函数注册的设备/dev/fuse。
首先，fuse fs捕获到文件操作请求后，直接将请求写入一个内核请求队列，然后在一个事件上等待，直到请求结束时wakeup。其次，我们重新考察libfuse的实现。在我们运行我们用户空间的自定义文件系统时，libfuse除了前面说的进行mount之外，还在用户空间打开了/dev/fuse这个设备文件，并启动一个线程循环地读取/dev/fuse设备的数据。而用户空间对/dev/fuse的read调用，转化为内核空间fuse dev的ops->read，即fuse_dev_read。这个函数直接从前面提到的fuse fs写入的请求队列读取请求，返回给用户空间即可。这时，libfuse在用户空间获得了请求，并通过回调调用用户实现的钩子函数来处理请求。
用户处理完请求之后的结果，又通过写入设备/dev/fuse的方式传入到内核空间，fuse_dev_write函数响应用户的反馈，将相应的数据交给原先等待的fuse fs请求，fuse fs再返回到vfs，最终反馈给用户空间的调用者

## 调用过程
以open为例，整个调用的过程如下:
1. 用户态app调用glibc open接口，触发sys_open系统调用
1. sys_open 调用fuse中inode节点定义的open方法
1. inode中open生成一个request消息，并通过/dev/fuse发送request消息到用户态libfuse
1. Libfuse调用fuse_application用户自定义的open的方法，并将返回值通过/dev/fuse通知给内核
1. 内核收到request消息的处理完成的唤醒，并将结果放回给VFS系统调用结果
1. 用户态app收到open的返回结果

## FAQ
### 查看fuse version
`fusermount -V`或`dpkg --get-selections | grep fuse`

### 使用fuse3
`sudo apt install fuse3 libfuse3-dev`

### fuse_operations
在[`fuse.h`](http://libfuse.github.io/doxygen/structfuse__operations.html)或[libfuse的fuse.h](https://github.com/libfuse/libfuse/blob/master/include/fuse.h#L302)

在fuse_operations中所有的方法都是可选的.

[各个操作的触发情况](https://github.com/mingforpc/fuse-go/blob/master/doc/%E5%90%84%E4%B8%AA%E6%93%8D%E4%BD%9C%E7%9A%84%E8%A7%A6%E5%8F%91%E6%83%85%E5%86%B5.md):

* `getattr()`: 用于查询节点是否存在、查询节点属性等动作

    * 当调用系统`stat()`fuse程序目录中的文件时，会对Ino为1的文件（fuse根目录）调用`getattr()`，会有缓存。

* `lookup()`: 对应着系统函数`stat()`，会对`stat()`函数的各级目录及最后的文件依次调用`lookup()`。

* `readdir()`: 获取目录的内容，`readdir()`会在读取目录的时候触发，并且需要`releasedir()`释放打开。获取目录文件下内容的操作顺序一般为`opendir()` -> `readdir()` -> `releasedir()`，操作系统一般会对获取到的文件调用`lookup()`的操作。

* `fsyncdir()`: 同步目录的内容，一般是文件夹对象调用`sync()`函数（封装后），或者是系统函数`fsync()`。

* `mkdir()`: 创建文件夹，一般系统会先调用`lookup()`检查文件是否存在。

* `rmdir()`: 删除目录，一般系统会先调用`lookup()`检查文件夹是否存在和是否是文件夹。部分语言（比如golang，所以单元测试中是调用`syscall.rmdir()`）封装的文件删除，会先调用`unlink()`再调用`rmdir()`，并且会检查`unlink()`返回的错误是否是`ENOTDIR`。

* `setxattr()`: 设置文件的扩展属性，一般系统会先调用`lookup()`检查文件是否存在，但不会打开文件。

* `getxattr()`: 获取文件的扩展属性，一般系统会先调用`lookup()`检查文件是否存在，但不会打开文件(PS: `lookup()`会有缓存的，比如`setxattr()`前调用了`lookup()`，后面的`getxattr()`就不需要了)。

* `listxattr()`: 列出文件所有扩展属性名，一般系统会先调用`lookup()`检查文件是否存在，但不会打开文件(PS: 同上)。

* `removexattr()`: 删除文件指定的扩展属性，一般系统会先调用`lookup()`检查文件是否存在，但不会打开文件(PS: 同上)。

* `symlink()`: 创建软连接。

* `readlink()`: 读取软连接文件指向的路径。

* `mknod()`: 创建一个文件，一般系统会先调用`lookup()`检查文件是否已经存在。

* `unlink()`: 删除文件。

* `link()`: 创建硬链接。

* `open()`: 打开文件。

* `read()`: 读取文件内容。

* `write()`: 写文件内容，注意`offset`偏移量。

* `fsync()`: 同步文件内容，将文件同步到“硬盘”上。

* `flush()`: 和`fsync()`是不同的，它会在文件`close`的时候调用，主要是为了返回写错误和删除文件锁。因为文件描述符可以被复制，比如(dup, dup2, fork)，每次`close`都会调用`flush()`，所以可能会被调起多次。

* `release()`: 关闭释放文件描述符时调用。和`flush()`不用，每个打开的文件只有当其没有任何引用才会调起`release()`，所以对于打开的文件只会调用1次。

* `statfs()`: 获取文件系统信息，详见[http://man7.org/linux/man-pages/man2/statfs.2.html](http://man7.org/linux/man-pages/man2/statfs.2.html)

* `access()`: 检查文件权限。如果mount时设置了`default_permissions`选项，或者fuse初始化时设置了`FUSE_CAP_POSIX_ACL`，此方法将不会被调用。PS: 如果`access()`返回错误码`ENOSYS`，那么之后将不会调用`access()`。

* `create()`: 创建并打开文件。如果没有实现，将会调用`mknod()`和`open()`代替。PS: 如果`create()`返回错误码`ENOSYS`，那么之后将不会调用`create()`。

* `bmap()`: 仅当是块设备是触发，`mount()`的时候需要参数`blkdev`和`fsname={块设备路径}`

## FAQ
### entry_timeout和attr_timeout
在文件系统实现中增大返回参数entry_timeout和attr_timeout的值，这会增加对应文件在内核缓存中的过期时间，从而减少元数据的交互次数