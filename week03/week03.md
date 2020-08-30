Week 03: 通过工具寻找 TiDB 瓶颈
===============================

@yfaming, 2020.08.30 周日

【High Performance TiDB】Lesson 03：通过工具寻找 TiDB 瓶颈
https://www.bilibili.com/video/BV1Pp4y1i7zu

# Performace & Bottleneck 漫谈
## 如何衡量性能
性能 = 产出 / 资源消耗
产出: 事务次数(QPS)，吞吐数据量(Bytes)
资源消耗: CPU 时间片，磁盘、网络 IO 次数，流量

在资源固定的情况下，提高产出:
* 每秒事务数: QPS
  * 如果独立作为产出的衡量单位，不够准确
  * 可通过 batch 方式，在 QPS 不变的情况下，提升系统产出
* 每秒吞吐: bytes/s

时间轴与对应的资源消耗

# Pipeline
目标: 时间轴折叠
## Pipeline 的 batch 粒度
* 更小的 batch 粒度
  * "batch 划分"消耗的资源相对更多
    * 锁竞争
    * batch 提交的代价
  * 小事务运行的资源占用更少，整体资源使用峰值更低
  * 失败代价更低
  * latency 有保障
* 更大的 batch 粒度
  * "batch 划分"消耗的资源更少
  * 整体占用资源多
  * 失败代价大
  * 首、尾 batch 不能从 pipeline 中受益
  * 通过可以带来更好的吞吐，但 latency 相对更高
* 平衡的艺术

## 谨慎决定 batch 粒度
例一: 7200 转的系统性 HDD 上进行文件拷贝
* HDD 寻道时间 3-6ms，随机读写最大 QPS 为 400-800 (这是 iops)
* HDD 读写带宽 100MB/s
* 低于 (100MB/800) ~= 100KB 的 batch 粒度，可能造成性能下降

相当于说，磁盘的 iops 是其固有属性。不同的磁盘，其 iops 是不同的。
而且读写带宽也是固有属性。

例二: 将数据划分小块，并发进行编码
* 使用 mutex 对数据状态(线程分配等信息)加锁
* 在高竞争状态，mutex 的 QPS 约为 10-20 万
* 该编码的总体吞吐不高于 10 万 * 块大小
* 若块大小为 4KB，则整体吞吐不高于 400MB/s

关键在于，此二例的计算方式，思考方式。


# 常见的瓶颈类型
* CPU
  * CPU usage 高
    * 减少计算开销: 更优的算法，减少重复计算(cache 中间结果), 更优的工程实现
  * CPU load 高 (应该就是指日常说的 "Load" 吧?)
    * 线程太多，频繁切换: 跨线程交互是否是必要的?
    * 等 IO, top 看到 iowait 比较高: IO 可能是限制性能的瓶颈

* IO
  * iops 上限
    * 减少读写的次数: 提高 cache hit ratio
  * IO bandwith 上限
    * 减少磁盘读写流量，更紧凑的数据存储格式，更小的读写放大(如只需要读写 100 字节，结果触发了好多个 page 的读写，从而产生放大)
  * fsync 次数上限
    * 减少 fsync 次数，合并提交 group commit

iops 和 io bandwith 是磁盘的物理特性。

* Network
  * bandwith 上限: 降低传输的数据量(计算下推，更紧凑的数据格式)
  * send/recv 系统调用太频繁: 指发送和接收

TiDB 中有 node exporter，暴露了系统的一些指标，值得关注之。

# CPU Profile 原理
在一段时间内对正在执行的程序进行大量的采样，每个采样点，是某个线程当前的栈信息，通过对采样的聚合统计分析，我们
可以得到各个函数的 CPU 消耗资源比例，然后针对性的优化。

采样时间越长，自然越准确。通过抓取 30s 就可以得到一个足够精度的结果。

# Profile TiDB CPU usage (1/3)
`curl http://{TiDBIP}:10080/debug/zip?seconds=60 --output debug.zip` 会把 tidb 性能问题所需的多个 profile 文件打包生成一个 zip 文件

解析后得到以下文件:
* config
* goroutine
* heap
* mutex
* profile
* version

# Profile TiDB CPU usage (2/3)
暂时只关心 profile 文件。

执行 `go tool pprof -http=:8080 debug/profile`，在打开的 web 界面上查看。

* 在 Graph 格式里，每个函数节点都是唯一的，有多个指向它的调用
* Graph 格式让我们比较容易发现开销大的子函数，指引我们对子函数本身进做优化。
* 在 Graph 格式里，函数节点除了函数名称，还包含了两个时间，前端的时间是函数自身消耗的 CPU 时间，后面的时间是「函数自身 + 调用的子函数」汇总的整体时间。
* 函数自身 CPU 时间越长，函数节点面积越大；函数整体 CPU 越长，颜色越深。
* 选择一个函数节点，颜色变紫色。此时选择 `Refine -> Focus`，可以让整个 graph 只展示和这个节点相关的 CPU 开销。

# Profile TiDB CPU usage (3/3)
* 在 FlameGraph 中，子函数的开销是分散在不同的父函数中的。
  对比：「Graph 格式让我们比较容易发现开销大的子函数，指引我们对子函数本身进做优化」。
* FlameGraph 格式会让我们从整体上看到各个线程的开销比例和子函数占用的比例，指引我们从整体上找到优化的优先级。
* Graph 和 FlameGraph 都可以方便的让我们找到函数调用关系，指导我们阅读代码。


# Profile TiKV CPU usage (1/3)
* 方式一: TiDB Dashboard -> 高级调试 -> 实例性能分析 -> 浏览器查看 svg 文件。
* 方式二: perf 命令
```sh
git clone https://github.com/pingcap/tidb-inspect-tools
cd tracing_tools/perf

# FlameGraph
sudo sh ./cpu_tikv.sh $tikv_pid
# Graph
sudo sh ./dot_tikv.sh $tikv_pid
```


# IO
grafana 中有个叫做 Disk-Performance 的 Dashboard。

# iostat
`iostat -x 1 -m`
Mac 下的 iostat 既没有 `-x` 也没有 `-m` 选项。。。
Ubuntu 下，iostat 命令是由 sysstat 包提供的，其中还包含 cifsiostat, iostat, mpstat, pidstat, sadf, sar.sysstat, tapestat 等命令。

```txt
vagrant@vagrant:~$ iostat -x 1 -m
Linux 4.15.0-91-generic (vagrant) 	08/30/2020 	_x86_64_	(2 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           5.18    0.18    3.40    1.47    0.00   89.76

Device            r/s     w/s     rMB/s     wMB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
loop0            0.02    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     1.60     0.00   0.00   0.00
sda             72.67    9.27      2.15      0.46     7.34    23.90   9.17  72.05    1.31    4.67   0.14    30.30    51.07   0.45   3.69
dm-0            78.86   32.38      2.13      0.58     0.00     0.00   0.00   0.00    2.45    6.50   0.40    27.60    18.22   0.33   3.68
dm-1             0.73    0.00      0.02      0.00     0.00     0.00   0.00   0.00    0.63    0.00   0.00    22.20     0.00   0.63   0.05
```

几个注意指标
* await: 平均每次设置 IO 操作的等待时间(ms)
* r_await/w_await: 读/写操作等待时间
  如果此值比较高(如超过 5ms)，说明磁盘压力比较大
* 可以看 io size, io 连续性，读写瞬时流量、读写分别的 iops

# iotop
iostat 是从磁盘的角度来看 io，而 iotop 是从进程角度看 io。
* iotop 用于看各个线程的 io 累积量，有没有超出预期，顺便作为 fsync 不足的佐证(jdb2 流量超大)
  jdb2 是个内核线程。
* `sudo iotop -o`
  `-o` 只显示有 io 输出的进程。

# iosnoop
http://www.brendangregg.com/blog/2014-07-16/iosnoop-for-linux.html
可检查磁盘延迟毛刺

# 其他 IO 工具
[fio](https://github.com/axboe/fio)
fio 可用于测试我们关注的三个磁盘指标: 读写带宽、iops、fsync/秒，另外还可获取延迟分布。
pg_test_fsync



# Profile TiDB Memory
* 查看 in-use 内存
  * `go tool pprof -http=:8080 debug/heap`
  * 从 heap profile 图里，我们可以看 in-use 的内存，都是由哪些函数分配出来的
* 查看历史上 alloc 的 space
  `go tool pprof -alloc_space -http=:8080 debug/heap`
  指定 `-alloc_space`，可以看到总共被分配过的内存，即使已经释放了。
* 查看 mutex 竞争情况
  `go tool pprof -contentions -http=:8080 debug/mutex`

# Profile TiKV memeory
方式一: perf
```
git clone https://github.com/pingcap/tidb-inspect-tools
cd tracing_tools/perf
sudo sh mem.sh $tikv_pid
```
针对 TiKV 需要做一些事情，`sudo perf probe -x {tikv-binary} -a malloc`，然后用 `perf record -e probe_tikv:malloc -F 99 -p $1 -g -- sleep 10` 替代 mem.sh 的第一条命令。

方式二: bcc
Linux 4.9 以上版本
```sh
sudo /usr/share/bcc/tools/stackcount -p $tikv-pid -U $tikv-binary-path:malloc > out.stacks
./stackcollapse.pl < out.stacks | ./flamegraph.pl --color=mem --title="malloc() Flame Graph" --countname="calls" > out.svg
```
[Memory Leak (and Growth) Flame Graphs](http://www.brendangregg.com/FlameGraphs/memoryflamegraphs.html)

方式三: jemalloc 统计信息
```
tiup ctl tikv --host=$tikv-ip:$tikv-status-port metrics --tag=jemalloc
tiup ctl:nightly tikv --host=$tikv-ip:$tikv-status-port metrics --tag=jemalloc
```

# VTune
略。


# 总结一下
由于 Go 内置有 pprof，因此 TiDB 的 profiling 比较方便。
而 TiKV 是 Rust 开发的，需要其他 profiling 工具，如 perf。其中 https://github.com/pingcap/tidb-inspect-tools
包装出了一些脚本，以方便使用。

profile 涉及 CPU, IO, memory, network 等方面。各方面关注的点，及可用的工具，总结如下 ：

CPU: pprof, perf
* cpu usage
* cpu load
go pprof web 界面中，graph 与 flame graph 的特点注意之

IO: iostat, iotop, iosnoop, fio
* iops
* io bandwith
* fsync

Memory: pprof, perf, bcc, jemalloc

Network: 未提相关工具
* bandwith
* send/recv
