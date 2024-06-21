# raft
ref:
- [raft官网](raft.github.io)

	raft动画
- [深度解析 Raft 分布式一致性协议（长文）](https://mp.weixin.qq.com/s/IA4aGJ0o-xsQtU9H0JliYw)
- [Raft 在 etcd 中的实现](https://blog.betacat.io/post/raft-implementation-in-etcd/)
- [分布式理论 6 - 一致性协议Raft.md](https://github.com/loveincode/notes/blob/master/15%20-%20Distributed%20%E5%88%86%E5%B8%83%E5%BC%8F/%E5%88%86%E5%B8%83%E5%BC%8F%E7%90%86%E8%AE%BA/%E5%88%86%E5%B8%83%E5%BC%8F%E7%90%86%E8%AE%BA%206%20-%20%E4%B8%80%E8%87%B4%E6%80%A7%E5%8D%8F%E8%AE%AERaft.md)
- [深入浅出etcd/raft](https://blog.mrcroxx.com/categories/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAetcd/raft/)
- [etcd教程(十五)---leader选取源码分析](https://www.lixueduan.com/posts/etcd/15-raft-leader-election/)
- [simple-raft](https://github.com/nananatsu/simple-raft)
    - [用go实现Raft](https://juejin.cn/post/7239238662692569148)


Paxos 协议有一个很大的设计假设, 它要求支持多个投票, 也就是数据库里的多条日志之间是可以**乱序**提交的, 可以**并行**处理的. 但是 raft 协议的做了一个约束, 数据库的多个投票多条日志一定要按照**顺序执行**, 只能前一个日志被确认了才能确认后一个日志.

> MultiRaft概念上等价于PaxosGroup.

这种简化使得 Paxos 协议更易实现, 但确定是:
1. 并发能力变得差

	以前支持并发的提交, 现在只能支持一个结束以后再来下一个

1. 可用性问题

	如果采用 Paxos 协议, 当一台机器新上线的时候很快就能提供服务了, 因为不需要等前面的数据确认它就能提供服务, 但是如果使用的是 raft 协议, 需要等前面的所有日志确认以后才能提供服务???(不是应是新节点补全数据后才提供访问吗?).

其实将分布式系统中如何对某个值达成一致可分解成3个子问题:
1. 如何选主（Leader Election）
1. 如何把数据复制到各个节点上（Entity Replication）
1. 如何保证过程是安全的（Safety）

raft属于 Multi-Paxos 算法, 但做了一些简化和限制, 它将问题分解为:
- leader election : leader选举
- log replication: 日志复制
- membership changes : 成员关系变化

在 Raft 中，不是所有节点都能当选领导者，只有日志最完整的节点，才能当选领导者；其次，在 Raft 中，日志必须是连续的.

raft算法基本操作只需要2种rpc:
- RequestVote : 由候选人在选举期间发起，通知各节点进行投票
- AppendEntries : leader触发, 用于向其他节点复制log和发送心跳

Raft 算法是现在分布式系统开发首选的共识算法。绝大多数选用 Paxos 算法的系统（比如 Cubby、Spanner）都是在 Raft 算法发布前开发的，当时没得选；而全新的系统大多选择了 Raft 算法（比如 Etcd、Consul、CockroachDB）.

## 角色
常见raft是Leader-Follower模式, 每个节点维持了一个状态机, 有3中状态:
- leader

	负责处理所有的client请求. 当它收到client的写请求时, 会在本地append一条相应的log记录, 然后将其封装成消息发送给其他follower. 当follower收到该消息会对其进行响应. 如果cluster中超过半数的节点都收到了该请求对应的log记录时, leader认为该log记录已提交(committed), 可以向client返回响应了.

	leader还会处理clientt的只读请求.

	leader还会定期向cluster中的follower发送心跳, 这主要是为了防止cluster中其他follower的选举计时器超时而触发新一轮选举.
- follower

	follower不发送任何请求, 只响应来自leader或candidate的请求. follower也不处理client的请求, 而是将请求重定向到leader.
- candidate

	由follower转换而来. 当follower长时间没收到leader发送的心跳消息时, 该节点的选举计时器会过期, 同时将自身转化为candidate, 开始新一轮的选举

任意时刻, cluster中的任意节点必定处于这3个状态之一. 大多数情况下, cluster中一个leader, 其他都是follower.


所有请求都是leader处理, client写操作有两种方法:
1. 重定向

	**推荐**: leader比较稳定, 不会经常切换

1. 转发

	增加大量的不必要的消息和性能消耗, 同时增加了问题分析排查的复杂度

> 一般情况下, 在绝大部分的时间内（比如 Google Chubby 团队观察到的值是数天），领导者是处于稳定状态的，某个节点一直是领导者，那么引入中间节点，就会增加大量的不必要的消息和性能消耗

### 选举
ref:
- [Raft 算法之领导人选举](https://qeesung.github.io/2020/04/14/Raft-%E7%AE%97%E6%B3%95%E4%B9%8B%E9%A2%86%E5%AF%BC%E4%BA%BA%E9%80%89%E4%B8%BE.html)
- [Raft协议之Leader选举](https://bloodhunter.github.io/2019/03/30/raft-xie-yi-zhi-leader-xuan-ju/)

raft中有两个时间用于控制leader选举发生:
1. 选举超时时间(election timeout)

	每个follower在接收不到leader节点的心跳消息后, 并不会立即发起新一轮选举, 而是会等待一段时间之后才切换成candidate状态发起新一轮选举.

	election timeout是150~300ms的随机数, 而非固定, 是为了避免leader发送的心跳消息因为瞬间的网络延迟或应用程序瞬间的卡顿而迟到甚至丢失.
1. 心跳超时时间(heartbeat timeout)

	leader节点向其他follower发送心跳消息的时间间隔

> election timeout和heartbeat timeout为什么不合并: 为了避免同时出现多个Candidate而导致选举无效的情况出现

任期意义: 任期是严格单调递增的, 以应付主节点陷入网络分区后重新恢复，但另外一部分节点仍然有多数派，且已经完成了重新选主的情况, 此时必须以任期编号大的主节点为准.

raft 算法中的任期不只是时间段，而且任期编号的大小，会影响领导者选举和请求的处理:
- 在 Raft 算法中约定，如果一个候选人或者领导者，发现自己的任期编号比其他节点小，那么它会立即恢复成跟随者状态。比如分区错误恢复后，任期编号为 3 的领导者节点 B，收到来自新领导者的，包含任期编号为 4 的心跳消息，那么节点 B 将立即恢复成跟随者状态
- 如果一个节点接收到一个包含较小的任期编号值的请求，那么它会直接拒绝这个请求。比如节点 C 的任期编号为 4，收到包含任期编号为 3 的请求投票 RPC 消息，那么它将拒绝这个消息

在 Raft 算法中，约定了选举规则，主要有这样几点:
1. 领导者周期性地向所有跟随者发送心跳消息（即不包含日志项的日志复制 RPC 消息）, 通知其他节点自身是领导者，阻止跟随者发起新的选举
1. 如果在指定时间内，跟随者没有接收到来自领导者的消息，那么它就认为当前没有领导者，推举自己为候选人，发起领导者选举
1. 在一次选举中，赢得大多数选票的候选人，将晋升为领导者
1. 在一个任期内，领导者一直都会是领导者，直到它自身出现问题（比如宕机），或者因为网络延迟，其他节点发起一轮新的选举
1. 在一次选举中，每一个服务器节点最多会对一个任期编号投出一张选票，并且按照“先来先服务”的原则进行投票。比如节点 C 的任期编号为 3，先收到了 1 个包含任期编号为 4 的投票请求（来自节点 A），然后又收到了 1 个包含任期编号为 4 的投票请求（来自节点 B）。那么节点 C 将会把唯一一张选票投给节点 A，当再收到节点 B 的投票请求 RPC 消息时，对于编号为 4 的任期，已没有选票可投了
1. 当任期编号相同时，日志完整性高的跟随者（也就是最后一条日志项对应的任期编号值更大，索引号更大），拒绝投票给日志完整性低的候选人。比如节点 B、C 的任期编号都是 3，节点 B 的最后一条日志项对应的任期编号为 3，而节点 C 为 2，那么当节点 C 请求节点 B 投票给自己时，节点 B 将拒绝投票

	在 Raft 中，日志不仅是数据的载体，日志的完整性还影响领导者选举的结果, 即日志完整性最高的节点才能当选领导者

raft通过随机超时时间, 避免一些会导致选举失败的情况，比如同一任期内，多个候选人同时发起选举，导致选票被瓜分(都投了自己)，选举失败.

在 Raft 算法中，随机超时时间是有 2 种含义的:
1. 跟随者等待领导者心跳信息超时的时间间隔，是随机的
2. 当没有候选人赢得过半票数，选举无效了，这时需要等待一个随机时间间隔，也就是说，等待选举超时的时间间隔，是随机的

#### 选举时间
Leader选举对于时间的要求比较严格, 一般要求整个集群的时间满足不等式: `广播时间 << 选举超时时间 << 平均故障间隔时间`.

广播时间指从一个节点发送心跳信息到其他节点收到信息并发出响应的平均时间.

平均故障时间是指一个节点，两次故障之间的平均时间.

为了保证集群可用，广播时间必须比选举超时时间小一个数量级，这样Leader才能发送心跳信息来重置其他Follower的选举计时器，从而防止他们切换为Candidate，触发新一轮选举.

选举超时时间是一个随机数 ，这样可以减少出现多个Candidate而瓜分选票的情况.

一般情况广播时间可以做到0.5ms~50ms，选举超时时间一般设置为200ms~1s之间.

## 日志
日志项是一种数据格式，它主要包含用户指定的数据，也就是指令（Command），还包含一些附加信息，比如索引值（Log index）、任期编号（Term）等:
- 指令：一条由客户端请求指定的、状态机需要执行的指令。你可以将指令理解成客户端指定的数据
- 索引值：日志项对应的整数索引值。它其实就是用来标识日志项的，是一个连续的、单调递增的整数号码
- 任期编号：创建这条日志项的领导者的任期编号

 Raft 的日志复制可理解成一个优化后的二阶段提交（将二阶段优化成了一阶段），减少了一半的往返消息，也就是降低了一半的消息延迟, 具体过程是:
 1. 领导者进入第一阶段，通过日志复制（AppendEntries）RPC 消息，将日志项复制到集群其他节点上
 1. 接着，如果领导者接收到大多数的“复制成功”响应后，它将日志项提交到它的状态机，并返回成功给客户端。如果领导者没有接收到大多数的“复制成功”响应，那么就返回错误给客户端。

 	领导者将日志项提交到它的状态机没通知跟随者提交日志项的原因: 这是 Raft 中的一个优化，领导者不直接发送消息通知其他节点提交指定日志项。因为领导者的日志复制 RPC 消息或心跳消息，包含了当前最大的，将会被提交的日志项索引值。所以通过日志复制 RPC 消息或心跳消息，跟随者就可以知道领导者的日志提交位置信息. 因此，当其他节点接受领导者的心跳消息，或者新的日志复制 RPC 消息后，就会将这条日志项提交到它的状态机。而这个优化，降低了处理客户端请求的延迟，将二阶段提交优化为了一段提交，降低了一半的消息延迟.


### 实现日志一致
在 Raft 算法中，领导者通过强制跟随者直接复制自己的日志项，处理不一致日志。也就是说，Raft 是通过以领导者的日志为准，来实现各节点日志的一致的. 具体有 2 个步骤。
1. 领导者通过日志复制 RPC  的一致性检查，找到跟随者节点上，与自己相同日志项的最大索引值。也就是说，这个索引值之前的日志，领导者和跟随者是一致的，之后的日志是不一致的了。
2. 领导者强制跟随者更新覆盖的不一致日志项，实现日志的一致

跟随者中的不一致日志项会被领导者的日志覆盖，而且领导者从来不会覆盖或者删除自己的日志.

## 成员变更
raft使用单节点变更（single-server changes）: 通过一次变更一个节点实现成员变更, 如果需要变更多个节点，那你需要执行多次单节点变更. 它可解决集群分裂, 比如3节点分区(2+1)后再加入2节点(2+3)会出现两个leader的场景.

> rafs最初实现成员变更的是联合共识（Joint Consensus），但这个方法实现起来难, 除了 Logcabin 外，未见到其他常用 Raft 实现采用了它.

## FAQ
### raft vs multi-paxos
类似或等价概念:
- leaer/proposer
- term/proposal ID : 标识leader的合法性
- LogEntry/Proposal
- LogIndex/InstanceID
- Leader选举/Prepare阶段
- 日志复制/Accept阶段

差异:
- leader: 强leader/弱leader

	Raft只允许日志从 Leader 流向其他服务器，当 Follower 数据与 Leader 不一致时，Leader 会强制 Follower 的日志复制自己的日志，用 Leader 的日志条目来覆盖 Follower 日志中的冲突条目, 这使得 Raft 成为一个强 Leader 的算法.

	Multi-Paxos虽然也选举Leader，但只是为了提高效率，并不限制提议只能由Leader发出（弱Leader）.

	强Leader在工程中一般使用Leader Lease和Leader Stickiness来保证:
	- Leader Lease：上一任Leader的Lease过期后，随机等待一段时间再发起Leader选举，保证新旧Leader的Lease不重叠
	- Leader Stickiness：Leader Lease未过期的Follower拒绝新的Leader选举请求

- leader选举权: 具有最新已提交日志的副本/任意副本

	Raft限制具有最新已提交的日志的节点才有资格成为Leader; Multi-Paxos 允许任何服务器成为 Leader，然后再从其他服务器补齐缺少的日志.

	Paxos 的 Candidate 在选举时，会根据服务器 ID 生成一个比已知的 term 更大的 term 来发送 RequestVotes RPC，不同 Candidate 生成的 term 不会冲突。当多个 Candidate 同时竞选时，总是由 term 更大的服务器获胜。因此 Paxos 选出一个新 Leader 是很快的。但这样新 Leader 可能会缺少一些日志条目，或者有一些日志条目与其他更新的节点有冲突，不能立即开始服务，必须首先通过从其他节点上复制缺失的、或者更新的日志条目来完成同步。而这个补日志的时间可能很花时间.

	在 Raft 中，一个新的 Leader 节点必须是一个具有最长日志的副本。因为不需要从其他节点学习日志，选出的 Leader 可以立刻开始工作.
- Leader 是否会修改日志的 term

	在 Raft 中，日志条目的 term 不会被未来的 leader 所修改。而在 Paxos 协议下，这是可能的。

	在 Raft 中，新的 Leader 选举出来后，对待上一个 term 未提交的日志条目，会继续沿用原有的 term，复制到其他节点。事实上，Raft Leader 甚至永远不会覆盖或删除其日志中的条目，只会追加写新条目.

	而在 Paxos 里，新的 Leader 选举出来后，会用新的 term 来覆盖所有未提交的日志条目.

- 日志复制: 保证连续/允许空洞

	Raft 的日志是严格顺序写入的，而 Paxos 通常允许无序写入，但需要额外的协议来填充可能因此发生的日志间隙.

	日志的连续性蕴含了这样一条性质：如果两个不同节点上相同序号的日志，只要 term 相同，那么这两条日志必然相同，且这和这之前的日志必然也相同的

	Raft在确认一条日志之前会检查日志连续性，若检查到日志不连续会拒绝此日志，保证日志连续性，Multi-Paxos不做此检查，允许日志中有空洞.

	无序写入的优势是可以实现更高的并发性和性能。但是,这需要付出额外的复杂性代价来处理日志的不一致情况。相反，Raft 的严格顺序写入降低了这种复杂性，代价是牺牲一定的并发性.
- 日志提交: 推进commit index/异步的commit消息

	Raft在AppendEntries中携带Leader的commit index，一旦日志形成多数派，Leader更新本地的commit index即完成提交，下一条AppendEntries会携带新的commit index通知其它节点；Multi-Paxos没有日志连接性假设，需要额外的commit消息通知其它节点.

Raft是基于对Multi-Paxos的两个限制形成的：
1. 发送的请求的是连续的, 也就是说Raft的append 操作必须是连续的， 而Paxos可以并发 (这里并发只是append log的并发, 应用到状态机还是有序的)
1. Raft选主有限制,必须包含最新、最全日志的节点才能被选为leader. 而Multi-Paxos没有这个限制，日志不完备的节点也能成为leader

### raft状态机一致性
ref:
- [线性一致性和 Raft](https://cn.pingcap.com/blog/linearizability-and-raft/)

raft只能保证不同节点对 Raft Log能达成一致, Log 后面的状态机（state machine）的一致性并没有做详细规定, 用户可以自由实现.

TiKV 将 commit 的 Log 应用到 RocksDB 上，由于 Input（即 Log）都一样，可推出各个 TiKV 的状态机（即 RocksDB）的状态能达成**最终**一致. 但实际多个 TiKV 不能保证同时将某一个 Log 应用到 RocksDB 上，也就是说各个节点不能**实时**一致, 加之 Leader 会在不同节点之间切换，所以 Leader 的状态机也不总有最新的状态。Leader 处理请求时稍有不慎，没有在最新的状态上进行，这会导致整个系统违反线性一致性。好在有一个很简单的解决方法：依次应用 Log，将应用后的结果返回给 Client.

这个方法的缺点很明显，性能差，因为所有请求在 Log 那边就被序列化了，无法并发的操作状态机.

优化的方法大致有两种:
- ReadIndex
- LeaseRead

## FAQ
### no-op LogEntry
raft切成leader后除了已经提交的 LogEntry, 还可能有一些较新的 LogEntry 是未提交的，raft 需要确认这些未提交 LogEntry 的状态.

由于 raft 不要求持久化 commit index，所以新 Leader 其实是不知道具体哪些 LogEntry 已经提交的. 但raft 只需要在当前 term 尝试对一条新的空日志（no-op LogEntry）达成共识即可，一旦它达成了共识，鉴于 commit index 的语义，所有之前的日志状态都可确认为已经达成quorum了.

noop 日志，它作为一个分割线, 避免了拥有过时日志的节点当选 Leader, 以[主动地避免幽灵复现问题](https://zhuanlan.zhihu.com/p/652849109).
