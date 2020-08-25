Week 02: 对 TiDB 进行基准测试
============================

@yfaming, 2020.08.22 周六

【High Performance TiDB】Lesson 02：对 TiDB 进行基准测试
https://www.bilibili.com/video/BV1TD4y1m7AF/

# Deploy
* 测试 TiDB 集群的性能最好选取不低于三台物理机/虚拟机节点部署 TiKV
* TiDB 与 TiKV 不要部署在同一节点上
* 单节点部署 TiKV 时，CPU 最好在 16-24 之间。如果一台机器的 CPU 资源较多，可考虑部署 2 个 TiKV 节点。
  TiKV 需要一部分内存用于磁盘的缓存，建议内存总大小不要低于数据总量的 10%
* 单节点部署了多个 TiKV 实例时，需要调整一些线程池朽比。见：
  https://github.com/pingcap-incubator/tidb-in-action/blob/master/session4/chapter8/threadpool-optimize.md

但没提 pd 与 tidb？


# TiUP

https://tiup.io/

TiUP is a component manager for TiDB.

安装：

```sh
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
```

安装到 `~/.tiup` 目录下，可执行文件的路径具体为 `~/.tiup/bin/tiup`。并在 shell 的 profile 中添加一行，以将 tiup 放到 `PATH` 中。

如使用 zsh，则向 `.zshrc` 中添加了一行：
```zsh
export PATH=/Users/yfaming/.tiup/bin:$PATH
```

文档见 [使用 TiUP 部署 TiDB 集群](https://docs.pingcap.com/zh/tidb/stable/production-deployment-using-tiup)。

使用 tiup 的话，这个文档一定读一下，并试验之。

视频中显示，创建一个数据库，竟然花了 1 min 18.84 sec。。


# sysbench
https://github.com/akopytov/sysbench

最流行的数据库测试工具。lua 编写，是比较简单的测试数据库负载的程序。
(github 页面显示，C 占 42%，Perl 占 24.6%，lua 只占 8.7%)

主要包括随机点查询(point select)，简单的点更新(point update)和范围查询。

用户可以根据自己的业务场景，改写 sysbench 的 .lua 脚本来进行测试。

Mac 安装：`brew install sysbench`

```
==> Installing dependencies for sysbench: mysql-client
==> Installing sysbench dependency: mysql-client
==> Pouring mysql-client-8.0.21.catalina.bottle.tar.gz
==> Caveats
mysql-client is keg-only, which means it was not symlinked into /usr/local,
because it conflicts with mysql (which contains client libraries).

If you need to have mysql-client first in your PATH run:
  echo 'export PATH="/usr/local/opt/mysql-client/bin:$PATH"' >> ~/.zshrc

For compilers to find mysql-client you may need to set:
  export LDFLAGS="-L/usr/local/opt/mysql-client/lib"
  export CPPFLAGS="-I/usr/local/opt/mysql-client/include"

For pkg-config to find mysql-client you may need to set:
  export PKG_CONFIG_PATH="/usr/local/opt/mysql-client/lib/pkgconfig"
```

Ubuntu 安装
```
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench
```


## Run
常见测试
* Point select 测试
`sysbench --config-file=config oltp_point_select --threads=128 --tables=32 --table-size=5000000 run`

* Update index 测试
`sysbench --config-file=config oltp_update_index --threads=128 --tables=32 --table-size=5000000 run`

* Read-only 测试
`sysbench --config-file=config oltp_read_only --threads=128 --tables=32 --table-size=5000000 run`


# go-ycsb
是 PingCAP 在雅虎开源的 ycsb 基础上用 golang 开发的数据库性能测试工具。

https://github.com/pingcap/go-ycsb

ycsb 可通过调用 workloads 下的各个配置文件夹
* 调整 requestdistribution 以更改随机分布的方式，默认是 zipfan
* 调整fieldcount 以修改 column 的数量。默认是 10
* 更多配置见 https://github.com/brianfrankcooper/YCSB/wiki/Core-Properties

# go-tpc
包括 tpc-c 和 tpc-h 两部分。

* tpc-c 是专门针对联机效果处理系统(OLTP 系统)的规范，一般也将之称为业务处理系统。1992.07 月发布
* tpc-h 是国际上认定的分析性数据库测试的标准。
* PingCAP 用 golang 编写了 tpc-c 和 tpc-h 的测试程序，见 https://github.com/pingcap/go-tpc


# Homework

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
