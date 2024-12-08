+++
title = "Docker Compose 部署饥荒服务器"
date = 2019-12-23 21:05:11
slug = "201912232105"

[taxonomies]
tags = ["Docker"]
+++

参考的是这个项目：

<https://github.com/mathielo/dst-dedicated-server>

<!-- more -->

## 项目 Clone

```sh
git clone https://github.com/mathielo/dst-dedicated-server.git
```

## Token 配置

游戏中可以在控制台里输入如下命令生成 Token：

```lua
TheNet:GenerateClusterToken()
```

也可以用图形化界面生成（进游戏之后 xjb 乱按就能找到）

然后把得到的 Token 存入项目中的文件 `dstclusterconfig/cluster_token.txt` 中，注意只有一行

## 游戏服务器配置

### `DSTClusterConfig/cluster.ini`

各种房间设置，大概有这些：

- `cluster_name`：房间名
- `cluster_description`：房间描述
- `cluster_password`：密码

### `DSTClusterConfig/mods/dedicated_server_mods_setup.lua`

用于安装 Mod，同目录下有个示例，照着写就行

### `DSTClusterConfig/mods/modoverrides.lua`

用于配置 Mod，同目录下有个示例，还是照着写就行

## 服务器启动

切到项目根目录，执行以下命令即可

```sh
sudo docker-compose up -d
```

## 服务器暂停

如果需要停止服务器，需要 `attach` 到 `dst_master` 容器中手动停止

```sh
sudo docker attach dst_master
```

然后会进入饥荒服务器的控制台。<br>
我们可以输入各种命令进行操作：

- `c_shutdown()`：停止
- `c_save()`：保存
- `c_rollback(count)`：回档

存档会被保存到对应的数据卷中。
所以此时将容器删除都不用担心。

## 服务器强制关闭

注意，这样关闭，数据不会被保存：

```sh
sudo docker-compose down
```

## 服务器重启或更新

饥荒更新之后，一般服务器也要相应更新。

先依照上文所述暂停服务器。然后还是依照上文所述启动就行了，容器启动的时候会自动更新。
