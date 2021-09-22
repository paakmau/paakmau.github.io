+++
title = "Unity ECS框架摸索 第二章 HelloWorld"
date = 2020-01-31 15:15:00

[taxonomies]
tags = ["Unity ECS"]
categories = ["Unity"]
+++

## 最终效果

<!-- more -->

跟随本文，创建一个立方体Entity，利用一个System控制它不断旋转。

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191210-142039-HD.gif)

## 步骤

首先写一个控制旋转速度的组件  
RotationSpeed.cs

```cs
[GenerateAuthoringComponent]
public struct RotationSpeed : IComponentData
{
    public float RadiansPerSecond;
}
```

然后在Unity编辑器中创建一个Cube的GameObject，挂载ConvertToEntity与刚刚写好的RotationSpeed，并设置RotationSpeed的初始值。具体如下图。

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191210-143136@2x.png)

解释：  
这个Cube本身是一个GameObject，但是挂上ConvertToEntity脚本之后，他会被转化为Entity，他的Transform与Mesh信息也会被转化为相应的Component。  
其中ConversionMode选择ConvertAndDestroy代表转化为Entity之后原本的GameObject会被删除。  
  
此外，我们挂载的RotationSpeed同样也会成为这个Entity的组件。

至此，我们添加了一个具有RotationSpeed组件的Cube实体

接下来写一个控制旋转的系统  
RotationSpeedSystem.cs

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

解释：JobComponentSystem的抽象方法OnUpdate会在每帧调用，重载该方法并在其中规划Job就能实现对相应Component的并行读写。  
  
上面的代码中利用ForEach传入一个Lambda函数，这个函数筛选拥有Rotation（Unity自带）与RotationSpeed（我们刚刚写的）组件的Entity，然后根据RotationSpeed的信息修改Rotation，从而实现旋转

然后直接在Unity编辑器中运行即可

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191210-142039-HD.gif)
