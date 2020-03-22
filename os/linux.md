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

## arch
参考:
- [Linux内核arch目录，各个处理器的介绍](https://blog.csdn.net/u014379540/article/details/52263342)