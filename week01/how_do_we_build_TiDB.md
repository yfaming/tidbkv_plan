How do we build TiDB
====================

https://pingcap.com/blog-cn/how-do-we-build-tidb/
2016.10.01 刘奇

# 为什么我们要创建另外一个数据库？
基于 MySQL 的方案的天花板太低了。

不能通过 MySQL 的 server 把 InnoDB 变成分布式数据库。
(这里应该是指，不动 InnoDB，而只改造上层的方式？)
其执行计划是单机的。比如认为读取一行和读取下一行的开销很小。这在分布式系统中不成立。

在分布式系统中，把数据都拿回来计划，太慢了。需要把运算往下推，向上只返回最终结果。
因此一定要用分布式的 plan。

把 MySQL 改造成分布式，另一方案是 Proxy。
但 Proxy 不支持分布式的 transaction，不支持跨节点的 join，无法理解复杂的 plan。

又，MySQL 支持的复制方式是半同步或者异步。但半同步可以降级成异步。
任何时候出了问题你不敢切换。因为可能是异步复制，有一部分数据还没有过来，切换的话数据就不一致了。

另外，多数据中心的复制和数据中心的容灾，MySQL 在这上面是做不好的。

分布式数据库 NewSQL 之理论基础，最重要的几篇论文:
* Google Spanner, 2013
* Raft, 2014


# Google Spanner / F1
Spanner 是全球分布的数据库。
Google 高级别数据有 7 副本，美国保存 3 个副本，另外两个国家各保存 2 个副本。

国内主流做法是两地三中心，但基本上不能自动切换。

事务支持方面，Google 用了 GPS 时钟和原子钟。
Google 先有了 BigTable，然后是 MegaStore，然后是 Spanner，F1 是在 Spanner 上构建的。


# TiDB and TiKV
TiKV 和 TiDB 基本上对应 Google Spanner 和 Google F1。

逻辑分层:
* MySQL protocol server
* SQL layer
* Transaction
* MVCC
* Raft
* Local KV storage (RocksDB)

TiKV 的 Raft 实现，是从 etcd port 过来的。

# 分布式事务模型
基于 Google Percolator, 2006 年的论文。
Google 在内部做增量处理的时候发现的方法，本质上是二阶段提交。用的乐观锁(v3.0.8 开始默认用悲观锁了)。

注意，事务是在 TiKV 上做的。不是在 TiDB 上。

# Placement Driver
监控整个系统的状态。
如果节点十分钟内无法被其他节点探测到，则认为其挂了。此时要在系统中重新选一台机器，复制数据，以保持足够副本数。
PD 还会根据系统负载，不断 move 数据。
Raft 还支持 leader transfer，即不移动数据，而换 leader。这样的好处是，不会形成抖动。

# DDL
方案来自于: Online, Asynchronous Schema Change in F1 paper
大意，先标记，然后不断迭代，直到迭代到一个最终可以对外公开的状态。
没太明白。
