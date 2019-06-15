# nftables 基于[nftables HOWTO](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)

nftables是新的数据包分类框架，旨在替代现存的{ip,ip6,arp,eb}_tables, 即它是新一代包过滤框架.

其他:
- linux kernel >= 3.13, 推荐 newer kernel >= 4.10. debain 10 默认使用nftables
- 它有一个新的命令行工具ntf，它的语法与iptables不同
- 它也包含了一个兼容层[iptables-nft](https://wiki.debian.org/nftables)，让你在新的nftables内核框架之上运行iptables命令, **不推荐**

使用理由:
- 更新速度更快 : 在iptables中添加一条规则，会随着规则数量增多而变得非常慢(线性); nftables使用原子的快速操作来更新规则集合
- 内核更新更少 : 使用iptables，每一个匹配或投递都需要内核模块的支持．因此如果需要更新或者添加新的功能时都需要重新编译内核. 在nftables中，大部分工作是在用户态完成的, 因此不存在这种状况.

其他的理由:
- 更易用易理解的语法
- 表和链是完全可配置的
- 匹配和目标之间不再有区别
- 在一个规则中可以定义多个动作
- 每个链和规则都没有内建的计数器
- 更好的动态规则集更新支持
- 简化IPv4/IPv6双栈管理
- 支持set/map等
- 支持级连（需要内核4.1+）

![nftables](images/nftables-the-evolution-of-linux-firewall-8-1024.jpg)
![nftables with ingress filter](images/nftables-the-evolution-of-linux-firewall-9-638.jpg)
![netfilter数据流图(更直观)](images/Netfilter_packet_flow2.png)
![netfilter数据流图(更细节)](images/Netfilter-packet-flow.svg)

参考:
- [UCloud 基于 Linux 内核新特性的下一代外网网关设计及相关开源工作](https://www.infoq.cn/article/Ep1KRyW*zFLGbSpK7Cq7)
- [Load Balancing with nftables](https://www.netdevconf.org/1.1/proceedings/papers/Load-balancing-with-nftables.pdf)
- [Nftables负载平衡比LVS快10倍](https://www.zevenet.com/blog/nftables-load-balancing-10x-faster-lvs/)
- [Quick reference-nftables in 10 minutes](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes)

组件
- libmnl : 提供了使用Netlink进行内核和用户空间通信的接口
- libnftnl : 提供了把netlink消息转换到对象的底层API

验证是否已安装`lsmod | grep nf_tables`

> 检查当前的内核支持哪些模块`grep CONFIG_NFT_ /boot/config-4.15.0-30deepin-generic`

## 基本操作
格式:
    nft [ options ] [ cmds... ] // 需要root权限

选项:
- `-n` : 关闭主机名解析(域名-> ip)
- `-nn` : 关闭设备名解析(应用名->端口)
- `-a` : 显示rule中的handle(句柄)编号
- `--debug` : 使用debug功能

### 配置table
在nftables中，表是链的容器. 所以开始使用nftables时你首先需要做的是添加至少一个表. 然后，你可以向你的表里添加链，然后往链里添加规则.

> 与 iptables 不同，nftables 中并没有默认表

[Nftables families](https://wiki.nftables.org/wiki-nftables/index.php/Nftables_families):
- ip  // **默认**, for ipv4
- arp // for arp
- ip6 // for ipv6
- bridge // 处理通过桥接设备的ethernet包
- inet // 是IPv4和IPv6混合使用的表(在inet表中注册的链在IPv4和IPv6中都能看到), 旨在改善双栈支持, 在Linux内核3.14之后可用
- netdev //  处理从ingress过来的包. 它有一个进入时的钩子，可以使用它注册一个链以便在路由之前更早的阶段进行过滤, 常用于负载平衡或丢弃由DDOS攻击引起的数据包，在Linux内核4.2之后可用,是tc命令的替代

nftables簇与iptables工具的对应
- ip	    iptables
- ip6	    ip6tables
- inet	    iptables和ip6tables
- arp	    arptables
- bridge	ebtables

语法:
```sh
# nft list tables [family]
# nft list table [<family>] <name>
# nft (add | delete | flush(清空)) table [family] {table}
```

> `nft delete`只能删除不包含链的表

### 配置chain
chain是rule的容器, 它包含两种类型: 基础链（base chain）和常规链（regular chain）. base chain是网络栈中数据包的入口; regular chain则可用于jump到目标以便更好地组织规则.

语法:
```sh
# nft (add | create) chain [<family>] <table> <name> [ { type <type> hook <hook> [device <device>] priority <priority> \; [policy <policy> \;] } ] // 编辑chain时只需省略`(add | create)`即可
# nft (delete | list | flush) chain [family] {table} {chain}
# nft rename chain [family] {table} {chain} {newname}

> create ≈ add , 但create已存在的chain时会报错
> 当指定hook和priority时, 该chain就是base chain.

type:
- filter: Supported by arp, bridge, ip, ip6 and inet table families. // 用于数据包过滤层
- route:  Mark packets (like mangle for the output hook, for other hooks use the type filter instead), supported by ip and ip6. // 用于数据包路由
- nat:    In order to perform Network Address Translation, supported by ip and ip6. // 用于网络地址转换（Network Address Translation，NAT）

hooks:
IPv4/IPv6/Inet address families, 用于处理 IPv4和IPv6包，其在network stack中在不同的包处理阶段一共包含了5个hook:
- prerouting : 刚到达且并未被 nftables 的其他部分所路由或处理的数据包. 它在routing流程之前就被发起，用于靠前阶段的包过滤或者更改影响routing的包属性
- input	: 已经被接受而且已经经过 prerouting 钩子的传入数据包
- forward :  被转发到其他主机的包会经由forward hook处理
- output : 由本地进程发送出去的包将被output hook处理
- postrouting : 所有离开系统的包都将被postrouting hook处理

ARP address family, 一般在集群环境中对ARP包进行mangle处理以支持clustering:
- input : 分发到本机的包会经过input hook
- output :	由本机发出的包会经过output hook

Netdev address family:
- ingress :	所有进入系统的包都将被ingress hook处理. 它在进入layer 3之前的阶段就开始处理.

priority用来对chain排序(比如钩子有多个链,它可以决定哪一个在另一个之前看到的数据包, 小的优先)，可以使用的值有：
- NF_IP_PRI_CONNTRACK_DEFRAG (-400), 碎片整理
- NF_IP_PRI_RAW (-300), // ???
- NF_IP_PRI_SELINUX_FIRST (-225) // 数据包在selinux的入口
- NF_IP_PRI_CONNTRACK (-200) // 连接追踪的入口
- NF_IP_PRI_MANGLE (-150) // ???
- NF_IP_PRI_NAT_DST (-100) // 目的NAT
- NF_IP_PRI_FILTER (0) // 过滤操作
- NF_IP_PRI_SECURITY (50) // 设置SECmark的安全表的位置
- NF_IP_PRI_NAT_SRC (100), 源NAT
- NF_IP_PRI_SELINUX_LAST (225) // 数据包在selinux的出口
- NF_IP_PRI_CONNTRACK_HELPER (300) // 连接追踪的出口

policy是对报文采取的动作：
- accept
- drop
- queue
- continue
- return

### 配置rule
规则由语句或表达式构成，包含在链中.

> iptables-translate命令可以将iptables规则转换成nftables格式
> [iptables -> nftables](https://wiki.nftables.org/wiki-nftables/index.php/Moving_from_iptables_to_nftables), tools in iptables-nftables-compat. 注意:
低版本Ubuntu16.04的iptables-nftables-compat里没有iptables-restore-translate, 请使用高版本Ubuntu或使用其他有iptables-restore-translate机器帮忙转换.
> `iptables-compat` = `iptables-nft` = iptables语法 with nftables interface.

语法:
```sh
# nft (add | insert) rule [<family>] <table> <chain> [position <position>] <matches> <statements> // 注意matches,statements都允许多个, 从左到右逐个解析
# nft replace rule [<family>] <table> <chain> [handle <handle>] <matches> <statements>
# nft delete rule [<family>] <table> <chain> [handle <handle>]
```

> insert ≈ add , 但add是追加在position之后, 默认添加到链的末尾; insert是追加在position之前, 默认是插入到链的开头

nftables 提供了下面的内建操作：
- eq:等于, 也可以使用==
- ne：不等于，也可以用 !=
- lt：小于，也可以用 <
- gt：大于，也可以用 >
- le：小于等于，也可以用 <=
- ge：大于等于，也可以用 >=

nftables 表达式中可以指定地址族或所处理的数据包类型. nftables 使用有效载荷表达（palyload expressions）式以及元表达式（meta expressions）. 有效载荷表达式是从数据包信息那里收集的. 一些特定的报头表达式，例如，sport 和 dport（分别是源端口和目的端口）会被应用到 TCP 和 UDP 数据包，但它们对于 IPv4 和 IPv6 层来说没有意义，因为这些层不使用端口. 元表达式可以广泛应用的规则链与常用数据包或接口属性相关的规则中使用

matches是报文需要满足的条件:
matches的内容非常多，可以识别以下几种类型的报文：
- ip          :  ipv4协议字段
- ip6         :  ipv6协议字段
- tcp         :  tcp协议字段
- udp         :  udp协议字段
- udplite     :  udp-lite协议
- sctp        :  SCTP协议 
- dccp        :  链接跟踪
- ah
- esp
- comp
- icmp        : ICMP协议
    - type	icmp数据包类型, `icmpv type {destination-unreachable,...}`
- icmpv6      : ICMP6协议
     - type	icmp6数据包类型, `icmpv6 type {destination-unreachable,...}`
- ether       :  以太头
- dst
- frag        :
- hbh
- mh
- rt
- vlan        :  vlan
- arp         :  arp协议
- ct          :  conntrack, 连接状态追踪
    - state 连接状态
        - new：一个新的数据包到达防火墙，例如，一个设置了 SYN 标志的 TCP 数据包. 
        - established：数据包是已经被处理或跟踪的连接的一部分
        - invalid：一个不符合协议规则的数据包
        - related：一个数据包与某个连接相关，该连接的协议不使用其他手段跟踪其状态，例如 ICMP 或被动 FTP
        - untracked：一个用于绕开连接跟踪的管理员状态，典型用于特殊情况
- meta        :  报本的基本信息
    - iif	接收数据包的接口索引, `meta iif eth0`
    - iifname	接收数据包的接口名称, `meta iifname "eth0"`
    - iiftype	接收数据包的接口类型, `meta iiftype {ether, ppp, ipip, ipip6, loopback, sit, ipgre}`
    - length	数据包的字节长度, `meta length 1000`, `meta length { 33-55, 67-88 }`
    - mark	数据包标记, `meta mark 0x4`
    - oif	传出数据包的接口索引, `meta oif lo`
    - oifname	传出数据包的接口的名称, `meta oifname "eth0"`
    - oiftype	传出数据包的接口的类型, `meta oiftype {ether, ppp, ipip, ipip6, loopback, sit, ipgre}`
    - prority	TC 数据包的优先级(tc class id), `meta priority 0x1:0x1`, `meta priority set 0x1:0x1`
    - protocol	以太网类型的协议, `meta protocol { ip, arp, ip6, vlan }`
    - rtclassid	路由数据包的领域, `meta rtclassid cosmos`
    - skgid	原始套接字的组标识符, `meta skgid {bin, root, daemon}`,`meta skgid != root`,`meta skgid { 2001-2005 }`
    - skuid	原始套接字的用户标识符, `meta skuid {bin, root, daemon}`,`meta skuid != root`,`meta skuid { 2001-2005 }`
    - pkttype Packet type, `meta pkttype { broadcast, unicast, multicast}`
    - cpu cpu id, `meta cpu != 1-2`
    - iifgroup 接收数据包的接口组, `meta iifgroup {default}`, `meta iifgroup { 11,33 }`
    - oifgroup 传出数据包的接口组, `meta oifgroup {default}`, `meta oifgroup { 11,33 }`
    - cgroup cgroup, `meta cgroup {1048577-1048578}`
    - l4proto l4proto, `meta l4proto { 33-55 }`
    - nfproto nfproto, `meta nfproto { ipv4, ipv6 }`

![ct](images/nft_ct_expression.png)
![ipv4+ipv6](images/nft_ipv4_ipv6_expression.png)
![tcp+upd](images/nft_tcp_udp_expression.png)

对每一种类型，又可以检查多个字段，具体查看[Quick reference-nftables in 10 minutes](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes#Matches)或[Linux-nftables](https://www.cnblogs.com/sztom/p/10947111.html)

statement是报文匹配rule时触发的操作：
- Verdict : 裁决
    - accept : 接受数据包并且停止处理
    - drop : 停止处理并丢弃此数据包, 即drop是终止行为，你不能在它之后添加任何其它行为
    - queue : 发送数据包到用户空间并停止处理
    - continue : 继续处理此数据包
    - return : 返回到调用的规则链继续处理
    - jump <chain> : 跳到指定的规则链进行处理, 而且当完成时或执行了return时返回到调用的规则链
    - goto <chain> : 发送到指定的规则链进行处理但不返回到调用的规则链
    - Log : 日志
        - 没有指定group时, kernel将日志打到kernel log, 能用dmesg或syslog读到
        - 指定group时, kernel会将数据包传递给nfnetlink_log, 它会通过netlink socket广播到指定的group以供应用订阅并接收该数据包 
    - Reject : 拒绝, 停止处理
    - Counter : 计数
    - Limit : 限速
        格式:`rate [over] <value> <unit> [burst <value> <unit>]` // burst, 指明可以超过速率限制的数据包或字节的数量
    - Nat
    - Queue : 队列

#### 原子性的替换规则
类似iptables-restore的作用.

```sh
# nft list table filter > filter-table
# nft -f filter-table // `-f`, 原子性的更新规则集
```

### ruleset(规则集)
允许将规则集作为一个整体进行操作

语法:
```sh
# nft list ruleset [family] // 输出给定family的所有表／链／集合／规则
# nft flush ruleset [family] // 清空整个规则集, 包括
```

#### 备份/恢复
备份：
```sh
# echo "nft flush ruleset" > backup.nft
# nft list ruleset >> backup.nft
```

恢复:
```sh
# nft -f backup.nft
```

规则集导出成XML 或JSON 格式:
```bash
# nft export (xml | json) // all rules
```
> import导入XML和JSON还未实现

### nft monitor
显示nft的变更

过滤的对象或事件：
- 对象：'tables', 'chains', 'sets', 'rules', 'elements', 'ruleset'
- 事件：'new', 'destroy'

输出格式：
- 纯文本（比如：原生的nft格式）
- xml
- json

语法:
```sh
# nft monitor [new | destroy] [tables | chains | sets | rules | elements | ruleset] [xml | json]
```

demo:
```sh
# nft monitor                      // Listen to all events, report in native nft format
# nft monitor new tables xml       // Listen to added tables, report in XML format
# nft monitor destroy rules json   // Listen to deleted rules, report in JSON format
# nft monitor chains               // Listen to both new and destroyed chains, in native nft format
# nft monitor ruleset              // Listen to ruleset events such as table, chain, rule, set, counters and quotas, in native nft format
```

### 脚本
在文件开头添加[`#!/usr/sbin/nft -f`](https://wiki.nftables.org/wiki-nftables/index.php/Scripting)即可, 该操作是原子的.

## 进行数据包分类的高级数据结构结构
### [sets](https://wiki.nftables.org/wiki-nftables/index.php/Sets)
使用任何支持的选择器创建集合, 集合可以使用字典（dictionaries）和映射组（maps）表示.

集合的元素在内部通常使用性能好的数据结构比如哈希表和红黑树表示.

匿名集合是：
1. 绑定到一条规则，规则被删除时，集合也会被删除
1. 它们没有特定的名字，内核会为它们分配一个标识符
1. 它们不能被更新，所以一旦绑定到规则上后就不能添加和删除元素

创建一个简单的集合：
```
# nft add rule filter output tcp sport { 22, 23 } counter // 捕获所有发送到TCP 22 和23 端口的数据包，同时更新计数器
```

命名集合
创建命名集合：
```
# nft add set filter blackhole { type ipv4_addr\;} // filter是表名, blackhole是集合的名字，type选项指示集合中数据的类型
# nft add element filter blackhole { 192.168.1.4, 192.168.1.5 }
# nft add rule test tchain ip saddr @blackhole drop
# nft list set test blackhole
```

当前type支持的数据类型有：
- ipv4_addr：IPv4 地址
- ipv6_addr：IPv6 地址
- ether_addr：以太网（Ethernet）地址
- inet_proto：网络协议
- inet_service：网络服务
- mark：标记类型

### [Dictionaries](https://wiki.nftables.org/wiki-nftables/index.php/Dictionaries)
使用Dictionarie来构建规则集, 此规则集允许减少线性列表检查的数量，以对数据包进行分类

匿名字典
demo:
```sh
# nft add rule filter icmp-chain counter
# nft add rule filter tcp-chain counter
# nft add rule filter udp-chain counter
# nft add rule ip filter input ip protocol vmap { tcp : jump tcp-chain, udp : jump udp-chain , icmp : jump icmp-chain } // 通过附加简单的规则来计算流量
```

命名字典
demo:
```sh
# nft add map filter mydict { type ipv4_addr : verdict\; }
# nft add element filter mydict { 192.168.0.10 : drop, 192.168.0.11 : accept }
# nft add rule filter input ip saddr vmap @mydict
```

### [maps](https://wiki.nftables.org/wiki-nftables/index.php/Maps)
使用map根据输入的某个特定key查找数据

匿名map
demo:
```sh
# nft add rule ip nat prerouting dnat tcp dport map { 80 : 192.168.1.100, 8888 : 192.168.1.101 }
```


命名map
demo:
```sh
# nft add map nat porttoip  { type inet_service: ipv4_addr\; }
# nft add element nat porttoip { 80 : 192.168.1.100, 8888 : 192.168.1.101 }
# nft add rule ip nat postrouting snat tcp dport map @porttoip
```

### [区间](https://wiki.nftables.org/wiki-nftables/index.php/Intervals)
区间（Intervals）由 value-value表示

demo:
```sh
# nft add rule filter input ip daddr 192.168.0.1-192.168.0.250 drop // 如何丢弃目标地址区间在 192.168.0.1 和 192.168.0.250 之间的数据包
# nft add rule filter input tcp ports 1-1024 drop // 也可以在 TCP 端口中使用
# nft add rule ip filter input ip saddr { 192.168.1.1-192.168.1.200, 192.168.2.1-192.168.2.200 } drop // 也可以在集合（sets）中使用集合，比如拉黑两个 IP 区间的数据包
# nft add rule ip filter forward ip daddr vmap { 192.168.1.1-192.168.1.200 : jump chain-dmz, 192.168.2.1-192.168.20.250 : jump chain-desktop } // 也可以在字典（dictionaries）中使用
```