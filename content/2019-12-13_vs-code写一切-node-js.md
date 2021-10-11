+++
title = "VS Code 写一切 Node.js"
date = 2019-12-13 21:36:11
slug = "201912132136"

[taxonomies]
tags = ["Node.js", "VS Code"]
categories = ["VS Code"]
+++

开发 Node.js 项目可能还是 WebStorm 功能多一点？？  
相比之下 VS Code 比较轻量级

<!-- more -->

VSC 天生支持 Node.js 不装插件就能调试  
但是，对于要装各种依赖的 Node.js 项目，VSC 没有自带图形化的管理工具，重度依赖终端，有点蛋疼（虽然有插件我也不乐意装毕竟命令行多方便）

## 项目新建与调试

打开一个空目录，ctrl + \` 呼出终端

$ npm init

一路回车，然后编辑工作区中的 index.js 如下

console.log("Hello World")

F9 下断点，F5 直接跑
