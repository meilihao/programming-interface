# iscsi
initiator:
```sh
# yum install iscsi-initiator-utils -y
# apt install open-iscsi -y
```

> [open-iscsi's git repo](https://github.com/open-iscsi/open-iscsi)

target:
```
# apt install targetcli # [ubuntu 16.04](https://packages.ubuntu.com/search?suite=xenial&section=all&arch=any&keywords=targetcli&searchon=contents)
# apt install targetcli-fb # ubuntu 18.04
```

> targetcli的官方git repo已不再维护, 推荐使用**[targetcli-fb](https://github.com/open-iscsi/targetcli-fb)**

挂载iscsi存储:
```sh
# iscsiadm -m discovery -t sendtargets -p 192.168.10.10   # -m：动作，discovery：发现; -t：类型，sendtargets(简写st)：发送终端类型; -p：指定target地址
# iscsiadm -m node --targetname iqn.2013-02.node2 -p 192.168.35.17:3260 --login # 如果target使用了多张网卡时会存在多路径问题, 挂载磁盘数=target提供的磁盘数*路径数
# iscsiadm -m session -P 3 # 获取挂载信息
# umount /dev/sdb1 /mnt/iscsi/    # 如果磁盘正在挂载使用，建议先卸载再登出
# iscsiadm -m node -T iqn.2013-02.node2 -u # 登出
```

# multipath
```sh
# yum install device-mapper-multipath -y
# apt install multipath-tools -y
# 配成开机启动
# 真正配置前需停止
# multipath -F # 删除之前的配置, 不建议
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
```