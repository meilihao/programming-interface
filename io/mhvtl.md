# [mhvtl](http://www.mhvtl.com)

[最新版v1.7(2022-05-10)](https://github.com/markh794/mhvtl/releases/tag/1.7-0_release)

## 构建
### [ubuntu](https://petertc.medium.com/%E5%9C%A8ubuntu%E5%AE%89%E8%A3%9D%E7%A3%81%E5%B8%B6%E6%AB%83%E6%A8%A1%E6%93%AC%E5%99%A8-f10b327ae3db)
```bash
# mhVTL_install_ubuntu.sh
#!/bin/sh 
# Script to download, compile and install mhvtl
# Tested on ubuntu 16.04
# Origin: http://mhvtl-a-linux-virtual-tape-library.966029.n3.nabble.com/Easy-Install-for-Debian-Ubuntu-td4025413.html
# 03/04/13 
#	Added libconfig-general-perl (ensures tgt-admin can run)
# 04/04/13
#	Added line to append www-data to sudoers automatically
#	Added check for sudo/root
#	Added copy for tgt-admin in sbin (fixes persistant config in tgt mhvtl-gui)
#	Added logging (log output to /tmp/log-mhvtl-install)
# 02/27/18
#	Fetch Makefile fixed forked repos
# 	Remove GUI install part since it is not work anymore
# Last Updated 02/27/2018

# config script vars
NONE='\033[00m'
RED='\033[01;31m'
GREEN='\033[01;32m'
BOLD='\033[1m'
LOGFILE='/tmp/log-mhvtl-install'

# function logs text string to screen and file
logText()
{
	TEXT=$1
	TLEN=${#TEXT}
	a=0
	TOUT=""
	while [ $a -lt $TLEN ]; do
		TOUT="$TOUT#"
		a=$((a+1))
	done
	if [ $2!="simple" ]; then
		echo "########$TOUT\n### $TEXT ###\n$TOUT########" >> $LOGFILE		
	else	
		echo "$TEXT" >> $LOGFILE
	fi
	echo "${GREEN}${BOLD}$TEXT${NONE}"
	tput sgr0
}

echo "" > $LOGFILE
logText "Script Begin"

# check our script has been started with root auth
if [ "$(id -u)" != "0" ]; then
	echo "${RED}${BOLD}This script must be run with root privileges. Please run again as either root or using sudo.${NONE}"
	tput sgr0
	exit 1
fi

# install required packages
logText "Downloading and Installing Packages"
sudo apt-get update >> $LOGFILE && sudo apt-get install sysstat lzop liblzo2-dev liblzo2-2 mtx mt-st sg3-utils zlib1g-dev git lsscsi build-essential gawk alien fakeroot linux-headers-$(uname -r) -y >> $LOGFILE; cd $HOME 

# make a src directory if it doesnt already exist 
logText "Check $HOME/src exists"
if [ ! -d $HOME/src ]; then
    logText "Creating directory $HOME/src" simple
    mkdir $HOME/src 
fi 

# create user, group and folders 
logText "Create Users, Groups and Folders"
sudo groupadd -r vtl >> $LOGFILE
sudo useradd -r -c "Virtual Tape Library" -d /opt/mhvtl -g vtl vtl -s /bin/bash >> $LOGFILE
sudo mkdir -p /opt/mhvtl >> $LOGFILE
sudo mkdir -p /etc/mhvtl >> $LOGFILE
sudo chown -Rf vtl:vtl /opt/mhvtl >> $LOGFILE
sudo chown -Rf vtl:vtl /etc/mhvtl >> $LOGFILE

# download, compile and install mhvtl 
logText "Download, Compile and Install mhvtl"
cd $HOME/src/ >> $LOGFILE
git clone https://github.com/markh794/mhvtl.git
cd mhvtl >> $LOGFILE
make distclean >> $LOGFILE
cd kernel/ 
make >> $LOGFILE && sudo make install >> $LOGFILE
cd .. 
make >> $LOGFILE && sudo make install >> $LOGFILE

# fix some errors 
logText "Fix some errors"
sudo mkdir /etc/tgt >> $LOGFILE
sudo ln -s /usr/lib64/libvtlscsi.so /usr/lib/ >> $LOGFILE 
sudo /etc/init.d/mhvtl start >> $LOGFILE

sleep 3
logText "Show your tape libraries now!"
lsscsi 
```

### rpm 
[To build an RPM](https://github.com/markh794/mhvtl/blob/master/INSTALL#L48)

### make
```
# --- build ko
# cd mhvtl-1.7/kernel
# make
# make install
# --- build userspace
# cd mhvtl-1.7/kernel
# make
# make install
# systemctl enable --now mhvtl-load-modules.service
# systemctl enable --now mhvtl.target
# systemctl daemon-reload
# systemctl start mhvtl.target
# systemctl status mhvtl.target
# ps -ax | fgrep vtl
# lsscsi -g # mediumx类型为磁带柜. 使用飞康vtl发现tape driver名称分配不是顺序的, 可能多个带库穿插分配.
```

### [gui](https://github.com/markh794/mhvtl-gui)

## example
ref:
- [第 15 章 管理磁带设备](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/managing_storage_devices/managing-tape-devices_managing-storage-devices)
- [小型多卷磁带库综合使用手册](https://blog.csdn.net/weixin_33725126/article/details/85589333)

> 默认情况下，在磁带设备中，块大小为 10KB(bs=10k)

工具:
```bash
# dnf install mt-st
```

op:
```bash
# tapeinfo -f /dev/sg14 # 磁带柜信息
# tapeinfo -f /dev/sg15 # 驱动器信息
# --- 磁带的条形码在/etc/mhvtl/library_contents.N中定义, 可用edit_tape来添加磁带
# vtlcmd 11 load E01001L4 # 11 是/etc/mhvtl/device.conf中的Library 11, 实际测试无效果???
# dump_tape -l 11 -f E01001L8|head # dump tape
# mtx -f /dev/sg13 status # mtx控制SCSI介质转换设备(control SCSI media changer devices), 控制单一或者多个SCSI介质转换器例如tape changers(磁带转换器) autoloaders(自动装填器), tape libraries(磁带库)或者optical media jukeboxes(光盘库). 由于历史的原因驱动器被编号从0开始，存储插槽被编号从1开始.
Storage Changer /dev/sg13:4 Drives, 43 Slots ( 4 Import/Export ) # Import/Export是邮件槽, 方便用户导出/导入磁带, 因为仅邮件槽是用户可打开的槽位, 类似于中转站.
Data Transfer Element 0:Empty           #4个磁带机
Data Transfer Element 1:Empty
Data Transfer Element 2:Empty
Data Transfer Element 3:Empty
    Storage Element 1:Full :VolumeTag=E01001L4          #多个存储磁带
    Storage Element 2:Full :VolumeTag=E01002L4
    Storage Element 3:Full :VolumeTag=E01003L4
    Storage Element 4:Full :VolumeTag=E01004L4
    Storage Element 5:Full :VolumeTag=E01005L4
    Storage Element 6:Full :VolumeTag=E01006L4
    Storage Element 7:Full :VolumeTag=E01007L4
    Storage Element 8:Full :VolumeTag=E01008L4
    Storage Element 9:Full :VolumeTag=E01009L4
    Storage Element 10:Full :VolumeTag=E01010L4
    Storage Element 11:Full :VolumeTag=E01011L4
    Storage Element 12:Full :VolumeTag=E01012L4
    Storage Element 13:Full :VolumeTag=E01013L4
    Storage Element 14:Full :VolumeTag=E01014L4
    Storage Element 15:Full :VolumeTag=E01015L4
    Storage Element 16:Full :VolumeTag=E01016L4
    Storage Element 17:Full :VolumeTag=E01017L4
    Storage Element 18:Full :VolumeTag=E01018L4
    Storage Element 19:Full :VolumeTag=E01019L4
    Storage Element 20:Full :VolumeTag=E01020L4
    Storage Element 21:Empty
    Storage Element 22:Full :VolumeTag=CLN101L4
    Storage Element 23:Full :VolumeTag=CLN102L5
    Storage Element 24:Empty
    Storage Element 25:Empty
    Storage Element 26:Empty
    Storage Element 27:Empty
    Storage Element 28:Empty
    Storage Element 29:Empty
    Storage Element 30:Full :VolumeTag=F01030L5
    Storage Element 31:Full :VolumeTag=F01031L5
    Storage Element 32:Full :VolumeTag=F01032L5
    Storage Element 33:Full :VolumeTag=F01033L5
    Storage Element 34:Full :VolumeTag=F01034L5
    Storage Element 35:Full :VolumeTag=F01035L5
    Storage Element 36:Full :VolumeTag=F01036L5
    Storage Element 37:Full :VolumeTag=F01037L5
    Storage Element 38:Full :VolumeTag=F01038L5
    Storage Element 39:Full :VolumeTag=F01039L5
    Storage Element 40 IMPORT/EXPORT:Empty
    Storage Element 41 IMPORT/EXPORT:Empty
    Storage Element 42 IMPORT/EXPORT:Empty
    Storage Element 43 IMPORT/EXPORT:Empty
# ls /opt/mhvtl/ # 磁带库的磁盘所存放的位置每一个目录就是一盘磁带. 它们槽位可用`mtx -f /dev/sg<N> status`查看
# ls /opt/mhvtl/CLN101L8 # data: 数据文件; meta: 设备描述文件; indx: 索引文件
data indx  meta
# vim /etc/mhvtl/mhvtl.conf
...
# Default media capacity (500 M)       指定每个磁盘大小

CAPACITY=500
...
# mtx -f /dev/sg13 inquiry # 获取产商信息
# mtx -f /dev/sg13 load 1 0 # 将StorageElement 1位置上的磁带设备放置到Data Transfer Element 3的磁带机中. L80是如果驱动器为空就可用放入磁带; L700是驱动器必须是逐个顺序使用, 否则报错. load模拟的是真实操作因此是存在延迟的, 即load后查看其status时file_number可能还是-1. 但`General status bits`有变化: `DR_OPEN`->`ONLINE`->`BOT ONLINE`.
# mt -f /dev/st0 status # 有磁带: file_number=0;没有磁带: file_number=-1. file_number表示磁带上的文件序号,从0开始. mt传送命令到磁带驱动器. [Density code](https://github.com/markh794/mhvtl/blob/master/usr/vtltape.h#L171)
# tar -cvf /dev/nst0 /etc/hosts # 测试向磁带中归档数据. st0为/dev/sg13磁带库中的第一个磁带机, 可通过`lsscsi -g`获取
# tar -tvf /dev/nst0
# mt -f /dev/nst0 tel # tel用于获取磁头位置. 如果使用的是自动回带设备st0，那么第2次备份会replace
At block 6.
# mt -f /dev/nst0 fsf 1 # 让磁头前进一个文件.
# tar -xvf /dev/st0 -C /tmp # 恢复文件
# mt -f /dev/nst0 rewind # 倒带
# mt -f /dev/nst0 erase # 清除磁带内容
# mt -f /dev/nst0 offline # ???没效果
# cat /opt/mhvtl/E01001L4/data # 假设是E01001L4在st0中
# mtx -f /dev/sg13 unload 2 0 # 将磁带从DataTransfer Element 0的磁带机中的移出到StorageElement 2
# --- Linux 备份 /home 分区 ###
# dump 0uf /dev/nst0 /dev/sda5
# dump 0uf /dev/nst0 /home
# --- 恢复分区
# restore rf /dev/nst0
```

推测:
1. 磁带上写入的文件间存在空隙, 导致`tar -tvf`读取一次正常一次错误交替出现, 可用`mt -f /dev/nst0 fsf 1`跳过空隙.
1. 每次`tar -tvf`读取操作会导致磁头移动到下一个文件块/空隙.

General status bits(磁带设备的状态):
- DR_OPEN : 磁带设备为空
- IM_REP_EN : 即时报告模式
- BOT ONLINE : [Beginning Of Tape](https://github.com/markh794/mhvtl/blob/master/usr/vtlcart_v1.c#L136)
- EOF ONLINE : 在file_number-1文件的末尾

磁头操作:
- fsf : 将磁带前进指定的文件标记数目, 即跳过指定个 EOF 标记, 磁带会定位在下一文件的第一个块
- bsf : 将磁带后退指定的文件标记数目，即倒带指定个 EOF 标记, 磁带会定位在 EOF 标记之后即前一个文件的最后一块. 操作后需要再fsf才可读取文件
- fsfm : 前进指定的文件标记数目。磁带定位在前一文件的最后一块
- bsfm : 后退指定的文件标记数目。磁带定位在下一个文件的第一块
- asf : 磁带定位在指定文件标记数目的开始位置。定位通过先倒带，再前进指定的文件标记数目来实现。

	asf 3 = `rewind`+`fsf 2`
- fsr : 前进指定的记录数
- bsr : 后退指定的记录数
- fss :（SCSI tapes）前进指定的 setmarks
- bss :（SCSI tapes）后退指定的 setmarks

### iscsi tape
ref:
- [Using AWS Virtual Tape Library as Storage for Bacula Bareos by Alberto Gimen](https://osbconf.org/wp-content/uploads/2015/10/Using-AWS-Virtual-Tape-Library-as-Storage-for-Bacula-Bareos-by-Alberto-Gimen%C3%A9z.pdf)

```bash
# iscsiadm --mode discovery --type sendtargets -p <IP of tape machine>
# iscsiadm --mode node [--targetname "iqn.2011-04.com.nia:mhvtl:mhvtl:stgt:1"] -p <IP of tape machine> --login
# udevadm info /dev/sg11 # sg11是sch0. 如果是iscsi导出的vtl, 那么ID_PATH包含target iqn; 会根据ID_SERIAL和ID_SCSI_SERIAL在`/dev/tape/by-id`下创建软连接, 但ID_SCSI_SERIAL可能没有比如mhvtl
# udevadm info /dev/st0 # 会根据ID_SERIAL和ID_SCSI_SERIAL在`/dev/tape/by-id`下创建软连接
# sg_inq -p 0x80 /dev/sg11 # 可获取`Unit serial number`, 但与`udevadm info`获取的ID_SERIAL有差异. 比如mhvtl: Unit serial number=XYZZY_A; ID_SERIAL=SSTK_L700_XYZZY_A
# iscsiadm -m session -P 3 -o show # 可先找到指定target iqn, 再通过其中的`Attached SCSI devices`解析出其SCSI设备地址映射, 第一个就是磁盘柜的, 其他是tape的.
```

## sys
```bash
# ls /sys/class/scsi_changer # 磁带柜设备
# ls /sys/class/scsi_tape # 磁带驱动器设备
```

## FAQ
### kernel mod make报`static declaration of ‘sysfs_emit’ follows non-static declaration`
ref:
- [Unable to compile kernel module Ubuntu/Rocky](https://github.com/markh794/mhvtl/issues/95)

解决方法: [kernel: Re-work compat symbols detection](https://github.com/markh794/mhvtl/pull/96)

### kernel mod make报`unrecognized command-line option ‘-ftrivial-auto-var-init=zero’`
`-ftrivial-auto-var-init=zero`需要gcc 12才支持, 而编译模块需要和内核编译选项保持一致, 要么把系统上的gcc升级到12，要么换内核或者用当前系统上的gcc重新编译.

我这里是kernel用了gcc 12(by `cat /proc/version`), 而系统自带的是gcc 11.

解决方法: 安装gcc-12

### st0和nst0区别
两者都表示磁带驱动器, 并指向同一设备, 主要区别在于执行任务后: st打头的表示**写入完成后会自动倒带, 下次写入会覆盖之前的数据即每次都是重头开始写**; nst打头的表示写入完成后不会自动倒带

通常使用 nst0 进行每日备份（不更换磁带）, nst0 进行每周/每月备份（每次备份使用单个磁带）.

### 厂商
只有 HP, Quantum 和 IBM 能生产 LTO 磁带机, 其它都是贴牌
只有 Fujitsu 和 SONY 能生产 LTO 磁带, 其它都是贴牌