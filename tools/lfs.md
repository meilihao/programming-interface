# lfs 8.4
参考:
- [Linux From Scratch SVN-20200815 中文翻译版](https://bf.mengyan1223.wang/lfs/zh_CN/)
- [LCTT/LFS-BOOK](https://github.com/LCTT/LFS-BOOK)

> 编译过程不要随意关机或退出环境, 否则再次进入编译环境进行操作时会碰到未知错误.
>
> lfs里的相对路径操作要留意, 有时候真的很模糊, 必要时通过locate定位.
>
> [必须严格按照给出的顺序安装软件包,且不要同时构建多个软件包，这可能节约时间， 但往往会导致某个程序包含一个硬编码的，指向 /tools 的路径， 结果删除该目录后程序不能工作](https://bf.mengyan1223.wang/lfs/zh_CN/development/chapter06/introduction.html)

## 1. 准备, 先看FAQ的问题
1. [宿主系统需求](https://bf.mengyan1223.wang/lfs/zh_CN/8.4-systemd/chapter02/hostreqs.html)
1. 创建lfs用户, 建立干净的编译环境, 以避免其他环境变量的影响

    ```sh
    $ sudo useradd -s /bin/bash -m -k /dev/null lfs // `-k`需与`-m`连用表示当useradd创建主目录时，将缺省文件从 指定目录(通常是/etc/skel) 复制到用户的主目录, 这里用`/dev/null`表示不复制
    $ sudo mkdir /home/lfs/lfs # 建立 base 目录
    $ sudo mount -v -t ext4 /dev/sdb /home/lfs/lfs # 挂载硬盘, 但这里仅单纯的新建目录也可以
    $ sudo chown -R lfs:lfs /home/lfs/lfs # 挂载后需修改所属
    $ su - lfs # 切换用户
    $ export LFS=/home/lfs/lfs
    $ mkdir -v $LFS/sources # 建立sources, 放需要下载的资源
    $ mkdir -v $LFS/tools # 建立tools, 编译生成的临时工具都会安装到这里
    $ cat > ~/.bash_profile << EOF
    exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
    EOF # 新建一个除了 HOME, TERM 以及 PS1 外没有任何环境变量的 shell ，替换当前 shell ， 防止宿主环境中不必要和有潜在风险的环境变量进入编译环境
    $ cat > ~/.bashrc << EOF
    set +h                  # 关闭了 bash 的哈希功能, 使得程序执行的时候就会一直搜索 PATH
    umask 022
    unset LANG # 避免字符集影响
    LFS=/home/lfs/lfs
    LC_ALL=POSIX
    LFS_TGT=x86_64-lfs-linux-gnu
    PATH=$LFS/tools/bin:/bin:/usr/bin
    MAKEFLAGS='-j 8' # 根据机器cpu配置
    export LFS LC_ALL LFS_TGT PATH MAKEFLAGS
    EOF
    $ exit
    $ su - lfs
    $ env # 输出env并检查
    ```

1. 环境/依赖检查
    1. `echo $SHELL`是否`bash`
    1. `/bin/sh`是否指向`bash`, 否则通过`sudo dpkg-reconfigure bash`来修改或重建ln
    1. `readlink -f /usr/bin/awk`是否`gawk`
    1. `readlink -f /usr/bin/yacc`是否`bsion`, 否则编译glic时会报错

1. 下载所需all Packages, `bison`使用`3.0.5`,否则glic.2.29无法通过编译

    **每次需要编译的包使用完成后需要删除**.

    根据[lfs 的 `Files Mirrors`](http://www.linuxfromscratch.org/mirrors.html#files)下载所需的版本的[all packages, **推荐**](https://mirror-hk.koddos.net/lfs/lfs-packages/lfs-packages-8.4.tar), 该tar已打包所有所需, 各包的作用在[这里](https://lctt.github.io/LFS-BOOK/lfs-systemd/prologue/package-choices.html).

    其他下载方式(**不推荐**): 根据相应版本的wget-list和md5sums逐个下载, 比如这里的[8.4](http://mirrors-usa.go-parts.com/lfs/lfs-packages/8.4/wget-list), 命令如下:
    ```
    $ wget --input-file=wget-list --continue --directory-prefix=sources # 支持断点恢复
    $ md5sum -c md5sums        # 校验下载
    ```

    > 也可使用[中国科学技术大学镜像站提供的 LFS 软件包mirror](http://mirrors.ustc.edu.cn/lfs/lfs-packages/)

## 2. 第一次编译
将`$LFS/tools`软连接到`/tools`

1. Binutils
    ```
    $ cd /home/lfs/lfs/tools # 在 x86_64 上构建需建立lib64
    $ mkdir lib
    $ ln -s lib lib64
    ...
    $../configure --prefix=/tools \
    --with-sysroot=$LFS \
    --with-lib-path=/tools/lib \
    --target=$LFS_TGT \
    --disable-nls \
    --disable-werror
    $ make
    $ make install
    ```

    `--prefix`表示`Binutils 程序的安装位置`

1. gcc
    ```
    $ cd gcc-8.2.0
    $ ./contrib/download_prerequisites # 使用gcc指定的依赖. 我这使用lfs提供的mpfr, gmp, mpc碰到问题`mpfr/missing xxx --gnu`
    $ for file in gcc/config/{linux,i386/linux{,64}}.h
    do
    cp -uv $file{,.orig}
    sed -e 's@/lib\(64\)\?\(32\)\?/ld@/tools&@g' \
    -e 's@/usr@/tools@g' $file.orig > $file
    echo '
    #undef STANDARD_STARTFILE_PREFIX_1
    #undef STANDARD_STARTFILE_PREFIX_2
    #define STANDARD_STARTFILE_PREFIX_1 "/tools/lib/"
    #define STANDARD_STARTFILE_PREFIX_2 ""' >> $file
    touch $file.orig
    done
    $ sed -e '/m64=/s/lib64/lib/' \
    -i.orig gcc/config/i386/t-linux64 # 修改`.orig`后缀的文件 + `gcc/config/i386/t-linux64`
    $ mkdir build && cd build
    $ ../configure \
    --target=$LFS_TGT \
    --prefix=/tools \
    --with-glibc-version=2.11 \
    --with-sysroot=$LFS \
    --with-newlib \
    --without-headers \
    --with-local-prefix=/tools \
    --with-native-system-header-dir=/tools/include \
    --disable-nls \
    --disable-shared \
    --disable-multilib \
    --disable-decimal-float \
    --disable-threads \
    --disable-libatomic \
    --disable-libgomp \
    --disable-libmpx \
    --disable-libquadmath \
    --disable-libssp \
    --disable-libvtv \
    --disable-libstdcxx \
    --enable-languages=c,c++
    ```

3. glibc
    ```
    $ cd glibc-2.29
    $ patch -p1 < ../glibc-2.29-fhs-1.patch # 这里其实不用打patch, 是在`6. 安装基本系统软件`编译glic时打
    ...
    ```

3. 第二次编译

4. 备份
在`6. 安装基本系统软件`前备份`$LFS/lfs`, 避免之后的流程出错可供还原.

5. `6.9.1. 安装 Glibc`
`../lib/ld-linux-x86-64.so.2`不明, 在宿主机用了locate查找得到.

```sh
(lfs chroot) root:/tools/lib# cp ld-linux-x86-64.so.2 /lib64
(lfs chroot) root:/tools/lib# cp ld-linux-x86-64.so.2 /lib64/ld-lsb-x86-64.so.3
```
因此`ln -sfv ../lib`应是指`/tools/lib`

6. `6.21. GCC-9.1.0`
`ln -sv ../usr/bin/cpp /lib` -> `ln -sv /usr/bin/cpp /lib`
`ln -sfv ../../libexec/gcc/$(gcc -dumpmachine)/9.1.0` -> `ln -sfv /usr/libexec/gcc/x86_64-pc-linux-gnu/9.1.0`

## FAQ
### Binutils(链接器), GCC(编译器)编译了三遍
为了构建交叉编译环境, 解除目标系统与宿主系统的关系. 比如:
```
$ ldd /home/lfs/lfs/tools/bin/x86_64-lfs-linux-gnu-ar # 第一遍编译Binutils
	linux-vdso.so.1 (0x00007ffea9f22000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fd6ffa59000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fd6ff6ba000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fd6fff8e000)
$ $ ldd /home/lfs/lfs/tools/bin/x86_64-lfs-linux-gnu-ar # 第二遍编译
	linux-vdso.so.1 (0x00007ffd79c87000)
	libdl.so.2 => /tools/lib/libdl.so.2 (0x00007f86cb493000)
	libc.so.6 => /tools/lib/libc.so.6 (0x00007f86cb2d9000)
	/lib64/ld-linux-x86-64.so.2 => /tools/lib64/ld-linux-x86-64.so.2 (0x00007f86cb7cb000)

```

Binutils, GCC, Glibc均设置了HOST,TARGET:
- HOST=TARGET : 编译脚本就构建本地编译工具
- HOST!=TARGET : 编译脚本就构建交叉编译工具, 指导宿主os上的工具链编译"运行在本机, 但是最后编译链接的程序/lib是运行在$TARGET上"的交叉二进制工具

编译次数:
1. Pass 1: 生成交叉工具Binutils, Freestanding GCC(它们运行在host, 生成代码是target), 它们会使用host /lib目录下的dynamic linker,这步仅是为了编译glic得到target glibc.
2. Pass 2: 利用交叉工具Binutils编译target Binutils, 并[为第6章的 “重新调整” 阶段准备链接器](https://bf.mengyan1223.wang/lfs/zh_CN/development/chapter05/binutils-pass2.html). 再利用Pass1的target glibc编译生成全功能的hosted target GCC.
3. Pass 2生成的target Binutils和GCC也是在host上编译, 因此再在chroot环境self compile一次,检查编译环境是否正常. 如果不打算在目标系统中安装编译工具链，这个就可以不做了.

[交叉编译gcc时为什么一定要编译glibc](https://www.zhihu.com/question/41672814):
为了解决“鸡和蛋”的循环依赖问题.

1. 编译 glibc 需要目标一致的gcc
编译运行 (--host=$TARGET) 在目标平台上的 glibc，需要 (--target=$TARGET) 设置一致的 gcc，而不是目标为本地的 gcc
2. 编译完整版gcc需要glibc
    1. 理论上，gcc不应该依赖于C库。但是，gcc 需要包含一些 glibc 的头文件来了解要支持哪些 C 库的特性
    2. gcc 中也使用了一些C库的接口 (有现成的接口，谁还想再发明一遍轮子)
    3. gcc 不光是编译器，它还包含了一些库，这些库依赖于 glibc.
    
freestanding 的gcc不需要依赖于glibc，所以用它先编译glibc，然后完整版的gcc就有了.

### 5.5. GCC-8.2.0 - 第一遍 报错`../.././gcc/libgcc.mvars: No such file or directory`
编译gcc时，需要注意一个原则：不要再gcc的源码中直接执行./configure、make、make install等命令，需要在源码目录下另外新建一个目录，在新建的目录中执行以上命令.

### 5.7. Glibc-2.29的`$LFS_TGT-gcc dummy.c`报错`x86_64-lfs-linux-gnu/bin/ld: cannot find crt1.o: No such file or directory`+`x86_64-lfs-linux-gnu/bin/ld: cannot find /home/lfs/lfs/tools/lib/libc.so.6 inside /home/lfs/lfs`
之前编译Binutils,gcc,glibc时使用了`--prefix=$LFS/tools`, 即未将`$LFS/tools`软连接到`/tools`而导致报错.

看了`x86_64-lfs-linux-gnu/bin/ld --verbose | grep SEARCH`的输出`有SEARCH_DIR("/home/lfs/tools/lib")`且也有`/home/lfs/lfs/tools/lib/crt1.o`, 怎么就找不到呢. 而且`/home/lfs/lfs/tools/lib/libc.so.6`也是存在的.
看了` $LFS_TGT-gcc dummy.c -Wl,--verbose 2>&1 | grep succeeded`发现`attempt to open /home/lfs/home/lfs/tools/lib/libc.so.6 failed`

> libc.so.6是glibc的软链接.

参考[Can not hardcode library search path of binutils](https://unix.stackexchange.com/questions/350944/can-not-hardcode-library-search-path-of-binutils)

解决方法:
1. 将`$LFS/tools`软连接到`/tools`, 具体原因未知,应该是编译时不了解参数而配置错误(**推荐**, 毕竟参数组合太多, 编译又耗时).
1. [执行`ln -s /home/lfs/lfs/tools/lib/crt*.o /home/lfs/lfs/tools/lib/gcc/x86_64-lfs-linux-gnu/8.2.0`后`crt1.o`错误消失](https://unix.stackexchange.com/questions/154083/how-to-provide-crt1-o-and-crti-o-for-lfs). 但`libc.so.6`错误无解

    实在想修改`--prefix`的话, 可参考[clfs](http://clfs.org/view/clfs-embedded/arm/), 未尝试, **不推荐**.

### bison : cannot stat 'examples/c/reccalc/scan.stamp.tmp': No such file or directory
参考[bug#36238: Problems cross-compiling on core-updates](https://www.mail-archive.com/bug-guix@gnu.org/msg13512.html), 是并行编译问题, 使用`make -j1`即可.

### 6.9. Glibc-2.29 报错`bison --yacc --name-prefix=__gettext --output /sources/glibc-2.29/build/intl/plural.c plural.y`
使用`bison-3.0.5`.

### 6.12. File-5.37 报错`/sources/file-5.36/src/.libs/lt-file: error while loading shared libraries: libz.so.1: cannot open shared object file: No such file or directory`
检查依赖`ldd src/.libs/lt-file`

解决方法: `export LD_LIBRARY_PATH=/lib:/usr/lib`

查看[FHS 兼容性说明](https://bf.mengyan1223.wang/lfs/zh_CN/systemd/chapter06/zlib.html)

我的建议是等所有的编译都完成了,一次性移动到`/lib`.

### 6.15. Bc-1.07.1 报错`/bin/sh: /tools/bin/makeinfo: /home/lfs/lfs/tools/bin/perl: bad interpreter: No such file or directory`
makeinfo使用了错误的依赖`/home/lfs/lfs/tools/bin/perl`, 先重新编译texinfo, 再编译bc即可.

### 6.17. GMP-6.1.2报错`ar: error while loading shared libraries: libbfd-2.32.so: cannot open shared object file: No such file or directory`
使用`whereis ar`确定ar位置, 再使用`ldd /usr/bin/ar`查看其使用的共享库.

通过宿主机查找`libbfd-2.32.so`
```
$ sudo updatedb --localpaths='/home/lfs'                    
$ locate libbfd-2.32.so |grep lfs
/home/lfs/lfs/sources/binutils-2.32/build/bfd/.libs/libbfd-2.32.so
```
不知之前编译安装binutils时`libbfd-2.32.so`没有复制到`/lib`,复制即可.

### 6.18. MPFR-4.0.2 的测试结果tests/test-suite.log报错`./tversion: error while loading shared libraries: libgmp.so.10: cannot open shared object file: No such file or directory`
gmp安装时so安装在`/usr/lib`修正LD_LIBRARY_PATH即可.

### 6.21. GCC-9.1.0 检查编译结果时报错`su: Cannot drop the controlling terminal`
```
$ su nobody -s /bin/bash -c "PATH=$PATH make -k check"
su: Cannot drop the controlling terminal
```

参考[Chroot problem](https://lfs-support.linuxfromscratch.narkive.com/sZIeWLZk/chroot-problem): `Before you enter chroot, mount /dev, /proc, /sys, etc according to Section 6.2.`

根本原因是:中途关机了, 再次chroot时编译环境重置. 切记编译中途不许退出环境.

### 6.21. GCC-9.1.0 `grep found dummy.log`返回`found ld-linux-x86-64.so.2 at /tools/lib/ld-linux-x86-64.so.2`
`grep 'SEARCH.*/usr/lib' dummy.log |sed 's|; |\n|g'`返回结果也包含了`/tools`, 原因未知.

### 6.25. Attr-2.4.48 make报错`gcc: error: ./.libs/libattr.so: No such file or directory`
未知