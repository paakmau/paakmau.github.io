+++
title = "CMake从零开始快速上手"
date = 2020-12-11 20:35:12

[taxonomies]
tags = ["C++", "CMake"]
categories = ["C++"]
+++

CMake是一个跨平台的构建、测试、打包工具，被广泛使用于C++开源项目中

<!-- more -->

本文将介绍CMake在C++项目中的通常用法

本文的例子使用CMake常用的规范项目结构，因此可以直接当作C++项目的脚手架

## 环境配置

### macOS

首先我们需要C++编译器和调试器，于是我们可以输入以下命令直接安装Xcode的命令行工具来使用它自带的Clang

```sh
$ xcode-select --install
```

然后使用brew安装CMake就行了

```sh
$ brew install cmake
```

### Windows

推荐使用MSYS2直接安装集成到mingw-w64上的CMake和Clang，打开MSYS2 Shell输入以下命令，当然你最好要把放着CMake和Clang的bin文件夹扔到PATH环境变量里

```sh
$ pacman -S mingw-w64-x86_64-cmake
$ pacman -S mingw-w64-x86_64-clang
```

当然也可以直接到CMake官网下载安装包，再去装mingw-w64的编译器集成，甚至可以装个Visual Studio来使用微软提供的C++编译器，但是很麻烦

## Helllo World

### 项目的目录结构

创建一个文件夹命名为HelloWorld，在其中创建一个CMakeLists.txt文件，build和src这两个文件夹把Hello.cpp放src文件夹里，项目结构大概长这样

```
- HelloWorld
  - build
  - src
    - Hello.cpp
  - CMakeLists.txt
```

### HelloWorld/CMakeLists.txt

这个是CMake项目的配置文件，用来描述项目中包含哪些目标，有什么源文件、头文件等等

它会被CMake读取，用于配置HelloWorld项目  
看下面它的代码，显然就是字面意义上的限定CMake最低版本的要求，然后项目名叫做HelloWorld，添加一个名为Hello的可执行目标，它包括了src文件夹里的Hello.cpp这个源文件  
最后编译出来就是一个二进制可执行文件，并以src/Hello.cpp里的main函数作为入口点

```cmake
cmake_minimum_required(VERSION 3.0.0)
project(HelloWorld)

add_executable(Hello src/Hello.cpp)
```

### HelloWorld/src/Hello.cpp

这是即将被编译的源文件，包含main函数，输出一个字符串

```cpp
#include <iostream>

int main(int, char**) {
    std::cout << "Hello, world!\n";
    return 0;
}
```

### 项目构建

于是我们就可以开始编译了，CMake不会直接构建项目，它会生成构建所需的与平台有关的中间文件，这是CMake跨平台的实现方法，这样对于同一个项目，我们在不同的平台上都能生成对应的中间文件来进行构建  
比如在Windows上可以生成Visual Studio项目文件，然后用VS打开它来编译；类UNIX上可以生成Makefile；此外也可以选择生成跨平台的build.ninja、macOS上的Xcode项目文件等等

但是CMake生成的这些中间文件很混乱，因此习惯上我们会新建一个build文件夹来存放它们，也就是上面项目目录结构里的那个build文件夹  
于是在终端中切到HelloWorld文件夹中，再切到build，使用生成器生成中间文件，依据操作系统可能需要选择不同的生成器

#### Linux和macOS

类UNIX系统上默认会生成Makefile，于是使用cmake命令之后再用make命令就能编译出可执行文件，直接运行就可以了

```sh
$ cd build
$ cmake ..
$ make
$ ./Hello
```

#### Windows

Windows上不知道默认会生成什么，CMake文档中没说，Windows也不会自带VS或者mingw32-make这样的东西，而且不同VS版本的项目还不怎么兼容，因此我们最好使用-G参数来手动指定，比如在PowerShell中输入下面的命令会在build目录下生成VS 2019项目

```ps1
PS cd build
PS cmake.exe .. -G "Visual Studio 16 2019"
```

build目录下应该会有一个HelloWorld.sln，接下来就直接用VS 2019打开它，正常构建就行了

你也可以安装一个mingw32-make，然后选择对应的生成器来生成中间文件，这样就能用类似make的方式来构建项目

在MSYS2 Shell中安装mingw32-make

```sh
$ pacman -S mingw-w64-x86_64-make
```

PowerShell切到HelloWorld目录之后

```ps1
PS cd build
PS cmake.exe .. -G "MinGW Makefiles"
PS mingw32-make.exe
PS .\Hello.exe
```

具体能生成什么可以输入下面的命令来查询CMake支持的所有生成器

```sh
$ cmake --help
```

## 库的配置与链接

刚刚我们构建了一个可执行目标Hello，但是我们可能需要引入外部的开源库或者项目本身的库，这时我们就需要把Hello跟这堆库链接起来，并且把库的头文件暴露给Hello

假设有一个外部库HelloExternalLib，一般习惯把外部库单独放在一个文件夹里，于是我们创建一个external文件夹把它扔进去  
项目结构长这样

```
- HelloWorld
  - build
  - external
    - HelloExternalLib
      - include
        - HelloExternalLib
          - HelloExternalLib.h
      - src
        - HelloExternalLib.cpp
      - CMakeLists.txt
    - CMakeLists.txt
  - src
    - Hello.cpp
  - CMakeLists.txt
```

HelloWorld/external/HelloExternalLib/include/HelloExternalLib/HelloExternalLib.h

是这个库的一个头文件，随便声明一个方法好了

```cpp
#ifndef HELLO_EXTERNAL_LIB_H
#define HELLO_EXTERNAL_LIB_H

void sayHello();

#endif  // HELLO_EXTERNAL_LIB_H
```

HelloWorld/external/HelloExternalLib/src/HelloExternalLib.cpp

在这个文件中定义那个方法

```cpp
#include <HelloExternalLib/HelloExternalLib.h>

#include <iostream>

void sayHello() {
    std::cout << "Hello, world!\n";
}
```

HelloWorld/external/HelloExternalLib/CMakeLists.txt

这个文件用于构建HelloExternalLib这个库本身，添加了HelloExternalLib的库目标，包括HelloExternalLib.cpp这个源文件  
其中的target\_include\_directories用于指定库的头文件路径，PUBLIC代表如果库被链接这些头文件对外部是可见的，改成PRIVATE的话就是对外部不可见，只有库本身的源文件能够找到这些头文件，比如Hello目标链接了HelloExternalLib，include文件夹中的头文件都是对Hello可见的

```cmake
cmake_minimum_required(VERSION 3.0.0)
project(HelloExternalLib)

add_library(HelloExternalLib src/HelloExternalLib.cpp)

target_include_directories(HelloExternalLib PUBLIC include)
```

HelloWorld/external/CMakeLists.txt

这个文件仅仅是通过add\_subdirectory把子文件夹添加到CMake项目中，也就是把HelloExternalLib这个文件夹加进来，这样CMake才会读取子文件夹中的CMakeLists.txt文件并进行相应的构建工作  
注意子文件夹可以有多级目录嵌套，并且是递归进行的，当然需要在每一级目录下都编写CMakeList.txt然后加上add\_subdirectory

```cmake
add_subdirectory(HelloExternalLib)
```

HelloWorld/CMakeLists.txt

这个文件就是刚刚用来配置HelloWorld项目的

我们加一行add\_subdirectory来让它添加external子目录

并且使用target\_link\_libraries把Hello这个可执行目标跟HelloExternalLib这个库目标链接起来，并使用PRIVATE的链接方式  
PRIVATE这种链接方式表示HelloExternalLib不会被暴露给链接了Hello的其他目标，如果使用PUBLIC链接方式，其他的目标链接Hello时也会自动链接HelloExternalLib，尽可能使用PRIVATE，这样可以避免很多潜在的问题  
而对于可执行目标来说，它不会被其它的目标链接，所以可以全部使用PRIVATE

```cmake
cmake_minimum_required(VERSION 3.0.0)
project(HelloWorld)

add_subdirectory(external)

add_executable(Hello src/Hello.cpp)

target_link_libraries(Hello PRIVATE HelloExternalLib)
```

HelloWorld/src/Hello.cpp

这时我们的源文件中就可以包含外部库的头文件并调用里面声明的方法了

```cpp
#include <HelloExternalLib/HelloExternalLib.h>

int main(int, char**) {
    sayHello();
}
```

然后生成中间文件  
macOS和Linux下可以这样，Windows下就根据上文添加-G参数并选择合适的生成器

```sh
$ cd build
$ cmake ..
```

最后使用make或者Windows下的其他工具构建就行了
