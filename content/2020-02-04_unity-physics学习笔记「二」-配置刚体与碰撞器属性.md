+++
title = "Unity Physics学习笔记「二」 配置刚体与碰撞器属性"
date = 2020-02-04 23:56:57

[taxonomies]
tags = ["Unity ECS", "Unity Physics"]
categories = ["Unity"]
+++

前文展示了一个简单的HelloWorld，本文将进一步介绍其中用到的Physics Shape和Physics Body这两个脚本的参数设置

<!-- more -->

这两个脚本其实都是Authoring，它们会根据编辑器中设定的值为Entity相应地添加组件

## Physics Shape

可以理解为碰撞体，能为Entity添加碰撞功能

### 寻常属性

在Inspector面板中，我们可以设置它的形状、大小、重心、材质、摩擦系数、恢复系数等属性  
这些属性在主流物理引擎中十分常见，高中物理也都学过，就不细说了

### Is Trigger与Raises Collision Events  

如果勾选了Is Trigger，碰撞体就不会与其他碰撞体产生物理碰撞，但是在碰撞体重叠时会产生碰撞事件  
可以用来检测角色是否进入一个区域  

而Raises Collision Events与Is Trigger类似，会产生碰撞事件，但是它能够与其他碰撞体产生物理碰撞

### Collision Filter

Collision Filter能屏蔽碰撞体与某些碰撞体的碰撞，也就是，只跟指定分组的Physics Shape发生碰撞  

具有32个分组，一个碰撞体可以属于多个分组，并且可以指定与哪些分组发生碰撞  
内部实现其实就是uint，用位运算做判断  

另外，碰撞查询也具有这个特性，例如发射射线或发射碰撞器等

## Physics Body

Physics Body使Entity具有运动的能力

### 寻常的属性

在Inspector面板中，我们可以看到质量、重力系数、线性阻尼、角度阻尼等，基本上也是高中物理

### Motion Type

Dynamic，动态。普通的可以运动的刚体，根据质量计算碰撞后的运动状态  
Kinematic，运动学。可以运动的刚体，可以理解为质量无穷大的Dynamic刚体，看起来就是能把其他普通Dynamic刚体弹开，自己不受碰撞的影响。可以用于电梯  
Static，静态。无法运动的刚体，可以理解为质量无穷大

### 质量分配

勾选Override Default Mass Distribution以手动配置重心与惯性张量

## 复合碰撞体

多个不同的Physics Shape可以组合到一起

个人猜测应该就是把多个碰撞体整合为一个碰撞体，最终组件数量不会变  
但并不是，验证后发现每个带有Physics Shape的GameObject都转化为了Entity，然后用Parent组件跟父组件捆起来，感觉是Unity偷懒了  
因此为了效率，可以把用作静态碰撞体的所有模型在导入Unity之前先合并为同一个模型。不要在Unity里蛇皮操作

根据官方手册，可以这样操作  
在Hierarchy面板中将多个具有Physics Shape的子GameObject挂在一个具有Physics Body的父GameObject下  
注意，只有父GameObject拥有Convert to Entity组件，子GameObject不需要
