# boot
ref:
- [systemd时代的开机启动流程](https://www.junmajinlong.com/linux/systemd/systemd_bootup/)

计算机启动流程可以分为几个大阶段:
1. 内核加载前
1. 本阶段和操作系统无关，Linux或Windows或其它系统在这阶段的顺序是一样的
1. 内核加载中–>内核启动完成
1. 内核加载后–>系统环境初始化完成
1. 终端加载、用户登录

# 细节

## 1. cpu加电开始引导
cpu加电运行相关代码后调到第一个程序bios/uefi, 将cpu控制权交给它们.

> Bios典型支持的是MBR分区类型，但也支持GPT分区类型。UEFI典型支持的是GPT分区类型，但也支持MBR。**从系统启动的角度看，无需在乎是MBR还是GPT，其基本目的都是找到各阶段的bootloader**.


### bios
对于BIOS来说，BIOS的工作包括：
1. POST，即加电对部分硬件进行检查
1. POST自检之后，BIOS初始化一部分启动所需的硬件(比如会启动磁盘，某些机器可能还会启动键盘)
1. 根据启动顺序找到排在第一位的磁盘
1. BIOS跳转到所选择磁盘的前446字节，这446字节的代码是第一个bootloader程序，BIOS加载bootloader并将CPU控制权交给bootloader程序

	- 磁盘的第一个扇区(前512字节)称为MBR，其中前446字节是bootloader程序，中间64字节是磁盘分区表，最后两个字节是固定0x55AA的magic标记，标记该磁盘的MBR是否有效，如果无效，则读取启动顺序中的第二位磁盘
	- MBR中的bootloader是硬编码在磁盘第一个扇区中的，像grub类的启动管理器会安装各个阶段的boot loader，包括这个MBR bootloader

1. 执行MBR中的bootloader程序，这个bootloader可能会直接加载内核，也可能会跳转到更多的bootloader程序上

	- 因为MBR中的bootloader只有446字节，代码量非常有限，所以有些系统会使用多段bootloader的方式。比如grub在安装MBR的时候，所安装的MBR bootloader最后的代码逻辑是跳转到下一个阶段的bootloader(也是grub安装的)，如果下一个阶段的bootloader还不够，那么还可以有更多阶段的bootloader。这种模式称为链式启动

	- 因为内核镜像和启动管理器配置文件等内核启动所必须的文件都在boot分区下，所以中间某个bootloader程序会加载boot分区所在文件系统驱动，使之能够识别boot分区并读取boot分区中的文件
1. 最后一个bootloader将获取内核启动参数(比如从boot分区读取grub配置文件获取内核参数)，并加载操作系统的内核镜像(grub配置文件中指定了内核镜像的路径), 同时向内核传递内核启动参数，然后将CPU控制权交给内核程序

至此，内核已经加载到内存中，并进入到了内核启动阶段，CPU控制权将转移到内核，内核开始工作.

### uefi
![systemd时代的开机启动流程(UEFI+systemd)](/misc/img/os/boot/boot-uefi.png)

UEFI支持读取分区表，也支持直接读取一个文件系统。UEFI不会启动任何MBR扇区中的bootloader(即使有安装MBR)，而是从非易失性存储中找到启动条目(boot entry)并启动它。

典型的UEFI支持的非易失性存储有FAT12、FAT16和FAT32文件系统(即EFI系统分区)，但是发行商也可以加入额外的文件系统，只要提供对应文件系统的驱动程序即可。比如Macs支持HFS+文件系统。此外，UEFI也支持ISO-9660光盘。

UEFI会启动EFI系统分区中的EFI程序，所谓的EFI程序即类似bootloader的程序，比如单纯的bootloader程序，类似GRUB的启动管理器、UEFI shell程序等。这些程序通常位于efi系统分区下的/EFI/vendor_name目录中，不同发行商可加入不同的efi程序。在EFI分区的/efi/目录下还有一个boot目录，这个目录中保存了所有的启动条目.

EFI目录下除了Boot目录外，可能还有其他发行商(比如deepin、Microsoft、Ubuntu)各自的EFI程序目录.

> systemd-boot工具也可以制作基于UEFI的bootloader.


当使用UEFI固件时，CPU通电后，UEFI的工作内容主要包括：
1. POST, 即加电对部分硬件进行检查
1. POST自检后, uefi程序初始化一部分启动所需的硬件(比如会启动磁盘, 某些机器可能启动键盘)
1. uefi读取保存在efi分区中的启动记录, 从而决定启动哪个efi程序以及从哪里启动该程序
1. uefi启动efi程序, 并获取内核启动参数, 加载内核镜像到内存等.

至此，内核已经加载到内存中，并进入到了内核启动阶段，CPU控制权将转移到内核，内核开始工作

## 2. 内核启动阶段
到目前为止，内核已经被加载到内存掌握了控制权，且收到了boot loader最后传递的内核启动参数，包括init ramdisk镜像的路径

但注意，所有的内核镜像都是以bzImage方式压缩过的, 压缩后CentOS 7的内核大小大约为5M，CentOS 8的内核大小约9M。所以内核要能正常运作下去，它需要进行解压释放.

> 可通过`ls -lh /boot/vmlinuz-*`查看压缩后内核的大小

> 内核引导协议要求bootloader最后将内核镜像读取到内存中，内核镜像是以bzImage格式被压缩。bootloader读取内核镜像到内存后，会调用内核镜像中的startup_32()函数对内核解压，也就是说，内核是自解压的。解压之后，内核被释放，开始调用另一个startup_32()函数，startup32函数初始化内核启动环境，然后跳转到start_kernel()函数，内核就开始真正启动了，PID=0的0号进程也开始了.

当内核真正开始运行后，将从/boot分区找到initial ram disk image(后面简称init ramdisk)并解压。init ramdisk要么是initrd，要么是initramfs，它们可使用dracut工具生成，早期系统用initrd比较多，因为现在就会都用initramfs，所以现在以initramfs来描述init ramdisk.

init ramdisk解压之后就得到了内核空间的根文件系统(这是启动过程中的早期根分区，也称为虚根)。对于使用systemd的系统来说，得到虚根之后就可以调用集成在initramfs中的systemd程序，其PID=1。从现在开始，就进入了早期的用户空间(early userspace)，systemd进程可以在此时做一些内核启动剩余的必要操作。

完成内核启动的必要操作后，systemd最后会将虚根切换为真实的根分区(即系统启动后用户看到的根分区)，并进入真正的用户空间阶段。systemd从此开始就成为了用户空间的总管进程，也是所有用户空间进程的祖先进程。

### 部分细节
1. 如何找到init ramdisk

	实际上，内核运行后，会创建一个负责内核运行环境初始化的进程，该进程会根据boot loader传递过来的内核启动参数找到init ramdisk的路径。正常情况下，该路径在boot分区中。**之所以能够读取Boot分区是因为在固件阶段，bootloader程序已经加载了boot分区的文件系统驱动**。

1. 为什么要用init ramdisk

	因为直到现在还无法读取根分区，甚至目前还没有根分区的存在，但显然，之后是要挂载根分区的。但是要挂载根分区进而访问根分区，要求知道根分区的类型(比如是xfs还是ext4)，从而装在根文件系统的驱动模块。

	由于不同用户在安装操作系统时，可能会选择不同的文件系统作为根文件系统的类型，所以不同用户的操作系统在内核启动过程中，根文件系统类型是不同的。如何知道该用户所装的操作系统的根文件系统是哪种类型的？

	事实上，在安装操作系统的最后阶段，会自动收集当前所装操作系统的信息，并根据这些信息动态生成一些文件，包括用户所选择的根文件系统的类型以及对应的文件系统驱动模块。这个过程收集的内容会保存在init ramdisk镜像中. 比如安装centos时, 安装器的进度条上会提示`Generating initramfs`.

1. 内核中根分区的来源

	内核会将init ramdisk镜像解压在一个位置下，这个位置称为rootfs，即根文件系统，这个根分区称为虚根，因为这个根分区和系统启动后用户看到的根分区不是同一个根分区。

	对于使用initramfs镜像的ramdisk来说，这个rootfs即为ramfs(ram file system)，它是一个在解压initramfs镜像后就存在且挂载的文件系统，但是系统启动之后用户并不能找到它，因为在内核启动完成后它就会被切换到真实的根文件系统。

	可以手动解压/boot/initrd.img所指向的镜像文件, 我这里是initrd.img-5.11.0-25-generic:
	```bash
	# os: Ubuntu 20.04.4 LTS
	# file initrd.img-5.11.0-25-generic # 如果返回`gzip compressed data,...`, 则需要先执行`gunzip initrd.img-5.11.0-25-generic`
	initrd.img-5.11.0-25-generic: ASCII cpio archive (SVR4 with no CRC)
	# binwalk initrd.img-5.11.0-25-generic # 分析initrd.img-5.11.0-25-generic结构: 前部是intel和amd的microcode, 后部是具体内容

	DECIMAL       HEXADECIMAL     DESCRIPTION
	--------------------------------------------------------------------------------
	0             0x0             ASCII cpio archive (SVR4 with no CRC), file name: ".", file name length: "0x00000002", file size: "0x00000000"
	112           0x70            ASCII cpio archive (SVR4 with no CRC), file name: "kernel", file name length: "0x00000007", file size: "0x00000000"
	232           0xE8            ASCII cpio archive (SVR4 with no CRC), file name: "kernel/x86", file name length: "0x0000000B", file size: "0x00000000"
	356           0x164           ASCII cpio archive (SVR4 with no CRC), file name: "kernel/x86/microcode", file name length: "0x00000015", file size: "0x00000000"
	488           0x1E8           ASCII cpio archive (SVR4 with no CRC), file name: "kernel/x86/microcode/AuthenticAMD.bin", file name length: "0x00000026", file size: "0x00007752"
	31184         0x79D0          ASCII cpio archive (SVR4 with no CRC), file name: "TRAILER!!!", file name length: "0x0000000B", file size: "0x00000000"
	31744         0x7C00          ASCII cpio archive (SVR4 with no CRC), file name: "kernel", file name length: "0x00000007", file size: "0x00000000"
	31864         0x7C78          ASCII cpio archive (SVR4 with no CRC), file name: "kernel/x86", file name length: "0x0000000B", file size: "0x00000000"
	31988         0x7CF4          ASCII cpio archive (SVR4 with no CRC), file name: "kernel/x86/microcode", file name length: "0x00000015", file size: "0x00000000"
	32120         0x7D78          ASCII cpio archive (SVR4 with no CRC), file name: "kernel/x86/microcode/.enuineIntel.align.0123456789abc", file name length: "0x00000036", file size: "0x00000000"
	32284         0x7E1C          ASCII cpio archive (SVR4 with no CRC), file name: "kernel/x86/microcode/GenuineIntel.bin", file name length: "0x00000026", file size: "0x00465400"
	4641456       0x46D2B0        ASCII cpio archive (SVR4 with no CRC), file name: "TRAILER!!!", file name length: "0x0000000B", file size: "0x00000000"
	4641792       0x46D400        LZ4 compressed data, legacy # 这里看到`LZ4 compressed data` # 看到`gzip compressed data`时用: dd if=/initrd.img-5.11.0-25-generic bs=4641792 skip=1 | zcat | cpio -id --no-absolute-filenames -v
	5171198       0x4EE7FE        SHA256 hash constants, little endian
	5243845       0x5003C5        LUKS_MAGIC
	5332953       0x515FD9        mcrypt 2.2 encrypted data, algorithm: RC2, mode: CBC, keymode: 8bit
	5399677       0x52647D        Unix path: /usr/share/locale
	5503849       0x53FB69        Copyright string: "Copyright (C) 2005-2007 Yura Pakhuchiy'"
	5554144       0x54BFE0        Copyright string: "Copyright (C) 1994 Ian Jackson,  "
	5579920       0x552490        gzip compressed data, ASCII, from VM/CMS, last modified: 1982-10-25 21:49:56 (bogus date)
	6208732       0x5EBCDC        LZO compressed data
	6208865       0x5EBD61        xz compressed data
	6529848       0x63A338        Unix path: /lib/firmware/amdgpu/bonaire_uvd.bin
	6706042       0x66537A        Certificate in DER format (x509 v3), header length: 4, sequence length: 1417
	6708257       0x665C21        Unix path: /lib/firmware/amdgpu/bonaire_vce.bin
	6776791       0x6767D7        Certificate in DER format (x509 v3), header length: 4, sequence length: 1417
	6820782       0x6813AE        Unix path: /lib/firmware/amdgpu/carrizo_mec2.bin
	6853138       0x689212        Unix path: /lib/firmware/amdgpu/carrizo_pfp.bin
	7064279       0x6BCAD7        Unix path: /lib/firmware/amdgpu/carrizo_vce.bin
	7281234       0x6F1A52        Unix path: /lib/firmware/amdgpu/dimgrey_cavefish_mec.bin
	7325218       0x6FC622        Unix path: /lib/firmware/amdgpu/dimgrey_cavefish_mec2.bin
	7388387       0x70BCE3        Unix path: /lib/firmware/amdgpu/dimgrey_cavefish_rlc.bin
	7579498       0x73A76A        Unix path: /lib/firmware/amdgpu/dimgrey_cavefish_sos.bin
	7852092       0x77D03C        Unix path: /lib/firmware/amdgpu/dimgrey_cavefish_vcn.bin
	...
	# lsinitramfs initrd.img-5.11.0-25-generic # 可查看initrd.img-5.11.0-25-generic内容
	# dd if=initrd.img-5.11.0-25-generic bs=4641792 skip=1 of=initrd.img.lz4
	# lz4 --test initrd.img.lz4 --verbose # 查看initrd.img.lz4是否ok
	# lz4 initrd.img.lz4
	# cat initrd.img | cpio -id --no-absolute-filenames -v
	# ll
	总用量 36K
	lrwxrwxrwx  1 chen chen    7 3月   4 19:23 bin -> usr/bin/
	drwxr-xr-x  3 chen chen 4.0K 3月   4 19:23 conf/
	drwxr-xr-x  2 chen chen 4.0K 3月   4 19:23 cryptroot/
	drwxr-xr-x 12 chen chen 4.0K 3月   4 19:23 etc/
	-rwxr-xr-x  1 chen chen 7.3K 3月   4 19:23 init*
	lrwxrwxrwx  1 chen chen    7 3月   4 19:23 lib -> usr/lib/
	lrwxrwxrwx  1 chen chen    9 3月   4 19:23 lib32 -> usr/lib32/
	lrwxrwxrwx  1 chen chen    9 3月   4 19:23 lib64 -> usr/lib64/
	lrwxrwxrwx  1 chen chen   10 3月   4 19:23 libx32 -> usr/libx32/
	drwxr-xr-x  2 chen chen 4.0K 3月   4 19:23 run/
	lrwxrwxrwx  1 chen chen    8 3月   4 19:23 sbin -> usr/sbin/
	drwxr-xr-x 10 chen chen 4.0K 3月   4 19:23 scripts/
	drwxr-xr-x 10 chen chen 4.0K 3月   4 19:24 usr/
	drwxr-xr-x  4 chen chen 4.0K 3月   4 19:24 var/
	```

	> [centos7 initramfs解包 打包](https://www.jianshu.com/p/218544a3531b)

	可以想象一下，init ramdisk镜像解压在/tmp/hhh目录下，那么这个目录就可以看作是在内核启动过程中的rootfs。解压得到的根目录和系统启动后的根目录内核很相似:
	```bash
	# ll /
	总用量 84K
	lrwxrwxrwx   1 root root    7 12月 17 12:09 bin -> usr/bin/
	drwxr-xr-x   4 root root 4.0K 3月   4 09:16 boot/
	drwxr-xr-x   2 root root 4.0K 1月  26 20:56 cdrom/
	drwxr-xr-x  20 root root 4.2K 3月   4 09:50 dev/
	drwxr-xr-x 153 root root  12K 3月   4 09:21 etc/
	drwxr-xr-x   4 root root 4.0K 1月  26 21:01 home/
	lrwxrwxrwx   1 root root    7 12月 17 12:09 lib -> usr/lib/
	lrwxrwxrwx   1 root root    9 12月 17 12:09 lib32 -> usr/lib32/
	lrwxrwxrwx   1 root root    9 12月 17 12:09 lib64 -> usr/lib64/
	lrwxrwxrwx   1 root root   10 12月 17 12:09 libx32 -> usr/libx32/
	drwx------   2 root root  16K 1月  26 20:50 lost+found/
	drwxr-xr-x   3 root root 4.0K 1月  27 09:17 media/
	drwxr-xr-x   2 root root 4.0K 12月 17 12:09 mnt/
	drwxr-xr-x  11 root root 4.0K 2月  28 09:51 opt/
	dr-xr-xr-x 335 root root    0 3月   4 09:50 proc/
	drwx------   8 root root 4.0K 2月  22 11:04 root/
	drwxr-xr-x  32 root root  940 3月   4 14:05 run/
	lrwxrwxrwx   1 root root    8 12月 17 12:09 sbin -> usr/sbin/
	drwxr-xr-x   2 root root 4.0K 1月  26 21:07 snap/
	drwxr-xr-x   2 root root 4.0K 12月 17 12:09 srv/
	dr-xr-xr-x  13 root root    0 3月   4 09:50 sys/
	drwxrwxrwt  30 root root  12K 3月   4 19:25 tmp/
	drwxr-xr-x  14 root root 4.0K 12月 17 12:14 usr/
	drwxr-xr-x  13 root root 4.0K 2月  25 12:56 var/
	```

	再深入一点看，会发现ramdisk中已经包含了当前操作系统多种fs的驱动模块:
	```bash
	# ll usr/lib/modules/5.11.0-25-generic/kernel/fs/ # 没有ext4是因为它已内置进kernel
	总用量 48K
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 btrfs/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 f2fs/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 fscache/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 isofs/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 jfs/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 lockd/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 nfs/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 nfs_common/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 nls/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 reiserfs/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 udf/
	drwxr-xr-x 2 chen chen 4.0K 3月   4 19:24 xfs/
	# cat /boot/config-5.11.0-25-generic |grep -i ext4
	CONFIG_EXT4_FS=y
	CONFIG_EXT4_USE_FOR_EXT2=y
	CONFIG_EXT4_FS_POSIX_ACL=y
	CONFIG_EXT4_FS_SECURITY=y
	# CONFIG_EXT4_DEBUG is not set
	```

1. PID=1进程的来源

	或者说，PID=1的init程序集成在init ramdisk中？在内核启动过程中就加载了它？

	对于早期的initrd来说，init程序并没有集成在initrd中，所以那时的内核会在装载完根分区驱动模块后去根分区寻找/sbin/init程序并调用它。且对于使用initrd的启动过程来说，加载/sbin/init的时机比较晚，很多内核启动过程中的环境都是由内核进程而非init进程完成的。

	对于initramfs，它已经将init程序集成在init ramdisk镜像文件中了。如下:
	```bash
	# ll init # init是脚本, 它会选择systemd最为init
	# /sbin/init -> /lib/systemd/systemd
	```

	由于内核加载到这里已经初始化一些运行环境了，有些环境是可以保留下来的，这样系统启动后就能直接使用这些已经初始化的环境而无需再次初始化。比如内核的运行状态等参数保存在虚根的/proc和/sys(当然，内核运行环境是保存在内存中的，这两个目录只是内核暴露给用户的路径)，再比如，已经收集到的硬件设备信息以及设备的运行环境也要保存下来，保存的位置是/dev。

	sysroot则是最重要的，它就是系统启动后用户看到的真正的根分区。没错，真正的根分区只是ramdisk中的一个目录???(没找到sysroot, 要系统启动时进入initramfs吗?)。


## 3. systemd在内核启动阶段做的事
在内核启动阶段，当调用了集成在initramfs中的systemd之后，systemd将接管后续的启动操作。但是在这个阶段，systemd具体会做哪些操作呢？

![启动流程中的systemd](/misc/img/os/boot/systemd-start.png)

在启动过程中，systemd有几个大目标，每个目标以.target表示。systemd的target主要作用是对多个任务进行分组，比如basic.target中，可能包含了很多任务:

1. 第一个大目标：sysinit.target

	sysinit.target主要目的是读系统运行环境做初始化，初始化完成后进入下一个大目标

2. 第二个大目标：basic.target

	basic.target的作用主要是在环境初始化完成后执行一些基本任务，算是做一些早期的开机自启动任务，basic.target完成后进入第三个大目标

3. 第三个大目标：『default.target运行级别』

	default.target是一个软链接，链接到不同的target表示进入不同的『运行级别』，运行级别阶段的目的是为最终的登录做准备:
	- 如果是内核启动过程中(内核启动时调用了集成在initramfs中的systemd就处于这个过程)，那么大目标是initrd.target，该target的目标是为后续虚根切换到实根做初始化准备，比如检查并挂载根文件系统，最终虚根切换实根，并进入真正的用户空间，也即完成了内核启动或内核已经成功登录
	- 如果不是内核启动过程中，可能会选择不同的『运行级别』，通常是图形终端的graphical.target，或者多用户的运行级别multi-user.target，它们的目标都是完成系统启动，并准备让用户登录系统

所以，在内核启动阶段，要分析systemd的工作流程，沿着sysinit-->basic-->initrd-->kernel launched这条路线即可。

回到在内核启动阶段，systemd接管后续的启动操作，具体会做哪些事呢？在`man bootup`手册中给出了内核启动阶段中systemd的工作流程图:

![](/misc/img/os/boot/1594211364793.png)


图中所有涉及到的unit文件均来自initramfs镜像解压后的目录，也即虚根，而不是来自真实的根文件系统，因为现在还没有真实的根文件系统。例如<ramfs>/usr/lib/systemd/system/sysinit.target文件.

对于已经启动完成的正常系统来说，sysinit.target是用于做系统初始化的，basic.target则是系统初始化完成后执行的一些基本任务，比如启动所有定时器任务，开始监控指定文件等，算是系统启动过程中早期的开机自启动任务。

但是在内核启动阶段，sysinit.target和basic.target所代表的含义，显然与启动完成后这两个文件代表的含义有所不同。在内核启动阶段，sysinit.target中的sys指的是内核阶段的虚拟系统，basic则代表ramdisk决定要执行的基础任务。

换句话说，在内核启动阶段，systemd也将initramfs所启动的环境看作是一个操作系统，只不过是一个虚拟的操作系统。

当basic.target完成后，进入initrd.target，之所以是initrd.target，是因为initramfs中的default.target软链接指向的是initrd.target。
```bash
ll /usr/lib/systemd/system/default.target # 我这里是graphical.target
lrwxrwxrwx 1 root root 16 1月  10 12:56 /usr/lib/systemd/system/default.target -> graphical.target
```

![](/misc/img/os/boot/1594211722595.png)

在initrd阶段，systemd为后续切换到真实的根文件系统做准备，比如检查根文件系统并将其挂载在<ramfs>/sysroot下，再比如从/etc/fstab中找出部分需要在这个阶段挂载的分区。如果一切没问题，将进入最后的阶段：从initramfs的虚根<ramfs>/切换到实根<ramfs>/sysroot

![](/misc/img/os/boot/1594211753716.png)

从此开始，<ramfs>/sysroot将代表真实的根文件系统，systemd也将从这个根文件系统调用init程序(systemd)替换当前的systemd进程(所以PID仍然为1)。从此内核引导阶段退出舞台，开始进入真正的用户空间，systemd进程也将在这个用户空间开始下一阶段的流程：sysinit->basic->default->login。

所以，总结一下内核启动阶段中systemd接管控制权后到用户登录中间的流程:
![](/misc/img/os/boot/1594295417512.png)

## 4. 内核启动后，用户登录前
当systemd(这个是来自真实根文件系统的systemd进程)进入到用户空间后，systemd将执行下一轮工作流程，全局路线为：sysinit.target->basic.target->default.target->...->login.

其中default.target是一个软链接，一般指向graphical.target(图形界面)或multi-user.target(多用户模式)，对应于SysV系统中的『运行级别』阶段。

![](/misc/img/os/boot/systemd-from-root.png)

需注意，在这里涉及到的所有unit文件都来自于真实的根文件系统，例如/usr/lib/systemd/system/sysinit.target。而内核启动阶段systemd工作路线中涉及到的unit文件都来自于initramfs构建的虚根，例如<ramfs>/usr/lib/systemd/system/sysinit.target，而且前文也提到过，在systemd眼中，initramfs构建的也是一个系统，只不过是虚拟系统，最终systemd会从这个虚拟系统中切换到真实的系统中，切换的内容主要包括两项：切换根分区，切换systemd进程自身.

在流程的每一个大阶段，和前面介绍的initramfs中的systemd是类似的。

![](/misc/img/os/boot/1594341612340.png)

basic.target完成后，将通过default.target决定进入哪一个『运行级别』。如果是进入graphical.target，那么会启动和图形终端相关的服务任务，如果是进入multi-user.target，那么:
```bash
# ll /usr/lib/systemd/system/multi-user.target.wants
总用量 0
lrwxrwxrwx 1 root root 15 6月  12  2020 dbus.service -> ../dbus.service
lrwxrwxrwx 1 root root 15 1月  10 12:56 getty.target -> ../getty.target
lrwxrwxrwx 1 root root 24 11月  2  2020 plymouth-quit.service -> ../plymouth-quit.service
lrwxrwxrwx 1 root root 29 11月  2  2020 plymouth-quit-wait.service -> ../plymouth-quit-wait.service
lrwxrwxrwx 1 root root 33 1月  10 12:56 systemd-ask-password-wall.path -> ../systemd-ask-password-wall.path
lrwxrwxrwx 1 root root 25 1月  10 12:56 systemd-logind.service -> ../systemd-logind.service
lrwxrwxrwx 1 root root 39 1月  10 12:56 systemd-update-utmp-runlevel.service -> ../systemd-update-utmp-runlevel.service
lrwxrwxrwx 1 root root 32 1月  10 12:56 systemd-user-sessions.service -> ../systemd-user-sessions.service
```

其中几项需要注意：
- getty.target：启动虚拟终端实例(或容器终端实例)并初始化它们
- systemd-logind：负责管理用户登录操作
- systemd-user-sessions：控制系统是否允许登录
- systemd-update-utmp-runlevel：切换运行级别时在utmp和wtmp中记录切换信息
- systemd-ask-password-wall：负责查询用户密码

除了这几个multi-user.target所依赖的服务外，在/etc/systemd/system/multi-user.target.wants下也有需要启动的服务，这里的服务是用户定义的systemd类的开机自启动服务:
```bash
# ll /etc/systemd/system/multi-user.target.wants/
总用量 0
lrwxrwxrwx 1 root root 35 12月 17 12:19  anacron.service -> /lib/systemd/system/anacron.service
lrwxrwxrwx 1 root root 35 2月  21 09:27  anydesk.service -> /etc/systemd/system/anydesk.service
lrwxrwxrwx 1 root root 40 12月 17 12:22  avahi-daemon.service -> /lib/systemd/system/avahi-daemon.service
lrwxrwxrwx 1 root root 42 12月 17 12:20  binfmt-support.service -> /lib/systemd/system/binfmt-support.service
lrwxrwxrwx 1 root root 52 12月 17 12:21  biometric-authentication.service -> /lib/systemd/system/biometric-authentication.service
lrwxrwxrwx 1 root root 41 12月 17 12:22  console-setup.service -> /lib/systemd/system/console-setup.service
lrwxrwxrwx 1 root root 32 12月 17 12:19  cron.service -> /lib/systemd/system/cron.service
lrwxrwxrwx 1 root root 35 1月  27 11:52  ctxlogd.service -> /lib/systemd/system/ctxlogd.service
lrwxrwxrwx 1 root root 40 12月 17 12:22  cups-browsed.service -> /lib/systemd/system/cups-browsed.service
lrwxrwxrwx 1 root root 29 12月 17 12:22  cups.path -> /lib/systemd/system/cups.path
lrwxrwxrwx 1 root root 33 12月 17 12:20  dmesg.service -> /lib/systemd/system/dmesg.service
lrwxrwxrwx 1 root root 37 12月 17 12:21  dns-clean.service -> /lib/systemd/system/dns-clean.service
lrwxrwxrwx 1 root root 39 1月  26 21:44  grub-common.service -> /lib/systemd/system/grub-common.service
lrwxrwxrwx 1 root root 48 12月 17 12:22  grub-initrd-fallback.service -> /lib/systemd/system/grub-initrd-fallback.service
lrwxrwxrwx 1 root root 38 12月 17 12:20  irqbalance.service -> /lib/systemd/system/irqbalance.service
lrwxrwxrwx 1 root root 38 12月 17 12:22  kerneloops.service -> /lib/systemd/system/kerneloops.service
lrwxrwxrwx 1 root root 45 12月 17 12:22  kylin-kmre-daemon.service -> /lib/systemd/system/kylin-kmre-daemon.service
lrwxrwxrwx 1 root root 49 12月 17 12:22  kylin-send-updateinfo.service -> /lib/systemd/system/kylin-send-updateinfo.service
lrwxrwxrwx 1 root root 47 12月 17 12:22  kylin-send-updateinfo.timer -> /lib/systemd/system/kylin-send-updateinfo.timer
lrwxrwxrwx 1 root root 38 12月 17 12:22  lm-sensors.service -> /lib/systemd/system/lm-sensors.service
lrwxrwxrwx 1 root root 40 12月 17 12:22  ModemManager.service -> /lib/systemd/system/ModemManager.service
lrwxrwxrwx 1 root root 47 12月 17 12:22  networkd-dispatcher.service -> /lib/systemd/system/networkd-dispatcher.service
lrwxrwxrwx 1 root root 38 12月 17 12:21  networking.service -> /lib/systemd/system/networking.service
lrwxrwxrwx 1 root root 42 12月 17 12:22  NetworkManager.service -> /lib/systemd/system/NetworkManager.service
lrwxrwxrwx 1 root root 32 12月 17 12:22  nmbd.service -> /lib/systemd/system/nmbd.service
lrwxrwxrwx 1 root root 36 12月 17 12:14  ondemand.service -> /lib/systemd/system/ondemand.service
lrwxrwxrwx 1 root root 41 12月 17 12:22  open-vm-tools.service -> /lib/systemd/system/open-vm-tools.service
lrwxrwxrwx 1 root root 36 12月 17 12:20  pppd-dns.service -> /lib/systemd/system/pppd-dns.service
lrwxrwxrwx 1 root root 36 12月 17 12:14  remote-fs.target -> /lib/systemd/system/remote-fs.target
lrwxrwxrwx 1 root root 33 12月 17 12:21  rsync.service -> /lib/systemd/system/rsync.service
lrwxrwxrwx 1 root root 35 12月 17 12:20  rsyslog.service -> /lib/systemd/system/rsyslog.service
lrwxrwxrwx 1 root root 45 12月 17 12:22 'run-vmblock\x2dfuse.mount' -> '/lib/systemd/system/run-vmblock\x2dfuse.mount'
lrwxrwxrwx 1 root root 41 12月 17 12:21  secureboot-db.service -> /lib/systemd/system/secureboot-db.service
lrwxrwxrwx 1 root root 41 12月 17 12:19  smartmontools.service -> /lib/systemd/system/smartmontools.service
lrwxrwxrwx 1 root root 32 12月 17 12:22  smbd.service -> /lib/systemd/system/smbd.service
lrwxrwxrwx 1 root root 42 12月 17 12:22  snapd.apparmor.service -> /lib/systemd/system/snapd.apparmor.service
lrwxrwxrwx 1 root root 44 12月 17 12:22  snapd.autoimport.service -> /lib/systemd/system/snapd.autoimport.service
lrwxrwxrwx 1 root root 44 12月 17 12:22  snapd.core-fixup.service -> /lib/systemd/system/snapd.core-fixup.service
lrwxrwxrwx 1 root root 58 12月 17 12:22  snapd.recovery-chooser-trigger.service -> /lib/systemd/system/snapd.recovery-chooser-trigger.service
lrwxrwxrwx 1 root root 40 12月 17 12:22  snapd.seeded.service -> /lib/systemd/system/snapd.seeded.service
lrwxrwxrwx 1 root root 33 12月 17 12:22  snapd.service -> /lib/systemd/system/snapd.service
lrwxrwxrwx 1 root root 44 12月 17 12:14  systemd-resolved.service -> /lib/systemd/system/systemd-resolved.service
lrwxrwxrwx 1 root root 36 12月 17 12:22  thermald.service -> /lib/systemd/system/thermald.service
lrwxrwxrwx 1 root root 41 1月  26 21:46  ua-license-check.path -> /lib/systemd/system/ua-license-check.path
lrwxrwxrwx 1 root root 42 12月 17 12:20  ua-reboot-cmds.service -> /lib/systemd/system/ua-reboot-cmds.service
lrwxrwxrwx 1 root root 31 12月 17 12:21  ufw.service -> /lib/systemd/system/ufw.service
lrwxrwxrwx 1 root root 46 12月 17 12:22  ukui-group-manager.service -> /lib/systemd/system/ukui-group-manager.service
lrwxrwxrwx 1 root root 55 12月 17 12:22  ukui-media-control-mute-led.service -> /lib/systemd/system/ukui-media-control-mute-led.service
lrwxrwxrwx 1 root root 47 12月 17 12:21  unattended-upgrades.service -> /lib/systemd/system/unattended-upgrades.service
lrwxrwxrwx 1 root root 36 12月 17 12:22  whoopsie.service -> /lib/systemd/system/whoopsie.service
lrwxrwxrwx 1 root root 42 12月 17 12:22  wpa_supplicant.service -> /lib/systemd/system/wpa_supplicant.service
lrwxrwxrwx 1 root root 30 12月 17 12:22  zfs.target -> /lib/systemd/system/zfs.targe
```

不仅如此，为了兼容早期sysV的rc.local开机自启动功能，systemd会检查/etc/rc.local是否具有可执行权限，如果具备，systemd会在此阶段自动执行rc.local中的命令.