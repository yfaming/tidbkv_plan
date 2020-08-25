Week02 Homework
===============

@yfaming, 2020.08.23 周日

# homework
分值：300

题目描述：

使用 sysbench、go-ycsb 和 go-tpc 分别对
 TiDB 进行测试并且产出测试报告。

测试报告需要包括以下内容：

* 部署环境的机器配置(CPU、内存、磁盘规格型号)，拓扑结构(TiDB、TiKV 各部署于哪些节点)
* 调整过后的 TiDB 和 TiKV 配置
* 测试输出结果
* 关键指标的监控截图
	    * TiDB Query Summary 中的 qps 与 duration
	    * TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
	    * TiKV Details 面板中 grpc 的 qps 以及 duration

输出：写出你对该配置与拓扑环境和 workload 下 TiDB 集群负载的分析，提出你认为的 TiDB 的性能的瓶颈所在(能提出大致在哪个模块即 可)

截止时间：下周二（8.25）24:00:00(逾期提交不给分)

# 准备机器

机器要求(生产环境)，见 https://docs.pingcap.com/zh/tidb/stable/hardware-and-software-requirements#%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83

+--------------+-------+----------+--------------------+--------------------+
| 组件 | CPU   | 内存  | 硬盘类型 | 网络               | 实例数量(最低要求) |
+------+-------+-------+----------+--------------------+--------------------+
| TiDB | 16核+ | 32GB+ |   SAS    | 万兆网卡(2 块最佳) |       2            |
+--------------+-------+----------+--------------------+--------------------+
| PD   | 4核+  |  8GB+ |   SSD    | 万兆网卡(2 块最佳) |       3            |
+--------------+-------+----------+--------------------+--------------------+
| TiKV | 16核+ | 32GB+ |   SSD    | 万兆网卡(2 块最佳) |       3            |
+--------------+---------------------------------------+--------------------+

省点钱，准备使用 6 台机器，其中 TiKV 3，TiDB 2，PD 1 节点。
所有机器都用相同规格： 8CPU, 32GB 内存, 640GB SSD。
直接在 DigitalOcean 购买，按小时计费，每节点每小时费用为 $0.12 ($160 每月)。

没用 Vultr，因为账户可购买的实例数有限制，最多 5 个(现在已经加到 10 个了，但是还有个每月费用限制为 $100，导致无法购买更高配置的机器)。
没用 Linode，因为貌似无法一次购买之台虚拟机，需要一台一台地手动创建。



# 安装软件
```sh
# ntp
sudo apt install ntpstat ntp
sudo systemctl start ntp.service
# 用 ntpstat 查看输出
ntpstat

# 添加用户
# 还得设置好 home 目录之类的才行
# 最后全都使用了 root 用户
sudo useradd tidb && passwd tidb
# 执行，并添加一行： tidb ALL=(ALL) NOPASSWD: ALL
# root ALL=(ALL) NOPASSWD: ALL
sudo visudo

# 集群中各机器，生成密钥
ssh-keygen
# 只在中控机上执行，将 <ip> 替换为集群中的机器的 IP
# 如果中控机包含在集群中，需要对中控机自己也执行这一条命令
ssh-copy-id -i ~/.ssh/id_rsa.pub <ip>


# sshpass (没用上)
sudo apt-get install sshpass
# tiup
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
```

# 部署

拓扑结构:
```yaml
## 现有的机器
## 104.156.238.151 tione
## 66.42.38.84 titwo

# # Global variables are applied to all deployments and used as the default value of
# # the deployments if a specific deployment value is missing.
global:
  user: "root"
  ssh_port: 22
  deploy_dir: "/tidb-deploy"
  data_dir: "/tidb-data"

pd_servers:
  - host: 104.156.238.151

tidb_servers:
  - host: 104.156.238.151
  - host: 66.42.38.84

tikv_servers:
  - host: 104.156.238.151
  - host: 66.42.38.84
  - host: 66.42.38.84

# 以下这些都加上，不然打开 dashboard 后，会提示没有 prometheus
monitoring_servers:
  - host: 104.156.238.151

grafana_servers:
  - host: 104.156.238.151

alertmanager_servers:
  - host: 104.156.238.151
```

```sh
# deploy
tiup cluster deploy tibench v4.0.4 ./topology.yaml

# display
tiup cluster display tibench

# start cluster
tiup cluster start tibench

# destroy
tiup cluster destropy tibench
```

验证集群运行状态：
```
mysql -u root -h 104.156.238.151 -P 4000
```
另，参考 [验证集群运行状态](https://docs.pingcap.com/zh/tidb/stable/post-installation-check)


# sysbench
https://github.com/akopytov/sysbench

Ubuntu 下，可用 `dpkg -L sysbench` 查看安装的文件。

[如何用 Sysbench 测试 TiDB](https://docs.pingcap.com/zh/tidb/stable/benchmark-tidb-using-sysbench)

sysbench 的配置文件：

```
mysql-host=165.22.253.27
mysql-port=4000
mysql-user=root
mysql-password=
mysql-db=sbtest
time=600
threads=8
report-interval=10
db-driver=mysql
```

修改 TiDB 设置，在 MySQL client 中执行 `set global tidb_disable_txn_auto_retry = off;`。
创建用于测试的 DB，`create database sbtest;`，与上面配置中的保持一致。


```sh
# (done)oltp_point_select
# 10 个表，1 百万条记录
# --threads=48 参数可以调整
sysbench --config-file=sysbench_config oltp_point_select --tables=10 --table-size=1000000 prepare
sysbench --config-file=sysbench_config oltp_point_select --tables=10 --table-size=1000000 run
sysbench --config-file=sysbench_config oltp_point_select --tables=10 --table-size=1000000 cleanup

# (done)oltp_insert
sysbench --config-file=sysbench_config oltp_insert --tables=10 --table-size=1000000 prepare
sysbench --config-file=sysbench_config oltp_insert --tables=10 --table-size=1000000 run
sysbench --config-file=sysbench_config oltp_insert --tables=10 --table-size=1000000 cleanup

# oltp_delete
sysbench --config-file=sysbench_config oltp_delete --tables=10 --table-size=1000000 prepare
sysbench --config-file=sysbench_config oltp_delete --tables=10 --table-size=1000000 run
sysbench --config-file=sysbench_config oltp_delete --tables=10 --table-size=1000000 cleanup

# oltp_update_index
sysbench --config-file=sysbench_config oltp_update_index --tables=10 --table-size=1000000 --threads=10 prepare
sysbench --config-file=sysbench_config oltp_update_index --tables=10 --table-size=1000000 run
sysbench --config-file=sysbench_config oltp_update_index --tables=10 --table-size=1000000 cleanup

# oltp_update_non_index
sysbench --config-file=sysbench_config oltp_update_non_index --tables=10 --table-size=1000000 --threads=10 prepare
sysbench --config-file=sysbench_config oltp_update_non_index --tables=10 --table-size=1000000 --threads=8 run
sysbench --config-file=sysbench_config oltp_update_non_index --tables=10 --table-size=1000000 cleanup

# bulk_insert
sysbench --config-file=sysbench_config bulk_insert --tables=10 --table-size=1000000 --threads=10 prepare
sysbench --config-file=sysbench_config bulk_insert --tables=10 --table-size=1000000 --threads=8 run
sysbench --config-file=sysbench_config bulk_insert --tables=10 --table-size=1000000 cleanup

# oltp_read_only
sysbench --config-file=sysbench_config oltp_read_only --tables=10 --table-size=1000000 --threads=10 prepare
sysbench --config-file=sysbench_config oltp_read_only --tables=10 --table-size=1000000 --threads=8 run
sysbench --config-file=sysbench_config oltp_read_only --tables=10 --table-size=1000000 cleanup

# oltp_read_write
sysbench --config-file=sysbench_config oltp_read_write --tables=10 --table-size=1000000 --threads=10 prepare
sysbench --config-file=sysbench_config oltp_read_write --tables=10 --table-size=1000000 --threads=8 run
sysbench --config-file=sysbench_config oltp_read_write --tables=10 --table-size=1000000 cleanup

# oltp_write_only
sysbench --config-file=sysbench_config oltp_write_only --tables=10 --table-size=1000000 --threads=10 prepare
sysbench --config-file=sysbench_config oltp_write_only --tables=10 --table-size=1000000 --threads=8 run
sysbench --config-file=sysbench_config oltp_write_only --tables=10 --table-size=1000000 cleanup

# select_random_points
sysbench --config-file=sysbench_config select_random_points --tables=10 --table-size=1000000 --threads=10 prepare
sysbench --config-file=sysbench_config select_random_points --tables=10 --table-size=1000000 --threads=8 run
sysbench --config-file=sysbench_config select_random_points --tables=10 --table-size=1000000 cleanup

# select_random_ranges
sysbench --config-file=sysbench_config select_random_ranges --tables=10 --table-size=1000000 --threads=10 prepare
sysbench --config-file=sysbench_config select_random_ranges --tables=10 --table-size=1000000 --threads=8 run
sysbench --config-file=sysbench_config select_random_ranges --tables=10 --table-size=1000000 cleanup
```



# go-ycsb
## workloada
```sh
./bin/go-ycsb load mysql -P workloads/workloada -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
./bin/go-ycsb run mysql -P workloads/workloada -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
```
结果:
```txt
Run finished, takes 10.498999555s
READ   - Takes(s): 10.5, Count: 517, OPS: 49.3, Avg(us): 2124, Min(us): 1461, Max(us): 6819, 99th(us): 4000, 99.9th(us): 7000, 99.99th(us): 7000
UPDATE - Takes(s): 10.5, Count: 483, OPS: 46.1, Avg(us): 19411, Min(us): 7126, Max(us): 840349, 99th(us): 131000, 99.9th(us): 841000, 99.99th(us): 841000
```

## workloadb
```sh
./bin/go-ycsb load mysql -P workloads/workloadb -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
./bin/go-ycsb run mysql -P workloads/workloadb -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
```
结果:
```txt
Run finished, takes 2.104958855s
READ   - Takes(s): 2.1, Count: 934, OPS: 444.7, Avg(us): 1677, Min(us): 1144, Max(us): 6974, 99th(us): 3000, 99.9th(us): 7000, 99.99th(us): 7000
UPDATE - Takes(s): 2.1, Count: 66, OPS: 31.9, Avg(us): 7939, Min(us): 6339, Max(us): 16428, 99th(us): 17000, 99.9th(us): 17000, 99.99th(us): 17000
```

## workloadc
```sh
./bin/go-ycsb load mysql -P workloads/workloadc -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
./bin/go-ycsb run mysql -P workloads/workloadc -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
```
结果:
```txt
Run finished, takes 1.79157677s
READ   - Takes(s): 1.8, Count: 1000, OPS: 559.1, Avg(us): 1777, Min(us): 1231, Max(us): 9076, 99th(us): 3000, 99.9th(us): 9000, 99.99th(us): 10000
```

## workloadd
```sh
./bin/go-ycsb load mysql -P workloads/workloadd -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
./bin/go-ycsb run mysql -P workloads/workloadd -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
```
结果:
```txt
Run finished, takes 2.168015872s
INSERT - Takes(s): 2.1, Count: 59, OPS: 27.6, Avg(us): 7779, Min(us): 6399, Max(us): 16558, 99th(us): 17000, 99.9th(us): 17000, 99.99th(us): 17000
READ   - Takes(s): 2.2, Count: 941, OPS: 434.9, Avg(us): 1796, Min(us): 1105, Max(us): 9658, 99th(us): 3000, 99.9th(us): 10000, 99.99th(us): 10000
```

## workloade
```sh
./bin/go-ycsb load mysql -P workloads/workloade -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
./bin/go-ycsb run mysql -P workloads/workloade -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
```
结果:
```txt
Run finished, takes 2.664580016s
INSERT - Takes(s): 2.6, Count: 37, OPS: 14.2, Avg(us): 7152, Min(us): 6134, Max(us): 8525, 99th(us): 9000, 99.9th(us): 9000, 99.99th(us): 9000
SCAN   - Takes(s): 2.7, Count: 963, OPS: 361.9, Avg(us): 2474, Min(us): 1754, Max(us): 12870, 99th(us): 5000, 99.9th(us): 13000, 99.99th(us): 13000
```

## workloadf
```sh
./bin/go-ycsb load mysql -P workloads/workloadf -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
./bin/go-ycsb run mysql -P workloads/workloadf -p mysql.host=165.22.253.27 -p mysql.port=4000 -p mysql.db=sbtest
```
结果:
```txt
Run finished, takes 5.486332174s
READ   - Takes(s): 5.5, Count: 1000, OPS: 182.4, Avg(us): 1836, Min(us): 1225, Max(us): 4002, 99th(us): 3000, 99.9th(us): 4000, 99.99th(us): 5000
READ_MODIFY_WRITE - Takes(s): 5.5, Count: 504, OPS: 92.1, Avg(us): 9053, Min(us): 7067, Max(us): 16289, 99th(us): 13000, 99.9th(us): 17000, 99.99th(us): 17000
UPDATE - Takes(s): 5.5, Count: 504, OPS: 92.1, Avg(us): 7183, Min(us): 5596, Max(us): 14610, 99th(us): 11000, 99.9th(us): 15000, 99.99th(us): 15000
```

# go-tpc
https://github.com/pingcap/go-tpc

## tpcc
warehouse=8
调整 threads 数量，以获得不同的 tpcc 值。
```
go-tpc tpcc prepare -H 165.22.253.27 -P 4000 -D tpcc --warehouses 8
go-tpc tpcc check -H 165.22.253.27 -P 4000 -D tpcc --warehouses 8
go-tpc tpcc run -H 165.22.253.27 -P 4000 -D tpcc --warehouses 8 --time 4m --threads 8
go-tpc tpcc cleanup -H 165.22.253.27 -P 4000 -D tpcc --warehouses 8
```

# tpch
```
go-tpc tpch prepare -H 165.22.253.27 -P 4000 -D tpch --sf 4
go-tpc tpch run -H 165.22.253.27 -P 4000 -D tpch --sf 4
go-tpc tpch clenup -H 165.22.253.27 -P 4000 -D tpch --sf 4
```
