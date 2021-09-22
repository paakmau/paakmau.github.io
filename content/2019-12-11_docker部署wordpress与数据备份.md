+++
title = "Docker部署WordPress与数据备份"
date = 2019-12-11 19:14:31

[taxonomies]
tags = ["Docker"]
categories = ["Docker"]
+++

阅读本文了解Docker上的WordPress快速部署，及其数据备份获取

<!-- more -->

## 大致步骤

先创建Volume用于备份  
然后运行MySQL容器  
最后运行WordPress容器

## 创建Volume

创建分别用于MySQL和WordPress的volume

```sh
$ docker volume create data_mysql_wordpress
$ docker volume create data_wordpress
```

## 运行MySQL容器

先拉取mysql镜像（如果需要）  
然后创建并运行mysql容器

```sh
$ docker pull mysql

$ docker run -d --name mysql_wordpress \
   -v data_mysql_wordpress:/var/lib/mysql \
   -e MYSQL_ROOT_PASSWORD=root \
   mysql --default-authentication-plugin=mysql_native_password
```

参数解释：  
\-d 后台运行  
\--name 设置容器的name  
\-v 映射数据卷  
\-e 设置环境变量  
mysql --default-authentication-plugin=mysql\_native\_password  
设置mysql的默认加密插件  
  
加密插件问题  
因为这里使用了MySQL 8.0，它的默认认证插件是caching\_sha2\_password，WordPress连接比较麻烦，改回mysql\_native\_password省事一点。  
或者可以直接使用mysql:5.7镜像

## 运行WordPress容器并链接MySQL

先拉取wordpress镜像（如果需要）  
然后创建并运行wordpress容器

```sh
$ docker pull wordpress

$ docker run -d --name wordpress \
   -p 80:80 -v data_wordpress:/var/www/html \
   --link mysql_wordpress:mysql wordpress
```

参数解释：  
\-d 后台运行  
\--name 设置容器的name  
\-p 映射容器80端口到本机的80端口  
\-v 映射数据卷  
\--link 链接MySQL

## 获取备份

使用docker volume inspect命令获取mysql和wordpress的数据存储路径  
输出里的Mountpoint即是数据存储路径

```sh
$ docker volume inspect data_mysql_wordpress
[
    {
        "CreatedAt": "2019-12-10T01:21:23+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/data_mysql_wordpress/_data",
        "Name": "data_mysql_wordpress",
        "Options": {},
        "Scope": "local"
    }
]

$ docker volume inspect data_wordpress
[
    {
        "CreatedAt": "2019-12-10T01:24:43+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/data_wordpress/_data",
        "Name": "data_wordpress",
        "Options": {},
        "Scope": "local"
    }
]
```
