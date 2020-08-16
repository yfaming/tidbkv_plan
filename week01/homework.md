# clone 代码

https://github.com/pingcap/pd

https://github.com/pingcap/tidb

https://github.com/tikv/tikv

pd 和 tidb 是用 Go 开发的，tikv 是用 Rust 开发的，build 之前需要安装好各自开发环境，不用细说。

PS: 今天偶然看到，pd 将转移到 https://github.com/tikv 下。
这也暴露了 Golang 包管理弱鸡之处，项目转移后，得把所有 import pd 的代码给改一遍。

# build
pd 和 tidb 的 repo 中都有 Makefile，直接执行 `make` 即可构建出各自的可执行文件。

pd 的可执行文件在 `bin` 目录中，包括 `pd-ctl`，`pd-recover` 和 `pd-server`。
tidb 的可执行文件也在 `bin` 目录中，为 `tidb-server`。

tikv 是 Rust 写的，自然用 Cargo 作为构建工具，执行 `cargo build` 即可。可执行文件是 `target/debug/tikv-ctl` 和 `target/debug/tikv-server`。
这里只构建了 debug 版，若要构建 release 版，则执行 `cargo build --release` 即可。

没想到的是，tikv 竟然还在使用 nightly Rust，简单搜了一下 TiKV 的 issue，没找到讨论这一问题的，暂略过。

构建过程，需要下载依赖，费时较长，而且有墙的干扰，网络需要很给力才行。
发现个人搭建的翻墙工具还是不太行，最后切换到公司 VPN 才好起来。

# 部署 TiDB 集群
部署要求是:
* 1 PD
* 1 TiDB
* 3 TiKV

因为要运行自己编译的可执行文件，因此最好不要用 [tiup](https://docs.pingcap.com/zh/tidb/dev/quick-start-with-tidb) 来部署了。

先把玩一下。pd, tidb, tikv 各自的 server，即使不给出任何命令行选项，也可正常运行的，可从日志中看到一些有意思的信息，也可通过 `-h` 了解了可用的选项。

另外，查到 https://docs.pingcap.com/tidb/v2.1/test-deployment-from-binary-tarball 有详细说明，据此执行即可。

```sh
127.0.0.1

# pd
./bin/pd-server --name=pd1 \
                --data-dir=pd \
                --client-urls="http://127.0.0.1:2379" \
                --peer-urls="http://127.0.0.1:2380" \
                --initial-cluster="pd1=http://127.0.0.1:2380" \
                --log-file=pd.log &

# 3 个 tikv
# 因为都是在开发机上运行，因此 IP 都用 127.0.0.1，而各个 tikv 实例的端口，data-dir, log-file 都需要不同
./target/debug/tikv-server --pd="127.0.0.1:2379" \
                  --addr="127.0.0.1:2021" \
                  --data-dir=tikv1 \
                  --log-file=tikv1.log &

./target/debug/tikv-server --pd="127.0.0.1:2379" \
                  --addr="127.0.0.1:2022" \
                  --data-dir=tikv2 \
                  --log-file=tikv2.log &


./target/debug/tikv-server --pd="127.0.0.1:2379" \
                  --addr="127.0.0.1:2023" \
                  --data-dir=tikv3 \
                  --log-file=tikv3.log &

# tidb
# 需要指定 store, 否则默认是 mocktikv
# 可不指定 log-file，这样就会把日志打到控制台，方便查看
./bin/tidb-server --store=tikv \
                  --path="127.0.0.1:2379" \
                  --log-file=tidb.log

# 访问, 注意 tidb 默认监听 4000 端口
# (Mac 没安装 mysql 官方 client。因此下面这个命令并没有实际验证过。用的是 Sequel Pro 这个 GUI 工具测试的。)
mysql -h 127.0.0.1 -P 4000 -u root -D test
```

# 改写代码
要求：使得 TiDB 启动事务时，能打印出一个 “hello transaction” 的日志。

TiDB 有 47w+ 行 Golang 代码，显然慢慢看是不太现实的。
而 `rg -i transaction` 又得到超过 1k 条结果，除掉 mock 和 test 相关的，仍然太多。

浏览代码结构，也没看到有顶级模块叫 transaction 的。

不过倒是发现了 session 和 sessionctx 这两者看起来很有希望。

打开 session/session.go，看到 `type Session interface` 的各种方法，看起来就在这里了。
但遗憾的是，虽然有 `CommitTxn()`, `RollbackTxn()` 却没有 `BeginTxn()`/`StartTxn()`。

挨个翻下去，最后终于发现 `PrepareTxnCtx()` 才是我们要找的方法。其文档也说明了这一点：

> // PrepareTxnCtx starts a goroutine to begin a transaction if needed, and creates a new transaction context.
>
> // It is called before we execute a sql query.

OK，加一行日志看看: `log.Info("[session/session.go] PrepareTxnCtx(): hello transaction!")`。

然后 `make` 并重新启动 TiDB server。可以看到会周期性地打印，如

```
[2020/08/16 23:35:47.029 +08:00] [INFO] [session.go:2185] ["[session/session.go] PrepareTxnCtx(): hello transaction!"]
[2020/08/16 23:35:47.064 +08:00] [INFO] [session.go:2185] ["[session/session.go] PrepareTxnCtx(): hello transaction!"]
[2020/08/16 23:35:47.064 +08:00] [INFO] [session.go:2185] ["[session/session.go] PrepareTxnCtx(): hello transaction!"]
[2020/08/16 23:35:47.066 +08:00] [INFO] [session.go:2185] ["[session/session.go] PrepareTxnCtx(): hello transaction!"]
```

通过客户端连接到 TiDB 然后执行一条 SQL，也可以看到有日志打印出来。(与 TiDB 周期性的日志混到了一起，需要通过打印时间来区分。)

另外，在尝试的过程中，也发现了一些看起来像但并非我们想要的方法，如 `NewTxn()`，`kv/kv.go` 的 `Transaction interface` 等。
