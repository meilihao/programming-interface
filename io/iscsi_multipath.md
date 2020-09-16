# iscsi
参考:
- [JohnHufferd-IP Storage Protocols-V3.pdf](/misc/pdf/JohnHufferd-IP Storage Protocols-V3.pdf)

iscsi 架构基于客户/服务器模型，其主要功能是在TCP/IP网络上的主机系统（启动器initlator）和存储设备（目标 target） 之间进行大量的数据封装和可靠传输过程，此外，iscsi 提供了在IP网络封装SCSI命令，切运行在TCP上.

ISCSI架构上分为服务端（target）与客户端（initiator）：
- ISCSI target ：就是存储设备端，存放磁盘,RAID的设备或模拟的scsi设备等，目的在提供其他主机使用的磁盘
- ISCSI initiator： 就是能够使用target的客户端，通常是服务器，只有装有iscsi initiator的相关功能后才能使用ISCSI target 提供的磁盘

iscsi在传输数据的时候考虑了安全性，可以通过IPSEC 对流量加密，并且iscsi提供CHAP认证机制以及ACL访问控制，但是在访问iscsi-target的时候需要IQN（iscsi完全名称，区分唯一的initiator和target设备），格式iqn.年月.域名后缀(反着写)：[target服务器的名称或IP地址等形式].

![在以太网上比较NVMe-oF Target和iSCSI Target](/misc/img/io/Image00099.jpg)

Linux-IO Target(LIO)是linux原生实现了iscsi target, 是趋势.

iscsi数据包PDU格式见[这里](https://www.cnblogs.com/pipci/p/11622014.html).

# multipath
```sh
# yum install device-mapper-multipath -y
# apt install multipath-tools -y
# 配成开机启动
# 真正配置前需停止
# multipath -F # 删除之前的配置, 不建议
# lsblk
# /lib/udev/scsi_id --whitelisted --device=/dev/sda        #获取设备的wwid
# for i in `cat /proc/partitions | awk {'print $4'} |grep sd`; do
val=`/sbin/blockdev --getsize64 /dev/$i`; val2=`expr $val / 1073741824`; echo "/dev/$i:$val2 `/lib/udev/scsi_id -gud /dev/$i`"; done # 获取所有lun的wwid, VMware   Virtual disk没有wwid
# mpathconf --enable --find_multipaths y --with_module y --with_chkconfig y # 生成multipath.conf文件. 没删除过配置时不用重新生成
# vim /etc/multipath.conf
defaults {
	find_multipaths yes
	path_grouping_policy multibus
	user_friendly_names yes
}

blacklist {
	wwid xxx # 本地磁盘, 不需要多路径
}

multipaths {
	multipath { # 将不重复的wwid作为multipath内容追加到multipaths
		wwid xxx1
		alias yyy1
	}
	multipath {
		wwid xxx2
		alias yyy2
	}
	...
}
# systemctl restart multipath-tools.service # 启动multipath, 启动后会在/dev/mapper下生成多路径逻辑盘(即/dev/dm*), 此时盘权限全部都是root用户, 有文章介绍要通过udev来修改权限, 不知能否通过chown修改.
# multipath -ll # 查看多路径状态
# ls -1 /dev/mapper/control  # 查看设备
```