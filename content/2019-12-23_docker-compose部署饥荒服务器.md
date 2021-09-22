+++
title = "Docker Compose部署饥荒服务器"
date = 2019-12-23 21:05:11

[taxonomies]
tags = ["Docker"]
categories = ["Docker"]
+++

参考的是这个项目  

[https://github.com/mathielo/dst-dedicated-server](https://github.com/mathielo/dst-dedicated-server)

<!-- more -->

## 项目Clone

```sh
$ git clone https://github.com/mathielo/dst-dedicated-server.git
```

## Token配置

游戏中可以在控制台里输入如下命令生成Token

```
TheNet:GenerateClusterToken()
```

也可以用图形化界面生成（进游戏之后xjb乱按就能找到）

然后把得到的Token存入项目中的这个文件，注意只有一行  
DSTClusterConfig/cluster\_token.txt

## 游戏服务器配置

DSTClusterConfig/cluster.ini  
各种房间设置，大概有这些  
cluster\_name 房间名  
cluster\_description 房间描述  
cluster\_password 密码

DSTClusterConfig/mods/dedicated\_server\_mods\_setup.lua  
用于安装Mod，同目录下有个示例，照着写就行

DSTClusterConfig/mods/modoverrides.lua  
用于配置Mod，同目录下有个示例，还是照着写就行

## 服务器启动

切到项目根目录，执行以下命令即可

```sh
$ sudo docker-compose up -d
```

## 服务器暂停

如果需要停止服务器，需要attach到dst\_master容器中手动停止

```sh
$ sudo docker attach dst_master
```

然后会进入饥荒服务器的控制台  
我们可以输入各种命令进行操作  
c\_shutdown() 停止  
c\_save() 保存  
c\_rollback(count) 回档

存档会被保存到对应的数据卷中  
所以此时将容器删除都不用担心

## 服务器强制关闭

注意，这样关闭，数据不会被保存

```sh
$ sudo docker-compose down
```

## 服务器重启或更新

饥荒更新之后，一般服务器也要相应更新

先依照上文所述暂停服务器  
然后还是依照上文所述启动就行了，容器启动的时候会自动更新