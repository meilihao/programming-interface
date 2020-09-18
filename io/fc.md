# FC(Fibre Channel)

/sys/class目录下有三个主要文件夹跟 fibre channel相关的文件夹fc_transport、fc_remote_ports，fc_hosts. 具体信息见[filesystem/fs.md](filesystem/fs.md).

## 创建initiator
只需要client的WWN, hostname,ip仅用于信息备注, 与fc管理无关.

## fc信息
```
# lspic | grep "Fibre" # 查看fc卡
# cat /sys/class/fc_host/*/port_name # 查看fc卡的wwn
# systool -v -c fc_host # 获取详细的光纤卡信息, from `apt install sysfsutils`
```