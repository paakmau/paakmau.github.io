+++
title = "UE4材质编辑器中材质结点的数据结构 源码浅析"
tags = []
categories = ["UE4"]
date = 2020-05-15 23:47:36
+++

UE4的材质编辑器中我们可以创建材质结点然后给它们连线来创建材质

本文将根据源码解析材质结点在内存中的数据结构

<!-- more -->

## UEdGraph

先简单提一下UEdGraph

它用来管理一张可视化的有向图，而且其实蓝图就是用这个东西来做的  
一个UEdGraph对象会包括图中所有的UEdGraphNode也就是结点  
一个结点包括位置等基础信息，然后它还有一个UEdGraphPin的指针数组  
一个UEdGraphPin其实就是一个插槽，用一个字节表示是输入插槽还是输出插槽，然后持有它所属的UEdGraphNode指针，还有一个UEdGraphNode指针数组保存这个插槽连接到的结点

## UMaterialGraph

它派生自UEdGraph，保存一个材质在编辑器中的所有结点形成的有向图。虽然它主要用于管理由材质结点组成的可视化的有向图，但它也持有UMaterial指针用来获取材质输入插槽等信息  
然后用户在编辑器中对材质的修改都会作用到这个对象上

它有一个UMaterialGraphNode\_Root类型的成员变量RootNode，这个RootNode其实就是编辑器中的材质结果结点，通过这个结点向前递归我们可以找到这个材质结果结点连接的所有结点，不过其实它的基类UEdGraph本身就具有这样的功能

## UMaterialGraphNode\_Base

根据名字我们知道它是材质结点的基类，派生自UEdGraphNode

比如前面提到的UMaterialGraphNode\_Root就是它的一个派生类；此外它的派生类还有UMaterialGraphNode，这个派生类用于管理材质结果结点之外的其他结点

这两个派生类之间的主要区别在于获取插槽等结点数据的方式不同

UMaterialGraphNode\_Root获取插槽数据是通过UEdGraphNode的GetGraph方法获取所属的UEdGraph指针，然后类型转换到UMaterialGraph指针，再通过UMaterialGraph的MaterialInputs得到材质结果结点上的各个输入

下面贴一下它的CreateInputPins的实现

```cpp
void UMaterialGraphNode_Root::CreateInputPins()
{
    UMaterialGraph* MaterialGraph = CastChecked<UMaterialGraph>(GetGraph());

    for (const FMaterialInputInfo& MaterialInput : MaterialGraph->MaterialInputs)
    {
        UEdGraphPin* InputPin = CreatePin(EGPD_Input, UMaterialGraphSchema::PC_MaterialInput, *FString::Printf(TEXT("%d"), (int32)MaterialInput.GetProperty()), *MaterialInput.GetName().ToString());
    }

}
```

然后UMaterialGraphNode管理的是其他结点，其实也就是表达式结点，它持有一个UMaterialExpression指针，于是通过这个指针直接就能拿到结点表达式的输入输出的类型和名称

UMaterialExpression  
结点表达式  
  
因为不同类型的结点会具有不同的输入输出和功能，所以一个结点就是一个具有若干输入和输出表达式  
  
但是UMaterialExpression只是一个接口，因为不同表达式的输入输出不同，因此具体的表达式都会派生自UMaterialExpression，比如UMaterialExpressionAdd就是具体的加法表达式

这里简单贴一下它的CreateInputPins的实现

```cpp
void UMaterialGraphNode::CreateInputPins()
{
    const TArray<FExpressionInput*> ExpressionInputs = MaterialExpression->GetInputs();

    for (int32 Index = 0; Index < ExpressionInputs.Num() ; ++Index)
    {
        FExpressionInput* Input = ExpressionInputs[Index];
        FName InputName = MaterialExpression->GetInputName(Index);

        InputName = GetShortenPinName(InputName);

        const FName PinCategory = MaterialExpression->IsInputConnectionRequired(Index) ? UMaterialGraphSchema::PC_Required : UMaterialGraphSchema::PC_Optional;

        UEdGraphPin* NewPin = CreatePin(EGPD_Input, PinCategory, InputName);
        if (NewPin->PinName.IsNone())
        {
            // Makes sure pin has a name for lookup purposes but user will never see it
            NewPin->PinName = CreateUniquePinName(TEXT("Input"));
            NewPin->PinFriendlyName = SpaceText;
        }
    }
}
```
