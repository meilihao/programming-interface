# iscsi
参考:
- [JohnHufferd-IP Storage Protocols-V3.pdf](/misc/pdf/JohnHufferd-IP Storage Protocols-V3.pdf)
- [rfc 7143 Internet Small Computer System Interface (iSCSI) Protocol](https://tools.ietf.org/html/rfc7143)

iscsi 架构基于客户/服务器模型，其主要功能是在TCP/IP网络上的主机系统（启动器initlator）和存储设备（目标 target） 之间进行大量的数据封装和可靠传输过程，此外，iscsi 提供了在IP网络封装SCSI命令，切运行在TCP上.

ISCSI架构上分为服务端（target）与客户端（initiator）：
- ISCSI target ：就是存储设备端，存放磁盘,RAID的设备或模拟的scsi设备等，目的在提供其他主机使用的磁盘
- ISCSI initiator： 就是能够使用target的客户端，通常是服务器，只有装有iscsi initiator的相关功能后才能使用ISCSI target 提供的磁盘

iscsi在传输数据的时候考虑了安全性，可以通过IPSEC 对流量加密，并且iscsi提供CHAP认证机制以及ACL访问控制，但是在访问iscsi-target的时候需要IQN（iscsi完全名称，区分唯一的initiator和target设备），格式iqn.年月.域名后缀(反着写)：[target服务器的名称或IP地址等形式].

![在以太网上比较NVMe-oF Target和iSCSI Target](/misc/img/io/Image00099.jpg)

Linux-IO Target(LIO)是linux原生实现了iscsi target, 是趋势.

iscsi数据包PDU格式见[这里](https://www.cnblogs.com/pipci/p/11622014.html).

## 协议
iscsi协议支持重定向功能, 见ZBS的[iSCSI重定向程序](https://max.book118.com/html/2023/0405/8142011001005054.shtm).

> nfs协议不支持重定向, zbs通过在nfs client端部署IO Rerouter解决.