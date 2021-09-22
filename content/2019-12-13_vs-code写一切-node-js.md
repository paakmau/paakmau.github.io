+++
title = "VS Code写一切 Node.js"
date = 2019-12-13 21:36:11

[taxonomies]
tags = ["Node.js", "VS Code"]
categories = ["VS Code"]
+++

开发Node.js项目可能还是WebStorm功能多一点？？  
相比之下VS Code比较轻量级

<!-- more -->

VSC天生支持Node.js不装插件就能调试  
但是，对于要装各种依赖的Node.js项目，VSC没有自带图形化的管理工具，重度依赖终端，有点蛋疼（虽然有插件我也不乐意装毕竟命令行多方便）

## 项目新建与调试

打开一个空目录，ctrl+\`呼出终端

$ npm init

一路回车，然后编辑工作区中的index.js如下

console.log("Hello World")

F9下断点，F5直接跑
