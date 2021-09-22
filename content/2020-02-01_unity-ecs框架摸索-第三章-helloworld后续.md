+++
title = "Unity ECS框架摸索 第三章 HelloWorld后续"
date = 2020-02-01 20:34:04

[taxonomies]
tags = ["Unity ECS"]
categories = ["Unity"]
+++

上一章介绍了ECS并行处理的实现，接下来就这个HelloWorld进行深入的研究。

<!-- more -->

## 添加Component的技巧

第二章中为Entity添加Component的方式是直接在GameObject上挂载脚本，但是Unity编辑器目前并非为ECS设计，如果为每一个Component都挂载一个脚本会很麻烦，所以考虑使用代码添加组件。

我们编写如下脚本，把它挂载在Cube上，它就能为Cube生成的Entity添加RotationSpeed组件。  
RotatingCubeAuthoring.cs

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

解释：IConvertGameObjectToEntity接口的Convert方法会在GameObject转换的过程中被调用，可以在其中直接用EntityManager为Entity添加任意Component，并能对其赋值，更加灵活。  
在该段代码中我们就把直观的Degree转换成了Radian。

再把之前的RotationSpeed组件修改一下  
RotationSpeed.cs

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

解释：原本的GenerateAuthoringComponent特性被改成了Serializable。GenerateAuthoringComponent的作用是允许ECS的组件直接挂载在GameObject上，但是使用代码添加组件时不需要这个特性，因此只需要Serializable即可。

在编辑器中修改Cube的RotatingCubeAuthoring的旋转速度为每秒180度，如下

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191210-160141@2x.png)

## IJobForEach

上一章中，我们使用Lambda对Entity进行了遍历，写法很简洁。但是这样的写法会产生GC，为了更好的性能，可以使用IJobForEach接口进行遍历。

修改RotationSpeedSystem为如下代码

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

解释：原本使用Lambda对Job进行规划。修改后使用一个结构体实现IJobForEach接口。在其中规定了Entity的筛选条件为拥有Rotation和RotationSpeed组件，并实现Execute方法规划Job。有效避免了GC。
