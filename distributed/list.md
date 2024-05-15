# list
## base
- [分布式系统设计中的通用方法](https://zhuanlan.zhihu.com/p/498068994)
- [深入浅出paxos](https://rebootcat.com/2020/12/05/paxos/)
- [分布式系统设计模式](https://colobu.com/2022/06/26/distributed-system-design-patterns/)

## 进阶
- [深度解析：分布式存储系统实现快照隔离的常见时钟方案](https://www.tuicool.com/articles/eEJB7rI)

## paxos
- [MySQL · 引擎特性 · RDS三节点企业版 一致性协议](http://mysql.taobao.org/monthly/2019/11/06/)
- [阿里如何实现高性能分布式强一致的独立 Paxos 基础库？](https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2650997287&idx=1&sn=4b3ef76bb90c2e28e259802866dc934e)
- [PolarDB-X 一致性共识协议 (X-Paxos)](https://developer.aliyun.com/article/781308)

## etcd
- [etcd技术架构以及其内部的实现机制](https://zhuanlan.zhihu.com/p/566090538)

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

## 实现
- [oceanbase/src/logservice/palf/election](https://github.com/oceanbase/oceanbase/tree/v4.0.0_CE_BP3/src/logservice/palf/election)

	- [OceanBase原生分布式数据库内核实战进阶版.pdf](https://obcommunity-private-oss.oceanbase.com/prod/blog/2023-09/%E4%BB%8E0%E5%88%B01%20OceanBase%E5%8E%9F%E7%94%9F%E5%88%86%E5%B8%83%E5%BC%8F%E6%95%B0%E6%8D%AE%E5%BA%93%E5%86%85%E6%A0%B8%E5%AE%9E%E6%88%98%E8%BF%9B%E9%98%B6%E7%89%88.pdf)
	- [开源数据库OceanBase代码导读](https://www.zhihu.com/column/c_1386628099518402560)
	- [OceanBase 数据库源码解读之模块结构](https://developer.aliyun.com/article/785281)
	- [Oceanbase PaxosStore 源码阅读](https://zhuanlan.zhihu.com/p/395197545)
	- [oceanbase源码阅读（1）程序启动](https://wangcy6.github.io/post/plan/oceanbase_day1/)
	- [万字解析：从 OceanBase 源码剖析 paxos 选举原理](https://zhuanlan.zhihu.com/p/630468476)
	- [OBCE V3.0教材: 07_第七章_OceanBase高可用_V3.0.pdf](https://mdn.alipayobjects.com/huamei_22khvb/afts/file/A*PIj4T6BIPtwAAAAAAAAAAAAADiGDAQ/07_%E7%AC%AC%E4%B8%83%E7%AB%A0_OceanBase%E9%AB%98%E5%8F%AF%E7%94%A8_V3.0.pdf)
- [PolarDB-X]()

	- [PolarDB-X 三副本存储引擎](https://zhuanlan.zhihu.com/p/535496764)
	- [第七节：X-Paxos 三副本与高可用](https://edu.aliyun.com/course/316505/lesson/15168)

## Distributed Consensus Framework
ref:
- [OceanBase的一致性协议为什么选择 Paxos 而不是 Raft?](modb.pro/db/27698)
- [Have we reached consensus on consensus?](https://tanxinyu.work/have-we-reached-consensus-on-consensus/)
	- [**Have we reached consensus on consensus? pptx**](https://vevotse3pn.feishu.cn/file/boxcnBKfW8q9E61Bfi314R0hOfe)

		[本地](/misc/pdf/dc/Have we reached consensus on consensus 公开版.pptx)


- [opengauss : Distributed Consensus Framework](https://gitee.com/opengauss/DCF)

	- [DCF](https://docs.opengauss.org/zh/docs/3.1.1/docs/CharacteristicDescription/DCF.html)
- [X-Paxos代码](https://github.com/polardb/polardbx-engine/blob/ed663bd0017042e7088ba34b46ad4e2fc0c01150/extra/IS/VERSION)

	最新版本VERSION已删除, 信息在CMakeLists.txt了
- [oceanbase paxos](https://github.com/oceanbase/oceanbase/tree/v4.3.0_CE_BETA/src/logservice/palf)

	- [多副本日志同步](https://www.oceanbase.com/docs/community-observer-cn-10000000000901312)
	- [万字解析：从 OceanBase 源码剖析 paxos 选举原理](https://zhuanlan.zhihu.com/p/630468476)
- [浅析华为Cantian引擎](https://www.modb.pro/db/1701776671271636992)

	- [【创新项目探索】浅析openEuler Cantian引擎](https://www.openeuler.org/zh/blog/20230915-Cantian/20230915-Cantian.html)
	- [openEuler/cantian](https://gitee.com/openeuler/cantian)