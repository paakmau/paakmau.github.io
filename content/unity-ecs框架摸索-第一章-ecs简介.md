+++
title = "Unity ECS框架摸索 第一章 ECS简介"
date = 2020-01-31 13:48:00

[taxonomies]
tags = ["Unity ECS"]
categories = ["Unity"]
+++

之前接触ECS是因为看了守望先锋的架构分析。了解了一下发现这种架构天生支持多线程，而且对缓存非常友好，似乎做联网同步也很方便，感觉这应该是个挺有前途的设计思路。  

<!-- more -->
这几天翻了一下Unity官网，看到ECS有文档了，例子也更新了许多，突然兴奋，准备写个Demo试试。顺便想记录看看Unity的正式版api能改成个什么鬼样子（大雾）

本系列基于 Entities 0.5.1-preview.11，并非正式版。  
因此更倾向于科普而不是实用（学了也没用早晚要改API），另外需要对旧版Unity有少许了解。

## ECS架构

ECS，是Entity-Component-System的缩写，该架构也就只有这三个东西组成（严格意义上）。这个架构由数据驱动逻辑，有别于我们熟知的OOP（面向对象）

## 名词解释

Entity，实体。可以近似理解为OOP中的对象。Entity由Component组成，此外不含有其他任何成分。可以理解为一个Entity编号为e1，他身上所有组件都被标记为e1，并且具体实现中Entity通常也只是一个用于标记的id。

Component，组件。存储数据，例如Position组件中会有x, y, z表示一个实体的位置。

System，系统。不含数据，只处理逻辑，固定帧率调用。且它会筛选拥有指定若干Component的Entity并对指定的Component进行读写。

## 举个例子描述一下ECS的运作

从前有一只鸭子Entity叫「老王」，它由「嘴」、「双脚」、「位置」几个Component组成。  
另外有一只大头鱼Entity叫「老鱼」，显然它没有脚，它只有「嘴巴」和「位置」两个Component。  
然后有「叫系统」、「跑系统」两个System。

「叫系统」以「嘴」组件为筛选条件。显然「老王」和「老鱼」都满足要求。它从这个组件中获取了「老王」的音量是100以及声音“嘎嘎”，然后每帧都调用系统API播放音量为100的“嘎嘎”的声音；同样，对于「老鱼」，它获取了音量为50的“啊啊”声，于是也播放相应的声音。

「跑系统」以「双脚」和「位置」组件为筛选条件。因为「老鱼」没有脚，所以它只会对「老王」作出反应。它从「双脚」中拿到「老王」的奔跑速度，每一帧根据DeltaTime和速度计算出「老王」的位移并更新老王的「位置」。

这样，两个简陋的System控制着「老王」一边向前奔跑一边嘎嘎叫，而「老鱼」只会“啊啊”叫。

其中理解ECS的重点有两个。

一个是数据存储，即Entity由多个不同种的Component组成。

另一个就是逻辑执行，即System只会筛选出拥有特定Component的Entity，并只对那些特定的Component进行读写。System针对的是Component而不是Entity。

总结一下就是，System执行逻辑，Component存储数据，Entity则是将Component关联起来。

## 瞎几把分析

天生对并行友好。观察不难发现，无论是「叫系统」还是「跑系统」，他们每次只会处理同一个Entity的某几个固定组件。  
因此假设有四只鸭子和一枚四核的Intel8代酷睿i5处理器，是不是可以每个核心分别处理一只鸭子而完全不需要线程锁？（我对硬件一无所知不要打我）

天生对缓存友好。因为System只关心特定的Component，因此如果将同种Component存储在线性内存空间里，System遍历Component的时候缓存命中率会非常高。Unity的具体实现中是将同类Entity的Component连续存储，可以实现缓存完全命中

耦合性低。理想状态下，系统只会对Component作出反应，因此系统之间的影响只有执行顺序，以及一些公共耦合。但是代码实现中，系统之间信息传递比较困难，不如直接调用更加高性能、高效率（省事）（包括Unity），因此需要不瞎几把乱写才能发挥ECS低耦合的优势。

框架设计比较困难。个人感觉ECS有一些麻烦的地方，比如没有回调，系统执行顺序会影响实际的效果。目前成熟的ECS框架也不算多。

## 其他学习资料

感觉我写的有一点过于简单，想要深入了解可以看看下面这些文章。

[https://www.cnblogs.com/zhaoqingqing/p/9718973.html](https://www.cnblogs.com/zhaoqingqing/p/9718973.html)  
[https://blog.codingnow.com/2017/06/overwatch\_ecs.html](https://blog.codingnow.com/2017/06/overwatch_ecs.html)
