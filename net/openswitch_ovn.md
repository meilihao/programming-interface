# openswitch
ref:
- [Openvswtich.pptx](/misc/ppt/net/Openvswtich.pptx)

# ovn
ref:
- [Latest Release](https://www.ovn.org/en/)
- [HCIP-OpenStack组件之网络服务Neutron(ovs、ovn)](https://blog.csdn.net/ma1782072/article/details/132484386)

## 架构
ovn本身主要有几个模块：
- ovn-northd服务运行于控制节点，主要是翻译Northbound DB的数据到Southbound DB

	OVN北向数据库, 用于描述上层的逻辑网络组件, 比如逻辑交换机/逻辑端口; 南向数据库，其将北向数据库的逻辑网络数据格式转换为物理网络数据格式并进行存储
- ovn-controller服务运行于计算节点，主要是根据Southbound DB的数据翻译为流表下发到本地的OVS
- ovn-controller-vtep是可以配置支持OVSDB协议的物理交换机
- ovn-nbctl和ovn-sbctl命令是分别查看Northbound DB和Southbound DB数据
- ovn-trace是用于查看报文的路径的

## FAQ
### ovs和ovn区别
OVN是OpenvSwitch项目组为OpenvSwitch开发SDN控制器. ovn是建立在ovs之上的，ovn必须有底层的ovs，ovs可理解为二层交换机，ovn可理解为三层交换机

### linux bridge和ovs区别
linux bridge适合小规模, 主机内部间通信场景
ovs适合大规模, 多主机间通信场景

### 安全组
未使用OVN时，安全组需要在bridge用iptables实现
使用OVN时，安全组使用OVS的ACL流表实现即可，这对于以后使用OVS-DPDK更容易平滑过渡