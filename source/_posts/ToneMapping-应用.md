---
title: ToneMapping-应用
date: 2022-05-18 14:00:42
tags: [ Shader, Unity]
categories: [ Shader]
about:
description: "UnityToneMapping URP 应用方案"
password: Xm4399
---

## ToneMapping

​	ToneMapping概念介绍请参考末尾[参考文献5](https://crazyink47.xyz/2022/02/09/HDR-ToneMapping-Bloom/)。

+ 首先，将ToneMapping添加到Volume组件

+ HDRP下面有四种模式

+ Neutral 中性映射方案，没有参数可以调整，使用该方案对比度可能降低
+ ACES 影视常用映射方案 没有参数 常用选取
+ Custom 自定义映射方案 暂略
+ Extrnal 外部映射方案 暂略

因笔者在这里使用URP作为渲染管线，因而没有Custom和Extrnal两种方案，在编辑器情况如下，需要上述介绍的请查看[参考文献3](https://www.jianshu.com/p/2ba6bb4e72da)。

![URPToneMapping](/images/URPToneMapping.png)

问题到这，事情就复杂了起来，我知道我想要什么曲线，但是要怎么让Unity也知道我知道的曲线呢..问题往下延伸..

+ 什么是Volumes?
+ 如何修改Volumes?
+ 如何向其中添加自定义效果？

## Volumes

高清渲染管线（HDRP）使用了Volumes架构。每一个Volumes都可以被设置为全局或者有局部边界（Global/Local）的模式。它们每一个都含有在Scence内设置的属性（property values），HDRP可以根据相机位置参与计算，并决定最终呈像值。举例而言，你可以采用local模式下的Volumes改变环境设置，比如修改雾气的颜色与浓度来改变你的场景气氛。

你可以添加 **Volume** 组件到任何一个GameObject上，包括Camera相机。但这里更推荐创建一个专门的GameObject负载每一个Volume组件。这个Volume组件本身并不含有任何实质上的数据，他通过引用一个处理数据的VolumeProfile来进行操作。这个VolumeProfile包含每一项参数的默认值，并在默认条件下将他们隐藏。为了查看或者修改这些参数，你必须添加Volume Component overrides，他们含有覆盖默认值的结构体，用于改变VolumeProfile。

Volumes同时包含其他参数来控制他们影响其他volumes。一个场景内可以含有多个Volumes。当Volume是 **Global** 的情况下，他会影响所有相机，在局部条件下，当相机观察到Volumes设计的边界区域时，他会参与计算。

在运行时，HDRP检测场景内所有启用（enabled）且添加到已激活的GameObject上的Volumes，并决定每个Volumes对场景最后设置的影响度。HDRP使用Camera参数和上述的Volumes参数来计算这个值。然后他们使用所有影响度不为0的Volumes来参与计算。

Volumes可以由多个不同的Volume Component组成。举例而言，一个Volume可以负责天空Volume而其他Volume负责雾气。

### Properties

![Properties](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@6.7/manual/Images/Volumes1.png)

| 参数           | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| Is Global      | 选取之后，Volume变为全局模式，影响整个场景                   |
| Blend Distance | 当Is Gloabl未勾选的情况下会显示此项，可以设置一个混合区域，该区域为从一个Volume到当前Volume时的渐变区域。 |
| Weight         | 混合程度，当Cmaera进入当前区域之后，决定最后的混合结果占比   |
| Priority       | 当两个Volume重叠时，根据此项来判断当前使用哪一个Volume的配置。数值越大越优先。 |
| Profile        | 使用Volume配置文件时，点击“New"新建一个配置文件。            |

+ Blend Distance 当Is Global未勾选时，需要在当前Volume所在物体上挂载Box Collider组件，并且勾选Box Collider组件的Is Trigger。

+ 混合区域是从Collider外侧边缘开始计算。

  例如Collider为4，Blend Distance为5，混合区域位置就是4到9。

+ 混合程度是 摄像机距离和混合距离 运算结果后，再乘以混合程度数值最终所得的结果。例如，当前Volume区域设置亮度为100，混合程度为0.5，在该Volume中最终亮度为50，当进入混合区域时，所有渐变的数值也会乘以0.5。

那么，什么是Volume得到了答案，那么要如何修改Volume？我之前做了一份官方文件的翻译。

但是很悲伤的是 ![PostProcessing](/images/PostProcessing.png)

显然这份文档过期了，后面的翻译就大多没有必要，需要的话可以在旁边的博客里找到他。

这个故事告诉我要先看源码再看文档。

## Override

现在可以在using UnityEngine.Rendering.Universal下找到新的处理方式。

![Bloom](/images/Bloom.png)

如上图，Volume下添加新的处理类需要继承VolumeComponent,IPostProcessComponent,并添加特性。

``` csharp
//此类用于调整Volume内参数，此处仅用于示范用法
public class VolumeTest : VolumeComponent, IPostProcessComponent
{
    [Range(0,100),Tooltip("整数参数")]
    public IntParameter IntTest = new IntParameter(0);

    [Range(0, 100), Tooltip("浮点参数")]
    public FloatParameter FloatTest = new FloatParameter(0);

    [Tooltip("布尔参数")]
    public BoolParameter BoolTest = new BoolParameter(false);
    
    public bool IsActive()=>true;

    public bool IsTileCompatible()=>true;
}
```

可以看到，这里使用IntParameter等参数代替原有的Int类型。

![VolumeParameter](/images/VolumeParameter.png)

如上图，这里的参数类型被Unity封装，继承VolumeParameter。

## ScriptableRenderPass

用于实现继承实现一个渲染通道，以拓展UPR。

``` csharp
	//这里截取了部分源码，源码里注释很详细
    [MovedFrom("UnityEngine.Rendering.LWRP")] public abstract class ScriptableRenderPass
    {
        /// <summary>
        /// 执行此通道，将自定义渲染效果呈现。这里实现具体细节。
        /// </summary>
        /// <param name="context">使用render context在执行期间进行渲染命令，你不需要手动调用ScriptableRenderContext.submit，渲染管线会在特定位置调用它。</param>
        /// <param name="renderingData">当前呈现的渲染状态信息</param>
        public abstract void Execute(ScriptableRenderContext context, ref RenderingData renderingData);
        
        /// <summary>
        /// 当完成camera渲染的时候触发回调，你可以使用这个回调函数来完成创建过程中的资源释放
        /// pass 需要在完成一次camera渲染的时候进行一次清理
        /// 这个方法可以被camera stack中的所有camera调用
        /// </summary>
        /// <param name="cmd">Use this CommandBuffer to cleanup any generated data</param>
        public virtual void FrameCleanup(CommandBuffer cmd)
        {}
    }
```

## ScriptableRendererFeature

你可以通过加入一个ScriptableRendererFeature到ScriptableRenderer来将这个scriptablerendererfeature加入到渲染器的渲染通道中。

``` csharp
    //同样截取部分源码
	[ExcludeFromPreset]
    [MovedFrom("UnityEngine.Rendering.LWRP")] public abstract class ScriptableRendererFeature : ScriptableObject
    {
        /// <summary>
        /// 初始化该特性的资源。每次序列化发生时都会调用这个函数。
        /// </summary>
        public abstract void Create();
        
        /// <summary>
        /// 在渲染器中注入一个或多个ScriptableRenderPass。
        /// </summary>
        /// <param name="renderPasses">List of render passes to add to.要添加到的渲染通道列表</param>
        /// <param name="renderingData">Rendering state. 用于设置render passes.</param>
        public abstract void AddRenderPasses(ScriptableRenderer renderer,
            ref RenderingData renderingData);
    }
```

### ScriptableRenderer

注意到这里，AddRenderPasses中传入第一个参数为ScriptableRenderer

ScriptableRenderer是为渲染逻辑设计的抽象类。他定义了剔除与照明等工作的基调。

渲染器可以被所有相机或每个被覆盖的基础相机使用。他将会在每一帧的执行，继承光照剔除，设置并定义一系列ScriptableRenderPass的步骤。渲染器可以通过添加ScriptableRendererFeature的方法来拓展额外的渲染效果。渲染的资源在ScriptableRendererData中序列化。

## CommandBuffer

综上所述，我们需要继承VolumeComponent, IPostProcessComponent的参数类，负责相机渲染实现的ScriptableRendererPass，以及最后将其加入渲染流程的ScriptableRendererFeature。道理是这么讲，但是实际需要怎么才能进行详细步骤完成渲染呢？

这里可以参照一下我之前写的博客。

[CommandBuffer and RenderTexture](https://crazyink47.xyz/2022/05/16/CommandBuffer%20and%20RenderTexture/)

## GTTonemap

这个映射曲线来自于[GTTonemap](https://www.desmos.com/calculator/gslcdxvipg?lang=zh-CN)，因为[相关的网站](https://www.artstation.com/artwork/wJZ4Gg)我暂时打不开，所以我也不知道有没有理论支撑，暂时作为可选项之一使用。

曲线如图所示。

![URPToneMapping](/images/GTTonemap.png)

```c++
static const float e = 2.71828;

float W_f(float x,float e0,float e1) {
	if (x <= e0)
		return 0;
	if (x >= e1)
		return 1;
	float a = (x - e0) / (e1 - e0);
	return a * a*(3 - 2 * a);
}
float H_f(float x, float e0, float e1) {
	if (x <= e0)
		return 0;
	if (x >= e1)
		return 1;
	return (x - e0) / (e1 - e0);
}

float GranTurismoTonemapper(float x) {
	float P = 1;
	float a = 1;
	float m = 0.22;
	float l = 0.4;
	float c = 1.33;
	float b = 0;
	float l0 = (P - m)*l / a;
	float L0 = m - m / a;
	float L1 = m + (1 - m) / a;
	float L_x = m + a * (x - m);
	float T_x = m * pow(x / m, c) + b;
	float S0 = m + l0;
	float S1 = m + a * l0;
	float C2 = a * P / (P - S1);
	float S_x = P - (P - S1)*pow(e,-(C2*(x-S0)/P));
	float w0_x = 1 - W_f(x, 0, m);
	float w2_x = H_f(x, m + l0, m + l0);
	float w1_x = 1 - w0_x - w2_x;
	float f_x = T_x * w0_x + L_x * w1_x + S_x * w2_x;
	return f_x;
}
```

需要其他ToneMapping曲线也可以参照[HDR_ToneMapping_Bloom](https://crazyink47.xyz/2022/02/09/HDR-ToneMapping-Bloom/)或者自拟映射曲线。

## 额外

``` csharp
int Shader.PropertyToID(string name);//用于获取Shader在Pass里的参数赋予ID
CoreUtiles.CreateEngineMaterial(shader);//UnityEngine.Rendering命名空间下，该方法将创建一个新材质并将其设置为在编辑器中隐藏，以确保不会将其另存为Asset，因此我们不必自己专门进行此操作。
```

## 修改文件

引用于 [参考文献9](https://developer.unity.cn/projects/618ce1e7edbc2a05bb615020)，该部分可用于直接修改Unity中已有的Volume效果。

![](https://connect-cn-cdn-public-prd.unitychina.cn/h1/20211111/p/images/9e0867e1-ad66-44e0-9064-70c74e1ba236___2021_11_11___5.48.43.png)

Unity的package包都放在Library\PackageCache路径下面，这个路径下面的包是受到Unity管理的，如果我们对它进行修改的话，Unity就会把我们的修改回滚掉。所以，我们需要把这个包拷贝出来，比如说拷贝到Assets目录的同级目录。

Package路径下面有一个叫做manifest.json文件，在这里面我们可以直接指定我们使用包的文件路径。在这里设置为我们拷贝的路径之后，我们就可以对它进行修改了，这个时候Unity就不会回滚修改。

![](https://connect-cn-cdn-public-prd.unitychina.cn/h1/20211111/p/images/f794072a-eb4f-4b81-a134-a3dd2cdfea27___2021_11_11___5.49.07.png)

**补充** ：我尝试过这个方案，ACES的效果可以按照上述方法修改，但可能会导致其他引用ACES的方案出错，同时又因直接修改文件的方法不利于维护，所以没有继续深入。

## 参考文献

1. [HDRCatlike](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/)
2. [机翻HDRCatlike](https://cloud.tencent.com/developer/article/1771261)
3. [Volume-ToneMapping](https://www.jianshu.com/p/2ba6bb4e72da)
4. [How to do custom tone mapping instead of neutral/aces in URP?](https://forum.unity.com/threads/how-to-do-custom-tone-mapping-instead-of-neutral-aces-in-urp.849280/)
5. [HDR_ToneMapping_Bloom](https://crazyink47.xyz/2022/02/09/HDR-ToneMapping-Bloom/)
6. [VolumesUnity官方文档](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@7.5/manual/Volumes.html)
7. [Volumes翻译参考1](https://zhuanlan.zhihu.com/p/72615352)
8. [如何在UnityURP中定义ToneMapping](https://www.cnblogs.com/yaoling1997/p/16029385.html)
9. [腾讯卡通渲染分享](https://developer.unity.cn/projects/618ce1e7edbc2a05bb615020)
10. [Manipulating the Stack](https://blog.csdn.net/cangod/article/details/92787721)
11. [Manipulating the Stack-Unity](https://docs.unity3d.com/Packages/com.unity.postprocessing@2.1/manual/Manipulating-the-Stack.html)
12. [Writing Custom Effects-Unity](https://docs.unity3d.com/Packages/com.unity.postprocessing@2.1/manual/Writing-Custom-Effects.html)
13. [Writing Custom Effects](https://blog.csdn.net/cangod/article/details/92789951)
14. [如何在URP中定义ToneMapping](https://www.cnblogs.com/yaoling1997/p/16029385.html)
15. [URP自定义后处理](https://zhuanlan.zhihu.com/p/385830661)
16. [GTTonemap](https://www.desmos.com/calculator/gslcdxvipg?lang=zh-CN)
17. [Unity为了更好用的后处理-拓展URP后处理踩坑记录](https://www.jianshu.com/p/b9cd6bb4c4aa?ivk_sa=1024320u)
