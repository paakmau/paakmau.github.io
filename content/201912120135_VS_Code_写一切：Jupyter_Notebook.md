+++
title = "VS Code 写一切：Jupyter Notebook"
date = 2019-12-12 01:35:40
slug = "201912120135"

[taxonomies]
tags = ["Python", "VS Code"]
+++

因为院里不少课程要炼丹，于是 Notebook 用的挺频繁的。当时用 VSC 写还行吧，就是快捷键有点问题，语法提示不太行，现在似乎好多了

<!-- more -->

## Jupyter 环境配置

安装 Python 与 Jupyter<br>
Mac 下用 brew 安装即可

```sh
brew install python jupyter
```

## 需要安装的 VSC 插件

- Python
- VS Code Jupyter Notebook Previewer

## Notebook 创建与运行

打开一个空文件夹，`Cmd+Shift+P` 呼出命令面板<br>
输入 `python create` 进行搜索，选择 `Python: Create New Blank Jupyter Notebook`

在当前目录保存为 `hello.ipynb`<br>
然后编辑内容如下并按 `Ctrl+Enter` 运行

操作跟 Web 版的差不多，但是 `macOS` 下快捷键很蛋疼，就这样吧
