+++
title = "Unity Physics学习笔记「一」 简要介绍与HelloWorld"
date = 2020-02-04 22:09:43

[taxonomies]
tags = ["Unity ECS", "Unity Physics"]
categories = ["Unity"]
+++

现在Unity的Package Manager中已经有了适用于ECS框架的物理引擎，功能完善  

<!-- more -->
可以说Unity的DOTS（面向数据技术栈）已经能够用于制作大部分游戏类型

这个系列会介绍目前Unity Physics中的绝大多数内容，与Havok Physics关系不大

需要旧版Unity的基础，Unity ECS基础，并对常见物理引擎有一定了解

本系列文章基于  
com.unity.entities 0.5.1-preview.11  
com.unity.physics 0.2.5-preview.1

## 相关Package介绍

目前相关的包有两个，一个是com.unity.physics，另一个是com.havok.physics

com.unity.physics是Unity自己写的

- 无状态。现代物理引擎通常会维护大量的状态缓存以提高性能，但这对网络同步中状态的回退与恢复不太友好，因此Unity舍弃了这一特性以提高简单性与可控性
- 模块化。该物理引擎可以脱离jobs和ECS单独存在，这使得用户在使用时更加灵活
- 高性能。该引擎比较轻量化，跟其他商业引擎相比，功能相似，但性能持平或更优

com.havok.physics是微软写的，是Havok引擎在Unity上的发行版，本系列文章不做具体介绍

- 基于com.unity.physics
- 性能更好
- 质量更好
- 调试更好

## 开始前的准备

在Package Manager中安装Hybrid Renderer和Unity Physics两个包就行了

其中Hybrid Renderer依赖Entities，Entities依赖Collections、Mathematics等包，Package Manager会自动安装所有依赖

另外注意，如果在mac上使用VS Code作为编辑器，Visual Studio Code Editor这个包的1.1.4版本会导致Omnisharp出问题，需要退至1.1.3版本

## HelloWorld

创建一个Cube GameObject命名为CubeA，移除Box Collider，添加Convert To Entity、Physics Shape，这样他就是一个static的刚体

再创建一个Cube GameObject命名为CubeB，移除Box Collider，添加Convert To Entity、Physics Shape、Physics Body，这样他就是一个dynamic的刚体，注意Physics Body默认会添加重力

然后把CubeA移到CubeB的正上方，运行就会看到CubeA落下砸到CubeB上并发生碰撞

![](https://hebomou.top/wp-content/uploads/2020/02/image-258x300.png)

## Debug Display

在场景中新建一个空的GameObject，添加ConvertToEntity与PhysicsDebugDisplay，并在PhysicsDebugDisplay中勾选需要展示的信息，运行后就能在Scene窗体中看到了

注意是在Scene窗体中而不是Game窗体
