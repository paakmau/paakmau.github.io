+++
title = "通过 MySQL Connector/J 连接 openGauss"
date = 2025-01-18 22:47:18
slug = "202501182247"

[taxonomies]
tags = ["openGauss", "MySQL", "Java", "数据库"]
+++

MySQL Connector/J 是 MySQL 官方的 JDBC 驱动。
本文将记录使用该驱动连接 openGauss B 库的过程。

<!-- more -->

## 下载并解压

下载链接：

<https://downloads.mysql.com/archives/c-j/>

- `Product Version` 选择 `8.0.28`。
- `Operating System` 选择 `Platform Independent`。

```sh
wget https://downloads.mysql.com/archives/get/p/3/file/mysql-connector-java-8.0.28.tar.gz
tar -zxf mysql-connector-java-8.0.28.tar.gz

mv mysql-connector-java-8.0.28/mysql-connector-java-8.0.28.jar ./

rm -r mysql-connector-java-8.0.28
rm mysql-connector-java-8.0.28.tar.gz
```

## 准备 B 库

使用默认用户连上 openGauss 之后就可以直接创建 B 库：

```sql
-- 创建 B 库
CREATE DATABASE compat_b DBCOMPATIBILITY 'B';

-- 配置数据库级别 B 库常用参数
ALTER DATABASE compat_b SET enable_set_variable_b_format TO ON;
ALTER DATABASE compat_b SET dolphin.lower_case_table_names TO 0;
ALTER DATABASE compat_b SET dolphin.b_compatibility_mode TO ON;

-- 配置系统级别 B 库常用参数
ALTER SYSTEM SET enable_dolphin_proto TO ON;
ALTER SYSTEM SET dolphin_server_port TO 3306;
```

于是重启数据库：

```sh
gs_ctl restart
```

创建 B 库之后需要先使用默认用户连到这个库上，以加载 dolphin 插件：

```sh
gsql -d compat_b
```

创建数据库时的默认用户是不允许远程登陆的，因此需要创建其他用户：

```sql
-- 创建用户 `mysql`
CREATE USER mysql WITH PASSWORD '<password>';
```

可以尝试用 `127.0.0.1` 连接一下：

```sh
gsql -h 127.0.0.1 -p "${PGPORT}" -U mysql -d compat_b
```

但这时仍然无法用 `mysql` 命令等方式直接连接到 B 库上。
我们还需要继续为用户 `mysql` 配置兼容 MySQL 认证协议的 B 库密码。

```sql
\c compat_b

-- 以 native 认证方式为用户 `mysql` 配置 B 库密码
SELECT set_native_password('mysql', '<new-b-password>', '<old-b-password>');
```

为用户设置 B 库密码时，注意到会要输入新密码与旧密码，第一次配置时旧密码可以直接为空字符串。

于是就可以用 `mysql` 命令连接了：

```sh
mysql -h 127.0.0.1 -P 3306 -u mysql -p
```

## 准备 Java 源文件

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

import com.mysql.cj.jdbc.Driver;

public class Demo {
    public static final String URL = "jdbc:mysql://localhost:3306/compat_b";
    public static final String USER = "mysql";
    public static final String PASSWORD = "<b-password>";

    public static void main(String[] args) throws Exception {
        // 注册驱动
        DriverManager.registerDriver(new Driver());

        // 创建连接
        Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);

        // 执行语句并输出结果
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT version();");
        while (rs.next()) {
            System.out.println(rs.getString("version"));
        }
    }
}
```

## 编译并执行

```sh
# 编译
javac -cp "./mysql-connector-java-8.0.28.jar" Demo.java

# 执行
java -cp ".:./mysql-connector-java-8.0.28.jar" Demo
```

注意一下 `-cp` 参数会覆盖 `CLASSPATH` 环境变量，用于指定 `.class`、`.jar` 等文件的路径，各路径之间使用 `:` 分隔。
