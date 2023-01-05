即便是在同一幅画面中，人眼所能观察到不同物体的细节程度是不一样的。因此，我们可以根据这种不同来合理的分配算力资源，让**距离越近、运动速度越慢、越靠近人眼视域中心**的物体，呈现更多细节；反之，则呈现更少细节。这种“合理”的算法，就是LOD（Level of Detail）技术了。与此同时，根据“细节层次”的高低，我们可以将相应的三维模型（物体）进行分级，从而诞生了一个关于“三维模型的LOD层级”的概念。

一般LOD都是基于模型与相机的距离来划分的。

## 用途

一般LOD都是基于模型与相机的距离来划分的，所以仅限于特定的游戏类型（相机与模型的距离会发生变化）。一些固定视角、固定相机距离的就不适用了，比如top down视角的游戏、2D游戏等。

## 优缺点

优点：简单易用，优化GPU耗时明显（对于远处直接使用了低精度模型，相当于直接减面）

缺点：

- 所有级别的模型都需要加载，因此磁盘和内存占用都会增加。
- Unity中的LODGroup中只有最高层次细节的模型才会参与静态光照的烘焙，这种情况只能使用光照探针等实现。

## 过渡

不同LOD之间必然存在突变（不是连续变化的），一般通过颜色抖动（dither）、透明度渐变来过渡

## Unity中的LODGroup组件

[统一 - 手册：LOD 组 (unity3d.com)](https://docs.unity3d.com/Manual/class-LODGroup.html)

## Unity shader中的LOD

其实就是根据设备性能的不同编译不同版本的Shader。

注意LOD的数值不要随意定，一些自带shader是有LOD值的，因此自定义shader的LOD应该参考这些值。

必须按LOD降序放置子着色器。因为 Unity 会选择它找到的第一个有效的子着色器，因此如果它首先找到一个具有较低 LOD 的子着色器，它将始终使用该着色器。

``` glsl
Shader "LODTest" 
{
    Properties
    {
        _MyLOD("_MyLOD",Float) = 50
    }
    
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 200
        ...
    }

    SubShader
    {
        Tags { "RenderType"="Opaque"}
        LOD 100
        ...
    }
    
    SubShader
    {
        Tags { "RenderType"="Opaque"}
        LOD [_MyLOD]
        ...
    }
}
```

参考：[Unity - Manual: ShaderLab: assigning a LOD value to a SubShader (unity3d.com)](https://docs.unity3d.com/Manual/SL-ShaderLOD.html)