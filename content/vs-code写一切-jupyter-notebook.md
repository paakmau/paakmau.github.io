+++
title = "VS Code写一切 Jupyter Notebook"
date = 2019-12-12 01:35:40

[taxonomies]
tags = ["Python", "VS Code"]
categories = ["VS Code"]
+++

因为院里不少课程要机器学习，于是Notebook用的挺频繁的。当时用VSC写还行吧，就是快捷键有点问题，语法提示不太行，现在似乎好多了

<!-- more -->

## Jupyter环境配置

安装Python与Jupyter  
Mac下用brew安装即可

```sh
$ brew install python jupyter
```

## 需要安装的VSC插件

- Python  
- VS Code Jupyter Notebook Previewer

## Notebook创建与运行

打开一个空文件夹，cmd+shift+p呼出命令面板  
输入python create进行搜索，选择Python: Create New Blank Jupyter Notebook

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191222-011543@2x-1024x199.png)

在当前目录保存为hello.ipynb  
然后编辑内容如下并按ctrl+enter运行（为什么不是cmd+enter）

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191222-013041@2x.png)

操作跟web版的差不多但是Mac下快捷键很蛋疼就这样吧
