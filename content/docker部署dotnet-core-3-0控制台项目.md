+++
title = "Docker部署.NET Core 3.0控制台项目"
date = 2019-12-12 17:24:24

[taxonomies]
tags = [".NET Core", "Docker"]
categories = [".NET Core", "Docker"]
+++

为了方便描述，我们快速新建HelloWorld项目作为示例。

<!-- more -->

## 新建并发布.NET Core控制台项目

首先新建一个.NET Core控制台项目，并直接运行  
控制台应该会输出“Hello World!”，如果没有可以自己写一个。

```sh
$ dotnet new console --name hello
$ cd hello
$ dotnet run
```

解释：  
dotnet new用于创建新项目，console表示是控制台项目，--name指定项目名称为hello并在当前路径下创建hello文件夹，而项目在其中。

于是直接发布

```sh
$ dotnet publish -c Release
```

解释：该命令会把项目发布到hello文件夹中的bin/Release/netcoreapp3.0/publish里

然后在hello目录下编写Dockerfile文件用于生成镜像  
Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/core/runtime
COPY bin/Release/netcoreapp3.0/publish/ /publish
WORKDIR /publish
ENTRYPOINT ["dotnet", "hello.dll"]
```

解释：  
FROM  
表示我们的镜像基于dotnet core runtime  
  
COPY  
用于拷贝我们打包好的应用，这里第一个参数的路径应该替换为应用的发布路径  
  
WORKDIR  
用于切换当前工作路径  
  
ENTRYPOINT  
运行应用

接下来利用这个Dockerfile构建镜像

```sh
$ docker build -t dotnet-hello .
```

解释：dotnet-hello是镜像的名称

最后利用该镜像构建容器并运行

```sh
$ docker run -t --name hello dotnet-hello
```

解释：  
\-t 表示虚拟终端，用于在Docker命令行工具中输出应用的控制台输出。但一般项目中不需要  
\--name hello表示容器名为hello

最后看到终端输出“Hello World!”
