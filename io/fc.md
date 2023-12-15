# FC(Fibre Channel)
ref:
- [以太网协议与FC协议的带宽计算](https://www.talkwithtrend.com/Article/263169)

/sys/class目录下有三个主要文件夹跟 fibre channel相关的文件夹fc_transport、fc_remote_ports，fc_hosts. 具体信息见[filesystem/fs.md](filesystem/fs.md).

## 创建initiator
只需要client的WWN, hostname,ip仅用于信息备注, 与fc管理无关.

## fc信息
```
# lspic | grep "Fibre" # 查看fc卡
# cat /sys/class/fc_host/*/port_name # 查看fc卡的wwn
# systool -v -c fc_host # 获取详细的光纤卡信息, from `apt install sysfsutils`
```

## 成本
ref:
- [光纤通道与以太网交换机之间有什么区别呢？](https://www.sohu.com/a/480089290_121001591)

在大多数情况下，以太网交换机比光纤通道交换机便宜得多。但是，光纤通道交换机主要用于数据中心存储环境，而以太网交换机可以在各种类型的网络中找到：从小型家庭，大型办公室到大型数据中心.

对于数据中心网络，8Gbps 的光纤通道交换机通常比10Gbps以太网交换机便宜，而16G光纤交换机的成本与10GbE大致相同。