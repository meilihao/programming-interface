# paxos
参考:
- [如何评价阿里最近推出的paxos基础库？](https://www.zhihu.com/question/63479409)
- [阿里如何实现高性能分布式强一致的独立 Paxos 基础库？](https://www.ft12.com/article_378.html)
- [号称史上最晦涩的算法Paxos，如何变得平易近人？](https://developer.aliyun.com/article/156281)

X-Paxos当前的算法基于unique proposer的multi-paxos, 大量理论和实践已经证明了基于 unique proposer 的 multi-paxos，性能好于 multi-paxos/basic paxos，当前成熟的基于 Paxos 的系统，大部分都采用了这种方式.