+++
title = "VS Code 写一切 CMake"
date = 2020-12-11 21:41:47
slug = "202012112141"

[taxonomies]
tags = ["C++", "CMake", "VS Code"]
categories = ["VS Code"]
+++

VS Code 写 CMake 感觉十分舒适，比 VS 爽多了

<!-- more -->

## 环境配置

安装C++编译器、C++调试器和CMake，并确保它们在 PATH 中

### macOS

安装 Xcode 命令行工具使用它的Clang，然后用 brew 安装CMake

```sh
$ xcode-select --install
$ brew install cmake
```

然后通过查看它们的版本信息测试是否安装成功

```sh
$ clang --version
$ cmake --version
```

### Windows

使用MSYS2安装mingw-w64集成的 Clang 和CMake，并把它们所在的目录添加的环境变量，如果你使用的是64位 Windows 且使用默认路径安装了MSYS2，那么这个目录应该是C:\\msys64\\mingw32\\bin

```sh
$ pacman -S mingw-w64-x86_64-clang
$ pacman -S mingw-w64-x86_64_cmake
```

然后打开 PowerShell 验证是否安装成功，如果环境变量没有生效需要重启

```ps1
PS clang.exe --version
PS cmake.exe --version
```

## 需要安装的VS Code 插件

- C/C++插件

- CMake 插件用于CMakeLists.txt 的语法高亮

- CMake Tools 插件用于 CMake 项目的构建、安装、调试等

## CMake 项目创建

创建一个文件夹HelloWorld，用VS Code 打开

在 macOS 下键入Cmd + Shift + P，或者在 Windows 下键入Ctrl + Shift + P，呼出命令面板

搜索CMake: Quick Start 命令，找到后回车

![](https://hebomou.top/wp-content/uploads/2020/12/cmake_quick_start.jpg)

输入项目名为HelloWorld

![](https://hebomou.top/wp-content/uploads/2020/12/cmake_project_name.jpg)

然后选择 Executable 创建可执行文件

![](https://hebomou.top/wp-content/uploads/2020/12/cmake_target.png)

然后就会看到CMake Tools 插件在目录下生成了main.cpp 和CMakeLists.txt，打开可以看到它的代码就是输出一个字符串

## 调试运行

按下Ctrl + F5就行了

比较折磨的是，CMake Tools 插件一开始就没有集成到launch.json 和tasks.json，所以需要使用它自己定的快捷键来构建和调试，但是最近已经有人提了 issue 并且受到了重视所以应该也快了吧
