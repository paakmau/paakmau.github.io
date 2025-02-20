+++
title = "openGauss 的 NUMA-aware 部署"
date = 2025-02-20 22:37:50
slug = "202502202237"

[taxonomies]
tags = ["openGauss", "数据库", "NUMA"]
+++

openGauss 针对鲲鹏 NUMA 架构的设备做了一些优化，能在发挥多核并行优势的同时减少跨 NUMA 节点访存的问题。
本文记录以 NUMA-aware 方式部署 openGauss 的过程。

<!-- more -->

## 环境

两路鲲鹏 920 7260，每路 64 核；四个 NUMA 节点，每节点 32 核：

```sh
$ lscpu | grep NUMA
NUMA node(s):                       4
NUMA node0 CPU(s):                  0-31
NUMA node1 CPU(s):                  32-63
NUMA node2 CPU(s):                  64-95
NUMA node3 CPU(s):                  96-127
```

系统发行版：

```sh
$ distro
Name: openEuler 22.03 (LTS-SP4)
Version: 22.03 (LTS-SP4)
Codename: LTS-SP4
```

## 为 WAL writer 线程选择 CPU 核心

首先检查 `pg_xlog` 目录是否为软链接：

```sh
cd ${PGDATA}
file pg_xlog
```

随后查询目录挂载磁盘信息：

```sh
df -h
```

本文环境中查询到的挂载设备为 `/dev/nvme1n1`。
于是去查询该磁盘对应的 NUMA 节点：

```sh
cat /sys/class/nvme/nvme1/device/numa_node
```

这里我查询到的 NUMA 节点编号为 2。
接下来查询该 NUMA 节点对应的 CPU 核心编号范围：

```sh
lscpu
```

这里查询出来节点 2 中 CPU 核心编号为 `NUMA node2 CPU(s): 64-95`。
一般选择编号最小的核心就行，这里我们选择 64。
最后配置 GUC 参数将 WAL writer 线程绑定到 64 号 CPU 上：

```conf
walwriter_cpu_bind = 64
```

## 为线程池选择 CPU 核心范围

本文所用环境具有四个 NUMA 节点，每节点 32 个核心。
WAL writer 已经独占了一个核心（64）。
由于客户端一般与数据库分离部署，因此考虑每节点保留 3 个核心用于处理网络中断。
于是选择 0-28、32-60、65-92、96-124 总共 115 个核心用于线程池。
考虑为每个核心分配 3 个线程，于是线程池容量为 115 * 3 = 345。
线程分组数与 NUMA 节点数量保持一致，配置为 4。

最后配置 GUC 参数：

```conf
enable_thread_pool = on
thread_pool_attr = '345, 4, (cpubind: 0-28, 32-60, 65-92, 96-124)'
```

这里仅提供一个绑核思路，具体可依据实际情况（如并发量、客户端部署方式等）进行调整。

## 通过 `numactl` 启动数据库

通过 `numactl` 启动 openGauss 的目的是为主线程绑核，不会影响线程池的绑核。
主线程只需要避开 WAL writer 对应的核心就行，建议与线程池绑核保持一致：

```sh
numactl -C 0-28,32-60,65-92,96-124 gs_ctl start -D ${PGDATA}
```
