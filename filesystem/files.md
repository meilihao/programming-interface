# files
linux文件系统结构引用了[Linux Foundation Referenced Specifications](https://refspecs.linuxfoundation.org/)的[Filesystem Hierarchy Standard Specifications Archive](https://refspecs.linuxfoundation.org/fhs.shtml)

## index
- /boot : 启动linux所需的文件
	- initrd.img-5.4.0-39-generic : initramfs
- /bin : /usr/bin的软连接
- /dev : 包含系统所有的设备文件
- /dev/audio* : 声卡
- /dev/sd* : scsi 磁盘
- /dev/lp* : 并行串口
- /dev/pty* : 网络中登录的远程终端设备
- /dev/ram* : 系统内存
- /dev/null : 空设备
- /dev/console : 系统控制台, 可直接连接到显示器
- /dev/ttS* : 代表串行端口. 类似windows下的COM
- /dev/tty* : linux上的虚拟控制台
- /etc : 存放系统管理相关的配置
- /etc/crontab : 系统定时任务的配置
- /etc/fstab : 开机启动时自动挂载的分区列表
- /etc/group : 保存用户组的信息
- /etc/hosts : hostname配置
- /etc/passwd : 用户登录信息保存位置
- /etc/resolv.conf : 本地dns配置
- /etc/rsyslog.conf : 系统日志的配置
- /etc/profile : 系统全局环境变量的配置
- /etc/services : 常用服务与端口的对应关系
- /etc/shadow : 保存用户密码
- /etc/sysctl.conf : 系统kernel配置, 已转移到`/usr/lib/sysctl.d`, 但sysctl.conf任有效, 且可覆盖`/usr/lib/sysctl.d`的配置
- /etc/systemd : systemd的配置目录
- /etc/systemd/system/*.wants : 所有服务的启动脚步
- /etc/X11 : x-window的配置文件
- /home : 存放每个用户的主目录
- /lib* : /usr/lib*的软连接
- lost+found : 保存丢失的文件, 不恰当的关机操作和磁盘错误均会导致文件丢失, 这些会丢失的文件会临时放在这里. 系统重启后, 引导运行fsck时就会发现这些文件. 每个分区均有lost+found.
- /mnt : 用于挂载移动设备
- /root: root的主目录
- /run : 外部设备的自动挂载点
- /tmp : 存放临时文件
- /usr : 存放应用和文件
- /usr/lib64和/usr/local/lib64 : 64位系统中的函数库目录
- /usr/src : 包含所有应用程序的源码, 主要是linux 核心生态程序的源码
- /usr/local : 存放本地安装的软件和其他文件, 与linux系统无关
- /usr/lib和/usr/local/lib : 32位系统中的函数库目录
- /usr/bin和/usr/local/bin : 用户可用的可执行程序
- /usr/sbin和/usr/local/sbin : 管理员才可用的可执行程序
- /usr/include : 包含c语言的头文件
- /usr/share : 存放共享的文件和数据库
- /var : 存放系统运行以及软件运行的日志信息
- /var/log : 存放各种应用的日志文件, **需定期清理**
- /var/lib : 存放系统正常运行时需要改变的库文件
- /var/spool : 是mail, new, 打印机队列和其他队列输入, 输出的缓冲目录
- /var/tmp : 允许比/tmp存放更大的文件
- /var/lock : 存放被锁定的文件, 很多应用都会在/var/lock下生成一个锁文件, 以保证其他应用不会同时使用某些资源
- /var/local : 存放/usr/local中所安装应用的可变数据
- /var/account : 存放格式化的man页
- /var/run : 包含到下次系统启动前的系统信息
- /sbin : /usr/sbin的软连接

# boot
## initrd.img-$(uname -r)
基于内存的文件系统, 因为内存访问是不需要驱动的.

linux 发行版必须适应各种不同的硬件架构，将所有的驱动编译进内核是不现实的, 因此它们在内核中只编译基本的硬件驱动, 其他各种不同的硬件驱动放在initramfs中, 是一种即可行又灵活的解决方案.

在 boot loader 配置了 initrd 的情况下，内核启动被分成了两个阶段，第一阶段先执行 initrd 文件系统中的某个文件，完成加载驱动模块等任务，第二阶段才会执行真正的根文件系统中的 /sbin/init 进程.

Initrd 的主要用途：
1. linux 发行版和livecd的必备部件, 用于加载驱动.
1. 定制化启动过程, 比如 bootsplash.
1. 任何kernel不能做的，但在用户态可以做的 (比如执行某些命令)

# etc

## /etc/group
保存用户组的信息

格式:
```
组名:用户组密码:GID:用户组内的用户名
```

`password`部分被`x`替代表示没有密码. 密码默认保存在`/etc/gshadow`中, 可通过gpasswd来设置.

某个用户的GID字段表示"初始用户组";用户组内的用户名表示"支持用户组",而且在"支持用户组"列表中第一个出现的组名也叫"有效用户组"(通常也是初始用户组).
> 有效用户组就是用`newgrp`命令(要切换的用户组必须在支持用户组里)所切换到的用户组; 如果一次也没有使用newgrp那么有效用户组就是初始用户组
> linux支持使用gpasswd设置组口令, 可在newgrp中使用该密码.

使用`groups`命令可以查看当前用户的所有支持用户组.

## /etc/passwd
用户登录信息保存位置

格式(7个以冒号分隔的字段):

`登录名:加密后的登录密码(在[/etc/shadow](shadow.md)里):UID:GID:注释:用户home目录:用户登录shell`

比如:
```
...
chen:x:1000:1000::/home/chen:/usr/bin/fish
...
```

`password`部分被`x`替代表示密码实际存储在`/etc/shadow`.

`login shell`中的`/usr/sbin/nologin`和`/bin/false`:

- `/usr/sbin/nologin`,礼貌地拒绝登录(会显示一条提示信息),但可以使用其他服务,比如ftp.
- `/bin/false`,什么也不做只是返回一个错误状态,然后立即退出.即用户会无法登录,并且不会有任何提示,是最严格的禁止login选项，一切服务都不能用.

> 如果存在`/etc/nologin`文件,则系统**只允许root用户**登录,其他用户全部被拒绝登录,并向他们显示`/etc/nologin`文件的内容
> `/etc/shells`保存了支持的所有shell, 可使用`chsh`修改用户自身的shell.

## /etc/shadow
保存用户密码

字段:
1. 登录名, **非空**
1. 加密后的口令, **非空**
1. 上次修改口令的时间
1. 两次修改口令之间的最少天数
1. 两次修改口令之间的最多天数
1. 提前多少天警告用户口令即将过期
1. 在口令过期后多少天禁用账号
1. 账号过期的日期(从1970.1.1开始的天数), 为空表示不过期
1. 保留字段, 目前为空.

## /etc/login.defs
定义创建用户时的默认设置. 比如指定用户uid和gid的范围, 用户的过期时间, 是否创建用户主目录等.

## /etc/default/useradd
定义新建用户的一些默认属性, 比如用户的主目录, 使用的shell等. 命令`useradd -D`可修改该文件.

## /etc/skel
定义了新建用户在主目录下的默认配置文件, 比如`.bashrc`, `.bash_logout`,`模板`等.

## FAQ
### passwd与usermod锁用户及用户解锁的区别
passwd -l user 后shadow中user的密码前增加2个`!`
passwd -u user 将去除shadow中user密码前的2个`!`
usermod -L user 后shadow中user的密码前增加1个`!`
usermod -U user 将去除shadow中user密码前的1个`!`
