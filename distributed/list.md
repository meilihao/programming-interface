# list
## base
- [分布式系统设计中的通用方法](https://zhuanlan.zhihu.com/p/498068994)
- [深入浅出paxos](https://rebootcat.com/2020/12/05/paxos/)

## 进阶
- [深度解析：分布式存储系统实现快照隔离的常见时钟方案](https://www.tuicool.com/articles/eEJB7rI)

## paxos
- [MySQL · 引擎特性 · RDS三节点企业版 一致性协议](http://mysql.taobao.org/monthly/2019/11/06/)
- [阿里如何实现高性能分布式强一致的独立 Paxos 基础库？](https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2650997287&idx=1&sn=4b3ef76bb90c2e28e259802866dc934e)
- [PolarDB-X 一致性共识协议 (X-Paxos)](https://developer.aliyun.com/article/781308)

## 实践
- [老司机带你用 Go 语言实现 Raft 分布式一致性协议](https://happyer.github.io/2017/02/06/2017-02-06-raft/)
- [Go实现Raft : 共4篇](https://mp.weixin.qq.com/s?__biz=Mzg5NDYxNTYyMw==&mid=2247487619&idx=1&sn=af6ad71ff4fb3663b437e30f8deb07e4&source=41#wechat_redirect)

	代码在[eliben/raft](https://github.com/eliben/raft)

	成熟的Go实现的Raft项目代码，可以参考：
	- https://github.com/etcd-io/etcd/tree/master/raft 是etcd的Raft部分，它是一个分布式键值数据库
	- https://github.com/hashicorp/raft 是一个独立的Raft共识模块，可以绑定到不同的客户端
- [使用 Go 语言实现 Paxos 共识算法](https://github.com/tangwz/DistSysDeepDive)
- [手撸golang 学ectd 手写raft协议1~13](https://www.jianshu.com/u/4e1316a61bd2)

	代码在[learning.gooop/gooop](https://gitee.com/ioly/learning.gooop/tree/master/gooop/etcd/raft)
-[**如何实现一个 Paxos**](https://www.tuicool.com/articles/QRbiQzv)