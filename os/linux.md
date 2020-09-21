# linux
参考:
- [kernel changelog](https://kernelnewbies.org/LinuxVersions)
- [Linux and glibc API changes](https://man7.org/tlpi/api_changes/index.html)
- [Linux Performance tools map](http://www.brendangregg.com/linuxperf.html)
- [linux kernel map](https://makelinux.github.io/kernel/map/)

## ko
Linux内核是单内核（monolithic kernel），也就是所有的内核功能都集成在一个内核空间内. 但是kernel内核又有微内核的设计即具有模块功能，可以将磁盘驱动程序、文件系统等独立的内核功能制作成模块，并动态添加到内核空间或者删除.

内核模块是可以动态添加到Linux内核空间的二进制文件，文件扩展名为ko.

## doc
- [The Linux Kernel的翻译](https://www.kernel.org/doc/html/latest/translations/zh_CN/index.html)

## kernel git tree
- mainline : 由Linus Torvalds亲自制作的内核发布版，是官方当前最新版本的kernel source
- stable : 稳定分支
- longterm : 长期维护分支
- linux-next : 根据设计 linux-next 提前包含了下一个合并窗口要合并的patch，理论上应该是下一个合并窗口关闭之后主线应该要成为的样子.

    当Linus发布一个Mainline主线内核时，一个为期2周左右的主线合并窗口就会打开，在此期间，mainline分支会从linux-next以及各个子模块的维护者处接收合并patch，当合入一些patch后，就会形成下一个版本的rc候选版本，一般会经历多个rc版本，等待时机成熟，就会发布下一个版本的Mainline内核.

## header
参考:
- [ Cross-Compiled Linux From Scratch - 5.3. Linux-3.14.21 Headers](http://www.clfs.org/view/CLFS-3.0.0-SYSTEMD/x86_64-64/cross-tools/linux-headers.html)

构建kernel header:
```sh
$ git clone -b v5.6-rc7 --depth=1 https://mirrors.tuna.tsinghua.edu.cn/git/linux-stable.git
$ cd linux-stable
$ make mrproper
$ make ARCH=x86_64 headers_check # ARCH默认是当前系统的arch
$ make ARCH=x86_64 INSTALL_HDR_PATH=../kernel_header headers_install # headers_install将适用于用户空间的内核headers(即linux发行版上的linux-headers软件包)导出, 因此没有内核task_struct的定义
```

基本的头文件在`include`下, 比如`<linux/inotify.h>`对应`include/linux/inotify.h`
arch相关的头文件在`arch/<arch>/include/asm`下, 比如x86_64的`<asm/ioctl.h>`在`arch/x86/include/generated/uapi/asm/ioctl.h`(ioctl.h需要构建出来, uapi参考下面连接)

> [Linux kernel uapi header file（用户态头文件, from 3.7）](https://lwn.net/Articles/507794/):为了解决include recursive（循环包含头文件）的问题及减少与简化kernel-only header的size.

注意: `kernel:arch/xxx/include/asm/...`下的头文件是对应`kernel:include/asm-generic/...`下的平台相关实现，若arch目录下没有相同的头文件，则使用asm-generic目录下的，arch目录下的头文件可能直接include asm-generic目录下的相关头文件.

### syscall
```sh
$ grep -r __NR_socket kernel_header/include |grep -v "socketcall" |grep -v "socketpair"
asm/unistd_32.h:#define __NR_socket 359 # unistd_32.h是x86的syscall
asm/unistd_64.h:#define __NR_socket 41 # unistd_64.h是x86_64的syscall
asm/unistd_x32.h:#define __NR_socket (__X32_SYSCALL_BIT + 41) # unistd_x32.h是x32的syscall
asm-generic/unistd.h:#define __NR_socket 198 # asm-generic/unistd.h 是 统一预设的syscall, 用于没有特殊指定syscall的arch syscall, 比如arm64
asm-generic/unistd.h:__SYSCALL(__NR_socket, sys_socket)
```

> [所有arch的syscall number](https://fedora.juszkiewicz.com.pl/syscalls.html)

```sh
# include合并了linux 5.6和glic 2.31的headers, 因此该目录等价于/usr/include
# 从时间上分析, unistd.h大致分两种, 结合kernel包含的`asm-generic/unistd.h`, 因此断定`Mar 23 18:26`的是glibc, 而`Mar 23 17:59`为kernel的.
$ mkdir include && cd include
$ cp -rpd ../kernel_header/include/* .
$ cp -rpd ../glibc_install/include/* .
$ ls -alt `find . |grep "unistd"`
-rw-r--r-- 1 chen chen  1613 Mar 23 18:26 ./bits/unistd_ext.h
-rw-r--r-- 1 chen chen 13316 Mar 23 18:26 ./bits/unistd.h
-rw-r--r-- 1 chen chen    20 Mar 23 18:26 ./sys/unistd.h
-rw-r--r-- 1 chen chen 42804 Mar 23 18:26 ./unistd.h
-rw-r--r-- 1 chen chen   220 Mar 23 17:59 ./linux/unistd.h
-rw-r--r-- 1 chen chen 30514 Mar 23 17:59 ./asm-generic/unistd.h
-rw-r--r-- 1 chen chen 11529 Mar 23 17:59 ./asm/unistd_32.h
-rw-r--r-- 1 chen chen  9340 Mar 23 17:59 ./asm/unistd_64.h
-rw-r--r-- 1 chen chen   361 Mar 23 17:59 ./asm/unistd.h
-rw-r--r-- 1 chen chen 16465 Mar 23 17:59 ./asm/unistd_x32.h
$ cat ./linux/unistd.h |grep "include"
#include <asm/unistd.h>
$ cat ./asm/unistd.h |grep "include"
#  include <asm/unistd_32.h>
#  include <asm/unistd_x32.h>
#  include <asm/unistd_64.h>
$ cat ./sys/unistd.h |grep "include"
#include <unistd.h>
$ cat ./unistd.h |grep "include"
...
# include <bits/unistd.h>
#include <bits/unistd_ext.h>
```

各unistd.h区别???

> 系统标准头文件位置： /usr/include下，以及安装库的头文件位置：/usr/local/include/.

系统调用和异常处理程序是kernel明确定义的接口. 进程只有通过这些接口才能陷入内核执行即对内核的所有访问都必须通过这些接口.

## glibc
glibc是gnu发布的libc库，也即c运行库. glibc是linux系统中最底层的api（应用程序开发接口），几乎其它任何的运行库都会倚赖于glibc. glibc除了封装linux操作系统所提供的系统服务外，它本身也提供了许多其它一些必要功能服务的实现，主要的如下：
（1）string，字符串处理
（2）signal，信号处理
（3）dlfcn，管理共享库的动态加载
（4）direct，文件目录操作
（5）elf，共享库的动态加载器，也即interpreter
（6）iconv，不同字符集的编码转换
（7）inet，socket接口的实现
（8）intl，国际化，也即gettext的实现
（9）io
（10）linuxthreads
（11）locale，本地化
（12）login，虚拟终端设备的管理，及系统的安全访问
（13）malloc，动态内存的分配与管理
（14）nis
（15）stdlib，其它基本功能

### glibc 和 libc
glibc 和 libc 都是 Linux 下的 C 函数库. libc 是 Linux 下的 ANSI C 函数库；glibc 是 Linux 下的 GUN C 函数库.

ANSI C 函数库是基本的 C 语言函数库，包含了 C 语言最基本的库函数:

    <ctype.h>：包含用来测试某个特征字符的函数的函数原型，以及用来转换大小写字母的函数原型；
    <errno.h>：定义用来报告错误条件的宏；
    <float.h>：包含系统的浮点数大小限制；
    <math.h>：包含数学库函数的函数原型；
    <stddef.h>：包含执行某些计算 C 所用的常见的函数定义；
    <stdio.h>：包含标准输入输出库函数的函数原型，以及他们所用的信息；
    <stdlib.h>：包含数字转换到文本，以及文本转换到数字的函数原型，还有内存分配、随机数字以及其他实用函数的函数原型；
    <string.h>：包含字符串处理函数的函数原型；
    <time.h>：包含时间和日期操作的函数原型和类型；
    <stdarg.h>：包含函数原型和宏，用于处理未知数值和类型的函数的参数列表；
    <signal.h>：包含函数原型和宏，用于处理程序执行期间可能出现的各种条件；
    <setjmp.h>：包含可以绕过一般函数调用并返回序列的函数的原型，即非局部跳转；
    <locale.h>：包含函数原型和其他信息，使程序可以针对所运行的地区进行修改。
    地区的表示方法可以使计算机系统处理不同的数据表达约定，如全世界的日期、时间、美元数和大数字；
    <assert.h>：包含宏和信息，用于进行诊断，帮助程序调试

glibc是linux下面c标准库的实现，即GNU C Library. **glibc本身是GNU旗下的C标准库，后来逐渐成为了Linux的标准c库，而Linux下原来的标准c库Linux libc逐渐不再被维护**. Linux下面的标准c库不仅有这一个，如uclibc、klibc，以及上面被提到的Linux libc，但是glibc无疑是用得最多的. **glibc在/lib目录下的.so文件为libc.so.6**.

查看glibc version有两种方法:
```sh
$ /lib/x86_64-linux-gnu/libc.so.6
$ ldd --version
```

### 编译glibc
```sh
$ git clone -b glibc-2.31 --depth=1  https://mirrors.tuna.tsinghua.edu.cn/git/glibc.git
$ mkdir -v /glibc
$ cd glibc
$ mkdir build # glibc要求不能在其源码的root目录上编译
$ cd build
$ cat ../INSTALL # 查看编译要求
$ ../configure -h # 查看编译选项
$ ../configure --prefix=/glibc --with-headers=../../kernel_header/include
$ make
$ sudo make install
```

## kernel 5.6
参考:
- [linux 内核裁剪与编译](https://www.jianshu.com/p/3e5f9bc0aa54)
- [linux kernel data structures](https://docs.huihoo.com/doxygen/linux/kernel/3.7/annotated.html)
- [linux kernel 在线阅读](https://elixir.bootlin.com/linux/latest/source)或[kernel on sourcegraph](https://sourcegraph.com/github.com/torvalds/linux@v5.8-rc3)

各目录说明
- arch：包含和硬件体系结构相关的代码，每种平台占一个相应的目录，如 i386、arm、arm64、powerpc、mips 等. Linux 内核目前已经支持30种左右的体系结构.

    在 arch 目录下，存放的是各个平台以及各个平台的芯片对 Linux 内核进程调度、内存管理、中断等的支持，以及每个具体的 SoC 和电路板的板级支持代码.
    
    > [各个处理器的介绍](https://blog.csdn.net/u014379540/article/details/52263342)
- block：块设备I/O层(驱动程序,I/O调度等)
- certs : 认证相关
- crypto：常用加密和散列算法（如AES、SHA等)，还有一些压缩和 CRC 校验算法
- Documentation：内核各部分的通用解释和注释
- drivers：设备驱动程序每个不同的驱动占用一个子目录，如 char、block、net、mtd、 i2c 等

    - accessibility – 可访问设备，目前里面包括盲人设备
    - acpi – 高级配置和电源管理接口
    - amba – ARM研发的AMBA片上总线相关
    - android – Android平台支持的设备
    - ata – 硬盘接口技术Advanced Technology Attachment相关驱动
    - atm – 异步传输模式设备驱动
    - auxdisplay – 辅助显示设备驱动
    - base
    - bcma – Broadcom基于amba总线驱动上开发的
    - block – 块设备驱动
    - bluetooth – 蓝牙设备驱动
    - bus –
    - cdrom – CDROM设备驱动
    - char – 字符设备驱动
    - clk – 时钟驱动框架，与平台相关
    - clocksource – 时钟源设备驱动
    - connector – 内核空间和用户空间通信的新机制连接器
    - cpufreq – cpu动态变频
    - cpuidle –
    - crypto – 加解密设备驱动
    - dax – 直接访问，后面的X只是为了看起来酷，对于新兴的NVDIMM设备，可直接访问此设备上的文件系统
    - dca – 直接缓存访问??
    - devfreq – 设备相关的频率调节
    - dio – dio设备驱动
    - dma – dma设备驱动
    - dma-buf – 提供DMA缓存的设备驱动
    - edac – 错误检测和纠正设备驱动
    - eisa –
    - extcon – 外部连接器
    - firewire – IEEE1394 firewire设备驱动
    - firmware –
    - fpga – FPGA框架驱动
    - fsi –
    - gnss -
    - gpio – GPIO驱动，与处理器相关
    - gpu – 包括DRM图形渲染架构，访问图形界面的DMA引擎，IMX的IPU图像处理单元等
    - greybus -
    - hid – 人机交互设备驱动
    - hsi – 高速同步串口接口
    - hv – 微软的虚拟化技术驱动
    - hwmon – 硬件监控芯片驱动，监控类传感器的芯片驱动
    - hwspinlock – 硬件自旋锁框架接口
    - hwtracing – 硬件跟踪调试驱动
    - i2c – i2c子系统总线驱动
    - i3c -
    - ide – 管理ATA/IDE和ATAPI单元，主要还是硬盘驱动IDE和CD-ROM驱动ATAPI
    - idle – Intel处理器的idle处理驱动
    - iio – 工业I/O子系统，包括各种使用不同物理接口(i2c, spi, etc)传感器的驱动
    - infiniband – 支持多并发链接的”转换线缆”技术的硬件设备驱动
    - input – input设备子系统，包括各种输入设备的驱动，键盘、混合设备、鼠标、触摸屏、游戏接口、游戏操作杆、触控rmi4、触摸面板、串口IO输入设备

        - keyboard 键盘驱动
    - interconnect
    - iommu
    - ipack
    - irqchip
    - isdn
    - Kconfig
    - leds
    - lightnvm
    - macintosh
    - mailbox
    - Makefile
    - mcb
    - md
    - media
    - memory
    - memstick
    - message
    - mfd
    - misc
    - mmc
    - mtd
    - mux
    - net
    - nfc
    - ntb
    - nubus
    - nvdimm
    - nvme
    - nvmem
    - of
    - opp
    - oprofile
    - parisc
    - parport
    - pci
    - pcmcia
    - perf
    - phy
    - pinctrl
    - platform
    - pnp
    - power
    - powercap
    - pps
    - ps3
    - ptp
    - pwm
    - rapidio
    - ras
    - regulator
    - remoteproc
    - reset
    - rpmsg
    - rtc
    - s390
    - sbus
    - scsi
    - sfi
    - sh
    - siox
    - slimbus
    - soc
    - soundwire
    - spi
    - spmi
    - ssb
    - staging
    - target
    - tc
    - tee
    - thermal
    - thunderbolt
    - tty
    - uio
    - usb
    - vfio
    - vhost
    - video
    - virt
    - virtio
    - visorbus
    - vlynq
    - vme
    - w1
    - watchdog
    - xen
    - zorro
- fs：vfs和所支持的各种文件系统，如EXT、FAT、NTFS、JFFS2等
- include：内核 API 级別头文件，与系统相关的头文件放置在 include/linux 子目录下
- init：内核初始化代码著名的 stait_kemel() 就位于 init/main.c 文件中
- ipc：linux支持的进程间通信的实现
- kernel：内核最核心的部分，包括进程调度、定时器等，而和平台相关的一部分代码放在 arch/*/kemel 目录下
- lib：库文件代码
- mm：内存管理代码，和平台相关的一部分代码放在arch/*/mm目录下
- net：网络相关代码，实现各种常见的网络协议
- samples : 示例代码
- scripts：用于配置和编译内核的脚本文件
- security：linux安全模块,主要是一个 SELinux 的模块
- sound：ALSA、OSS 音频设备的驱动核心代码和常用设备驱动
- tools : 在linux开发中有用的工具
- usr：实现用于打包和压缩的 cpio 等
- virt : 虚拟化基础结构

### 配置kernel
> Kconfig,Makefile,.config的关系: Kconfig是编译选项的配置菜单, 配置好的结果就是.config, Makefile根据.config编译kernel.

`.config`的各种选项以CONFIG_FEATRUE形式表示, 前缀是CONFIG.

部分选项支持多选, 比如PREEMPT(内核抢占):
- n : 不编译
- y: yes, 会静态编入kernel image
- m : module, 编译这部分代码时以模块(一种可以动态安装的独立代码段)方式编译

> `#CONFIG_<xxx> is not set` : 该选项所对应的功能不编译

> kernel config不支持手动编辑, 必须使用相应的工具比如`make menuconfig`, 因为某些功能可能依赖其他功能.

通常驱动程序都会提供三选一的配置项.

配置工具:
- `make config` : 字符界面. 要求手动设定所有的选项，即使之前曾设定过
- `make oldconfig` : 所有选择都基于已有的.config文件，只对新特性和新设定提出询问
- `make localmodconfig` : 所有选择都基于已有的.config文件，只对新特性和新设定提出询问, 并会禁用**当前未被系统已加载的module**即不编译它们
- `make menuconfig` : 基于ncurse库编制的图形界面工具, **推荐**
- `make gconfig` : 基于gtk+的图形工具
- `make xconfig` : 基于qt的图形工具
- `make defconfig` : 使用本机arch对应的默认配置创建一个`.config`, 之后可用`make menuconfig`调整
- `make allyesconfig` : 生成将所有设置项启用并静态链接到kernel的.config
- `make allnoconfig` : 生成将允许范围内的设置项设为无效的.config
    
    每种arch的默认`.config`在`arch/<arch>/config/defconfig`

生成的配置会保存在kernel root的`.config`中.

> 可将make menuconfig 当做make oldconfig的图形版本. 在将新的设定更新到.config中去的同时，将原来的.config文件保存为.config.old

> 当前系统所用的kernel config在`/boot`里, 比如`/boot/config-4.19.0-8-amd64`

测试内核启动: `qemu-system-x86_64 -kernel  /home/chen/test/mnt/lfs/boot/vmlinuz-5.8.1-lfs-10.0-systemd-rc1 -initrd ../rootfs.gz -append "rw root=/dev/ram0  ramdisk_size=40960"`, 这里的root是指最终的根文件系统, `../rootfs.gz`为当前系统的`/boot/initrd.img`.

#### 获取最新kernel的config
1. 在[ubuntu kernel网站](https://kernel.ubuntu.com/~kernel-ppa/mainline/)选择指定的kernel并下载其header安装包, 然后解压, 再找到`usr/src/linux-headers-${kernel_version}-generic/.config`即可.

1. 在[fedora buildsystem](https://koji.fedoraproject.org/koji/packageinfo?packageID=8)搜索kernel, 选择指定版本的kernel, 根据指定的Source, 比如`https://src.fedoraproject.org/rpms/kernel.git#426b17af14a269cc24d57e3d1346cd06ba40e98e`, 转到`https://src.fedoraproject.org/rpms/kernel/tree/426b17af14a269cc24d57e3d1346cd06ba40e98e`下载指定的config即可.

1. 从[arch linux-headers](https://www.archlinux.org/packages/core/x86_64/linux-headers/)的构建仓库中获取.

##### .config配置
```config
CONFIG_EFI_VARS=n # from [Linux 内核中有关 UEFI 的配置选项](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface)
CONFIG_LOCALVERSION="-amd64-desktop" # kernel `make modules_install`时在`/lib/modules`下生成的文件名后缀, 比如`/lib/modules/5.4.50-amd64-desktop`
```

### 编译kernel & 替换linux内核
```
# sudo apt-get install libncurses5-dev libssl-dev build-essential openssl bison flex bc cpio
# wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.3.11.tar.xz
# xz -d linux-5.3.11.tar.xz
# tar -xf linux-5.3.11.tar
# cd linux-5.3.11
# cp /boot/config-4.15.0-30deepin-generic .config # 基于现有内核的配置来编译kernel
# make -j8
# make modules_install # 安装kernel的模块到`/lib/modules`
# make install # 将编译好的kernel image安装到/boot. 当安装完毕之后，grub 和 menu.lst 都会发生改变, 比如，grub.conf 里面会多一个新内核的项
# sudo reboot
```

##### [最简.config for v5.8.9](https://gist.github.com/chrisdone/02e165a0004be33734ac2334f215380e)
运行`qemu-system-x86_64 -kernel arch/x86/boot/bzImage`

```log
$ make allnoconfig
$ make menuconfig
General setup ---> Initial RAM filesystem and RAM disk (initramfs/initrd) support ---> yes
General setup ---> Configure standard kernel features ---> Enable support for printk ---> yes
64-bit kernel ---> yes
Executable file formats / Emulations ---> Kernel support for ELF binaries ---> yes
Executable file formats / Emulations ---> Kernel support for scripts starting with #! ---> yes
Device Drivers ---> Generic Driver Options ---> Maintain a devtmpfs filesystem to mount at /dev ---> yes
Device Drivers ---> Generic Driver Options ---> Automount devtmpfs at /dev, after the kernel mounted the rootfs ---> yes
Device Drivers ---> Character devices ---> Enable TTY ---> yes
Device Drivers ---> Character devices ---> Serial drivers ---> 8250/16550 and compatible serial support ---> yes
Device Drivers ---> Character devices ---> Serial drivers ---> Console on 8250/16550 and compatible serial port ---> yes
File systems ---> Pseudo filesystems ---> /proc file system support ---> yes
File systems ---> Pseudo filesystems ---> sysfs file system support ---> yes
$ time make -j2 bzImage # 不到4m
$ qemu-system-x86_64 -kernel arch/x86/boot/bzImage
```

最最精简的.config:
```log
$ make allnoconfig
$ make menuconfig
General setup ---> Configure standard kernel features ---> Enable support for printk ---> yes
64-bit kernel ---> yes # 如果不需要64bit支持, 这个也可不要
Device Drivers ---> Character devices ---> Enable TTY ---> yes
$ time make -j2 bzImage # 不到4m
$ qemu-system-x86_64 -kernel arch/x86/boot/bzImage
```

> `make tinyconfig`生成的bzImage, 没法用`qemu-system-x86_64 -kernel`运行

#### 编译核心与核心模块
- make vmlinux：未经压缩的核心
- make modules：仅核心模块
- make bzImage：经压缩过的核心（预设）
- make all：进行上述三个动作

比较常用的是modules和bzImage，bzImage可以制作出压缩过的核心.

通常会用`make -j N clean bzImage modules`连续动作来编译. clean用于清除之前的编译缓存.

> distcc和ccache可加速编译kernel.

> 编译并安装已有版本的kernel会覆盖同版本的kernel相关文件(kernel image, kernel modules等), 而CONFIG_LOCALVERSION可解决该问题, 因为该项指定的字符会成为version的一部分而避免发生覆盖.

调试kernel需要编译前激活 CONFIG_DEBUG_INFO 和 CONFIG_FRAME_POINTER 选项. 且在内核命令行中需要添加 nokaslr，来关闭 KASLR. 因为KASLR 会使得内核地址空间布局随机化，从而会造成打的断点不起作用.

其他可用的make命令:
- clean : 将源码恢复到编译前的状态, 但.config和编译过程中自动生成的部分文件不会被删除
- mrproper : 将源码恢复到刚下载时的状态, 删除的文件包括.config
- help : 显示可以使用的make命令
- tags : 生成tag文件, 可是vim等editor的tag jump功能可用(比如函数跳转), 便于源码浏览
- cscope : 生成用于cscope的索引文件. cscope是基于字符界面的源码浏览器
- allmodconfig : 生成将所有能编为module的设置项设为m的.config
- <dir> : 编译dir及其以下的所有文件
- <dir>/<file>.o : 仅生成指定的目标文件
- <dir>/<file>.ko : 仅生成指定的模块

make 选项:
- V=0|1|2 : 设置编译时console显示的详细程度
- O=<dir> : 将编译最终生成的文件全部输出到dir. 在源码目录禁止写入等情况下很实用

#### 生成内核包
1. fedora

    ```bash
    # make rpm-pkg # 会生成两个包, 二进制包在~/rpmbuild/rpms, 源码包在~/rpmbuild/SRPM
    ```
1. ubuntu

    ```bash
    # make deb-pkg # 同样会生成二进制包和源码包
    ```
#### 在kernel源码外编译module
```bash
# make -C /lib/modules/$(uname -r)/build M=$PWD # M=$PWD是为了告诉kernel编译在kernel源码外执行
# make -C /lib/modules/$(uname -r)/build M=$PWD modules_install # modules_install可用INSTALL_MOD_PATH来指定module的安装目录
```

## FAQ
### kernel开发与用户空间程序开发的差异
1. kernel开发既不能访问C库也不能访问标准的C头文件

    先有鸡还是先有蛋的悖论.

1. kernel开发必须使用gnu c
1. kernel开发缺乏像用户空间那样的内存保护机制

    用户空间程序进行非法内存访问, kernel会发送SIGSEGV信号并终止它.
    kernel发生内存错误会导致oops(错误信息).

    kernel的内存不分页.
1. kernel开发时难以执行浮点运算

    浮点的编码跟整数编码是不一样的，计算时需要专门的寄存器和浮点计算单元来处理，一个浮点运算指令使用的CPU周期也更长，因此对于内核来说就会想尽量回避浮点数运算，譬如说浮点数经过定点整数转换(`-msoft-float`)后进行运算，效率会高很多，即使CPU带有浮点数运算部件(fpu)，一般内核还是要避免直接进行浮点数运算，因为这些部件有可能被用户进程占用了，内核要判断这些浮点数部件是否被占用，保护现场，然后用浮点运算部件计算结果，恢复现场，开销会很大。如果CPU不支持浮点数运算，也就只能软件实现浮点数运算.
1. kernel给每个进程只有一个很小的定长堆栈

    在x86体系上栈的大小在编译时配置, 32位是8k, 64位是16k.
    > PAGE_SIZE为4KB(32位和64位相同)
1. kernel支持异步中断,抢占和SMP, 因此必须时刻注意同步和并发

    常用的解决竞争的方法是自旋锁和信号量.
1. 要考虑可移植的重要性

> 查看了[kernel releases](https://www.kernel.org/category/releases.html)的doc, 发现Longterm是以0.5 version的模式递增的.

### 提交kernel patch流程
1. 在`.git/config`配置`sendemail`

    ```conf
    [sendemail]
    smtpEncryption = tls
    smtpServer = smtp.gmail.com
    smtpUser = xxx@gmail.com
    smtpServerPort = 587
    ```
1. 准备patch

    ```bash
    # git format-path HEAD~ # 生成patch
    # ./scripts/checkpatch.pl --terse --file xxx.patch # 检查patch的格式
    ```
1. 提交patch

    ```bash
    # ./scripts/get_maintainer.pl xxx.patch # 获取patch相关代码的维护人员
    # git send-email --to xxx1@xxx.com --to xxx2@xxx.com --cc linux-kernel@vger.kernel.org xxx.patch # 发送Patch
    ```

    提交成功后，就能在[内核邮件列表](https://lkml.org/)中看到自己的邮件以及维护人员的回复.

    Linux内核被划分成不同的子系统，如网络、内存管理等，不同的子系统有相应的维护人员，一个Patch会首先提交到子系统分支，再被维护人员提交到上游分支, 最终由Linus Torvalds将Patch合并到主分支.

### kernel调试
- printk : 支持设置优先级
- kdump

    解析内核dump，首先需要符号表，也就是未经过压缩的内核镜像，通常我们也叫它vmlinux. 通常系统默认并没有自带vmlinux，但可以通过安装系统版本对应的debug包来获取它.

    redhat下要安装kernel-debug以及kernel-debug-info，而在ubuntu下，对应的包名是dbgsym.

    kernel dump调试工具主要有Trace32和crash两个，trace32是商业软件，图形化界面，功能强大，但是收费. 而crash则是开源工具，且基于命令行模式，但是功能并不逊于Trace32.

    crash工具启动时如果不给它传递kdump文件，那么它默认就是调试当前内存中的内核. `su root`然后直接在命令行输入`crash vmlinux-4.4.0-87-generic`即可.

    在ubuntu系统中，通过安装linux-crashdump工具，我们可以捕捉kdump. Kdump是一个Linux内核崩溃转储机制，这个机制的原理是在内存中保留一块区域，这块区域用来存放capture kernel，当前的内核发生crash后，通过kexec把保留区域的capture kernel运行起来，由capture kernel负责把crash kernel的完整信息--包括CPU寄存器、堆栈数据等--转储到文件中，文件的存放位置可以是本地磁盘，也可以是网络.
- sysrq

    要使用sysrq需要kernel config的`CONFIG_MAGIC_SYSRQ`.

    设置sysrq的方法:
    1. sysctl -w kernel.sysrq=1
    1. echo 1 > /proc/sys/kernel/sysrq, sysrq使用位图表示, 1表示所以sysrq键都可用

    发送sysrq方法:
    1. alt+sysrq+<命令键> # 如有热键冲突则需要先解决冲突
    1. echo <命令键> >/proc/sysrq-trigger # 必须root权限, sudo也不行

    常用sysrq命令键:
    - b reboot : 立即重新启动系统
    - c : 故意让系统崩溃(在使用netdump或者diskdump的时候有用)
    - d : 输出所有lock
    - l : 输出系统中所有cpu的栈
    - m : 导出关于内存分配的信息
    - p : 到处当前CPU寄存器信息和标志位的信息
    - q : 所有计时器的信息
    - t : 导出线程状态信息
    - s sync : 立即同步所有挂载的文件系统
    - u : 立即重新挂载所有的文件系统为只读
    - o : 立即关机(如果机器配置并支持此项功能)
    - z : 输出ftrace的缓冲区

# kernel启动
内核的启动从入口函数 start_kernel() 开始, 在 ${kernel_root}/init/main.c 文件中,start_kernel 相当于内核的main 函数.

start_kernel流程:
1. 在操作系统里面,先要有个创始进程`set_task_stack_end_magic(&init_task)`. 它是系统创建的第一个进程,我们称为0 号进程, 也是唯一一个没有通过 fork 或者 kernel_thread 产生的进程,是进程列表的第一个.
1. trap_init() 设置了很多中断门 (Interrupt Gate),用于处理各种中断
1. mm_init() 用来初始化内存管理模块
1. sched_init() 用于初始化调度模块
1. vfs_caches_init() 用来初始化基于内存的文件系统 rootfs
1. rest_init()用来做其他方面的初始化
    1. 用kernel_thread(kernel_init, NULL, CLONE_FS) 创建第二个进程,这个是1 号进程, 是用户态的进程
    1. 用kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES), 创建第2 号进程. 函数 kthreadd,负责所有内核态的线程的调度和管理,是内核态所有线程运行的祖先

## systemd启动过程
systemd 是所有进程的父进程. 它负责将 Linux 主机带到一个用户可操作状态（可以执行功能任务）.

首先，systemd 按照 /etc/fstab 挂载文件系统，包括内存交换文件或分区. systemd 借助其配置文件 /etc/systemd/system/default.target 决定 Linux 系统应该启动达到哪个状态（或目标态target）, 对于桌面系统，其链接到 graphical.target, 对于一个服务器操作系统来说，default.target 更多是默认链接到 multi-user.target.
然后, 直至执行/bin/login程序，跳出登录界面，等待用户输入用户名和密码, 至此，全部启动过程完成.

> /etc/systemd/system/default.target没有则使用/usr/lib/systemd/system/default.target
> target查看: systemctl get-default

## vmlinuz、initrd.img和System.map
vmlinuz是可引导的、压缩的内核. “vm”代表 “Virtual Memory”. Linux 支持虚拟内存, 能够使用硬盘空间作为虚拟内存，因此得名“vm”.vmlinuz是可执行的Linux内核，它位于/boot/vmlinuz，它一般是一个软链接.

> vmlinux是未压缩的内核.

initrd就是一个带有根文件系统的虚拟RAM盘，里面包含了根目录‘/’，以及其他的目录,比如：bin，dev，proc，sbin，sys等linux启动时必须的目录，以及在bin目录下加入了一下必须的可执行命令. 被成为最小根文件系统.

initrd是传递给内核用于挂载根文件系统的文件，现在它分为两种，一种是image格式zip压缩包，这是最原始的作用，传入的是ramdisk，另一种是cpio格式，传入的是initramfs(**常用且推荐**)，内核在解析时会自动解析它的格式，从而按照不同的方式进行处理.

> initramfs支持两种方式，可以和kernel编译在一起，也可以以initrd的方式传入. initramfs会通过switch_root切换到真实的rootfs上.

System.map: 内核符号映射表，顾名思义就是将内核中的符号（也就是内核中的函数）和它的地址能联系起来的一个列表. 是所有符号及其对应地址的一个列表.

## rootfs
root分区挂载的方式:
1.ramdisk挂载

    cmdline传入root=/dev/ramX，initrd=addr,size
1.物理分区挂载

    cmdline传入root=/dev/mmcblk0
1.initramfs挂载

    这种方式不需要传入"root=XXX"，因为这个roofs是内核自带的，但是要保证其中填充了内容，特别是根目录要存在init程序. 如果根目录没有init程序，那么根目录会挂载失败，当然也可以通过传入"rdinit=/XXX"来指定其他的程序名，只要这个名字存在也同样能挂载rootfs. 如果initramfs是独立于kernel存在的cpio压缩包，那么同样需要initrd=addr,size来传递给内核

### 构建根文件系统的工具
- debian/ubuntu : debootstrap
- fedora : febootstrap
- LFS(linux form scratch)
- busybox

### initramfs
参考:
- [initramfs](http://xstarcd.github.io/wiki/Linux/initramfs.html)
- [用qemu运行一个小小Linux系统](https://my.oschina.net/u/3258476/blog/1550537)
- [Build and run minimal Linux / Busybox systems in Qemu](https://gist.github.com/chrisdone/02e165a0004be33734ac2334f215380e)

#### 缘由 : "鸡生蛋，还是蛋生鸡"的悖论
bios和uefi携带的驱动有限, 只能识别有些种类的fs, 比如ext4, fat32等, 它们常用于efi, boot分区. 即bios/uefi可直接加载这里分区的内容.

但内核要想成功挂载rootfs，首先必须能识别rootfs所在设备的物理类型，还要读懂rootfs的文件系统类型. 所以，这就需要rootfs所在块设备驱动，需要相应的文件系统驱动，可能需要逻辑卷相关模块，还可能需要特殊的数据解密驱动.

这些驱动模块，如果放在rootfs中，就不能使用！因为承载rootfs的根文件系统设备还没就绪；如果放在内核image中，内核镜像可能会太臃肿，而且每增加支持一种新设备或文件系统，就要重新编译一次内核镜像，耦合太紧；再加上内核较严格的软件许可证制度，而很多设备厂商有一定的知识产权保护要求，这些驱动模块也许不方便放入内核镜像(另，Android采用了另外一种机制HAL规避知识产权冲突).

因此有了initramfs机制, 提供一个初始化最小文件系统(initial fs)来解决这个悖论.

#### 手动创建initramfs

CONFIG_INITRAMFS_SOURCE为空时编译的kernel不带initramfs.

> CONFIG_INITRAMFS_SOURCE默认即为空.

CONFIG_INITRAMFS_SOURCE支持三种参数:
1. 一个已经做好的cpio.gz的路径
1. 指定给内核一个text的配置文件或者文件夹(一个已经为制作cpio.gz准备好所有内容的文件夹)
1. 使用configuration文件initramfs_list来告诉内核initramfs在哪里

制作和加载独立的initramfs时，首先需要保证内核选项使能：`CONFIG_BLK_DEV_INITRD=y`.

常见的是通过提取busybox的rootfs来制作initramfs:
```bash
cd rootfs
find . | cpio -H newc -ov --owner root:root > ../initramfs.cpio
cd ..
gzip initramfs.cpio
```

最后会生成一个initramfs.cpio.zip的压缩包文件，这个就是想要的initramfs.

可用`sudo lsinitramfs ${initramfs}`查看生成的initramfs内容. unmkinitramfs可解压initramfs image.

#### 使用initramfs-tools by `apt install initramfs-tools`
`LC_ALL="en_US.UTF-8" mkinitramfs -o /boot/initrd.img ${kernel_version}`,  mkinitramfs是需要提供创建initramfs的kernel版本号，如果是给当前kernel制作initramfs，可以用uname -r查看当前的版本号, 提供kernel版本号的主要目的是为了在initramfs中添加指定kernel的驱动模块, 此时mkinitramfs会把/lib/modules/${kernel_version}/目录下的一些启动会用到的模块添加到initramfs中.

更新当前kernel的initramfs: `update-initramfs -u`

> 在fedora/centos/rhel下面一般是用mkinitrd,而在Ubuntu/Debian下是用mkintramfs, 两者类似.

或使用`update-initramfs -c -k 5.8.1 -v`, `5.8.1`是kernel_version在`/lib/modules`下的文件夹名, **已测试**.

#### build initramfs by busybox
通常使用轻量级的busybox来制作initramfs, 查看其使用的libc版本:
1. 以qemu+kernel+initrd的方式进入initramfs
1. `find / -name "libc*"
1. 执行找到的`/usr/lib/x86_64-linux-gun/libc.so.6`即可, 得到: "glibc 2.28.8"


后续参考[busybox制作initramfs以及切换rootfs](https://blog.csdn.net/m0_38096844/article/details/97786761), 未测试.

## 制作linux 启动盘 by Syslinux
Syslinux是一个启动加载器的集合, 包含了一系列的bootloaders, 用于引导启动os:
- SYSLINUX : fat fs bootloader
- EXTLINUX : ext2/3/4, btrfs, xfs fs bootloader
- PEXLINUX : network pxe bootloader
- ISOLINUX : iso-9660 for cd/dvd bootloading

### isolinux.cfg
光盘安装系统启动过程:
1. 读取MBR： isolinux/boot.cat # boot.cat的作用就相当于启动流程中的MBR的446字节
1. 读取isolinux/isolinux.bin # isolinux.bin的作用等价于grub的第二阶段
1. 加载配置文件： isolinux/isolinux.cfg # 这个文件就是启动菜单；自定义启动菜单可以修改此文件
1. 根据配置文件定义调用内核文件

    加载内核： isolinuz/vmlinuz向内核传递参数： append initrd=initrd.img
1. 装载根文件系统，并启动anaconda ###anaconda就是kickstart文件

```conf
# 第一行，DEFAULT vesamenu.c32，必须的，因为要用到菜单功能，必须有这个vesamenu.c32文件
# com32/menu/vesamenu.c32 (graphical) or com32/menu/menu.c32 (textmode only)
# 1, 向用户提示输入选择，直接回车就是缺省选项了; 0,不向用户提示输入选择
DEFAULT vesamenu.c32
prompt=1
# 60 = 6s
TIMEOUT 60
label linux   # label定义了启动系统可供选择的模式
  menu label ^Install or upgrade an existing system
  menu default
  kernel vmlinuz
  append initrd=initrd.img
```
## kernel编译
### LFS kernel 5.8.1 编译报"[kernel/Makefile:144: kernel/kheaders_data.tar.xz] Error 127"
`.config` from https://kernel.ubuntu.com/~kernel-ppa/mainline/v5.8.1/amd64/linux-headers-5.8.1-050801-generic_5.8.1-050801.202008111432_amd64.deb

解决步骤:
1. 使用`MAKEFLAGS=j1 make V=1`编译获取调试输出

    ```log
    need-modorder=1
    sh ./kernel/gen_kheaders.sh kernel/kheaders_data.tar.xz
    GEN     kernel/kheaders_data.tar.xz
    ```
1. 对`./kernel/gen_kheaders.sh`开启`set -x`, 重复步骤1.

    ```log
    + for f in $dir_list
    + find include/ -name '*.h'
    + cpio --quiet -pd /tmp/tmp.4Vqzgnf9tT.kernel/kernel/kheaders_data.tar.xz.tmp
    make[1]: *** [kernel/Makefile:144: kernel/kheaders_data.tar.xz] Error 127
    ```
1.  检查cpio是否存在

    ```log
    # cpio
    bash: cpio: command not found
    ```

> 网上的查到的结论也是缺cpio.

### /bin/sh: lz4c: command not found
```log
# make
/bin/sh: lz4c: command not found
make[2]: *** [arch/x86/boot/compressed/Makefile:147: arch/x86/boot/compressed/vmlinux.bin.lz4] Error 127
```

解决方法: `apt install liblz4-tool`

### zcat /boot/initrd.img报错"gzip: /boot/initrd.img: not in gzip format"
参考:
- [夾帶 microcode 的 initrd 解法](https://www.ubuntu-tw.org/modules/newbb/viewtopic.php?viewmode=compact&order=ASC&topic_id=108548&forum=11)

os: ubuntu 20.04.1

```
$ file -L /boot/initrd.img
/boot/initrd.img: ASCII cpio archive (SVR4 with no CRC)
$ cpio -idvm < /boot/initrd.img # 发现解压出的大小仍然不对, 仅解出了部分内容, 推测其他内容应该还有另外的压缩方式.
$ sudo unmkinitramfs /boot/initrd.img # 解压成功, **推荐**
```

或参考上面的文章手动解码(不推荐):
```
$ grep 'TRAILER!!!' -a -b -o initrd.img
31294:TRAILER!!!
3035934:TRAILER!!!
3080689:TRAILER!!!
50221283:TRAILER!!!
```

因此initrd.img的结构大致上是这样的:
```log
GenuineIntel.bin 第一层
----------------------- TRAILER!!!
AuthenticAMD.bin 第二层
----------------------- TRAILER!!!
gz 或 xz 或 lzma 或 lz4 第三层
----------------------- TRAILER!!!
cpio 第四层
----------------------- TRAILER!!!
```

也可通过`hexdump -C initrd.img  -n 51200 > a.log && vim a.log`搜索第一个`TRAILER!!!`来验证

### "VFS: Unable to mount root fs on unknown"
linux启动时报该错, 是因为没有配置initrd/initramfs.

### 测试kernel是否可以启动
qemu test uefi+kernel: `qemu-system-x86_64 -enable-kvm -m 512 -kernel vmlinuz ［-initrd initrd.img]`

建议内存最小是512M, 之前试过256M, 但卡在了uefi的界面.

### kernel panic - not syncing: No working init found.  Try passing init= option to kernel
os: Deepin 20
kernel: 5.4.50-amd64-desktop

同台电脑的另一个kernel是正常的:  vmlinuz-5.3.0-3-amd64 + initrd.img-5.3.0-3-amd64

将initrd.img-5.4.50-amd64-desktop解压后发现存在"main/init".

通过`qemu-system-x86_64 -nographic -enable-kvm -m 512 -kernel vmlinuz-5.4.50-amd64-desktop -initrd initrd.img-5.4.50-amd64-desktop -append console=ttyS0`将日志输出到终端, 找到错误日志:
```log
[    2.000673] Run /init as init process
[    2.001336] Failed to execute /init (error -2)
...
```

检查调试initramfs中的`${initramfs_uncompress}/main/init`即可.

### make modules_prepare用法
```bash
# mkdir /usr/src/linux-headers-5.8.7
# cp .config /usr/src/linux-headers-5.8.7
# cd ${KernelRoot}
# make mrproper
# make O=/usr/src/linux-headers-5.8.7 modules_prepare
```

make modules_prepare应在make之前执行, 问题是modules_prepare还是有include或软链接到${KernelRoot}, 具体处理方法可参考[5.8.7.arch1-1 PKGBUILD](https://github.com/archlinux/svntogit-packages/blob/packages/linux/trunk/PKGBUILD).

### CONFIG_PVH
参考:
- [QEMU 4.0 boots uncompressed Linux x86_64 kernel](https://stefano-garzarella.github.io/posts/2019-08-23-qemu-linux-kernel-pvh/)

Prerequisites
- QEMU >= 4.0
- Linux kernel >= 4.21

  CONFIG_PVH=y enabled
  vmlinux uncompressed image

`CONFIG_PVH=y`时`qemu-system-x86_64 -kernel vmlinux -nographic -append "console=ttyS0"`可运行, 否则`-kernel`必须使用bzImage.

### 关于Linux编译优化几个必须掌握的姿势
Linux内核如果用O0编译，是无法编译过的，Linux的内核编译，要么是O2，要么是Os，这点从Linux的Makefile里面可以看出.

> kernel用O0编译不过，是因为kernel本身也没有想用O0能够编译过，它的设计里面包含了编译会优化的假想.

```makefile
ifdef CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE
KBUILD_CFLAGS += -O2
else ifdef CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE_O3
KBUILD_CFLAGS += -O3
else ifdef CONFIG_CC_OPTIMIZE_FOR_SIZE
KBUILD_CFLAGS += -Os
endif
```

O2倾向于基于速度的优化，Os倾向于基于size更小的优化, O3目前社区不推荐.
```bash
$ gcc -c -Q -O2 --help=optimizers > /tmp/O2-opts
$ gcc -c -Q -Os --help=optimizers > /tmp/Os-opts
$ meld /tmp/O2-opts /tmp/Os-opts 
```

就2项有差别. O2和Os都使能了inline small函数和called once的函数，但是O2里面-finline-functions是关闭的,而Os里面是开的; O2里面optimize-strlen是开的，Os里面这个选项是关闭的. 相关选项的含义可以通过"man gcc"查看.


> 如果在全局优化的情况下，想针对某个局部避免优化，可以尝试用noinline，__attribute__((optimize("O0")))等进行外科手术式地调整.

#### 去掉编译内核的优化选项
1. 上面优化的位置替换成`KBUILD_CFLAGS    += -O1`, 只做基本的优化
1. .config

    Kernel hacking -> Compile-time checks and compiler options -> Enable full Section mismatch analysis = y : CONFIG_DEBUG_SECTION_MISMATCH这个宏本来是检测代码/数据段类型属性不匹配的，定义后同时有个功能是禁止将只有一个地方调用的函数变为inline函数
