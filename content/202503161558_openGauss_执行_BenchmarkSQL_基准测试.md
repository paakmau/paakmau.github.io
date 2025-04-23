+++
title = "openGauss 执行 BenchmarkSQL 基准测试"
date = 2025-03-16 15:58:34
slug = "202503161558"

[taxonomies]
tags = ["openGauss", "BenchmarkSQL"]
+++

BenchmarkSQL 是一种类似 TPC-C 的测试工具。
TPC-C 是一种 OLTP 的基准测试，通过并发事务测试数据库的性能，衡量的标准是每分钟事务数（tpmC）。

<!-- more -->

<https://github.com/pgsql-io/benchmarksql>

<https://www.tpc.org/tpcc>

## 工作路径

```sh
export BMSQL_WORKSPACE=${HOME}/BenchmarkSQL
```

## 安装 BenchmarkSQL

### 构建

原本的项目已经不维护了：

<https://github.com/petergeoghegan/benchmarksql>

本文使用的是 fork 出来的项目：

<https://github.com/pgsql-io/benchmarksql>

版本的话，使用 `REL5_1` 的 tag。

```sh
cd ${BMSQL_WORKSPACE}
git clone https://github.com/pgsql-io/benchmarksql.git
cd benchmarksql
git checkout REL5_1

ant
```

然后 `BenchmarkSQL-5.1.jar` 就会被生成在 `${BMSQL_WORKSPACE}/benchmarksql/dist` 目录中。

### 配置 BenchmarkSQL

```sh
cd ${BMSQL_WORKSPACE}/benchmarksql/run
cp sample.postgresql.properties og.properties
```

然后编辑这个 `og.properties`，配置一下连接、驱动、基准测试参数等。
这里提供的示例参数仅用于跑通，需要视实际情况调整：

```properties
db=postgres
driver=org.opengauss.Driver
conn=jdbc:opengauss://localhost:2333/benchmarksql?useUnicode=true&characterEncoding=utf-8
user=benchmarksql
password=<password>

// 数据规模，每个 warehouse 占用约 100MB 的初始数据，一般配置为内存的 2-5 倍
warehouses=10
// 导入数据的工作线程数，不影响基准测试的执行，配置依据是服务器核心数与 I/O 带宽
loadWorkers=4

// 并发量，一般配置为 CPU 核心数的 2-6 倍
terminals=10

// 为每个客户端指定需要执行的事务数。
// 如果配置该参数，`runMins` 必须配成 0。
runTxnsPerTerminal=0

// 指定执行的分钟数，一般配置为数小时甚至数天。
// 因为本测试构造的数据规模较大，需要执行足够长的时间才能使数据库达到稳定状态。
// 这样才能确保 checkpoint 与 vacuum 等性能相关的功能都被测量到。
// 如果配置该参数，`runTxnsPerTerminal` 必须配成 0。
runMins=2

// 每分钟的总事务数限制
limitTxnsPerMin=10000000

// Set to true to run in 4.x compatible mode. Set to false to use the
// entire configured database evenly.
terminalWarehouseFixed=false

// 使用存储过程执行事务，当对比不同数据库性能时不推荐开启。
// 因为存储过程的实现存在差异，可能影响性能，应该使用最通用的事务语句。
// 不过该参数可用于分析存储过程节省的网络 I/O。
useStoredProcedures=false

// 以下五个取值的总和必须为 100。
// 内部默认的比例与 TPC-C 标准中描述的 23 张牌实现相符合。
// 以下的取值与默认值相同：
newOrderWeight=45
paymentWeight=43
orderStatusWeight=4
deliveryWeight=4
stockLevelWeight=4

// Directory name to create for collecting detailed result data.
// Comment this out to suppress.
//resultDirectory=my_result_%tY-%tm-%td_%tH%tM%tS
//osCollectorScript=./misc/os_collector_linux.py
//osCollectorInterval=1
//osCollectorSSHAddr=user@dbhost
//osCollectorDevices=net_eth0 blk_sda
```

### 导入 openGauss JDBC

<https://opengauss.org/zh/download>

下载 6.0.1 版本的。

```sh
cd ${BMSQL_WORKSPACE}
mkdir pkg && cd pkg
wget https://opengauss.obs.cn-south-1.myhuaweicloud.com/6.0.1/openEuler22.03/arm/openGauss-JDBC-6.0.1.tar.gz
tar -zxf openGauss-JDBC-6.0.1.tar.gz
mv opengauss-jdbc-6.0.1.jar ${BMSQL_WORKSPACE}/benchmarksql/lib

cd ${BMSQL_WORKSPACE}
rm -rf pkg
```

## 部署 openGauss

### 安装 openGauss

环境变量：

```sh
export OG_WORKSPACE=${HOME}/og

export GAUSSHOME=${OG_WORKSPACE}/install
export PGDATA=${OG_WORKSPACE}/data
export PGPORT=2333
export PGDATABASE=postgres
export LD_LIBRARY_PATH=${GAUSSHOME}/lib:${LD_LIBRARY_PATH}
export PATH=${GAUSSHOME}/bin:${PATH}
```

<https://opengauss.org/zh/download>

下载 6.0.1 企业版，下载完之后解压到 `${GAUSSHOME}` 目录中：

```sh
cd ${OG_WORKSPACE}
mkdir pkg && cd pkg

wget https://opengauss.obs.cn-south-1.myhuaweicloud.com/6.0.1/openEuler22.03/arm/openGauss-All-6.0.1-openEuler22.03-aarch64.tar.gz
tar -zxf openGauss-All-6.0.1-openEuler22.03-aarch64.tar.gz

mkdir ${GAUSSHOME}
tar -C ${GAUSSHOME} -jxf openGauss-Server-6.0.1-openEuler22.03-aarch64.tar.bz2

cd ${OG_WORKSPACE}
rm -rf pkg
```

### 准备数据库节点

```sh
gs_initdb --nodename=single_node -w <password>

gs_ctl start
```

### 创建数据库与用户

```sh
gsql -r
```

```sql
CREATE USER benchmarksql SYSADMIN PASSWORD '<password>';

CREATE DATABASE benchmarksql;
```

## 装载数据集

```sh
cd ${BMSQL_WORKSPACE}/benchmarksql/run
./runDatabaseBuild.sh og.properties
```

这个脚本包含建表、创建存储过程、导入数据、建索引、建立外键约束等。

## 执行基准测试

```sh
cd ${BMSQL_WORKSPACE}/benchmarksql/run
./runBenchmark.sh og.properties
```

执行完成后输出大概长这样，指标的话要看 `Measured tpmC (NewOrders)` 的那行：

```txt
Term-00, Running Average tpmTOTAL: 98607.17    Current tpmTOTAL: 1306284    Memory Usage: 66MB / 1550MB
11:38:09,553 [Thread-7] INFO   jTPCC : Term-00,
11:38:09,554 [Thread-7] INFO   jTPCC : Term-00,
11:38:09,554 [Thread-7] INFO   jTPCC : Term-00, Measured tpmC (NewOrders) = 44511.25
11:38:09,554 [Thread-7] INFO   jTPCC : Term-00, Measured tpmTOTAL = 98585.38
11:38:09,554 [Thread-7] INFO   jTPCC : Term-00, Session Start     = 2025-03-18 11:36:09
11:38:09,554 [Thread-7] INFO   jTPCC : Term-00, Session End       = 2025-03-18 11:38:09
11:38:09,554 [Thread-7] INFO   jTPCC : Term-00, Transaction Count = 197224
```

观察到 BenchmarkSQL 执行输出过程中会不断地刷新当前的指标，这实际上是通过往 stdout 输出退格符（在同一行中）实现的。
退格符在 C/C++ 中是 `\b`，ASCII 码为 0x08。
当我们尝试将这些输出重定向到文件时，就会发现这些退格符也被写入文件了。
于是可以使用 `tr` 命令，把连续的退格符替换为换行符（`\n`）：

```sh
./runBenchmark.sh og.properties \
  | tr -s '\b' '\n'
```

但这些不断刷新的指标太多了，我们可以用 `awk` 命令控制每五行仅输出一行。
这里注意一下 `tr` 命令的输出，需要使用 `stdbuf` 将 stdout 缓冲模式修改为逐行缓冲。

```sh
./runBenchmark.sh og.properties \
  | stdbuf -oL tr -s '\b' '\n' \
  | awk '/^Term-/{ if (++i % 5 == 1) print; next } 1'
```

最后使用 `tee` 命令，在写入文件的同时输出到 stdout 中，同样需要注意缓冲模式：

```sh
./runBenchmark.sh og.properties \
  | stdbuf -oL tr -s '\b' '\n' \
  | stdbuf -oL awk '/^Term-/{ if (++i % 5 == 1) print; next } 1' \
  | tee bmsql.out
```

## 清除数据集

这个脚本会删除掉装载数据集时创建的表和存储过程。
它写的有一点小问题，依据报错改一下 `sql.postgres/storedProcedureDrops.sql` 文件就行。

```sh
cd ${BMSQL_WORKSPACE}/benchmarksql/run
./runDatabaseDestroy.sh og.properties
```
