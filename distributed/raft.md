# raft

Paxos 协议有一个很大的设计假设, 它要求支持多个投票, 也就是数据库里的多条日志之间是可以**乱序**提交的, 可以**并行**处理的. 但是 raft 协议的做了一个约束, 数据库的多个投票多条日志一定要按照**顺序执行**, 只能前一个日志被确认了才能确认后一个日志.

这种简化使得 Paxos 协议更易实现, 但确定是:
1. 并发能力变得差

	以前支持并发的提交, 现在只能支持一个结束以后再来下一个

1. 可用性问题

	如果采用 Paxos 协议, 当一台机器新上线的时候很快就能提供服务了, 因为不需要等前面的数据确认它就能提供服务, 但是如果使用的是 raft 协议, 需要等前面的所有日志确认以后才能提供服务???(不是应是新节点补全数据后才提供访问吗?).

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


### 选举
raft中有两个时间用于控制leader选举发生:
1. 选举超时时间(election timeout)

	每个follower在接收不到leader节点的心跳消息后, 并不会立即发起新一轮选举, 而是会等待一段时间之后才切换成candidate状态发起新一轮选举.

	election timeout是150~300ms的随机数, 而非固定, 是为了避免leader发送的心跳消息因为瞬间的网络延迟或应用程序瞬间的卡顿而迟到甚至丢失.
1. 心跳超时时间(heartbeat timeout)

	leader节点向其他follower发送心跳消息的时间间隔
