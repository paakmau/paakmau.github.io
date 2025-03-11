+++
title = "openGauss 执行 HyBench 基准测试"
date = 2025-03-11 23:08:40
slug = "202503112308"

[taxonomies]
tags = ["openGauss", "HyBench"]
+++

HyBench 是一个 HTAP 数据库基准测试，用于评估数据库在事务与分析混合处理场景下的性能。

<https://benchmark.tempos.cn/beachmarkDesc-htap.html>

<!-- more -->

## 总体流程

- 生成数据集

    八个 `.csv` 文件，分别对应八张表。
    可通过配置文件控制数据集的规模。

- 装载数据集

    HyBench 提供了一些语句模板，用于创建这八张表及其索引。
    表结构与索引是允许修改的，因此可以视情况进行适当修改。
    建表完成后，直接用 `COPY` 命令导入数据就行。

- 配置与调优

    在执行负载前允许修改数据库与系统配置，也允许通过 `ANALYZE` 等方式采集统计信息。
    但执行负载预热是不允许的。

- 执行负载

    负载总共 2 小时，分为三个阶段：OLAP、OLTP、OLXP，时间比例为 3:3:4。
    在不改变语义的情况下，允许对负载语句进行修改。
    通常的优化方法是为语句添加查询计划提示。

    - OLAP：复杂的只读查询，语句主要包含多表连接、聚合函数等。
    - OLTP：简单事务查询，主要是增删改查。
    - OLXP：并发地执行 OLAP 与 OLTP 负载

## 工作路径

为方便描述，本文把工作路径配置成环境变量：

```sh
export HY_WORKSPACE=${HOME}/hybench
```

## 安装 HyBench

### 下载 JRE

本文使用的是毕昇 JDK 17。
因为 HyBench 是直接把 `.jar` 包塞仓库里的，不需要编译，所以下载 JRE 就行。

<https://www.hikunpeng.com/developer/devkit/download/jdk>

```sh
cd ${HY_WORKSPACE}
wget https://mirrors.huaweicloud.com/kunpeng/archive/compiler/bisheng_jdk/bisheng-jre-17.0.14-b12-linux-aarch64.tar.gz
tar -zxf bisheng-jre-17.0.14-b12-linux-aarch64.tar.gz
rm bisheng-jre-17.0.14-b12-linux-aarch64.tar.gz
```

于是 JRE 就会被解压到 `${HY_WORKSPACE}/bisheng-jre-17.0.14` 目录中。

### 获取 HyBench

<https://benchmark.tempos.cn/downloadTool.html>

<https://gitee.com/cstc2023/hybench>

```sh
cd ${HY_WORKSPACE}
git clone https://gitee.com/cstc2023/hybench.git
```

### 配置 HyBench

首先修改配置文件 `${HY_WORKSPACE}/hybench/conf/og.props`，主要包括：

- 数据库连接配置

    包括 JDBC 驱动类名、用户名、密码、URL 等

- 数据规模

    缩放系数，默认是 `1x`。
    配置为 `1x` 时，生成的数据集文件总共约 500MB，其中最大的 `transfer.csv` 含有 6000000 条记录。

- 并发量

    HyBench 在执行负载时会多线程并发访问数据库执行随机语句。
    可为各个负载阶段分别配置并发量。

- 运行时间

    可对各个负载阶段分别配置执行时间。

```conf
db=openGauss
classname=org.opengauss.Driver
username=hybench
password=<password>
url=jdbc:opengauss://localhost:2333/hybench_1x?useUnicode=true&characterEncoding=utf-8
url_ap=jdbc:opengauss://localhost:2333/hybench_1x?useUnicode=true&characterEncoding=utf-8
classname_ap=org.opengauss.Driver
username_ap=hybench
password_ap=<password>

# 数据规模，可配置为 `1x` 或 `10x`
sf=1x

at1_percent=35
at2_percent=25
at3_percent=15
at4_percent=15
at5_percent=7
at6_percent=3

# AP 并发量
apclient=1

# TP 并发量
tpclient=1

fresh_interval=20

contention_num=100

# 运行时间
apRunMins=36
tpRunMins=36
xpRunMins=48

# XP 并发量
xtpclient=1
xapclient=1

apround=1
```

然后修改 `${HY_WORKSPACE}/hybench/hybench` 文件，依据文件里的提示，删除含有 `exit -1` 的第一行，并配置 `${JAVA_HOME}`：

```sh
export JAVA_HOME=${HY_WORKSPACE}/bisheng-jre-17.0.14
```

最后为该文件添加可执行权限：

```sh
chmod +x ${HY_WORKSPACE}/hybench/hybench
```

### 导入 openGauss JDBC

HyBench 通过 JDBC 访问数据库，JDBC 驱动的类名已经由上文中的 `og.props` 指定好了。
观察 `${HY_WORKSPACE}/hybench/hybench` 发现 HyBench 在执行时会加载 `${HY_WORKSPACE}/hybench/lib` 中的所有 `.jar` 包。
所以只需要把 openGauss JDBC 驱动移动到这个目录里就行。

<https://opengauss.org/zh/download>

这里的驱动下载 6.0.1 版本。

```sh
cd ${HY_WORKSPACE}
mkdir pkg && cd pkg

wget https://opengauss.obs.cn-south-1.myhuaweicloud.com/6.0.1/openEuler22.03/arm/openGauss-JDBC-6.0.1.tar.gz
tar -zxf openGauss-JDBC-6.0.1.tar.gz
mv opengauss-jdbc-6.0.1.jar ${HY_WORKSPACE}/hybench/lib

cd ${HY_WORKSPACE}
rm -rf pkg
```

## 部署 openGauss

### 安装 openGauss

为了方便描述，同样将 openGauss 的工作路径配置到环境变量中：

```sh
export OG_WORKSPACE=${HOME}/og
```

继续配置环境变量：

```sh
export GAUSSHOME=${OG_WORKSPACE}/install
export PGDATA=${OG_WORKSPACE}/data
export PGPORT=2333
export PGDATABASE=postgres
export LD_LIBRARY_PATH=${GAUSSHOME}/lib:${LD_LIBRARY_PATH}
export PATH=${GAUSSHOME}/bin:${PATH}
```

<https://opengauss.org/zh/download>

这里我下载的是 6.0.1 企业版，下载完之后解压到 `${GAUSSHOME}` 目录中。

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
```

数据库节点会被生成在 `${PGDATA}` 目录中，于是启动数据库：

```sh
gs_ctl start
```

### 创建数据库与用户

连接数据库：

```sh
gsql -r
```

为 HyBench 单独创建数据库与用户：

```sql
CREATE DATABASE hybench_1x;

CREATE USER hybench PASSWORD '<pasword>';
GRANT ALL PRIVILEGES ON DATABASE hybench_1x TO hybench;

\c hybench_1x
GRANT ALL PRIVILEGES ON SCHEMA public TO hybench;
```

## 装载数据集

### 建表

这里直接使用 HyBench 自带的建表脚本就行：

```sh
cd ${HY_WORKSPACE}/hybench
./hybench -t sql -f conf/ddl_og.sql -c conf/og.props
```

### 生成数据集

```sh
cd ${HY_WORKSPACE}/hybench
./hybench -t gendata -c conf/og.props
```

数据集是八个 `.csv` 文件，分别对应八张表，会被生成在 `${HY_WORKSPACE}/hybench/Data_1x` 目录内。

### 导入数据集

```sh
gsql -d hybench_1x -c "COPY checking FROM '${HY_WORKSPACE}/hybench/Data_1x/checking.csv' WITH (FORMAT 'text', DELIMITER ',');"
gsql -d hybench_1x -c "COPY checkingaccount FROM '${HY_WORKSPACE}/hybench/Data_1x/checkingAccount.csv' WITH (FORMAT 'text', DELIMITER ',');"
gsql -d hybench_1x -c "COPY company FROM '${HY_WORKSPACE}/hybench/Data_1x/company.csv' WITH (FORMAT 'text', DELIMITER ',');"
gsql -d hybench_1x -c "COPY customer FROM '${HY_WORKSPACE}/hybench/Data_1x/customer.csv' WITH (FORMAT 'text', DELIMITER ',');"
gsql -d hybench_1x -c "COPY loanapps FROM '${HY_WORKSPACE}/hybench/Data_1x/loanApps.csv' WITH (FORMAT 'text', DELIMITER ',');"
gsql -d hybench_1x -c "COPY loantrans FROM '${HY_WORKSPACE}/hybench/Data_1x/loanTrans.csv' WITH (FORMAT 'text', DELIMITER ',');"
gsql -d hybench_1x -c "COPY savingaccount FROM '${HY_WORKSPACE}/hybench/Data_1x/savingAccount.csv' WITH (FORMAT 'text', DELIMITER ',');"
gsql -d hybench_1x -c "COPY transfer FROM '${HY_WORKSPACE}/hybench/Data_1x/transfer.csv' WITH (FORMAT 'text', DELIMITER ',');"
```

### 建索引

一般习惯是导入数据之后再建索引，能节约一些时间。
这里同样使用 HyBench 自带的脚本：

```sh
cd ${HY_WORKSPACE}/hybench
./hybench -t sql -f conf/ddl_og_afterload.sql -c conf/og.props
```

## 执行基准测试负载

在执行之前，HyBench 要求查询各表记录数，以确认数据集正常：

```sh
gsql -r -d hybench_1x
```

```sql
SELECT count(*) FROM checking;
SELECT count(*) FROM checkingaccount;
SELECT count(*) FROM company;
SELECT count(*) FROM customer;
SELECT count(*) FROM loanapps;
SELECT count(*) FROM loantrans;
SELECT count(*) FROM savingaccount;
SELECT count(*) FROM transfer;
```

确认记录数正确后，需要先检查负载语句的配置是否正常。
HyBench 自带的负载语句配置文件是 `${HY_WORKSPACE}/hybench/conf/stmt_opengauss.toml`。
直接用这个文件跑的话会出问题，需要做一些修改：

```toml
[TP-9]
sql = [
    "SELECT balance FROM savingAccount WHERE accountid = ?;",
    "UPDATE savingaccount SET balance = balance - ? where accountid = ?;",
    "UPDATE savingaccount SET balance = balance + ? where accountid = ?;",
    "INSERT INTO transfer VALUES (DEFAULT, ?, ?, ?, ?, ?);"
]

[TP-13]
sql = [
    "INSERT INTO loantrans VALUES(DEFAULT, ?, ?, ?, ?, ?, ?, ?, ?);",
    "UPDATE customer SET loan_balance = loan_balance + ? where custid = ?;",
    "UPDATE company SET loan_balance = loan_balance + ? where companyid = ?;",
    "UPDATE loanapps SET status = ? WHERE id = ?;"
]

[TP-14]
sql = [
    "UPDATE savingaccount SET balance = balance + ? WHERE accountid = ?;",
    "UPDATE loantrans SET status = 'lent', timestamp = ? WHERE id = ?;"
]

[TP-16]
sql = [
    "SELECT balance FROM savingaccount WHERE accountid = ?;",
    "UPDATE savingaccount SET balance = balance - ? WHERE accountid = ?;",
    "UPDATE loantrans SET status = 'repaid', timestamp = ? WHERE id = ?;"
]

[AppQueue]
sql = "SELECT id, applicantid, amount, duration, status, timestamp FROM loanapps WHERE status= ? LIMIT 10000;"

[TransQueue]
sql = "SELECT id, applicantid, appid, amount, status, timestamp, duration, contract_timestamp, delinquency FROM loantrans WHERE status = ? LIMIT 10000;"
```

最后执行基准测试的负载：

```sh
./hybench -t runall -f conf/stmt_opengauss.toml -c conf/og.props
```

其中 `stmt_opengauss.toml` 指定了负载语句的模板。
HyBench 执行负载时会生成随机值填充到模板中。

`og.props` 指定了并发量与执行时间。
HyBench 负载的执行分为三个阶段（AP、TP、XP），每个阶段依据并发量创建线程分别执行负载语句。
每个线程会按照特定比例以随机的顺序执行该阶段的所有语句。
比如在 AP 阶段，每个线程实际上会被分配一个随机排序的语句队列，保证每条 AP 语句都各执行一次。
因此并发量足够高的情况下，语句的执行是均匀的。
