+++
title = "Unity ECS 框架摸索第五章：利用 Prefab 实例化 Entity 及其销毁"
date = 2020-02-03 14:05:00
slug = "202002031405"

[taxonomies]
tags = ["Unity ECS"]
+++

一般游戏中都避不开实体的动态生成与动态销毁。

本文演示使用 Prefab 创建旋转的立方体方阵并定时销毁。

<!-- more -->

## 总体思路

放置一个生成器实体在场景中，让一个 `SpawnerSystem` 根据它创建立方体方阵。<br>
旋转与销毁的思路都与前文相同，让旋转系统筛选旋转速度组件并控制旋转，让销毁系统筛选销毁时间组件并控制销毁即可。

## 制作旋转立方体的 Prefab

其实跟 Hello World 中基本一致，但是为了表述清晰这里还是写一下。

首先编写用于控制旋转速度的组件<br>
`RotationSpeed.cs`

```cs
using Unity.Entities;
using System;

[Serializable]
public struct RotationSpeed : IComponentData
{
    public float RadiansPerSecond;
}
```

然后还有控制自动销毁的组件<br>
`DestroyTime.cs`

```cs
using Unity.Entities;
using System;

[Serializable]
public struct DestroyTime : IComponentData
{
    public float TimeBeforeDestroy;
}
```

接下来则是用于添加上面两个组件的 Authoring<br>
`RotatingCubeAuthoring.cs`

```cs
using Unity.Entities;
using Unity.Mathematics;
using UnityEngine;

[RequiresEntityConversion]
public class RotatingCubeAuthoring : MonoBehaviour, IConvertGameObjectToEntity {
    public float DegreePerSecond = 180;

    public void Convert (Entity entity, EntityManager dstManager, GameObjectConversionSystem conversionSystem) {
        var rotationSpeedCmpt = new RotationSpeed { RadiansPerSecond = math.radians (DegreePerSecond) };
        var destroyTimeCmpt = new DestroyTime { TimeBeforeDestroy = 5 };
        dstManager.AddComponentData (entity, rotationSpeedCmpt);
        dstManager.AddComponentData (entity, destroyTimeCmpt);
    }
}
```

最后创建一个基于 Cube 的 Prefab，移除 `Collider`，挂载 `ConvertToEntity` 与 `RotatingCubeAuthoring` 脚本。

至此，就能使用该 Prefab 批量生成旋转立方体的 Entity 实例。

## 编写旋转系统

与 Hello World 完全一致

控制方块旋转的 System<br>
`RotateSpeedSystem.cs`

```cs
using Unity.Burst;
using Unity.Collections;
using Unity.Entities;
using Unity.Jobs;
using Unity.Mathematics;
using Unity.Transforms;

public class RotateSpeedSystem : JobComponentSystem {

    [BurstCompile]
    struct RotationJob : IJobForEach<Rotation, RotationSpeed> {
        public float DeltaTime;
        public void Execute (ref Rotation rotation, [ReadOnly] ref RotationSpeed rotationSpeed) {
            rotation = new Rotation {
                Value = math.mul (math.normalize (rotation.Value), quaternion.AxisAngle (math.up (), rotationSpeed.RadiansPerSecond * DeltaTime))
            };
        }
    }

    protected override JobHandle OnUpdate (JobHandle inputDeps) {
        return new RotationJob { DeltaTime = Time.DeltaTime }.Schedule (this, inputDeps);
    }
}
```

## 编写销毁 Entity 的系统

筛选 `DestroyTime` 组件，让 `TimeBeforeDestroy` 不断随时间减少，在它归零的时候销毁它所属的 Entity。

下面是销毁系统的具体代码<br>
`DestroySystem.cs`

```cs
using Unity.Burst;
using Unity.Collections;
using Unity.Entities;
using Unity.Jobs;

public class DestroySystem : JobComponentSystem {
    BeginInitializationEntityCommandBufferSystem m_EntityCommandBufferSystem;
    protected override void OnCreate () {
        m_EntityCommandBufferSystem = World.GetOrCreateSystem<BeginInitializationEntityCommandBufferSystem> ();
    }

    [BurstCompile]
    struct DestroyJob : IJobForEachWithEntity<DestroyTime> {
        public float DeltaTime;
        public EntityCommandBuffer.Concurrent CmdBuffer;
        public void Execute (Entity entity, int entityInQueryIndex, ref DestroyTime destroyTime) {
            destroyTime.TimeBeforeDestroy -= DeltaTime;
            if (destroyTime.TimeBeforeDestroy <= 0)
                CmdBuffer.DestroyEntity (entityInQueryIndex, entity);
        }
    }
    protected override JobHandle OnUpdate (JobHandle inputDeps) {
        var jobHandle = new DestroyJob { DeltaTime = Time.DeltaTime, CmdBuffer = m_EntityCommandBufferSystem.CreateCommandBuffer ().ToConcurrent () }.Schedule (this, inputDeps);
        m_EntityCommandBufferSystem.AddJobHandleForProducer (jobHandle);
        return jobHandle;
    }
}
```

解释：<br>
在一个 System 遍历 Entity 的时候，如果直接将某个 Entity 销毁，会导致遍历出现错乱。
因此需要将销毁操作放入一个命令缓冲区，只要在下次 Tick 该系统运行之前执行这个命令缓冲区中的所有操作，就能安全地销毁 Entity。<br>
然后每一个 Tick 里系统是有执行顺序的，我们找到所有系统中第一个执行的系统 `BeginInitializationEntityCommandBufferSystem`。
于是调用它的接口创建命令缓冲区，写入销毁命令，在下次 Tick 的时候这个系统就会执行我们写入的销毁命令。<br>
另外，`IJobForEachWithEntity` 跟前文的 `IJobForEach` 基本一致，但是使用它遍历可以拿到 Entity。
这里我们需要拿到 Entity 以便销毁它。

## 编写并放置 Spawner

首先是 `Spawner` 组件用于记录 Prefab 与生成数量信息<br>
生成的实体会形成一个方阵，`CountX`、`CountY` 分别表示方阵横纵方向上实体的数量<br>
`Spawner.cs`

```cs
using Unity.Entities;
using System;

[Serializable]
public struct Spawner : IComponentData {
    public Entity Prefab;
    public int CountX;
    public int CountY;
}
```

然后编写 Authoring 脚本<br>
`SpawnerAuthoring.cs`

```cs
using System.Collections.Generic;
using Unity.Entities;
using UnityEngine;

[RequiresEntityConversion]
public class SpawnerAuthoring : MonoBehaviour, IConvertGameObjectToEntity, IDeclareReferencedPrefabs {
    public GameObject Prefab;
    public int CountX;
    public int CountY;

    public void DeclareReferencedPrefabs (List<GameObject> referencedPrefabs) {
        referencedPrefabs.Add (Prefab);
    }

    public void Convert (Entity entity, EntityManager dstManager, GameObjectConversionSystem conversionSystem) {
        var spawner = new Spawner {
            Prefab = conversionSystem.GetPrimaryEntity (Prefab),
            CountX = CountX,
            CountY = CountY
        };
        dstManager.AddComponentData (entity, spawner);
    }
}
```

解释：<br>
它不仅把 `CountX`、`CountY` 传入组件，还使用 `GameObjectConversionSystem` 把 GameObject 类型的 Prefab 转化为 Entity。<br>
具体为什么这样写不用管，当作咒语记下来就好。（反正 api 还会改不少）

然后在 Scene 中新建一个空的 GameObject 命名为 `Spawner`，挂载 `ConvertToEntity` 与 `SpawnerAuthoring` 脚本

## 编写 SpawnerSystem

现在场景中已经存在一个生成器实体了，我们只需要写一个系统读取该实体储存的 Prefab 信息，就能实例化任意多个 Prefab 了。

当然，为了防止反复访问这个生成器实体而重复实例化 Prefab，需要在实例化 Prefab 之后就立刻销毁该实体

代码如下<br>
`SpawnerSystem.cs`

```cs
using Unity.Burst;
using Unity.Collections;
using Unity.Entities;
using Unity.Jobs;
using Unity.Mathematics;
using Unity.Transforms;

public class SpawnerSystem : JobComponentSystem {
    BeginInitializationEntityCommandBufferSystem m_EntityCommandBufferSystem;
    protected override void OnCreate () {
        m_EntityCommandBufferSystem = World.GetOrCreateSystem<BeginInitializationEntityCommandBufferSystem> ();
    }

    [BurstCompile]
    struct SpawnerJob : IJobForEachWithEntity<Spawner, LocalToWorld> {
        public EntityCommandBuffer.Concurrent CmdBuffer;
        public void Execute (Entity entity, int entityInQueryIndex, [ReadOnly] ref Spawner spawner, [ReadOnly] ref LocalToWorld local) {
            for (int i = 0; i < spawner.CountY; i++)
                for (int j = 0; j < spawner.CountX; j++) {
                    var instance = CmdBuffer.Instantiate (entityInQueryIndex, spawner.Prefab);
                    var position = math.transform (local.Value,
                        new float3 (j * 1.3F, noise.cnoise (new float2 (j, i) * 0.21F) * 2, i * 1.3F));
                    var destroyTime = 5 + noise.cnoise (new float2 (j, i) * 5.3F);
                    CmdBuffer.SetComponent (entityInQueryIndex, instance, new Translation { Value = position });
                    CmdBuffer.SetComponent (entityInQueryIndex, instance, new DestroyTime { TimeBeforeDestroy = destroyTime });
                }
            CmdBuffer.DestroyEntity (entityInQueryIndex, entity);
        }
    }
    protected override JobHandle OnUpdate (JobHandle inputDeps) {
        var jobHandle = new SpawnerJob { CmdBuffer = m_EntityCommandBufferSystem.CreateCommandBuffer ().ToConcurrent () }.Schedule (this, inputDeps);
        m_EntityCommandBufferSystem.AddJobHandleForProducer (jobHandle);
        return jobHandle;
    }
}
```

解释：<br>
同样的，在遍历实体过程中创建新的实体会导致遍历混乱，所以它也跟 `DestroySystem` 一样使用了命令缓冲区来创建新的实体。

## 运行流程

首先 `SpawnerSystem` 筛选出了一个拥有 `Spawner` 组件的实体，根据该组件中的 Prefab 与方阵信息创建了整个方阵的旋转方块实体，最后销毁这个拥有 `Spawner` 组件的实体防止重复实例化 Prefab。

然后 `RotateSpeedSystem` 控制拥有 `RotationSpeed` 组件的方块旋转

还有 `DestroySystem` 筛选出拥有 `DestroyTime` 组件的实体，每个 Tick 都让 `TimeBeforeDestroy` 这个计时器减去 `DeltaTime`，并把计时器已经归零的 Entity 销毁。
