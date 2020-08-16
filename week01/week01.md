Week 01
=======

@yfaming, 2020.08.16 周日

【High Performance TiDB】Lesson 01：TiDB 整体架构
https://www.bilibili.com/video/BV17K411T7Kd

# 课程概述
## 课程目标
受众: 一线研发人员
目标:
* 掌握 TiDB 内核实现原理
* 精通一个或多个模块
* 能够对一或多个模块给出源码级别的调优
* 具备成为 TiDB 社区 committer 的潜力

## 课程考核
考核目标: 深度参与 TiDB 社区源码级别的性能调优，在导师指导下，协作或独立完成
一定难度与数量的任务，对 TiDB 的性能作出实质性的贡献。

考核方式(部分积累至 3000 分即完成课程考核):
1. 选择感兴趣的模块
2. 挑选感兴趣的 issue
3. 找到导师
4. 提交 PR
5. 得到分数

3000 分大约是 10 天工作量。

## 课程组织方式
* 前一周周末，主办方公布课程预习材料
* 周三前，预习当周知识
* 周三晚 6:00: 主办方上线课程视频，包括
  * 本周重点知识
  * 本周课程相关 issue 或文档类作业发放
* 周四以后:
  * 文档类作业当周晶前提交，选择认领，人人可以认领
  * issue 可以根据自己兴趣点，选择认领，完成 PR 开发，合并到 master 分支，获取分数

issue 要提前认领，被别人认领了，就无法继续认领。


# TiDB 总体架构
TiDB: open-source, distributed, transactional, MySQL protocol
Key features
* horizontal scalability
* high availability
* ACID
* MySQL protocol & dialect

## 核心组件
* TiKV: distributed KV storage engine
* TiDB: stateless SQL layer
* Placement Driver(PD): metadata/timestamp request, control flow(balance, failover)


(以下是 TiKV 部分============:)
# TiKV
## What is TiKV
* a distributed transactional key-value database, storage engine for TiDB
* based on the design of Google Spanner and HBase
[TiKV](https://github.com/tikv/tikv)

## TiKV Architecture
集群形式，每个 tikv 实例由各层组成(从上到下):
* (TiKV API, Coprocessor API)
* Transaction
* Raft
* RocksDB

## Logical View of TiKV
* a giant Map
  * sorted kv map
  * both keys and values are byte arrays
  * keys are sorted by byte order
* key space is divided into pieces
  * data divided into chunks called "regions"

## Physical View of TiKV
* a distributed storage engine
  * each region has multiple "replicas"
  * replicas are distributed among the storage instances via Raft algorithm

## Raft
一致性协议，复制日志
* provide strong consistency guarantees over replicas
* Roles: Leader, Candidate, Follower and Learner
* read/write only go for leader and replicated to followers and learners
* leader election
* membership change

## Raft Group
一个 region 的多个副本，组成一个 raft group

## Dynamic Split/Merge
region 数量，及 region 包含的数据，是高度动态的。
* split/merge based on data size
  * 一个 region 数据量 `>96MB` by default to split
  * 两个相邻 region 数据量 `<20MB` by default to merge

并会根据负载，触发 region 的分裂，以应对热点问题。

## PD: the manager of the raft group
管理员调整 PD 的调度策略，PD 通过 TiKV 心跳命令，回复消息，下属调度命令。
tikv 实例则执行调度命令。

## Distributed Transactions
* tikv exposes a full transactional KV API
* 基于 [Google Percolator]() 并作了优化
* 引入中央 timestamp allocator 的方式，实现两阶段提交算法
  timestamp allocator 每秒可分配 4M 个时间戳，因此 tikv 理论上的事务写入吞可达 2M 每秒
* 与 percolator 相同，最初使用乐观事务模型。从 v3.0.8 开始默认改用悲观事务
* isolation level
  * 默认 snapshot level
  * 用户可根据需要可用 read committed

Google Percolator:
[Large-scale Incremental Processing Using Distributed Transactions and Notifications](https://research.google/pubs/pub36726/)

这里提到了 Google Percolator，但之前还提到了 Spanner。二者的关系是啥？弄清楚一下。


(以下是 TiDB 部分============:)
# TiDB: The SQL engine
## SQL layer(tidb server)
* stateless SQL layer
  * clients can connect to any existing tidb-server instance
* full-featured SQL layer
  * speaks MySQL wire protocol
  * CBO: cost-base optimization
  * secondary index support
  * online DDL

[TiDB](https://github.com/pingcap/tidb)

## Architecture
* protocol layer: 连接管理，codec
* SQL core layer:
  * SQL 执行流程 parse --> logical optimizer --> physical optimizer --> executor, wo
  * workers: DDL worker, GC worker, BG job worker

## Mapping relational model to key-value pairs
```
SQL model, in tidb-server:
+-------------+--------------+-----------------+-------+
| id(primary) | name(unique) | age(non-unique) | score |
+=============+==============+=================+=======+
|     1       |      Bob     |         12      |   99  |
+-------------+--------------+-----------------+-------+

key-value model in TiKV:
+-----------------+--------------------+---------------+
|    index_type   |       key          |   value       |
+=================+====================+===============+
|  primary_index  | t_tableID_r_1      | (Bob, 12, 99) |
+-----------------+--------------------+---------------+
|  name(uniuqe)   | t_tableID_i_Bob    | 1             |
+-----------------+--------------------+---------------+
| age(non-unique) | t_tableID_i_(12,1) | null          |
+-----------------+--------------------+---------------+
```
这个图很有意思。想想 MySQL 中索引的实现，对比一下。
主键，value 保存了该行记录的全部信息。
唯一键，value 保存了对应的主键
非唯一键，key 保存了该键对应的值及主键的值。value 为空。

## Write data into a TiDB Cluster
client:
```
INSERT INTO `t` VALUES(1, "Bob", 12, 99);
```

tidb-server: encode to kv pairs
TiKV: atomic set & commit

## Read data from a TiDB Cluster
比 write 更加复杂。
```SQL
SELECT t1.c1, t2.c3
FROM t1 JOIN t2 on t1.c1 = t2.c1
WHERE t1.c2 = 100 AND t2.c3 > 1;
```
解析 SQL 语句，形成初始执行计划。
依据规则，改写执行计划，获得逻辑上等价，但执行效率更优的执行计划。
利用数据的物理属性，以执行代价为目标，对执行计划进行进一步的改写，形成最终的物理执行计划。


# Tools
* BR: fast backup and restore
  具备分布式的备份与恢复能力
  在单一 TiKV 实例上，可获得 ~150MB/s 的速度。全局备份速度可随着 TiKV 实例数量而线性扩展
* TiCDC
  数据变更抓取工具。从 TiKV 拉取 kv/row change log
* Data Migration(DM)

# Homework
下载 tidb, tikv, pd 代码，改写之，编译，并部署一个集群，包括
* 1 tidb
* 1 pd
* 3 tikv
改写后，使得 tidb 启动事务时，打印 "hello transaction" 日志。

输出: 写一篇文章介绍以上过程。
