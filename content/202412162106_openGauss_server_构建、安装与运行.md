+++
title = "openGauss-server 构建、安装与运行"
date = 2024-12-16 21:06:18
slug = "202412162106"

[taxonomies]
tags = ["openGauss", "数据库"]
+++

`openGauss` 是一个开源的关系型数据库，兼容多个主流数据库的语法。
本文以 `master` 分支为例，记录 `openGauss-server` 的构建、安装与运行。
参考的是官方文档：<https://gitee.com/opengauss/openGauss-server#%E4%BD%BF%E7%94%A8%E5%91%BD%E4%BB%A4%E7%BC%96%E8%AF%91%E4%BB%A3%E7%A0%81>。

<!-- more -->

## 环境

可以通过一些命令查询 Linux 发行版信息与 CPU 架构：

```sh
# 查询 CPU 架构
arch
uname -a

# 查询发行版信息
distro
cat /etc/system-release
```

本文使用的环境是：

- `x86_64`
- `openEuler` 20.03 (LTS)

## 工作路径

为了便于描述，本文的操作都会在一个工作路径中执行。
所以直接给这个路径配一个环境变量 `OG_WORKSPACE`：

```sh
export OG_WORKSPACE=$HOME/og
```

## 下载第三方库

参考 `openGauss-server` 仓中 `README` 的说明，我们需要依据上文中查询出的 CPU 架构与发行版信息，下载对应的预编译第三方库依赖包。
下载好了之后解压，再把解压出来的目录重命名为 `binarylibs`。

```sh
wget https://opengauss.obs.cn-south-1.myhuaweicloud.com/latest/binarylibs/gcc10.3/openGauss-third_party_binarylibs_openEuler_x86_64.tar.gz
tar -zxf openGauss-third_party_binarylibs_openEuler_x86_64.tar.gz
mv openGauss-third_party_binarylibs_openEuler_x86_64 binarylibs
```
