+++
title = "Docker 部署 WordPress 与数据备份"
date = 2019-12-11 19:14:31
slug = "201912111914"

[taxonomies]
tags = ["Docker", "WordPress"]
+++

阅读本文了解 Docker 上的 WordPress 快速部署，及其数据备份获取

<!-- more -->

## 大致步骤

先创建 Volume 用于备份<br>
然后运行 MySQL 容器<br>
最后运行 WordPress 容器

## 创建 Volume

创建分别用于 MySQL 和 WordPress 的 Volume

```sh
docker volume create data_mysql_wordpress
docker volume create data_wordpress
```

## 运行 MySQL 容器

先拉取 `mysql` 镜像（如果需要）<br>
然后创建并运行 `mysql` 容器

```sh
docker pull mysql

docker run -d --name mysql_wordpress \
  -v data_mysql_wordpress:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=root \
  mysql --default-authentication-plugin=mysql_native_password
```

参数解释：

- `-d`：后台运行
- `--name`：设置容器的 name
- `-v`：映射数据卷
- `-e`：设置环境变量
- `--default-authentication-plugin=mysql_native_password`：设置 mysql 的默认加密插件

加密插件问题<br>
因为这里使用了 MySQL 8.0，它的默认认证插件是 `caching_sha2_password`，WordPress 连接比较麻烦，改回 `mysql_native_password` 省事一点。<br>
或者可以直接使用 `mysql:5.7` 镜像

## 运行 WordPress 容器并链接 MySQL

先拉取 `wordpress` 镜像（如果需要）<br>
然后创建并运行 `wordpress` 容器

```sh
docker pull wordpress

docker run -d --name wordpress \
  -p 80:80 -v data_wordpress:/var/www/html \
  --link mysql_wordpress:mysql wordpress
```

参数解释：

- `-d`：后台运行
- `--name`：设置容器的 name
- `-p` 映射容器 80 端口到本机的 80 端口
- `-v` 映射数据卷
- `--link` 链接到 MySQL 容器

## 获取备份

使用 `docker volume inspect` 命令获取 `mysql` 和 `wordpress` 容器的数据存储路径<br>
输出里的 `Mountpoint` 即是数据存储路径

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
