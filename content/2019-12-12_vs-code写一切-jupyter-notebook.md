+++
title = "VS Code 写一切 Jupyter Notebook"
date = 2019-12-12 01:35:40
slug = "201912120135"

[taxonomies]
tags = ["Python", "VS Code"]
categories = ["VS Code"]
+++

因为院里不少课程要炼丹，于是 Notebook 用的挺频繁的。当时用 VSC 写还行吧，就是快捷键有点问题，语法提示不太行，现在似乎好多了

<!-- more -->

## Jupyter 环境配置

安装 Python 与 Jupyter  
Mac 下用 brew 安装即可

```sh
$ brew install python jupyter
```

## 需要安装的 VSC 插件

- Python  
- VS Code Jupyter Notebook Previewer

## Notebook 创建与运行

打开一个空文件夹，cmd + shift + p 呼出命令面板  
输入 python create 进行搜索，选择 Python: Create New Blank Jupyter Notebook

![Create Jupyter Notebook](https://hebomou.top/wp-content/uploads/2019/12/QQ20191222-011543@2x-1024x199.png)

在当前目录保存为 hello.ipynb  
然后编辑内容如下并按 ctrl + enter 运行（为什么不是 cmd + enter）

![Result](https://hebomou.top/wp-content/uploads/2019/12/QQ20191222-013041@2x.png)

操作跟 web 版的差不多但是 Mac 下快捷键很蛋疼就这样吧
