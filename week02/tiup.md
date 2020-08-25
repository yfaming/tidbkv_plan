TiUP
====

@yfaming, 2020.08.22 周六

https://tiup.io/

https://github.com/pingcap/tiup

TiUP is a component manager for TiDB.

# 安装

```sh
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
```

安装到 `~/.tiup` 目录下，可执行文件的路径具体为 `~/.tiup/bin/tiup`。并在 shell 的 profile 中添加一行，以将 tiup 放到 `PATH` 中。

如使用 zsh，则向 `.zshrc` 中添加了一行：
```zsh
export PATH=/Users/yfaming/.tiup/bin:$PATH
```

文档见 [使用 TiUP 部署 TiDB 集群](https://docs.pingcap.com/zh/tidb/stable/production-deployment-using-tiup)。


`tiup -h` 的输出：

```txt
TiUP is a command-line component management tool that can help to download and install
TiDB platform components to the local system. You can run a specific version of a component via
"tiup <component>[:version]". If no version number is specified, the latest version installed
locally will be used. If the specified component does not have any version installed locally,
the latest stable version will be downloaded from the repository.

Usage:
  tiup [flags] <command> [args...]
  tiup [flags] <component> [args...]

Available Commands:
  install     Install a specific version of a component
  list        List the available TiDB components or versions
  uninstall   Uninstall components or versions of a component
  update      Update tiup components to the latest version
  status      List the status of instantiated components
  clean       Clean the data of instantiated components
  mirror      Manage a repository mirror for TiUP components
  telemetry   Controls things about telemetry
  completion  Output shell completion code for the specified shell (bash or zsh)
  help        Help about any command or component

Components Manifest:
  use "tiup list" to fetch the latest components manifest

Flags:
  -B, --binary <component>[:version]   Print binary path of a specific version of a component <component>[:version]
                                       and the latest version installed will be selected if no version specified
      --binpath string                 Specify the binary path of component instance
      --help                           Help for this command
      --skip-version-check             Skip the strict version check, by default a version must be a valid SemVer string
  -T, --tag string                     Specify a tag for component instance
  -v, --version                        Print the version of tiup

Component instances with the same "tag" will share a data directory ($TIUP_HOME/data/$tag):
  $ tiup --tag mycluster playground

Examples:
  $ tiup playground                    # Quick start
  $ tiup playground nightly            # Start a playground with the latest nightly version
  $ tiup install <component>[:version] # Install a component of specific version
  $ tiup update --all                  # Update all installed components to the latest version
  $ tiup update --nightly              # Update all installed components to the nightly version
  $ tiup update --self                 # Update the "tiup" to the latest version
  $ tiup list                          # Fetch the latest supported components list
  $ tiup status                        # Display all running/terminated instances
  $ tiup clean <name>                  # Clean the data of running/terminated instance (Kill process if it's running)
  $ tiup clean --all                   # Clean the data of all running/terminated instances

Use "tiup [command] --help" for more information about a command.
```

components 的概念，跟 rustup 有点像？
components 安装在 ~/.tiup/components 目录下。

# playground
利用本地 Mac 或者单机 Linux 环境快速部署 TiDB 集群。可以体验 TiDB 集群的基本架构，以及 TiDB、TiKV、PD、监控等基础组件的运行。
简单地说，playground 就是让你体验一下的。所有组件 pd, tidb, tikv, tiflash 都运行在本机上。

```sh
# 以最新的版本，启动 pd, tikv, tidb, tiflash 各一个实例。并且启动 prometheus 和 grafana。
tiup playground
# 指定版本和实例数
tiup playground v4.0.0 --pd 3 --db 2 --kv 3 --tiflash 0

# 展示实例列表
tiup playground display

# 通过 pid 指定组件，并扩容/缩容(只增减一个实例)
tiup playground scal-in --pid 71137
tiup playground scal-out --pid 71137
```

使用 playground 时，pd/tidb/tikv 各自的配置文件是怎样的？tikv 将数据保存在何处？不清楚。

# cluster
