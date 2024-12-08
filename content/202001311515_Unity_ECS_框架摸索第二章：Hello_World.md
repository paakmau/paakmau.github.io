+++
title = "Unity ECS 框架摸索第二章：Hello World"
date = 2020-01-31 15:15:00
slug = "202001311515"

[taxonomies]
tags = ["Unity ECS"]
+++

跟随本文，创建一个立方体 Entity，利用一个 System 控制它不断旋转。

<!-- more -->

## 步骤

首先写一个控制旋转速度的组件<br>
`RotationSpeed.cs`

```cs
[GenerateAuthoringComponent]
public struct RotationSpeed : IComponentData
{
    public float RadiansPerSecond;
}
```

然后在 Unity 编辑器中创建一个 `Cube` 的 GameObject，挂载 `ConvertToEntity` 与刚刚写好的 `RotationSpeed`，并设置 `RotationSpeed` 的初始值。具体如下图。

解释：<br>
这个 `Cube` 本身是一个 GameObject，但是挂上 `ConvertToEntity` 脚本之后，他会被转化为 Entity，他的 `Transform` 与 `Mesh` 信息也会被转化为相应的 Component。<br>
其中 `ConversionMode` 选择 `ConvertAndDestroy` 代表转化为 Entity 之后原本的 GameObject 会被删除。

此外，我们挂载的 `RotationSpeed` 同样也会成为这个 Entity 的组件。

至此，我们添加了一个具有 `RotationSpeed` 组件的 `Cube` 实体

接下来写一个控制旋转的系统<br>
`RotationSpeedSystem.cs`

```cs
public class RotationSpeedSystem : JobComponentSystem {
    protected override JobHandle OnUpdate (JobHandle inputDeps) {
        float deltaTime = Time.DeltaTime;

        var jobHandle = Entities
            .WithName ("RotationSpeedSystem")
            .ForEach ((ref Rotation rotation, in RotationSpeed rotationSpeed) => {
                rotation.Value = math.mul (
                    math.normalize (rotation.Value),
                    quaternion.AxisAngle (math.up (), rotationSpeed.RadiansPerSecond * deltaTime));
            })
            .Schedule (inputDeps);

        return jobHandle;
    }
}
```

解释：<br>
`JobComponentSystem` 的抽象方法 `OnUpdate` 会在每帧调用，重载该方法并在其中规划 Job 就能实现对相应 Component 的并行读写。<br>
上面的代码中利用 `ForEach` 传入一个 Lambda 函数，这个函数筛选拥有 `Rotation`（Unity 自带）与 `RotationSpeed`（我们刚刚写的）组件的 Entity，然后根据 `RotationSpeed` 的信息修改 `Rotation`，从而实现旋转

然后直接在 Unity 编辑器中运行即可
