+++
title = "Unity ECS 框架摸索第三章：Hello World 后续"
date = 2020-02-01 20:34:04
slug = "202002012034"

[taxonomies]
tags = ["Unity ECS"]
+++

上一章介绍了 ECS 并行处理的实现，接下来就这个 Hello World 进行更进一步的摸索。

<!-- more -->

## 添加 Component 的技巧

第二章中为 Entity 添加 Component 的方式是直接在 GameObject 上挂载脚本，但是 Unity 编辑器目前并非为 ECS 设计，如果为每一个 Component 都挂载一个脚本会很麻烦，所以考虑使用代码添加组件。

我们编写如下脚本，把它挂载在 `Cube` 上，它就能为 `Cube` 生成的 Entity 添加 `RotationSpeed` 组件。<br>
`RotatingCubeAuthoring.cs`

```cs
[RequiresEntityConversion]
public class RotatingCubeAuthoring : MonoBehaviour, IConvertGameObjectToEntity {
    public float DegreePerSecond = 180;

    public void Convert (Entity entity, EntityManager dstManager, GameObjectConversionSystem conversionSystem) {
        var rotationSpeedCmpt = new RotationSpeed { RadiansPerSecond = math.radians (DegreePerSecond) };
        dstManager.AddComponentData (entity, rotationSpeedCmpt);
    }
}
```

解释：<br>
`IConvertGameObjectToEntity` 接口的 `Convert` 方法会在 GameObject 转换的过程中被调用。
可以在其中直接用 `EntityManager` 为 Entity 添加任意 Component，并能对其赋值，更加灵活。<br>
在该段代码中我们就把直观的 `Degree` 转换成了 `Radian`。

再把之前的 `RotationSpeed` 组件修改一下<br>
`RotationSpeed.cs`

```cs
// 修改前
[GenerateAuthoringComponent]
public struct RotationSpeed : IComponentData
{
    public float RadiansPerSecond;
}

// 修改后
[Serializable]
public struct RotationSpeed : IComponentData
{
    public float RadiansPerSecond;
}
```

解释：
原本的 `GenerateAuthoringComponent` 特性被改成了 `Serializable`。
`GenerateAuthoringComponent` 的作用是允许 ECS 的组件直接挂载在 GameObject 上。
但是使用代码添加组件时不需要这个特性，因此只需要 `Serializable` 即可。

在编辑器中修改 `Cube` 的 `RotatingCubeAuthoring` 的旋转速度为每秒 180 度，如下

## `IJobForEach`

上一章中，我们使用 Lambda 对 Entity 进行了遍历，写法很简洁。但是这样的写法会产生 GC，为了更好的性能，可以使用 `IJobForEach` 接口进行遍历。

修改 `RotationSpeedSystem` 为如下代码

```cs
// 修改后
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

解释：
原本使用 Lambda 对 Job 进行规划。
修改后使用一个结构体实现 `IJobForEach` 接口。
在其中规定了 Entity 的筛选条件为拥有 `Rotation` 和 `RotationSpeed` 组件，并实现 `Execute` 方法规划 Job。
有效减少了 GC。
