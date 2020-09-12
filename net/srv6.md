# SRv6
参考:
- [SRv6浅谈](https://cloud.tencent.com/developer/article/1420074)

SRv6是一种网络转发技术，其中SR是Segment Routing的缩写，v6顾名思义是指IPv6.

SRv6是直接在IPv6的IP扩展头中进行新的扩展，这个扩展部分称为SRH（Segment Routing Header），而这部分扩展没有破坏标准的IP头，因此可以认为SRv6是一种native的IPv6技术.

> [SRv6 uSID在任何情况下的协议开销都要低于VxLAN over SR-MPLS.](https://www.cisco.com/c/dam/global/zh_cn/solutions/service-provider/segment-routing/pdf/usid_srv6.pdf)