# lfs
参考:
- [Linux From Scratch SVN-20190618 中文翻译版](https://bf.mengyan1223.wang/lfs/zh_CN/)
- [LCTT/LFS-BOOK](https://github.com/LCTT/LFS-BOOK)

## 1. 准备
1. 创建lfs用户, 建立干净的编译环境, 以避免其他环境变量的影响

    ```sh
    $ sudo useradd -s /bin/bash -m lfs
    $ sudo mkdir /home/lfs/lfs # 建立 base 目录
    $ sudo mount -v -t ext4 /dev/sdb /home/lfs/lfs # 挂载硬盘, 但这里仅单纯的新建目录也可以
    $ sudo chown -R lfs:lfs /home/lfs/lfs # 挂载后需修改所属
    $ su - lfs # 切换用户
    $ export LFS=/home/lfs/lfs
    $ mkdir -v $LFS/sources # 建立sources, 放需要下载的资源
    $ mkdir -v $LFS/tools # 建立tools, 编译生成的临时工具都会安装到这里
    $ cat > ~/.bashrc << EOF
    set +h                  # 关闭了 bash 的哈希功能, 使得程序执行的时候就会一直搜索 PATH
    umask 022
    LFS=/home/lfs/lfs
    LC_ALL=POSIX
    LFS_TGT=x86_64-lfs-linux-gnu
    PATH=$LFS/tools/bin:/bin:/usr/bin
    MAKEFLAGS='-j 8' # 根据机器cpu配置
    export LFS LC_ALL LFS_TGT PATH MAKEFLAGS
    EOF
    $ source ~/.bashrc # 使环境变量生效
    $ env # 输出env并检查
    ```

1. 环境/依赖检查
    1. `echo $SHELL`是否`bash`
    1. `/bin/sh`和`/usr/bin/sh`是否指向`bash`, 否则通过`sudo dpkg-reconfigure bash`来修改
    1. `readlink -f /usr/bin/awk`是否`gawk`
    1. `readlink -f /usr/bin/yacc`是否`bsion`, 否则编译glic时会报错

1. 下载所需all Packages

    根据[lfs 的 `Files Mirrors`](http://www.linuxfromscratch.org/mirrors.html#files)下载所需的版本的[all packages, **推荐**](http://mirrors-usa.go-parts.com/lfs/lfs-packages/lfs-packages-8.4.tar), 该tar已打包所有所需, 各包的作用在[这里](https://lctt.github.io/LFS-BOOK/lfs-systemd/prologue/package-choices.html).

    其他下载方式(**不推荐**): 根据相应版本的wget-list和md5sums逐个下载, 比如这里的[8.4](http://mirrors-usa.go-parts.com/lfs/lfs-packages/8.4/wget-list), 命令如下:
    ```
    $ wget --input-file=wget-list --continue --directory-prefix=sources # 支持断点恢复
    $ md5sum -c md5sums        # 校验下载
    ```

## 2. 第一次编译
这里没有将`/tools`软连接到`$LFS/tools`

1. Binutils
    ```
    $ cd /home/lfs/lfs/tools # 在 x86_64 上构建需建立lib64
    $ mkdir lib
    $ ln -s lib lib64
    ...
    $../configure --prefix=$LFS/tools \
    --with-sysroot=$LFS \
    --with-lib-path=$LFS/tools/lib \
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
    --prefix=$LFS/tools \
    --with-glibc-version=2.11 \
    --with-sysroot=$LFS \
    --with-newlib \
    --without-headers \
    --with-local-prefix=$LFS/tools \
    --with-native-system-header-dir=$LFS/tools/include \
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
    $ patch -p1 < ../glibc-2.29-fhs-1.patch
    ...
    ```

3. 第二次编译

## FAQ
### Binutils, GCC编译了两遍
为了构建交叉编译环境, 解除目标系统与宿主系统的关系. 比如:
```
$ ldd /home/lfs/lfs/tools/bin/x86_64-lfs-linux-gnu-ar # 第一遍编译Binutils
	linux-vdso.so.1 (0x00007ffea9f22000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fd6ffa59000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fd6ff6ba000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fd6fff8e000)
```