# linux编译

如何替换linux内核:
```sh
# sudo apt-get install libncurses5-dev libssl-dev build-essential openssl bison flex bc
# wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.3.11.tar.xz
# xz -d linux-5.3.11.tar.xz
# tar -xf linux-5.3.11.tar
# cd linux-5.3.11
# cp /boot/config-4.15.0-30deepin-generic .config # 基于现有内核的配置来编译kernel
# make -j4
# sudo make modules_install # 编译并安装kernel的模块
# sudo make install # 编译并安装新kernel
# sudo reboot
```

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
- ipc：进程间通信的代码
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
各种选项以CONFIG_FEATRUE形式表示, 前缀是CONFIG.

部分选项支持多选, 比如PREEMPT(内核抢占):
- no : 不支持
- yes: 支持, 会编入kernel image
- module : 编译这部分代码时已模块(一种可以动态安装的独立代码段)方式编译

通常驱动程序都会提供三选一的配置项.

配置工具:
- `make config` : 字符界面. 要求手动设定所有的选项，即使之前曾设定过
- `make oldconfig` : 所有选择都基于已有的.config文件，只对新特性和新设定提出询问
- `make menuconfig` : 基于ncurse库编制的图形界面工具, **推荐**
- `make gconfig` : 基于gtk+的图形工具
- `make defconfig` : 基于默认的配置为本机的体现结构创建一个配置

生成的配置会保存在kernel root的`.config`中.

> 可将make menuconfig 当做make oldconfig的图形版本. 在将新的设定更新到.config中去的同时，将原来的.config文件保存为.config.old
> 当前系统所用的kernel config在`/boot`里, 比如`/boot/config-4.19.0-8-amd64`

使用`make -j${N}`编译完kernel后可用`make modules_install`安装已编译的模块到`/lib/modules`.

> distcc和ccache可加速编译kernel.

> 获取最新kernel的config: 在[ubuntu kernel网站](https://kernel.ubuntu.com/~kernel-ppa/mainline/)选择指定的kernel并下载其header安装包, 然后解压, 再找到`usr/src/linux-headers-${kernel_version}-generic/.config`即可.

### 编译kernel
```
# apt install libssl-dev libelf-dev bc
# make -j8
# make modules_install
# make install # 安装编译好的kernel. 当编译完毕之后，grub 和 menu.lst 都会发生改变。例如，grub.conf 里面会多一个新内核的项
```

调试kernel需要编译前激活 CONFIG_DEBUG_INFO 和 CONFIG_FRAME_POINTER 选项. 且在内核命令行中需要添加 nokaslr，来关闭 KASLR. 因为KASLR 会使得内核地址空间布局随机化，从而会造成打的断点不起作用.

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

### kernel调试
- printk : 支持设置优先级
- kdump

    解析内核dump，首先需要符号表，也就是未经过压缩的内核镜像，通常我们也叫它vmlinux. 通常系统默认并没有自带vmlinux，但可以通过安装系统版本对应的debug包来获取它.

    redhat下要安装kernel-debug以及kernel-debug-info，而在ubuntu下，对应的包名是dbgsym.

    kernel dump调试工具主要有Trace32和crash两个，trace32是商业软件，图形化界面，功能强大，但是收费. 而crash则是开源工具，且基于命令行模式，但是功能并不逊于Trace32.

    crash工具启动时如果不给它传递kdump文件，那么它默认就是调试当前内存中的内核. `su root`然后直接在命令行输入`crash vmlinux-4.4.0-87-generic`即可.

    在ubuntu系统中，通过安装linux-crashdump工具，我们可以捕捉kdump. Kdump是一个Linux内核崩溃转储机制，这个机制的原理是在内存中保留一块区域，这块区域用来存放capture kernel，当前的内核发生crash后，通过kexec把保留区域的capture kernel运行起来，由capture kernel负责把crash kernel的完整信息--包括CPU寄存器、堆栈数据等--转储到文件中，文件的存放位置可以是本地磁盘，也可以是网络.

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