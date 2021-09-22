+++
title = "VS Code写一切 Maven与Spring Boot"
date = 2019-12-11 15:13:45

[taxonomies]
tags = ["Java", "Maven", "Spring Boot", "VS Code"]
categories = ["VS Code"]
+++

VSC用于开发Maven项目已经比较成熟  
直接打开就行，没什么好说的

因此本文介绍VS Code创建与调试Spring Boot项目

<!-- more -->

## Java环境配置

问题不大，xjb装就行了

## 需要安装的VS Code插件

- Java Extension Pack

- Spring Boot Extension Pack

## 创建Spring Boot项目

按下Shift-Cmd-P打开命令面板，输入spring搜索相关命令  
选择Spring Initializr: Generate a Maven Project

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-145332@2x-1024x342.png)

他会依次让你选择语言、Group Id、Artifact Id、Spring Boot版本、依赖等，除了依赖我们一路回车  
为了方便，依赖只选一个Spring Web

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-150634@2x-1024x202.png)

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-150641@2x-1024x127.png)

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-150649@2x-1024x121.png)

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-150659@2x-1024x241.png)

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-150304@2x-1024x620.png)

接下来选择项目生成路径

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-150514@2x-1024x600.png)

然后他会在路径下生成一个demo文件夹

## 工作区配置

用VSC打开这个文件夹，按F5运行，配置选择Java

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-151044@2x-1024x451.png)

他会生成launch.json，跟上一篇Java的类似，用于确定主类，就不贴了  
可以把Current File的那条删了，因为暂时不需要单文件运行

按F5运行，跑起来了

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-151311@2x-1024x285.png)
