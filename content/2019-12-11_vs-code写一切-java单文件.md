+++
title = "VS Code 写一切 Java 单文件"
date = 2019-12-11 00:44:50
slug = "201912110044"

[taxonomies]
tags = ["Java", "VS Code"]
categories = ["VS Code"]
+++

VSC 可以直接编译调试单个 Java 源文件（虽然没什么用）

<!-- more -->

## Java 环境配置

与 VSC 无关，但提一下 mac 的配置

macOS 下用 brew 装OpenJDK

```sh
$ brew install openjdk
```

然后根据提示创建符号链接以便系统找到JDK

```sh
$ sudo ln -sfn /usr/local/opt/openjdk/libexec/openjdk.jdk \
   /Library/Java/JavaVirtualMachines/openjdk.jdk
```

## 需要安装的VS Code 插件

- Java Extension Pack

## 工作区配置

打开一个空文件夹，在其中创建一个文件Hello.java，编辑内容如下

```java
public class Hello {
    public static void main(String[] args) {
        System.out.print("Hello World");
    }
}
```

按F5直接运行，VSC 自动生成launch.json，有时候会失败，可以重试几遍  
内容大概如下

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "java",
            "name": "Debug (Launch) - Current File",
            "request": "launch",
            "mainClass": "${file}"
        },
        {
            "type": "java",
            "name": "Debug (Launch)-Hello<java_single_file_9fc652ff>",
            "request": "launch",
            "mainClass": "Hello",
            "projectName": "java_single_file_9fc652ff"
        }
    ]
}
```

解释：它配置了两个launch，一个是以当前打开的 Java 源文件作为主类调试，一个是以Hello.java 作为主类进行调试

切回Hello.java 按F5，得到如下输出

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191221-004354@2x.png)
