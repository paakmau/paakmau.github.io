+++
title = "Unity Physics 学习笔记「一」 简要介绍与HelloWorld"
date = 2020-02-04 22:09:43
slug = "202002042209"

[taxonomies]
tags = ["Unity ECS", "Unity Physics"]
categories = ["Unity"]
+++

现在 Unity 的Package Manager 中已经有了适用于 ECS 框架的物理引擎，功能完善  
可以说 Unity 的DOTS（面向数据技术栈）已经能够用于制作大部分游戏类型

<!-- more -->

这个系列会介绍目前Unity Physics 中的绝大多数内容，与Havok Physics 关系不大

需要旧版 Unity 的基础，Unity ECS 基础，并对常见物理引擎有一定了解

本系列文章基于  
com.unity.entities 0.5.1-preview.11  
com.unity.physics 0.2.5-preview.1

## 相关 Package 介绍

目前相关的包有两个，一个是com.unity.physics，另一个是com.havok.physics

com.unity.physics 是 Unity 自己写的

- 无状态。现代物理引擎通常会维护大量的状态缓存以提高性能，但这对网络同步中状态的回退与恢复不太友好，因此 Unity 舍弃了这一特性以提高简单性与可控性
- 模块化。该物理引擎可以脱离 jobs 和ECS 单独存在，这使得用户在使用时更加灵活
- 高性能。该引擎比较轻量化，跟其他商业引擎相比，功能相似，但性能持平或更优

com.havok.physics 是微软写的，是 Havok 引擎在Unity 上的发行版，本系列文章不做具体介绍

- 基于com.unity.physics
- 性能更好
- 质量更好
- 调试更好

## 开始前的准备

在Package Manager 中安装Hybrid Renderer 和Unity Physics 两个包就行了

其中Hybrid Renderer 依赖Entities，Entities 依赖Collections、Mathematics 等包，Package Manager 会自动安装所有依赖

另外注意，如果在 mac 上使用VS Code 作为编辑器，Visual Studio Code Editor 这个包的1.1.4版本会导致 Omnisharp 出问题，需要退至1.1.3版本

## HelloWorld

创建一个Cube GameObject 命名为CubeA，移除Box Collider，添加Convert To Entity、Physics Shape，这样他就是一个 static 的刚体

再创建一个Cube GameObject 命名为CubeB，移除Box Collider，添加Convert To Entity、Physics Shape、Physics Body，这样他就是一个 dynamic 的刚体，注意Physics Body 默认会添加重力

然后把 CubeA 移到CubeB 的正上方，运行就会看到 CubeA 落下砸到CubeB 上并发生碰撞

![](https://hebomou.top/wp-content/uploads/2020/02/image-258x300.png)

## Debug Display

在场景中新建一个空的GameObject，添加 ConvertToEntity 与PhysicsDebugDisplay，并在 PhysicsDebugDisplay 中勾选需要展示的信息，运行后就能在 Scene 窗体中看到了

注意是在 Scene 窗体中而不是Game 窗体
