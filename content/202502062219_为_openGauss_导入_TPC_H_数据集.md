+++
title = "为 openGauss 导入 TPC-H 数据集"
date = 2025-02-06 22:19:07
slug = "202502062219"

[taxonomies]
tags = ["openGauss", "数据库", "基准测试"]
+++

TPC-H 是一个决策支持基准测试。
它由一套面向业务的即席查询与并发数据修改组成。
其数据集与查询具有广泛的行业相关性。
这个基准测试模拟了一个决策支持系统，该系统会对大规模的数据执行高复杂度的查询以回答关键商务问题。

本文会记录将 TPC-H 数据集导入 openGauss 的过程。

<!-- more -->

## 构建 TPC-H 数据生成工具

官网链接为 <https://www.tpc.org/tpc_documents_current_versions/current_specifications5.asp>。
在其中点击形如 `Download TPC-H_Tools_v3.0.1.zip` 的超链接，然后填写邮箱等信息，就可以收到包含下载地址的邮件。

下载好之后解压：

```sh
unzip D12D3B85-DBE4-45B4-96D2-7DAAF4292C10-TPC-H-Tool.zip
```

会解压出一个 `TPC-H V3.0.1` 目录，把目录内的 `dbgen/makefile.suite` 文件拷贝为 `Makefile`：

```sh
cd 'TPC-H V3.0.1/dbgen'
cp makefile.suite Makefile
```

于是修改这个 `Makefile`，对一些变量进行赋值：

```mk
CC        = gcc
DATABASE  = ORACLE
MACHINE   = LINUX
WORKLOAD  = TPCH
```

直接构建：

```sh
make
```

构建完成后会得到 `dbgen` 与 `qgen`，分别用于生成数据集与查询语句。

## 生成数据集

使用 `dbgen` 生成数据：

```sh
./dbgen -s 1
```

其中 `-s` 参数用于指定缩放系数，会影响生成数据的规模。
这个缩放系数与生成出数据的 GB 数大致相等，比如 `-s 10` 大约会生成 10GB 的数据。
生成完成后，我们会得到八个 `.tbl` 文件：

```sh
$ ls *.tbl
customer.tbl  lineitem.tbl  nation.tbl  orders.tbl  partsupp.tbl  part.tbl  region.tbl  supplier.tbl
```

## 创建表

先建一个库：

```sql
CREATE DATABASE tpch;
```

然后在 `dbgen` 目录里面有一个 `dss.ddl` 文件，里面是创建表的语句，可以直接丢给 openGauss 执行：

```sh
gsql -d tpch -f dss.ddl
```

于是就可以在 `tpch` 这个库里面创建出八张表。

## 导入数据

这些 `.tbl` 文件中，每行数据都会以 `|` 结尾，无法被 openGauss 导入。
因此需要把行末的 `|` 去掉，顺带把后缀名改为 `.csv`：

```sh
for file in *.tbl; do
  sed 's/|$//' $file > ${file/tbl/csv}
done
```

于是用 `\copy` 元命令导入就可以了：

```sql
\copy customer FROM 'customer.csv' WITH csv DELIMITER '|'
\copy lineitem FROM 'lineitem.csv' WITH csv DELIMITER '|'
\copy nation FROM 'nation.csv' WITH csv DELIMITER '|'
\copy orders FROM 'orders.csv' WITH csv DELIMITER '|'
\copy partsupp FROM 'partsupp.csv' WITH csv DELIMITER '|'
\copy part FROM 'part.csv' WITH csv DELIMITER '|'
\copy region FROM 'region.csv' WITH csv DELIMITER '|'
\copy supplier FROM 'supplier.csv' WITH csv DELIMITER '|'
```
