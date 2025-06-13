+++
title = "openGauss-server 构建、安装、运行与调试"
date = 2024-12-16 21:06:18
slug = "202412162106"

[taxonomies]
tags = ["openGauss", "数据库"]
+++

`openGauss` 是一个开源的关系型数据库，兼容多个主流数据库的语法。
本文以 `master` 分支为例，记录 `openGauss-server` 的构建、安装、运行与调试。
参考的是官方文档：<https://gitcode.com/opengauss/openGauss-server#%E4%BD%BF%E7%94%A8%E5%91%BD%E4%BB%A4%E7%BC%96%E8%AF%91%E4%BB%A3%E7%A0%81>。

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

- `aarch64`
- openEuler 20.03 LTS

## 环境变量

为了便于描述，本文的操作都会在一个工作路径中执行。
所以直接给这个路径配一个环境变量 `OG_WORKSPACE`：

```sh
export OG_WORKSPACE=${HOME}/og
```

于是继续配置其他环境变量：

```sh
# 源码路径
export CODE_BASE=${OG_WORKSPACE}/openGauss-server

# 第三方依赖包路径
export BINARYLIBS=${OG_WORKSPACE}/binarylibs

# GCC 编译器相关环境变量
export GCC_HOME=${BINARYLIBS}/buildtools/gcc10.3
export CC=${GCC_HOME}/gcc/bin/gcc
export CXX=${GCC_HOME}/gcc/bin/g++
export LD_LIBRARY_PATH=${GCC_HOME}/gcc/lib64:${GCC_HOME}/isl/lib:${GCC_HOME}/mpc/lib/:${GCC_HOME}/mpfr/lib/:${GCC_HOME}/gmp/lib/:${LD_LIBRARY_PATH}
export PATH=${GCC_HOME}/gcc/bin:${PATH}

# 数据库相关环境变量
export GAUSSHOME=${OG_WORKSPACE}/install
export PGDATA=${OG_WORKSPACE}/data
export PGPORT=2333
export PGDATABASE=postgres
export LD_LIBRARY_PATH=${GAUSSHOME}/lib:${LD_LIBRARY_PATH}
export PATH=${GAUSSHOME}/bin:${PATH}
```

## 下载第三方依赖包

参考 `openGauss-server` 仓中 `README` 文档的说明，我们需要依据上文中查询出的 CPU 架构与发行版信息，下载对应的预编译第三方依赖包。
文档链接是 <https://gitcode.com/opengauss/openGauss-server/#%E4%B8%8B%E8%BD%BDopengauss>。
依据上文查询出的环境信息，这里需要下载 `master` 分支的 `openEuler_arm` 依赖包。
下载好了之后解压，再把解压出来的目录重命名为 `binarylibs`。

```sh
cd ${OG_WORKSPACE}
wget https://opengauss.obs.cn-south-1.myhuaweicloud.com/latest/binarylibs/gcc10.3/openGauss-third_party_binarylibs_openEuler_arm.tar.gz
tar -zxf openGauss-third_party_binarylibs_openEuler_arm.tar.gz
mv openGauss-third_party_binarylibs_openEuler_arm binarylibs
```

需要注意的是，这个依赖包已经自带了 GCC，所以直接通过上文配置好的环境变量使用即可。
可以尝试用 `g++ --version` 验证第三方依赖包与环境变量是否正常。

## 构建并安装 `openGauss-server`

先把代码仓 clone 下来：

```sh
cd ${OG_WORKSPACE}
git clone https://gitcode.com/opengauss/openGauss-server.git
```

然后进入 clone 下来的源码路径：

```sh
cd ${CODE_BASE}
```

接下来使用 `configure` 生成构建脚本。
这里需要注意一下，目前 openGauss 在第三方依赖包中给不同 CPU 架构提供的 GCC 版本是不同的。
可以直接用 `g++ --version` 查出来。
比如 `aarch64` 依赖包中提供的版本是 10.3.1，而 `x86_64` 的是 10.3.0。

```sh
# 生成 debug 版本构建脚本
./configure \
  --gcc-version=10.3.1 CC=g++ CFLAGS='-O0 -g3' \
  --prefix=${GAUSSHOME} --3rd=${BINARYLIBS} \
  --enable-debug --enable-cassert --enable-thread-safety \
  --with-readline --without-zlib

# 生成带 ASan 的 debug 版本构建脚本
./configure \
  --gcc-version=10.3.1 CC=g++ CFLAGS='-O0 -g3' \
  --prefix=${GAUSSHOME} --3rd=${BINARYLIBS} \
  --enable-debug --enable-cassert --enable-thread-safety \
  --with-readline --without-zlib --enable-memory-check

# 生成 release 版本构建脚本
./configure \
  --gcc-version=10.3.1 CC=g++ CFLAGS="-O2 -g3" \
  --prefix=${GAUSSHOME} --3rd=${BINARYLIBS} \
  --enable-thread-safety \
  --with-readline --without-zlib
```

于是构建并安装即可：

```sh
make -s -j 8 && make install
```

如果之前的环境变量配置正确，可以在 `${GAUSSHOME}` 路径下面找到安装好的文件。
二进制可执行文件在 `${GAUSSHOME}/bin` 里面；动静态库在 `${GAUSSHOME}/lib` 里面。
此时可以通过 `gaussdb --version` 或者 `gsql --version` 确认安装是否正常。

## 构建并安装 dolphin

Dolphin 插件的功能是兼容 MySQL 的语法。
构建并安装 dolphin 插件前需要先参考上文安装 openGauss。
其过程跟上文类似，需要注意的是要把 `Plugin` 仓中的 `dolphin` 目录拷贝到源码路径里的 `contrib` 目录中。

```sh
cd ${OG_WORKSPACE}
git clone https://gitcode.com/opengauss/Plugin.git

cp -r ${OG_WORKSPACE}/Plugin/contrib/dolphin ${CODE_BASE}/contrib

cd ${CODE_BASE}/contrib/dolphin
make -s -j 8 && make install
```

如果安装成功的话，可以在 `${GAUSSHOME}/lib/postgresql` 里面看到 `dolphin.so`。

## 初始化数据库

我们可以通过 `gs_initdb` 初始化数据库。
这个命令会建立一个目录，数据库所有的配置与持久化数据文件都会存储在这个目录中。
这个目录的路径可以通过 `-D` 参数指定：

```sh
gs_initdb -D ${PGDATA} --nodename=single_node -w <password>
```

我们也可以通过环境变量 `${PGDATA}` 指定这个路径。
实际上在配置了上文的 `${PGDATA}` 环境变量之后，就可以省略掉 `-D` 参数：

```sh
gs_initdb --nodename=single_node -w <password>
```

如果需要修改数据库后端的监听端口号，可以修改 `${PGDATA}/postgresql.conf` 里的 `port` 参数。
但由于我们在上文已经配置了环境变量 `${PGPORT}`，就不需要再修改配置文件了。

如果需要允许远程连接，可以修改 `${PGDATA}/pg_hba.conf`。

## 运行数据库

这里提供数据库启动、停止、重启的命令，由于存在环境变量 `${PGDATA}`，这里所有的 `-D` 参数均可省略。

```sh
gs_ctl start -D ${PGDATA}
gs_ctl stop -D ${PGDATA}
gs_ctl restart -D ${PGDATA}

gs_ctl start
gs_ctl stop
gs_ctl restart
```

此外，也可以使用 `gaussdb -D ${PGDATA}` 或 `gaussdb` 直接在前台启动数据库，要停止的话按 `Ctrl+C` 即可。
这种启动方式可以直接看到运行过程中的日志输出。
如果需要通过调试器启动数据库，也需要使用这种启动方式。

## 连接数据库

可以通过 `openGauss-server` 安装后自带的 `gsql` 连接数据库。
使用 `-d` 参数或环境变量 `${PGDATABASE}` 指定数据库名；使用 `-p` 参数或环境变量 `${PGPORT}` 指定端口号。

```sh
gsql -r -d "${PGDATABASE}" -p "${PGPORT}"

gsql -r
```

可以尝试查询一下版本号：

```sql
SELECT version();
```

## 创建 B 库以兼容 MySQL 语法

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

这里由于修改了系统级别的 GUC 参数，需要重启一下数据库，比如直接：

```sh
gs_ctl restart
```

有时经常会见到提示 3306 端口被占用，导致重启失败。
直接打开 `${PGDATA}/postgresql.conf`，修改 `dolphin_server_port` 参数随便换个端口号就行。

重启后可以连到刚刚创建的 B 库上面：

```sh
gsql -r -d compat_b
```

只有在连接到 B 库的情况下 dolphin 插件才会加载，所以可以用 `\dx` 元命令确认一下 dolphin 插件是否正常：

```sql
\dx dolphin
```

## 调试

使用 VS Code 调试，需要安装 C/C++ 插件。
可以通过调试器启动 `gaussdb` 进程直接运行数据库；如果数据库已在运行，也可以直接 attach 上去。
有时候我们还会需要调试 `gsql` 客户端。
如果前面环境变量配置正确，可以直接使用这个 `launch.json`：

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) Attach gaussdb",
            "type": "cppdbg",
            "request": "attach",
            "program": "${env:GAUSSHOME}/bin/gaussdb",
            "MIMode": "gdb",
            "miDebuggerArgs": "-ex 'handle SIGUSR2 nostop noprint pass' -ex 'set print thread-events off' -ex 'set print elements 0'",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        },
        {
            "name": "(gdb) Launch guassdb",
            "type": "cppdbg",
            "request": "launch",
            "program": "${env:GAUSSHOME}/bin/gaussdb",
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "MIMode": "gdb",
            "miDebuggerArgs": "-ex 'handle SIGUSR2 nostop noprint pass' -ex 'set print thread-events off' -ex 'set print elements 0'",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        },
        {
            "name": "(gdb) Attach gsql",
            "type": "cppdbg",
            "request": "attach",
            "program": "${env:GAUSSHOME}/bin/gsql",
            "MIMode": "gdb",
            "miDebuggerArgs": "-ex 'handle SIGINT nostop noprint pass'",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

这个 `launch.json` 提供三个调试配置：

- `(gdb) Attach gaussdb`

    Attach 到 `gaussdb` 进程上。

- `(gdb) Launch gaussdb`

    通过调试器启动 `gaussdb`。

- `(gdb) Attach gsql`

    Attach 到 `gsql` 进程上。

需要注意一下这个 `launch.json` 通过 `miDebuggerArgs` 给调试器指定了一些参数：

- `handle SIGUSR2 nostop noprint pass`

    因为 `gaussdb` 通过自定义信号进行线程间通信，默认情况下每次信号产生都会导致调试器中断，所以我们需要让调试器忽略这些信号并重新传递给 `gaussdb` 处理。

- `set print thread-events off`

    忽略线程启停事件。

- `set print elements 0`

    用于关闭 `gdb` 打印变量的最大长度限制。
    因为我们可以通过 `nodeToString` 将各种结点结构体序列化为字符串，而这些字符串通常很长，所以需要关闭长度限制，否则会被截断。

- `handle SIGINT nostop noprint pass`

    `SIGINT` 信号会在用户输入中断字符时产生（通常是 `Ctrl+C`），`gsql` 会自行处理这个中断。
    所以直接让调试器忽略并传递回 `gsql` 即可。

## 清理

如果修改差异很大或者需要切分支，那就要把构建或者安装的产物给清理掉后再重新构建了。

这里提供一些常用的命令：

```sh
# 先停止数据库
gs_ctl stop

# 删除数据库目录
rm -rf ${PGDATA}

# 删除安装目录
rm -rf ${GAUSSHOME}

# 清理构建产物
cd ${CODE_BASE}
make clean
git clean -xfd
git reset --hard HEAD
```
