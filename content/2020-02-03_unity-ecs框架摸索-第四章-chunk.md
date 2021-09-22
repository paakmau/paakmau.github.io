+++
title = "Unity ECS框架摸索 第四章 Chunk"
date = 2020-02-03 01:37:42
slug = "202002030137"

[taxonomies]
tags = ["Unity ECS"]
categories = ["Unity"]
+++

本文将介绍Unity ECS中对于Component的内存管理

<!-- more -->

## Component的Chunk式存储

首先，一组Component是一个Archetype。多个Entity如果由相同的Component构成，那么他们属于同一个Archetype。

Unity会把属于同种Archetype的Entity归成同一类，同一类实体的组件会被集中存储。首先他们被划分为若干个Chunk，一个Chunk内部的同种组件会存储在连续的内存中；然后同类的Chunk之间可能用链表或指针数组相连，从而能够遍历这个Archetype所有实体的所有组件。如下图所示。

![](https://hebomou.top/wp-content/uploads/2019/12/ArchetypeChunkDiagram.jpg)

同种颜色的方块为同种组件，Chunk中的每列方块都是一个实体

## 针对Chunk的遍历

在前文中，我们使用的IJobForEach就是对符合筛选条件的所有Archetype中所有Chunk中的所有Entity进行遍历，它遍历的元素是Entity。

但是Unity提供了一种在Chunk层面上遍历的方法，即遍历符合条件的所有Chunk。

将第二章中的RotateSpeedSystem修改为如下代码，依然能正常运行

```cs
public class RotateSpeedSystem : JobComponentSystem {
    EntityQuery m_Group;
    protected override void OnCreate () {
        m_Group = GetEntityQuery (ComponentType.ReadWrite<Rotation> (), ComponentType.ReadOnly<RotationSpeed> ());
    }
    [BurstCompile]
    struct RotationJob : IJobChunk {
        public float DeltaTime;
        public ArchetypeChunkComponentType<Rotation> RotationType;
        [ReadOnly] public ArchetypeChunkComponentType<RotationSpeed> RotationSpeedType;
        public void Execute (ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex) {
            var chunkRotations = chunk.GetNativeArray (RotationType);
            var chunkRotationSpeeds = chunk.GetNativeArray (RotationSpeedType);
            for (var i = 0; i < chunk.Count; i++) {
                var rotation = chunkRotations[i];
                var rotationSpeed = chunkRotationSpeeds[i];

                chunkRotations[i] = new Rotation {
                    Value = math.mul (math.normalize (rotation.Value), quaternion.AxisAngle (math.up (), rotationSpeed.RadiansPerSecond * DeltaTime))
                };
            }
        }
    }
    protected override JobHandle OnUpdate (JobHandle inputDeps) {
        var rotationType = GetArchetypeChunkComponentType<Rotation> ();
        var rotationSpeedType = GetArchetypeChunkComponentType<RotationSpeed> (true);
        var job = new RotationJob {
            DeltaTime = Time.DeltaTime,
            RotationType = rotationType,
            RotationSpeedType = rotationSpeedType
        };
        return job.Schedule (m_Group, inputDeps);
    }
}
```

解释：  
m\_Group用于存储筛选Chunk的条件。在OnCreate中赋值为对Rotation可读可写，对RotationSpeed只读。在Job Schedule的时候被传入。  
RotationJob是用于规划Job的struct，Execute中传入的chunk变量就是当前遍历到的Chunk，根据RotationType和RotationSpeedType就能从chunk中拿到对应类型的原生数组，直接用for循环即可遍历该Chunk中的所有Entity的所有Component。

这样的遍历方式相对于IJobForEach更加灵活，它对Component的筛选跟访问是独立开的，也就是它筛选出的Component跟实际访问的Component可以不一样，更加灵活。  
比如说使用「或」条件筛选出的组件实际可能有多种情况，可以在内部判断是哪一种情况并灵活处理  
此外它还可以跟ChunkComponentData配合使用，可以提高性能（愚蠢的常数优化）

## ChunkComponentData介绍

它是一种奇怪的Component，Unity规定这类Component由同一个Chunk中每个Entity共享。  
举个例子，如果一个Archetype中包含一个ChunkComponentData类型的组件，那么它的每个Chunk就会有一个该组件。同一个Chunk中，Entity共享同一个该组件。不同Chunk的该组件是不同的实例，它们的值也可以不相同。

如果修改一个Entity上的该类组件，它所在Chunk中其他Entity的同类组件的值也会一并修改。

感觉这种组件应该可以用于在同种Archetype中细分小类型，或者用于共享公共数据。对它进行写访问的时候应该需要结合IJobChunk使用。
