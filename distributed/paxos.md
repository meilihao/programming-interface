# paxos
参考:
- [如何评价阿里最近推出的paxos基础库？](https://www.zhihu.com/question/63479409)
- [阿里如何实现高性能分布式强一致的独立 Paxos 基础库？](https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2650997287&idx=1&sn=4b3ef76bb90c2e28e259802866dc934e&scene=21)
- [X-Paxos — 阿里巴巴的高性能分布式强一致Paxos独立基础库](https://mp.weixin.qq.com/s/EN09RG8c695iDm0edoM5mg)
- [号称史上最晦涩的算法Paxos，如何变得平易近人？](https://developer.aliyun.com/article/156281)
- [**MySQL · 引擎特性 · RDS三节点企业版 一致性协议**](http://mysql.taobao.org/monthly/2019/11/06/)
- [AliSQL X-Cluster 基于X-Paxos的高性能强一致MySQL数据库](https://mp.weixin.qq.com/s?__biz=MzIxNTQ0MDQxNg==&mid=2247483994&idx=1&sn=633c0782f149d5ecc31ddc542634ff07)

X-Paxos当前的算法基于[unique proposer的multi-paxos](https://mp.weixin.qq.com/s/EN09RG8c695iDm0edoM5mg), 大量理论和实践已经证明了基于 unique proposer 的 multi-paxos，性能好于 multi-paxos/basic paxos，当前成熟的基于 Paxos 的系统，大部分都采用了这种方式.

## node role
整个Paxos算法中包含三种角色：Proposer、Accepter和Learner.

在X-Paxos中，节点的角色分为四类：

<table>
  <thead>
    <tr>
      <th>角色</th>
      <th>同步日志</th>
      <th>投票权</th>
      <th>状态机回放</th>
      <th>读写状态</th>
      <th>Paxos角色映射</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Leader</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>rw</td>
      <td>Proposer / Accepter / Learner</td>
    </tr>
    <tr>
      <td>Follower</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>ro</td>
      <td>Proposer / Accepter / Learner</td>
    </tr>
    <tr>
      <td>Logger</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>-</td>
      <td>Accepter / Learner</td>
    </tr>
    <tr>
      <td>Learner</td>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>ro</td>
      <td>Learner</td>
    </tr>
  </tbody>
</table>

整个一致性协议的持久存储分两块：日志和状态机. 日志代表了对状态机的更新操作，状态机存放了外部业务读写的实际数据.

Leader是集群中唯一可读写的节点. 它给集群所有节点发送新写入的日志，达成多数派后允许提交，并回放到本地的状态机. 众所周知，标准的Paxos存在活锁的问题（livelock），即两个Proposer交替发起Prepare请求，导致每一轮Prepare的Accept请求都失败，提案编号不断递增，陷入死循环永远达不成一致. 因此业界的最佳实践是选取一个主Proposer，来保证算法的活性. 另一方面，针对数据库场景，只允许主Proposer发起提案，简化了事务的冲突处理，保证了高性能. 这个主Proposer被称之为Leader.

Follower是灾备节点，用于收集Leader发送的日志，并负责把达成多数派的日志回放到状态机. 当Leader发生故障时，集群中的剩余节点会选一个新的Follower升级成Leader接受读写请求.

Logger是一种特殊类型的Follower，不对外提供服务. Logger做两件事：存储最新的日志用于Leader的多数派判定；选主阶段行使投票权. Logger不回放状态机，会定期清理老旧的日志，占用极少的计算和存储资源. 因此，基于Leader/Follower/Logger的部署方式，三节点相比双节点高可用版，只额外增加很少的成本.

Learner没有投票权，不参加多数派的计算，仅从Leader同步已提交的日志，并回放到状态机. 在实际使用中，我们把Learner作为只读副本，用于应用层的读写分离.此外，X-Paxos支持Learner和Follower之间的节点变更，基于这个功能可以实现故障节点的迁移和替换.