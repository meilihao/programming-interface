# lvs
ref:
- [lvs](https://tonydeng.github.io/sdn-handbook/linux/loadbalance.html)

	自研负载均衡:
	- Google Maglev
	- UCloud Vortex
- [LVS原理篇：LVS简介、结构、四种模式、十种算法](https://blog.csdn.net/lcl_xiaowugui/article/details/81701949)

Linux Virtual Server (lvs) 是Linux内核自带的负载均衡器，也是目前性能最好的软件负载均衡器之一。lvs包括ipvs内核模块和ipvsadm用户空间命令行工具两部分。

在lvs中，节点分为Director Server和Real Server两个角色，其中Director Server是负载均衡器所在节点，而Real Server则是后端服务节点。当用户的请求到达Director Server时，内核netfilter机制的PREROUTING链会将发往本地IP的包转发给INPUT链（也就是ipvs的工作链），在INPUT链上，ipvs根据用户定义的规则对数据包进行处理（如修改目的IP和端口等），并把新的包发送到POSTROUTING链，进而再转发给Real Server

> lvs在kernel中的专业命名是ipvs.

LVS的模型中有两个角色：
- 调度器:Director Server，又称为Dispatcher，Balancer

    调度器主要用于接受用户请求
- 真实主机:Real Server，简称为RS

    用于真正处理用户的请求

将所在角色的IP地址分为以下4种：
- Director Virtual IP:调度器用于与客户端通信的IP地址，简称为VIP
- Director IP:调度器用于与RealServer通信的IP地址，简称为DIP
- Real Server : 后端主机的用于与调度器通信的IP地址，简称为RIP
- CIP：Client IP，访问客户端的IP地址所

lvs的ip负载均衡是通过ipvs实现的, 具体机制分3种:
- NAT

	NAT模式通过修改数据包的目的IP和目的端口来将包转发给Real Server.

    用户请求LVS到达director，director将请求的报文的目的IP改为RIP和将报文的目标端口也改为realserver的相应端口，最后将报文发送到realserver上，realserver将数据返回给director，director再将响应报文的源地址和源端口换成vip和相应端口, 最后把数据发送给用户

    优点:
    - 支持端口映射
    - 不需要Real Server做任何特殊配置
    - 节省公有IP地址
    
        RIP和DIP必须处于同一个网段内, 且RS的网关是DIP. 使用nat另外一个好处就是后端的主机相对比较安全.

    缺点：
    - 效率低: 请求和响应报文都要经过Director转发;极高负载时，Director可能成为系统瓶颈

    ![](/misc/img/develop/2020-07-21_11-31-09.png)
- FULLNAT

	FULLNAT是阿里在NAT基础上增加的一个新转发模式，通过引入local IP（CIP-VIP转换为LIP->RIP，而LIP和RIP均为IDC内网IP）使得物理网络可以跨越不同vlan，代码维护在alibaba/LVS上面。其特点是

	- 物理网络仅要求三层可达, 即不需要DIP和RIP在同一网段
	- Real Server不需要任何特殊配置
	- SYNPROXY防止synflooding攻击
	- 未进入内核主线，维护复杂

    FULLNAT和NAT相比的话：会保证RS的回包一定可到达LVS
    FULLNAT需要更新源IP，所以性能正常比NAT模式下降10%
- TUN

    基于ip隧道技术, 通过将数据包封装在另一个IP包中（源地址为DIP，目的为RIP）将包转发给Real Server

    用户请求LVS到达director，director通过IP-TUN技术将请求报文的包封装到一个新的IP包里面，目的IP为VIP(不变)，然后director将报文发送到realserver，realserver基于IP-TUN解析出来包的目的为VIP，检测网卡是否绑定了VIP，绑定了就处理这个包，如果在同一个网段，将请求直接返回给用户，否则通过网关返回给用户；如果没有绑定VIP就直接丢掉这个包

    Real Server需要在lo上配置vip，但不需要Director Server作为网关.

    优点：

    - RIP,VIP,DIP都应该使用公网地址，且RS网关不指向DIP
    - 只接受进站请求，请求报文经由Director调度，但是响应报文不需经由Director, 解决了LVS-NAT时的问题，减少负载

    缺点：
    - 不指向Director所以不支持端口映射
    - RS的OS必须支持隧道功能
    - 隧道技术会额外花费性能，增大开销, 同时运维麻烦, 一般不用.

- DR

	通过修改数据包的目的MAC地址将包转发给Real Server.

    当Director接收到请求之后，通过调度方法选举出RealServer, 将目标地址的MAC地址改为RealServer的MAC地址. RealServer接受到转发而来的请求，发现目标地址是VIP, RealServer配置在lo接口上, 处理请求之后则使用lo接口上的VIP响应CIP.

    需要在Real Server的lo上配置vip，并配置arp_ignore和arp_announce忽略对vip的ARP解析请求.

    优点:
    - RIP可以使用私有地址，也可以使用公网地址, 只要求DIP和RIP的地址必须在同一个物理网络内，二层可达
    - 请求报文经由Director调度，但是响应报文不经由Director. 性能最高最好, 因此**最常用**.
    - RS可以使用大多数OS

    缺点：
    - 不支持端口映射
    - 不能跨局域网

四种模式的比较:
- 是否需要VIP和realserver在同一网段
    
    DR模式因为只修改包的MAC地址，需要通过ARP广播找到realserver，所以VIP和realserver必须在同一个网段，也就是说DR模式需要先确认这个IP是否只能挂在这个LVS下面；其他模式因为都会修改目的地址为realserver的IP地址，所以不需要在同一个网段内
- 是否需要在realserver上绑定VIP
    
    realserver在收到包之后会判断目的地址是否是自己的IP
    DR模式的目的地址没有修改，还是VIP，所以需要在realserver上绑定VIP
    IP TUN模式值是对包重新包装了一层，realserver解析后的包的IP仍然是VIP，所以也需要在realserver上绑定VIP
- 四种模式的性能比较
    
    DR模式、IP TUN模式都是在包进入的时候经过LVS，在包返回的时候直接返回给client, 但是NAT模式时Real Server回复的报文也会经过Director Server地址重写, 所以二者的性能比NAT高.
    但TUN模式更加复杂，所以性能不如DR
    FULLNAT模式不仅更换目的IP还更换了源IP，所以性能比NAT下降10%
    性能比较：DR>TUN>NAT>FULLNAT

LVS的八种调度方法
参考:
- [<<liunx就是这个范儿>>#15.6.5 LVS的负载均衡调度算法]

- 静态方法:仅依据算法本身进行轮询调度

    - RR:Round Robin,轮询 : 均等地对待每一台服务器，不管服务器上的实际连接数和系统负载
    - WRR:Weighted RR，加权轮询 : 加权，手动让能者多劳
    - SH:SourceIP Hash : 来源地址散列, 与目标地址散列调度算法类似，但它是根据源地址散列算法进行静态分配固定的服务器资源. 来自同一个IP地址的请求都将调度到同一个RealServer
    - DH:Destination Hash : 目标地址散列, 根据目标 IP 地址通过散列函数将目标 IP 与服务器建立映射关系，出现服务器不可用或负载过高的情况下，发往该目标 IP 的请求会固定发给该服务器. 不管IP，请求特定的东西，都定义到同一个RS上

- 动态方法:根据算法及RS的当前负载状态进行调度

    - LC:least connections(最小链接数) : 链接最少，也就是Overhead最小就调度给谁; 假如都一样，就根据配置的RS自上而下调度
    - WLC:Weighted Least Connection (加权最小连接数) : 这个是LVS的默认算法. 调度器可以自动问询真实服务器的负载情况，并动态调整权值
    - SED:Shortest Expection Delay(最小期望延迟) : WLC算法的改进. 不考虑非活动链接，谁的权重大，优先选择权重大的服务器来接收请求，但权重大的机器会比较忙
    - NQ:Never Queue : 最少队列, SED算法的改进
    - LBLC:Locality-Based Least-Connection,基于局部性的最少连接算法

        请求数据包的目标 IP 地址的一种调度算法，该算法先根据请求的目标 IP 地址寻找最近的该目标 IP 地址所有使用的服务器，如果这台服务器依然可用，并且有能力处理该请求，调度器会尽量选择相同的服务器，否则会继续选择其它可行的服务器
    - lblcr:Locality-Based Least Connections with replication, 带复制的基于局部性的最少连接算法 : 记录的不是要给目标 IP 与一台服务器之间的连接记录，它会维护一个目标 IP 到一组服务器之间的映射关系，防止单点服务器负载过高