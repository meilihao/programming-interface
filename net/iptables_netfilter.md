# iptables/netfilter
参考:
- [iptables/netfilter](https://mp.weixin.qq.com/s?__biz=MzkyMTIzMTkzNA==&mid=2247506496&idx=1&sn=c629e22f0de944c0940ffb3a665b726f)

淘汰原因:
1. 路径太长

    netfilter 框架在IP层, 报文需要经过链路层, IP层才能被处理, 如果是需要丢弃报文, 会白白浪费很多CPU资源, 影响整体性能

    ![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nibzSpsZlIHwzjIYJa7ZvTUTzgGyucfcmrV2oXL2ymlIdupS3CYy2PGO1giazNRUoPiblCIxQHWnwQA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. O(N)匹配
    
    如上图, 极端情况下, 报文需要依次遍历所有规则, 才能匹配中, 极大影响报文处理性能
1. 规则太多

    netfilter 框架类似一套可以自由添加策略规则专家系统, 并没有对添加规则进行合并优化, 这些都严重依赖操作人员技术水平, 随着规模的增大, 规则数量n成指数级增长, 而报文处理又是0（n）复杂度, 最终性能会直线下降