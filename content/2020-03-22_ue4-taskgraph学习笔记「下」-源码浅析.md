+++
title = "UE4 TaskGraph学习笔记「下」 源码浅析"
date = 2020-03-22 20:52:18

[taxonomies]
tags = ["图论", "并发", "拓扑排序"]
categories = ["UE4"]
+++

接下来阅读TaskGraph的源码来了解其实现细节

<!-- more -->

## TaskGraph实现概括

首先是线程管理。我们创建的TGraphTask并不能直接运行，需要把它交给某个FTaskThreadBase对象来控制执行。这个FTaskThreadBase可以理解为一个线程，可以把很多个TGraphTask交给他，它会依次执行这些任务（当然也要满足依赖）

另外注意到我们可以为TGraphTask指定想要让他执行的线程，于是我们发现对于游戏中的每一个线程都会产生一个FTaskThreadBase对象，这个对象持有他所对应的线程的指针。我们创建的TGraphTask对象会被交给对应的FTaskThreadBase对象，从而在正确的线程上执行

然后是任务之间依赖关系的处理。我们在CreateTask的时候看起来是把依赖传给了任务，但其实是TGraphTask被他所依赖的所有FGraphEvent持有。这样在一个TGraphTask结束时，他就可以立刻通过FGraphEvent更新依赖于自己的其他TGraphTask的依赖入度，然后如果入度为零就放入执行队列

## 线程管理

TaskGraph的线程管理是基于Runnable的，于是简单的介绍一下它

一个线程其实是由一个FRunnableThread对象管理，包括了线程优先级，创建等；而我们想要在线程里执行逻辑代码就要需要通过FRunnable，把逻辑代码写在它的Run方法里，然后把它丢给FRunnableThread让真实的线程去执行它

而TaskGraph里的FTaskThreadBase其实就继承自FRunnable；它同时还持有一个FWorkerThread对象，这个对象又持有FRunnableThread。也就是FTaskThreadBase自己不但能够管理线程，同时还能控制线程去执行逻辑代码  
另外所有的FTaskThreadBase都由一个FTaskGraphImplementation单例持有

最后看一下FTaskThreadBase的Run方法，就是不断地执行任务  
需要注意的是这些任务都是在满足了依赖之后才会被Run方法访问到，后文会详细描述  
这里贴一下Run的源码

```cpp
virtual uint32 Run() override
{
    check(OwnerWorker); // make sure we are started up
    ProcessTasksUntilQuit(0);
    FMemory::ClearAndDisableTLSCachesOnCurrentThread();
    return 0;
}
```

## FTaskThreadBase的派生类

查看源码我们发现这个东西有两个派生类，FTaskThreadAnyThread和FNamedTaskThread，这里简单介绍一下

FTaskThreadAnyThread比较符合我们的想象（如果你跟我想的一样）  
它由FTaskGraphImplementation单例在实例化的时候创建若干个。然后这个单例维护一个待执行任务的队列，每个FTaskThreadAnyThread在自己的任务执行完成后不断地调用这个单例的FindWork方法获取队列中的任务然后去执行它，可以说所有的FTaskThreadAnyThread共享同一个任务队列（如果不考虑优先级的话）  
还是贴一下FindWork的源码

```cpp
FBaseGraphTask* FindWork(ENamedThreads::Type ThreadInNeed)
{
    int32 LocalNumWorkingThread = GetNumWorkerThreads() + GNumWorkerThreadsToIgnore;
    int32 MyIndex = int32((uint32(ThreadInNeed) - NumNamedThreads) % NumTaskThreadsPerSet);
    int32 Priority = int32((uint32(ThreadInNeed) - NumNamedThreads) / NumTaskThreadsPerSet);
    check(MyIndex >= 0 && MyIndex < LocalNumWorkingThread &&
    MyIndex < (PLATFORM_64BITS ? 63 : 32) &&
    Priority >= 0 && Priority < ENamedThreads::NumThreadPriorities);

    return IncomingAnyThreadTasks[Priority].Pop(MyIndex, true);
}
```

而FNamedTaskThread比较特殊，根据类名我们知道它管理的是具名线程，也就是主线程、渲染线程这些。而且线程本身也不是由TaskGraph负责创建的  
并且每一个FNamedTaskThread的任务队列是自己负责管理的（因为每个具名线程都是唯一，所以不会存在共享任务队列的可能），这里贴一下它把任务塞进队列的小部分代码

```cpp
virtual void EnqueueFromThisThread(int32 QueueIndex, FBaseGraphTask* Task) override
{
    checkThreadGraph(Task && Queue(QueueIndex).StallRestartEvent); // make sure we are started up
    uint32 PriIndex = ENamedThreads::GetTaskPriority(Task->ThreadToExecuteOn) ? 0 : 1;
    int32 ThreadToStart = Queue(QueueIndex).StallQueue.Push(Task, PriIndex);
    check(ThreadToStart < 0); // if I am stalled, then how can I be queueing a task?
}
```

## 任务之间的依赖关系

上文中提到一个任务只有在满足了依赖的情况下才会被扔进任务队列里，并且任务的完成是通过FGraphEvent来通知依赖于自己的其他任务的

于是我们先看看FGraphEvent是怎么拿到其他任务的，在CreateTask的时候我们会传入一个FGraphEventRef的数组表示这个任务的依赖，然后跟随调用往下查我们发现有个SetupPrereqs函数它遍历了这个数组然后对每个FGraphEvent都调用它的AddSubsequent方法，把任务自身扔进这些事件的SubsequentList里  
简单贴一下源码

```cpp
bool AddSubsequent(class FBaseGraphTask* Task)
{
    return SubsequentList.PushIfNotClosed(Task);
}
```

然后在一个TGraphTask执行结束后，他会调用自己的FGraphEvent的DispatchSubsequents方法，这个方法把SubsequentList里的TGraphTask都Pop出来，调用它们的ConditionalQueueTask方法，把每个任务的依赖数（入度）减一，如果为零了意味着依赖满足，就可以把这个任务扔进任务队列里  
再贴一下源码

```cpp
void ConditionalQueueTask(ENamedThreads::Type CurrentThread)
{
    if (NumberOfPrerequistitesOutstanding.Decrement()==0)
    {
        QueueTask(CurrentThread);
    }
}
```

## 结语

源码终于啃完了，感觉很多写法非常的牛逼  
发现自己C++的高she级pi用cao法zuo还不是很熟悉，可能也是太久没碰UE4了，有空就去把改填的坑填了

但是实现的思路感觉还是很好理解，甚至有一种随xia心ji所ba欲xie的感觉（比如FTaskThreadAnyThread的公共任务队列就直接扔在FTaskGraphImplementation单例里）

结合了Runnable的TaskGraph是很好的跨平台基于任务的多线程实现方案，如果要自己写引擎的话这个很好用（抄就完事了）
