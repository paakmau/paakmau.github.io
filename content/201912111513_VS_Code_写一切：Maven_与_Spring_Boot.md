+++
title = "VS Code 写一切：Maven 与 Spring Boot"
date = 2019-12-11 15:13:45
slug = "201912111513"

[taxonomies]
tags = ["Java", "Maven", "Spring Boot", "VS Code"]
+++

VSC 用于开发 Maven 项目已经比较成熟<br>
直接打开就行，没什么好说的

因此本文介绍 VS Code 创建与调试 Spring Boot 项目

<!-- more -->

## Java 环境配置

问题不大，xjb 装就行了

## 需要安装的 VS Code 插件

- Java Extension Pack

- Spring Boot Extension Pack

## 创建 Spring Boot 项目

按下 `Shift+Cmd+P` 打开命令面板，输入 `spring` 搜索相关命令<br>
选择 `Spring Initializr: Generate a Maven Project`

他会依次让你选择语言、Group Id、Artifact Id、Spring Boot 版本、依赖等，除了依赖我们一路回车<br>
为了方便，依赖只选一个 `Spring Web`

接下来选择项目生成路径

然后他会在路径下生成一个 `demo` 文件夹

## 工作区配置

用 VSC 打开这个文件夹，按 `F5` 运行，配置选择 `Java`

他会生成 `launch.json`，跟上一篇 Java 的类似，用于确定主类，就不贴了<br>
可以把 `Current File` 的那条删了，因为暂时不需要单文件运行

按 `F5` 运行，跑起来了
