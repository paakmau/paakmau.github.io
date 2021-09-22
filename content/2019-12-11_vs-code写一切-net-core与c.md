+++
title = "VS Code写一切 .NET Core与C#"
date = 2019-12-11 19:37:00
slug = "201912111937"

[taxonomies]
tags = [".NET Core", "C#", "VS Code"]
categories = ["VS Code"]
+++

开发.NET Core项目VSC应该是首选了吧，毕竟跨平台而且它们还都是微软的私生子

<!-- more -->

## .NET Core环境配置

到官网下安装包  
或者Mac下

```sh
$ brew install dotnet
```

## 需要安装的VS Code插件

- C#

## .NET Core项目创建

新建一个文件夹hello并打开它，按下快捷键ctrl+\`呼出终端并输入如下命令创建控制台项目

```sh
$ dotnet new console
```

看到目录中生成了一些文件，其中  
Program.cs是入口函数所在的文件，打开发现里面已经写好了HelloWorld，所以就不改了  
hello.csproj是项目配置

## 调试运行

直接按F5，选择.NET Core，再按F5就行了

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-193323@2x-1024x410.png)

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-193406@2x.png)
