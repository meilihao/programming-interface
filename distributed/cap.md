# 分布式
![](/misc/img/distributed/bg2018071607.jpg)

参考:
- [分布式系统一致性（ACID、CAP、BASE、二段提交、三段提交、TCC、幂等性）原理详解](https://juejin.im/post/5c9443406fb9a070fe0dd9a9)
- [分布式事务数据库事务CAP定理BASE理论分布式事务案例](https://cloud.tencent.com/developer/article/1346890)
- [分布式系统的事务处理](http://coolshell.cn/articles/10910.html)

分布式的特点:
- 分布性 ：系统中的机器随意分布
- 对等性：组成分布式系统的所有节点都是对等或相对对等的
- 并发性：需要协调并发操作一些共享资源git
- 缺乏全局时钟：在分布式系统中，很难定义两件事情谁先谁后
- 故障总是会发生：任何在设计阶段考虑到的异常情况，一定会在系统实际运行中发生

分布式系统常见问题:
- 通讯异常：单机内存访问往往延时在纳秒数量级（通常是 10ns 左右）；网络则在 0.1～1ms，消息丢失和消息延迟变得非常普遍
- 网络分区：俗称”脑裂”，部分节点和整个分布式系统失去联系，自己单独组成了一个小集群

    当两（多）个节点同时认为自已是唯一处于活动状态的服务器从而出现争用资源的情况，这种争用资源的场景即是所谓的`脑裂（split-brain）`或`区间集群（partitioned cluster）`.
- 三态：成功、失败和超时. 无法预测超时的请求是否到达了接收方，还是在接收方返回的时候丢失了

## cap : 分布式系统的三个指标
- Consistency : 一致性, 这里特指强一致

    所有观测者看到的数据都是相同的, 即使存在并发的更新.

    所有节点上的数据**时刻**保持同步. 一致性严谨的表述是**原子读写**, 即所有读写都应该看起来是原子的或串行的. 所有的读写请求都好像是经全局排序过的一样,写后面的读一定能读到前面所写的内容.

    > 一致性与结果的正确性没有关系,而是系统对外主现的状态是否一致. 例如,所有节,最都达成一个错误的共识也是一致性的一种表现.
- Availability : 可用性

    任何**非故障节点**都应该在**有限的时间**内给出请求的响应,**不论请求是否成功**.
- Partition tolerance : 分区容忍(或容错)性

    当发生网络分区时(即**节点之间无法通信**),在丢失**任意多**消息的情况下,系统仍然能够保证对外提供满足一致性和可用性的服务.

### Consistency 和 Availability 的矛盾
一般来说，**分区容错无法避免**，因此可以认为 CAP 的 P 总是成立. **CAP 定理告诉我们，剩下的 C 和 A 无法同时做到**.

#### AP
一旦发生网络分区(P),节点之间将无法通信,为了满足高可用(A),每个节点只能用本地数据提供服务, 这样就会导致数据的不一致(!C).

#### CP
如果要求数据在各个服务器上是强一致的(C),然而网络分区(P)会导致**同步**时间无限延长,那么如此一来可用性就得不到保障了(!A). 坚持事务 ACID 的传统数据库以及对结果一致性非常敏感的应用(例如,金融业务)通常会做出这样的选择.

#### CA
如果不存在网络分区,那么强一致性和可用性是可以同时满足的, 但该系统不一定是分布式系统了, 同时**放弃了 P 也就等于放弃了系统的扩展性**.

### 拜占庭错误(Byzantine Fa ilure)
它在计算机科学领域特指分布式系统中的某些恶意节点扰乱系统的正常运行,包括选择性不传递消息,选择性伪造消息等. 很显然, 拜占庭错误是一个 overly pessimistic 模型(最悲观 、 最强的错误模型).

进程失败错误(fail - stop Failure ,如同若机)则是一个 overly optimistic 模型(最乐观、最弱的错误模型) . 这个模型假设当某个节点出错时, 这个节点会停止运行,并且其他所有节点都知道这个节点发生了错误.

结论:

一个 RSM(Replicated State Machine, 复制状态机)系统要容忍 N 个拜占庭错误,至少需要 2N+l 个复制节点. 如果只是把错误的类型缩小到进程失败,则至少需要 N+l 个复制节点才能容错.

综上所述, 对于一个通用的、具有复制状态机语义的分布式系统,如果要做到 N 个节点的容错,理论上最少需要 2N+l 个复制节点. 这也是典型的一致性协议都要求半数以上(N/2+ 1)的服务器可用才能做出一致性决定的原因.

### BASE
BASE(Basic Availability, Soft state, Eventually Consistency)理论是对CAP中的一致性和可用性进行一个权衡的结果，该理论的核心思想就是：我们无法做到强一致，但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性(Eventualconsistency). 例如, NoSQL 数据库(Cassandra 、 CouchDB 等)往往会放宽对一致性的要求(**满足最终一致性**即可),以此来换取基本的可用性.

基本可用(Basic Availability)：在系统出现故障的时候，允许损失部分可用性以保证核心功能可用. 比如响应时间上的损失和功能上的损失.
软状态(Soft state)：也称为弱状态，表示允许系统中的数据存在中间状态，即允许不同的数据副本之间数据同步存在延迟
最终一致性(Eventually Consistency)：经过一段时间的同步之后，系统最终能够达到一个数据一致的状态.

最终一致性的一些变种：
1. 因果一致性：进程 A 更新数据后通知了进程 B，进程 B 必须对修改后的数据可见
1. 读己之所写：进程 A 对数据做的更改，自己总是能够获得最新的值
1. 会话一致性：在单个会话中，进程 A 总是能够看到最新的值
1. 单调读一致性：在系统中读出数据项 Y 的值 y 后，后面的读取中不允许返回比 y 更旧的值
1. 单调写一致性：系统需要保证来自同一个进程的写操作被顺序的执行

## 2PC 与 3PC
参考:
- [分布式一致性协议学习笔记：2PC 和 3PC](https://www.tuicool.com/articles/eeIZJbe)

当一个事务需要跨越多个分布式节点的时候，需要引入一个成为 “协调者”（Coordinator） 的组件来统一调度所有分布式节点的执行逻辑，这些被调度的分布式节点则被称为 “参与者”（Participant）. 协调者负责调度参与者的行为，并最终决定这些参与者是否真的需要把事务进行真正的提交. 由此，衍生出二阶段提交和三阶段提交两种协议.

### 2PC
两阶段提交协议算法可以概括为参与者将操作的成功与否通知协调者，再由协调者根据所有参与者的反馈情况，来决定每个参与者的操作是否要提交.

两阶段提交主要包含 投票（Propose） 和 执行（Commit） 两个阶段.
1. Propose 阶段: 

    协调者向所有参与者（voter）发送事务内容(Prepare消息)，询问是否可以执行事务提交操作，并等待参与者的响应. 

    各个参与者为该事务预留资源，保证在下一个阶段能够提交事务，如果资源可以获取反馈给协调者 agree 响应，反之返回 disagree 响应.

1. 执行（Commit） 阶段

    如果协调者收到了任何一个参与者的失败消息或超时消息，则直接给每个参与者发送回滚消息，否则发送提交消息, 然后参与者根据协调者的指令进行提交或回滚.

    事务提交：如果在 Propose 阶段，所有参与者都返回的是 agree 信号，那么协调者会发送提交事务的请求；所有参与者收到事务提交请求后会正式执行事务提交操作并释放事务执行期间占用的资源，执行完成后将事务执行结果反馈给协调者；协调者收到所有参与者的反馈信息后，事务执行完毕.

    事务中断：如果在 Propose 阶段有参与者返回 disagree 或者有的节点超时未返回，没有投票达成一致，那么协调者会向所有参与者发送回滚请求；参与者收到回滚请求之后会释放掉在 Propose 阶段占用的资源，然后反馈信号给协调者；协调者收到反馈消息之后，完成事务终端.

两阶段提交协议关注的是**分布式事务的原子性**，这个分布式事务提交之后，数据自然就会保持一致.

二阶段提交协议的优点是：原理简单，实现方便.
二阶段提交协议的缺点是：
- 同步阻塞：2PC 协议的各个阶段都是同步阻塞的，任何节点故障都有会导致事务提交失败
- 单点问题：一旦协调者出现问题，将无法提交事务. 如果协调者在收到参与者的 Propose 确认之后，发送 Commit 信号之前发生故障，那么所有参与者占用的资源将无法得到释放.
- 数据不一致：如果在 Commit 阶段部分参与者发生了故障，导致事务无法提交，但是正常的参与者提交了事务, 导致数据不一致

### 3PC, 增加了超时机制
三阶段提交是在二阶段提交的基础上进行的改进，将二阶段提交的 Propose Phase 一分为二，形成了 CanCommit、PreCommit 和 doCommit 三个阶段.

CanCommit：协调者向所有参与者询问是否可以执行事务提交操作；各参与者想协调者反馈是否可以提交事务.

PreCommit：
- 如果所有参与者反馈的都是 agree， 那么就会执行事务预提交. 协调者发送 PreCommit 请求，参与者收到请求之后进行资源的分配工作，最后将资源分配结果反馈这协调者. 等待后面的 commit 命令或者 abort 命令.
- 如果任意一个参与者返回了 disagree，或者在协调者等待超时之后,协调者将会向所有参与者节点发送 abort 请求.

doCommit：
- 如果上面 PreCommit 全部成功，即协调者收到了全部参与者的成功反馈，那么协调者将会发送 doCommit 请求；参与者接收到 doCommit 会正式提交事务然后释放事务执行期间占用的资源，反馈事务提交结果给协调者；协调者收到参与者反馈的消息后，完成事务.
- 如果上面的 PreCommit 有部分失败反馈，或者是等待超时就会中断事务. 协调者会向所有参与者节点发送 abort 请求；参与者收到 abort 请求后执行回滚操作，并反馈给协调者；协调者收到反馈消息后，完成事务的中断.

需要注意的是，在 doCommit 阶段，可能出现下面的两种故障：
- 协调者出现问题
- 协调者和参与者之间网络故障

无论出现上面那种情况，都会造成参与者无法及时收到来自协调者的 doCommit 请求或者是 abort 请求，针对这种情况，参与者会在等待超时之后，**继续提交事务**, 原因: 因为能进到阶段3说明协调者已经完成了阶段2对所有参与者的资源预留锁定, 乐观推断事务能够执行成功.

针对上述问题，Paxos/Raft算法提供了解决方案: 每次处理经过分布式节点中的参与者投票决议是否允许提交，得到超过半数投票者的同意后提交.

三阶段提交协议的优点是：相比于 2PC 能够在单点故障后继续达成一致.
三阶段提交协议的缺点是：引入了新的问题，如果某个参与者与协调者出现在不同的网络分区，那么该参与者会提交事务，有可能出现数据不一致的情况.

### 柔性事务TCC(Try-Confirm-Cancel)
Try阶段：完成所有的业务检查，预留(锁定)业务资源
Confirm阶段：确认执行业务操作
Cancel阶段： 业务最终失败，或者部分业务资源锁定失败，释放已锁定的资源

TCC追求的是 最终一致性, 即TCC是BASE理论的一种体现,

### FLP 不可能性
参考:
- [分布式理论梳理——FLP定理](https://my.oschina.net/duofuge/blog/1512344)
- [FLP 不可能原理](https://www.iminho.me/wiki/docs/blockchain_guide/distribute_system-flp.md)

FLP **定理**实际上说明了在允许节点失效的场景下,基于**异步通信**方式的分布式协议,无法确保在有限的时间内达成一致性.

> FLP 不可能原理告诉我们: 不要浪费时间，去试图为异步分布式系统设计面向任意场景的共识算法.
>
> 在分布式系统中,“异步通信”与“同步通信”的最大区别是没有时钟、不能时间同步、不能使用超时、不能探测失败、消息可任意延迟、消息可乱序等.

根据 FLP 定理,实际的一致性协议( Paxos 、 Ra武等)在理论上都是有缺陷的, 最大的问题是理论上存在不可终止性! 至于 Paxos 和 Raft 协议在工程的实现上都做了调整(例如, Paxos 和 Raft 都通过随机的方式显著降低了发生算法无法终止的概率), 并使用同步假设(或保证 safety ,或保证 liveness ), 从而规避了理论上存在的问题.

### [Paxos 算法与 Raft 算法](https://www.iminho.me/wiki/docs/blockchain_guide/distribute_system-paxos.md)
参考:
- [raft动画](http://thesecretlivesofdata.com/raft/)
- [微信自研生产级paxos类库PhxPaxos实现原理介绍](http://mp.weixin.qq.com/s?__biz=MzI4NDMyNTU2Mw==&mid=2247483695&idx=1&sn=91ea422913fc62579e020e941d1d059e)
- [号称史上最晦涩的算法Paxos，如何变得平易近人](https://yq.aliyun.com/articles/156281)

与两阶段提交协议不同，在分布式系统中，Paxos算法用于保证同一份数据中多个副本之间的数据一致性. Paxos算法通过采用选举的方式，利用少数服从多数的思想来解决问题.

分布式协议Paxos和Raft算法演进过程:
1. Basic-Paxos解决的问题：在一个分布式系统中，如何就一个提案达成一致

    需要借助两阶段提交实现：
    1. Prepare阶段：

    Proposer选择一个提案编号n并将prepare请求发送给 Acceptor
    Acceptor收到prepare消息后，如果提案的编号大于它已经回复的所有prepare消息，则Acceptor将自己上次接受的提案回复给Proposer，并承诺不再回复小于n的提案

    2. Accept阶段：

    当一个Proposer收到了多数Acceptor对prepare的回复后，就进入批准阶段. 它要向回复prepare请求的Acceptor发送accept请求，包括编号n和根据prepare阶段决定的value（如果根据prepare没有已经接受的value，那么它可以自由决定value）.
    在不违背自己向其他Proposer的承诺的前提下，Acceptor收到accept请求后即接受这个请求.
2. Mulit-Paxos解决的问题：在一个分布式系统中，如何就一批提案达成一致

    当存在一批提案时，用Basic-Paxos一个一个决议当然也可以，但是每个提案都经历两阶段提交，显然效率不高. Basic-Paxos协议的执行流程针对每个提案（每条redo log）都至少存在三次网络交互：1. 产生log ID；2. prepare阶段；3. accept阶段

    所以，Mulit-Paxos基于Basic-Paxos做了优化，**在Paxos集群中利用Paxos协议选举出唯一的leader，在leader有效期内所有的议案都只能由leader发起**.

    这里强化了协议的假设：即leader有效期内不会有其他server提出的议案. 因此，对于后续的提案，我们可以简化掉产生log ID阶段和Prepare阶段，而是由唯一的leader产生log ID，然后直接执行Accept，得到多数派确认即表示提案达成一致（每条redo log可对应一个提案）.

    > 相关产品: X-DB、OceanBase、Spanner,  MySQL Group Replication的xcom都是使用Multi-Paxos来保障数据一致性.
3. Raft

    Raft也使用了分而治之的思想，把算法分为三个子问题:

    1. 选举（Leader election）
    1. 日志复制（Log replication）
    1. 安全性（Safety）

    分解后，整个raft算法变得易理解、易论证、易实现
4. Mulit-Raft

    许多NewSQL数据库的数据规模上限都定位在100TB以上，为了负载均衡，都会对数据进行分片（sharding），所以就需要使用多个Raft集群（即Multi-Raft），每个Raft集群对应一个分片.
    在多个Raft集群间可增加协同来减少资源开销、提升性能（如：共享通信链接、合并消息等）.

    > 相关产品: TiDB、CockroachDB、PolarDB都是使用Mulit-Raft来保障数据一致性

因为Paxos的难理解, 这里只考虑Raft.

Raft 算法主要使用两种方法来提高可理解性: 问题分解和减少状态空间.
- 问题分解

    Raft 算法把问题分解成了领袖选举(leader election )、日志复制( log replication ) 、 安全性( safety )和成员关系变化
(membership changes )这几个子问题:

    - 领袖选举:在一个领袖节点发生故障之后必须重新给出一个新的领袖节点
    - 日志复制:领袖节点从客户端接收操作请求,然后将操作日志复制到集群中的其他服务器上,并且强制要求其他服务器的日志必须和自己的保持一致.
    - 安全性: Raft 关键的安全特性是状态机安全原则( State Machine Safety ). 即如果一个服务器已经将给定索引位置的日志条目应用到状态机中,则所有其他服务器不会在该索引位置应用不同的条目.
    - 成员关系变化:配置发生变化的时候,集群能够继续工作
- 减少状态空间

    Raft 算法通过减少需要考虑的状态数量来简化状态空间. 这将使得整个系统更加一致并且能够尽可能地消除不确定性. 需要特别说明的是,日志条目之间不允许出现空洞,并且还要限制日志出现不一致的可能性. 尽管在大多数情况下, Raft 都在试图消除不确定性以减少状态空间. 但在一种场景下(选举), Raft 会用随机方法来简化选举过程中的状态空间.


Raft 算法采用的是非对称节点关系模型, 在一个由 Raft 协议组织的集群中,一共包含如下 3 类角色:
- Leader (领袖)
- Candidate (候选人)
- Follower (群众)

#### Raft选举过程
Raft协议中，一个节点有三个状态：Leader、Follower和Candidate，但同一时刻只能处于其中一种状态。Raft选举实际是指选举Leader，选举是由候选者（Candidate）主动发起，而不是由其它第三者。

并且约束只有Leader才能接受写和读请求，只有Candidate才能发起选举。如果一个Follower和它的Leader失联（失联时长超过一个Term），则它自动转为Candidate，并发起选举。

发起选举的目的是Candidate请求（Request）其它所有节点投票给自己，如果Candidate获得多数节点（a majority of nodes）的投票（Votes），则自动成为Leader，这个过程即叫Leader选举。

在Raft协议中，正常情况下Leader会周期性（不能大于Term）的向所有节点发送AppendEntries RPC，以维持它的Leader地位。

相应的，如果一个Follower在一个Term内没有接收到Leader发来的AppendEntries RPC，则它在延迟随机时间（150ms~300ms）后，即向所有其它节点发起选举。

采取随机时间的目的是避免多个Followers同时发起选举，而同时发起选举容易导致所有Candidates都未能获得多数Followers的投票（脑裂，比如各获得了一半的投票，谁也不占多数，导致选举无效需重选），因而延迟随机时间可以提高一次选举的成功性

## FAQ
### Raft和Multi-Paxos的区别
Raft是基于对Multi-Paxos的两个限制形成的：

发送的请求的是连续的, 也就是说Raft的append 操作必须是连续的， 而Paxos可以并发 (这里并发只是append log的并发, 应用到状态机还是有序的).
Raft选主有限制,必须包含最新、最全日志的节点才能被选为leader. 而Multi-Paxos没有这个限制，日志不完备的节点也能成为leader.

Raft可以看成是简化版的Multi-Paxos.

Multi-Paxos允许并发的写log,当leader节点故障后，剩余节点有可能都有日志空洞. 所以选出新leader后, 需要将新leader里没有的log补全,在依次应用到状态机里.