+++
title = "Unity ECS框架摸索 第五章 利用Prefab实例化Entity及其销毁"
date = 2020-02-03 14:05:00

[taxonomies]
tags = ["Unity ECS"]
categories = ["Unity"]
+++

一般游戏中都避不开实体的动态生成与动态销毁。

<!-- more -->

本文演示使用Prefab创建旋转的立方体方阵并定时销毁。

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191212-030831-HD.gif)

## 基本思路

放置一个生成器实体在场景中，让一个SpawnerSystem根据它创建立方体方阵。  
旋转与销毁的思路都与前文相同，让旋转系统筛选旋转速度组件并控制旋转，让销毁系统筛选销毁时间组件并控制销毁即可。

## 制作旋转立方体的Prefab

其实跟HelloWorld中基本一致，但是为了表述清晰这里还是写一下。

首先编写用于控制旋转速度的组件  
RotationSpeed.cs

```cs
using Unity.Entities;
using System;

[Serializable]
public struct RotationSpeed : IComponentData
{
    public float RadiansPerSecond;
}
```

然后还有控制自动销毁的组件  
DestroyTime.cs

```cs
using Unity.Entities;
using System;

[Serializable]
public struct DestroyTime : IComponentData
{
    public float TimeBeforeDestroy;
}
```

接下来则是用于添加上面两个组件的Authoring  
RotatingCubeAuthoring.cs

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

最后创建一个基于Cube的Prefab，移除Collider，挂载ConvertToEntity与RotatingCubeAuthoring脚本。如下图所示。

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191212-014200@2x-357x1024.png)

至此，就能使用该Prefab批量生成旋转立方体的Entity实例。

## 编写旋转系统

与HelloWorld完全一致

控制方块旋转的System  
RotateSpeedSystem.cs

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

## 编写销毁Entity的系统

筛选DestroyTime组件，让TimeBeforeDestroy不断随时间减少，在它归零的时候销毁它所属的Entity。

下面是销毁系统的具体代码  
DestroySystem.cs

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

解释：  
在一个System遍历Entity的时候，如果直接将某个Entity销毁，会导致遍历出现错乱。因此需要将销毁操作放入一个命令缓冲区，只要在下次Tick该系统运行之前执行这个命令缓冲区中的所有操作，就能安全地销毁Entity。  
  
然后每一个Tick里系统是有执行顺序的，我们找到所有系统中第一个执行的系统BeginInitializationEntityCommandBufferSystem。然后调用它的接口创建命令缓冲区，写入销毁命令，在下次Tick的时候这个系统就会执行我们写入的销毁命令。  
  
另外，IJobForEachWithEntity跟前文的IJobForEach基本一致，但是使用它遍历可以拿到Entity。这里我们需要拿到Entity以便销毁它。

## 编写并放置Spawner

首先是Spawner组件用于记录Prefab与生成数量信息  
生成的实体会形成一个方阵，CountX、CountY分别表示方阵横纵方向上实体的数量  
Spawner.cs

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

然后编写Authoring脚本  
SpawnerAuthoring.cs

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

解释：  
它不仅把CountX、CountY传入组件，还使用GameObjectConversionSystem把GameObject类型的Prefab转化为Entity。  
具体为什么这样写不用管，当作咒语记下来就好。（反正api还会改不少）

然后在Scene中新建一个空的GameObject命名为Spawner，挂载ConvertToEntity与SpawnerAuthoring脚本

## 编写SpawnerSystem

现在场景中已经存在一个生成器实体了，我们只需要写一个系统读取该实体储存的Prefab信息，就能实例化任意多个Prefab了。

当然，为了防止反复访问这个生成器实体而重复实例化Prefab，需要在实例化Prefab之后就立刻销毁该实体

代码如下  
SpawnerSystem.cs

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

解释：  
同样的，在遍历实体过程中创建新的实体会导致遍历混乱，所以它也跟DestroySystem一样使用了命令缓冲区来创建新的实体。

## 运行流程

首先SpawnerSystem筛选出了一个拥有Spawner组件的实体，根据该组件中的Prefab与方阵信息创建了整个方阵的旋转方块实体，最后销毁这个拥有Spawner组件的实体防止重复实例化Prefab。

然后RotateSpeedSystem控制拥有RotationSpeed组件的方块旋转

还有DestroySystem筛选出拥有DestroyTime组件的实体，每个Tick都让TimeBeforeDestroy这个计时器减去DeltaTime，并把计时器已经归零的Entity销毁。

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191212-030831-HD.gif)
