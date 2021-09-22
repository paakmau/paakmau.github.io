+++
title = "Docker简明介绍与安装"
date = 2019-12-11 20:16:43

[taxonomies]
tags = ["Docker"]
categories = ["Docker"]
+++

## 简介

<!-- more -->

根据官方文档，Docker是一个以容器形式构建、分享、运行应用的平台。

民间说法，Docker是一个虚拟机，只要制作了应用的虚拟机镜像，就能在几乎任何平台上一键部署我们的应用。而且这个虚拟机的性能非常好。

举个部署后端的例子说明Docker部署流程  
  
首先制作一个镜像，这个镜像的内容是一个配置了JRE然后还存放着我们写的Spring Boot后端的Ubuntu  
然后把镜像交给Docker，Docker就能根据镜像生成容器并运行它。  
这样我们的后端就部署成功了。

## 容器

是一个虚拟环境。这个环境可以是带了JRE和Java后端的Ubuntu，也可以只是一个JRE与Java后端，甚至可以自己从零开始编写。

总之就是容器中运行着我们的应用。

## 镜像

可以理解为容器的模板，可以用一个镜像生成很多个一模一样的容器。

一个镜像可以作为另一个镜像的基础。比如我们可以基于“Ubuntu”镜像制作一个“带JRE的Ubuntu”镜像。

而Docker提供了许多的基础镜像，比如包括Ubuntu在内的各种Linux发行版、DotNet、MySQL等等。我们可以基于这些镜像快速的构建我们的应用镜像。

## Docker Compose

在项目部署的时候，我们可能会用到多个容器，逐个配置就会很麻烦。可以使用Docker Compose同时配置管理多个容器

## Docker安装

这里以Debian 10为例

使用apt安装，首先根据处理器架构添加合适的apt仓库，然后直接apt install即可。

官网链接在这里  
[https://docs.docker.com/install/linux/docker-ce/debian/](https://docs.docker.com/install/linux/docker-ce/debian/)

注意：如果已有旧版的Docker，请移步上述官网教程按步骤卸载

首先安装依赖

```sh
$ sudo apt update
$ sudo apt install apt-transport-https ca-certificates
$ sudo apt install curl gnupg2 software-properties-common
```

然后添加Docker提供的GPGKey

```sh
$ curl -fsSL https://download.docker.com/linux/debian/gpg  sudo apt-key add -
```

接下来根据服务器的CPU架构，添加Docker的apt仓库

x86\_64 / amd64

```sh
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
```

armhf

```sh
$ sudo add-apt-repository \
   "deb [arch=armhf] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
```

arm64

```sh
$ sudo add-apt-repository \
   "deb [arch=arm64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
```

接下来更新apt

```sh
$ sudo apt update
```

安装Docker

```sh
$ sudo apt install docker-ce docker-ce-cli containerd.io
```

尝试HelloWorld  
观察到类似以下输出的东西就安装成功了

```sh
$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete 
Digest: sha256:4fe721ccc2e8dc7362278a29dc660d833570ec2682f4e4194f4ee23e415e1064
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

## 使用Snap安装Docker

喜闻乐见，在沙箱中运行沙箱

```sh
$ sudo snap install docker
```

## Docker Compose安装

官网在这里  
[https://docs.docker.com/compose/install/#alternative-install-options](https://docs.docker.com/compose/install/#alternative-install-options)

普通方式  
直接curl下载预编译的可执行文件然后改个权限就好了

```sh
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

然后发现Compose也能在容器中运行，Docker官方提供了脚本封装。因此还可以执行下面的命令安装（多此一举）

```sh
$ sudo curl -L --fail https://github.com/docker/compose/releases/download/1.25.0/run.sh -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```
