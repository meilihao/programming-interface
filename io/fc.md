# FC(Fibre Channel)

## 创建initiator
只需要client的WWN, hostname,ip仅用于信息备注, 与fc管理无关.

## fc信息
```
# lspic | grep "Fibre" # 查看fc卡
# cat /sys/class/fc_host/*/port_name # 查看fc卡的wwn
```