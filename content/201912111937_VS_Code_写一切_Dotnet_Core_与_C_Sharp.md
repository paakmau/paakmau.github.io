+++
title = "VS Code 写一切 .NET Core 与 C#"
date = 2019-12-11 19:37:00
slug = "201912111937"

[taxonomies]
tags = [".NET Core", "C#", "VS Code"]
categories = ["VS Code"]
+++

开发.NET Core 项目 VSC 应该是首选了吧，毕竟跨平台而且它们还都是微软的私生子

<!-- more -->

## .NET Core 环境配置

到官网下安装包  
或者 Mac 下

```sh
$ brew install dotnet
```

## 需要安装的 VS Code 插件

- C#

## .NET Core 项目创建

新建一个文件夹 hello 并打开它，按下快捷键 ctrl + \` 呼出终端并输入如下命令创建控制台项目

```sh
$ dotnet new console
```

看到目录中生成了一些文件，其中  
Program.cs 是入口函数所在的文件，打开发现里面已经写好了 HelloWorld，所以就不改了  
hello.csproj 是项目配置

## 调试运行

直接按 F5，选择.NET Core，再按 F5 就行了

![Select env](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-193323@2x-1024x410.png)

![Result](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-193406@2x.png)
