+++
title = "Unity Physics学习笔记「七」 精确的碰撞管理"
date = 2020-03-15 22:07:48
slug = "202003152207"

[taxonomies]
tags = ["Unity ECS", "Unity Physics"]
categories = ["Unity"]
+++

官方手册里有提到，但是没有具体描述。本文主要基于官方示例代码与实践

<!-- more -->

## 遇到的问题

前文和手册中都有介绍过CollisionFilter，但是这个东西是跟Collider绑在一起扔进Blob里的。而Blob是一种类似引用的东西，因此由同一个带有Collider的Prefab实例化出的Entity，他们的CollisionFilter也是相同的。如果想要改变，必须要为每个实例化的Entity单独创建一个Collider的Blob资源来让他们的CollisionFilter不同。这就非常蛋疼

## 简单高效但是不正常也不一定高效的解决方案

最简单的一种思路，为每一个实例化的Entity单独创建一个Collider的Blob资源

例如网格碰撞体，可以把Prefab上碰撞体中存着点和面的NativeArray的引用偷出来，重新定个CollisionFilter就行了，因为点和面都是共用的，所以内存上其实不亏

而且使用CollisionFilter的效率很高，因此在碰撞处理时的性能上也不亏

不过在一开始创建新的Blob的时候就很麻烦而且时间开销较大；并且如果涉及到Entity的销毁，可能要把相应的Blob资源一起销毁；为了降低Blob创建销毁的时间开销，就要开个Blob资源的内存池（ECS里开内存池是有毒吧？）。最终实现起来会变得很蛇皮，我个人认为作为一个正常的人类不应该做这样的事情。

## 正常的解决方案

根据手册的指示，使用IBodyPairsJob  
具体来说，我们写一个Struct实现IBodyPairsJob接口，并把它注入到StepPhysicsWorld中，把需要屏蔽的碰撞Disable掉。这样就能对碰撞为所欲为了

下面是个简单的代码，他禁用了所有拥有NonUniformScale组件的碰撞体的碰撞

```cs
[UpdateBefore (typeof (StepPhysicsWorld))]
class DisableCollisionSystem : SystemBase {
    struct DisableCollisionJob : IBodyPairsJob {
        public ComponentDataFromEntity<NonUniformScale> nonUniformScaleFromEntity;
        public void Execute (ref ModifiableBodyPair pair) {
            if (nonUniformScaleFromEntity.Exists (pair.Entities.EntityA)  nonUniformScaleFromEntity.Exists (pair.Entities.EntityB))
                pair.Disable ();
        }
    }
    BuildPhysicsWorld m_buildPhysicsWorld;
    StepPhysicsWorld m_stepPhysicsWorld;
    protected override void OnCreate () {
        m_buildPhysicsWorld = World.GetOrCreateSystem<BuildPhysicsWorld> ();
        m_stepPhysicsWorld = World.GetOrCreateSystem<StepPhysicsWorld> ();
    }
    protected override void OnUpdate () {
        if (m_stepPhysicsWorld.Simulation.Type == SimulationType.NoPhysics) return;
        SimulationCallbacks.Callback callback = (ref ISimulation simulation, ref PhysicsWorld world, JobHandle inDeps) => {
            return new DisableCollisionJob { nonUniformScaleFromEntity = GetComponentDataFromEntity<NonUniformScale> () }.Schedule (simulation, ref world, inDeps);
        };
        m_stepPhysicsWorld.EnqueueCallback (SimulationCallbacks.Phase.PostCreateDispatchPairs, callback);
    }
}
```

但是需要注意几个点

1.  使用CollisionFilter来实现的话，碰撞处理过程中的性能要注入IBodyPairsJob好；但是使用IBodyPairsJob不需要考虑上述CollisionFilter的诸多问题，也避免了销毁与创建的开销；
2.  上面的代码使用了lambda，却没有用Burst优化，因此会产生GC，可以暂时用蛇皮方法自己搞搞；但Unity Physic之后的版本应该会自己解决这个问题，不用我们操心
