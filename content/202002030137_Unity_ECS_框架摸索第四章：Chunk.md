+++
title = "Unity ECS 框架摸索第四章：Chunk"
date = 2020-02-03 01:37:42
slug = "202002030137"

[taxonomies]
tags = ["Unity ECS"]
+++

本文将介绍 Unity ECS 中对于 Component 的内存管理

<!-- more -->

## Component 的 Chunk 式存储

首先，一组 Component 是一个 Archetype。多个 Entity 如果由相同的 Component 构成，那么他们属于同一个 Archetype。

Unity 会把属于同种 Archetype 的 Entity 归成同一类，同一类实体的组件会被集中存储。
他们被划分为若干个 Chunk，一个 Chunk 内部的同种组件会存储在连续的内存中。
同类的 Chunk 之间可能用链表或指针数组相连，从而能够遍历这个 Archetype 所有实体的所有组件。

## 针对 Chunk 的遍历

在前文中，我们使用的 `IJobForEach` 就是对符合筛选条件的所有 Archetype 中所有 Chunk 中的所有 Entity 进行遍历，它遍历的元素是 Entity。

但是 Unity 提供了一种在 Chunk 层面上遍历的方法，即遍历符合条件的所有 Chunk。

将第二章中的 `RotateSpeedSystem` 修改为如下代码，依然能正常运行

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

解释：<br>
`m_Group` 用于存储筛选 Chunk 的条件。在 `OnCreate` 中赋值为对 `Rotation` 可读可写，对 `RotationSpeed` 只读。在 Job Schedule 的时候被传入。<br>
`RotationJob` 是用于规划 Job 的 struct，`Execute` 中传入的 `chunk` 变量就是当前遍历到的 Chunk，根据 `RotationType` 和 `RotationSpeedType` 就能从 `chunk` 中拿到对应类型的原生数组，直接用 `for` 循环即可遍历该 Chunk 中的所有 Entity 的所有 Component。

这样的遍历方式相对于 `IJobForEach` 更加灵活，它对 Component 的筛选跟访问是独立开的，也就是它筛选出的 Component 跟实际访问的 Component 可以不一样，更加灵活。<br>
比如说使用或条件筛选出的组件实际可能有多种情况，可以在内部判断是哪一种情况并灵活处理<br>
此外它还可以跟 `ChunkComponentData` 配合使用，可以提高性能（邪恶的常数优化）

## `ChunkComponentData` 介绍

它是一种奇怪的 Component，Unity 规定这类 Component 由同一个 Chunk 中每个 Entity 共享。<br>
举个例子，如果一个 Archetype 中包含一个 `ChunkComponentData` 类型的组件，那么它的每个 Chunk 就会有一个该组件。同一个 Chunk 中，Entity 共享同一个该组件。不同 Chunk 的该组件是不同的实例，它们的值也可以不相同。

如果修改一个 Entity 上的该类组件，它所在 Chunk 中其他 Entity 的同类组件的值也会一并修改。

感觉这种组件应该可以用于在同种 Archetype 中细分小类型，或者用于共享公共数据。对它进行写访问的时候应该需要结合 `IJobChunk` 使用。
