+++
title = "Win10下Perforce命令行快速上手"
date = 2020-07-11 20:38:43

[taxonomies]
tags = ["Perforce"]
categories = ["游戏开发"]
+++

相对于Git和SVN，Perforce在管理二进制文件上更方便

<!-- more -->

本文参考了官网教程

<https://www.perforce.com/manuals/p4guide/Content/P4Guide/tutorial.readmefirst.html>

## Git用户看这里

Stream Depot大概可以理解为一个Git项目，然后Stream就是Git里的分支

其中Stream必须要依赖于Stream Depot存在

## 安装

<https://www.perforce.com/downloads>

把服务端和命令行客户端都装上就行了，也可以整图形化界面

Windows下安装包会自动配置环境变量；并且服务端装完后会自动启动，如果没有的话就用p4d命令启动服务端

我们可以在PowerShell里输入下面的命令测试一下服务端是否正常

```ps1
PS p4 info
```

## 创建Stream Depot

```ps1
PS p4 depot -t stream TestDepot
```

解释一下，这行命令创建了一个名为TestDepot的Stream Depot，-t stream用于指定这个Depot的类型

## 创建Stream

Git中我们通常会开master和dev分支

在Perforce里我们也可以做同样的操作，使用下面两行命令创建main和dev两个Stream

```ps1
PS p4 stream -t mainline //TestDepot/main
PS p4 stream -t development -P //TestDepot/main //TestDepot/dev
```

解释一下，-t参数用于指定Stream的类型，-P参数用于指定父Stream；上面的例子里main这个Stream是dev的父Stream

需要注意mainline类型的Stream不能有父Stream

我们可以使用下面的命令来查看所有的Stream

```ps1
PS p4 streams
```

## 创建Workspace

之前Stream相关的操作都是作用在服务端上的

这里我们在本地创建Workspace

首先设置环境变量P4CLIENT，这个是PowerShell的写法  
配环境变量这种操作其实我是觉得挺离谱的

```ps1
PS $env:P4CLIENT='hello_client'
```

然后cd到一个空的文件夹，输入以下命令

```ps1
PS p4 client -S //TestDepot/main
```

这样这个文件夹就跟main这个Stream绑定起来了

## 添加文件

在这个文件夹里新建一个文件test\_file.txt

```ps1
PS p4 add .\test_file.txt
PS p4 submit -d 'Add a test file'
```

## 修改文件

Workspace里已有的文件通常都是只读的，我们使用下面这行命令修改test\_file.txt的读写权限

```ps1
PS p4 edit .\test_file.txt
```

然后用编辑器改一下里面的内容

```ps1
PS p4 submit -d 'Modify a test file'
```

## 删除文件

```ps1
PS p4 delete .\test_file.txt
PS p4 submit -d 'Delete a test file'
```

需要注意这个只是从服务端把该文件删除，Workspace里这个文件依然存在

## 与服务端同步

如果其他人对Stream进行了修改并提交到了服务端，我们可以把这些修改同步到本地的Workspace里

```ps1
PS p4 sync
```

## 切换Stream

我们可以使用下面这个命令查看所有的Stream

```ps1
PS p4 switch -l
```

然后我们切换到dev

```ps1
PS p4 switch -r dev
```

## 合并Stream

merge命令只能以父Stream为源，子Stream为目标

首先switch到子Stream，然后执行下面的命令，会自动从父Stream上合并过来

```ps1
PS p4 merge
PS p4 resolve
```

后面的resolve命令是解决两个Stream的冲突，不同于Git的自动合并，手动合并非常原始，建议使用图形化界面或者编辑器插件

resolve完了之后记得submit

```ps1
PS p4 submit -d "Merge"
```
