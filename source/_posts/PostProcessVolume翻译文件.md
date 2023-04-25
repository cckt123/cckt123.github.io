---
title: PostProcessVolume翻译文件
date: 2022-05-26 15:27:10
tags: [ Shader, Unity]
categories: [ Shader]
about:
description: "Manipulating the Stack"
---

## 注意

这玩意可能被迭代了。

## Manipulating the Stack/操作后期栈

在开发游戏的过程中，您经常需要为基于时间的事件或者临时情况下需要在栈上加入或者覆写效果。你可以在场景内动态创建一个全局Volume，创建profile，创建一个新的overriders，将他们完成配置并分配数据，但是这样的操作非常繁琐。

因而我们提供了一个QuickVolume方法在场景内快速生成Volumes

``` csharp
public PostProcessVolume QuickVolume(int layer,float priority,params PostProcessEffectSettings[] settings)
```

前两个参数不必多说，最后一个参数要接收在Volume中覆写的效果数据。

举例说明可以很直观的展现出用法，例如需要创建一个Vignette效果并覆盖它的enabled&intensity字段

``` csharp
var vignette = ScriptableObject.CreateInstance<Vignette>();
vignette.enabled.Override(true);
vignette.intensity.Override(1f);
```

现在让我们看一个稍微复杂一点的效果，让我们从脚本创建一个vignetter effect 效果

``` csharp
using UnityEngine;
using UnityEngine.Rendering.PostProcessing;

public class VignettePulse : MonoBehaviour
{
    PostProcessVolume m_Volume;
    Vignette m_Vignette;

    void Start()
    {
        m_Vignette = ScriptableObject.CreateInstance<Vignette>();
        m_Vignette.enabled.Override(true);
        m_Vignette.intensity.Override(1f);

        m_Volume = PostProcessManager.instance.QuickVolume(gameObject.layer, 100f, m_Vignette);
    }

    void Update()
    {
        m_Vignette.intensity.value = Mathf.Sin(Time.realtimeSinceStartup);
    }

    void OnDestroy()
    {
        RuntimeUtilities.DestroyVolume(m_Volume, true, true);
    }
}
```

这段代码创建了一个新的vignetter效果，并为其分配了一个新的Volume，将其优先级设置为100.然后再每一帧根据正弦函数来改变他的强度。

**注意** ：当你不再需要他的时候，要销毁Volume和profile。

### Fading Volumes/淡出体

基于距离的Volume混合对于大多数关卡设计来说都很不错，但是偶尔你也会向根据游戏玩法事件来触发淡入和淡出效果。你可以使用上一节描述的方法手动完成，也可以使用tweening库完成这项工作。

这里的例子使用DOTween.

``` csharp
using UnityEngine;
using UnityEngine.Rendering.PostProcessing;
using DG.Tweening;

public class VignettePulse : MonoBehaviour
{
    void Start()
    {
        var vignette = ScriptableObject.CreateInstance<Vignette>();
        vignette.enabled.Override(true);
        vignette.intensity.Override(1f);

        var volume = PostProcessManager.instance.QuickVolume(gameObject.layer, 100f, vignette);
        volume.weight = 0f;

        DOTween.Sequence()
            .Append(DOTween.To(() => volume.weight, x => volume.weight = x, 1f, 1f))
            .AppendInterval(1f)
            .Append(DOTween.To(() => volume.weight, x => volume.weight = x, 0f, 1f))
            .OnComplete(() =>
            {
                RuntimeUtilities.DestroyVolume(volume, true, true);
                Destroy(this);
            });
    }
}

```

本例中，同样使用quick volume来生成一个vignette。我们可以将他的权重设置为0，因为我们不希望他有任何影响。然后使用DOTween的排序特性来链接一组渐变事件，淡入，暂停，淡出并销毁体积和组件本身。

### Profile Editing 配置文件编辑

[原文文档](https://docs.unity3d.com/Packages/com.unity.postprocessing@2.1/manual/Writing-Custom-Effects.html)

您还可以手动编辑Volume上的现有配置文件。这非常类似于在Unity中编写材质脚本。有两种方法可以做到这一点:直接修改共享配置文件，或者只用于Volume的共享配置文件的克隆体。

每种方法都有一些优点和缺点:

Shared  profile editing

1. 更改将应用于使用相同配置文件的所有Volume
2. 修改实际的asset，在退出Playmode的情况下不会重置
3. 字段名:sharedProfile

Owned profile editing

1. 改变将只应用于指定的Volume
2. 退出播放模式的情况下重置
3. 注意销毁
4. 字段名：profile

## Writing Custom Effects

这个架构允许你书写自定义后处理代码效果，同时将他们插入栈中而不需要修改代码库。当然，所有在该架构下编写的效果都可以任意与Volume效果混合，除非某些依赖于循环的特性之外，他们还可以自动使用将要应用的SRP。

让我们写一个非常简单的灰度效果来举例。

自定义效果至少需要两项文件：一项C#和一项HLSL源文件（需要注意，HLSL被Unity交叉编译为GLSL,Metal和其他APL,所以他不止局限于DirectX）。

**注意** : 这个快速启动指南需要适当的C#与Shader编程知识，我们不会在这里讨论每一个细节，可以把它看做一个概述而不是深入的教程。

``` csharp
using System;
using UnityEngine;
using UnityEngine.Rendering.PostProcessing;

[Serializable]
[PostProcess(typeof(GrayscaleRenderer), PostProcessEvent.AfterStack, "Custom/Grayscale")]
public sealed class Grayscale : PostProcessEffectSettings
{
    [Range(0f, 1f), Tooltip("Grayscale effect intensity.")]
    public FloatParameter blend = new FloatParameter { value = 0.5f };
}

public sealed class GrayscaleRenderer : PostProcessEffectRenderer<Grayscale>
{
    public override void Render(PostProcessRenderContext context)
    {
        var sheet = context.propertySheets.Get(Shader.Find("Hidden/Custom/Grayscale"));
        sheet.properties.SetFloat("_Blend", settings.blend);
        context.command.BlitFullscreenTriangle(context.source, context.destination, sheet, 0);
    }
}

```

**注意** ： 此代码必须存储于Grayscale.cs文件中。因Unity中的序列化工作，你必须确保文件是使用你设置的类名，否则他将无法正确序列化。

我们需要两个类，一个用于存储数据，另一个用来处理渲染部分。

### Settings

Settings类用于保存我们效果的数据，这些都是为用户使用的字段，你可以在Inspector面板找到它们。

``` csharp
[Serializable]
[PostProcess(typeof(GrayscaleRenderer), PostProcessEvent.AfterStack, "Custom/Grayscale")]
public sealed class Grayscale : PostProcessEffectSettings
{
    [Range(0f, 1f), Tooltip("Grayscale effect intensity.")]
    public FloatParameter blend = new FloatParameter { value = 0.5f };
}
```

首先，你需要确保该类继承了PostProcessEffectSettings，并且序列化，所有不要忘记使用[Serializable]。

然后，你需要告诉Unity这是一个包含后处理数据的类，这也就是[PostProcess()]属性的作用。第一个参数将设置链接到渲染器，第二个参数效果的执行点，现有三种可选项。

1. BeforeTransparent:该效果在渲染透明Pass之前渲染。
2. BeforeStack：该效果将在内置栈启动之前应用，包括抗锯齿，景深，toneMapping等。
3. AfterStack: 在内置栈之后，和FXAA和final-pass抖动之前应用。

第三个参数是效果的菜单项，你可以使用/来进行子菜单类别的使用。

最后，还有一个可选项的第四个参数allowInSceneView，顾名思义，他代表支持或不支持在Scene场景中的显示效果。默认为true，但是你可能想要临时禁用这段代码或者不希望Scene场景变得太乱。

对于参数本身，您可以使用所需的任何类型，但是如果您希望这些参数在Volume中可覆写和可混合，则必须使用已装箱的字段。在本例中，我们只需添加一个浮动参数，其范围固定从0到1。您可以通过在/PostProcessing/Runtime/中浏览ParameterOverride.cs源文件来获得完整的内置参数类列表，也可以在同一源文件中按照下面方式完成轻松创建自己的内置参数。

注意，您还可以覆盖PostProcessEffectSettings的IsEnabledAndSupported()方法来设置您自己对效果的需求(如果需要特定的硬件)，甚至在满足条件之前悄悄的禁用掉效果。例如，在我们的例子中，如果混合参数为0，我们可以像这样自动禁用效果:

``` csharp
public override bool IsEnabledAndSupported(PostProcessRenderContext context)
{
    return enabled.value
        && blend.value > 0f;
}
```

这样效果将不会执行，除非 blend> 0。

### Renderer 渲染

现在让我们看看渲染逻辑。我们的渲染器拓展了PostProcessEffectRenderer，其中T是要附加到渲染器的设置类型。

``` csharp
public sealed class GrayscaleRenderer : PostProcessEffectRenderer<Grayscale>
{
    public override void Render(PostProcessRenderContext context)
    {
        var sheet = context.propertySheets.Get(Shader.Find("Hidden/Custom/Grayscale"));
        sheet.properties.SetFloat("_Blend", settings.blend);
        context.command.BlitFullscreenTriangle(context.source, context.destination, sheet, 0);
    }
}
```

Render()方法中发生的所有事情都以PostProcessRenderContext为参数，这个Context包含很多有用的数据，你可以使用这些数据并在处理渲染效果的阶段传递这些数据。通过检查/PostProcessing/Runtime/PostProcessRenderContext.cs，以获得可用内容的列表(该文件有大量注释)。

PostProcessEffectRenderer 还有一些可以覆写的方法，例如

+ void Init(): called when the renderer is created 渲染器被渲染的时候
+ DepthTextureMode GetLegacyCameraFlags():用于设置相机Flag和请求深度图、运动矢量等。
+ void ResetHistory(): 在调用"reset history"事件时调用，主要用于清除历史缓冲区等。
+ void Release(); 渲染器销毁时调用。

我们的效果还蛮简单的，只需要做两件事

+ 将blend参数发送到shader
+ 使用原图作为输入，将shader pass 全屏传输到目标。

因为我们只使用命令缓冲区，所以系统依赖于MaterialPropertyBlock来存储着色器数据。您不需要自己创建它们，因为框架会自动为您创建池，以节省时间并确保性能是最佳的。我们只需要为着色器请求一个属性表并在其中统一设置。

最后，我们使用context提供的CommandBuffer和源图、目标、属性表和pass序号传输一个全屏pass。

这就是c#部分的内容。

### Shader

编写自定义效果着色器也相当简单，但是在使用它之前，您应该知道一些事情。这个框架大量使用宏来抽象平台差异，使您的工作更轻松。兼容性是关键，对于即将到来的脚本化呈现管道更是如此。

``` c++
Shader "Hidden/Custom/Grayscale"
{
    HLSLINCLUDE

        #include "Packages/com.unity.postprocessing/PostProcessing/Shaders/StdLib.hlsl"

        TEXTURE2D_SAMPLER2D(_MainTex, sampler_MainTex);
        float _Blend;

        float4 Frag(VaryingsDefault i) : SV_Target
        {
            float4 color = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.texcoord);
            float luminance = dot(color.rgb, float3(0.2126729, 0.7151522, 0.0721750));
            color.rgb = lerp(color.rgb, luminance.xxx, _Blend.xxx);
            return color;
        }

    ENDHLSL

    SubShader
    {
        Cull Off ZWrite Off ZTest Always

        Pass
        {
            HLSLPROGRAM

                #pragma vertex VertDefault
                #pragma fragment Frag

            ENDHLSL
        }
    }
}

```

首先要注意的是:我们不再使用CG块了。如果将来与脚本渲染管道的兼容性对您很重要，那么不要使用它们，因为它们会在转换时破坏着色器，因为CG块添加了您不想添加到着色器中的隐藏代码。使用HLSL块代替。

至少您需要包含StdLib.hlsl。它包含预先配置的顶点着色器和不同的结构体(VertDefault、VaryingsDefault)，以及编写通用效果所需的大部分数据。

贴图声明是使用宏来完成的。要获得可用宏的列表，我们建议您查看/PostProcessing/Shaders/ api /中的一个api文件。

除此之外，其余的是标准的着色器代码。在这里，我们计算了当前像素的亮度，我们使用_Blend统一对像素颜色和亮度进行lerp，然后返回结果。

**重要提示** :如果着色器在你的任何场景中都没有被引用，当在编辑器之外运行游戏时，它就不会被编译，效果也不会起作用。要么将其添加到Resources文件夹，要么将其放入Edit -> Project Settings -> Graphics中的Always include着色器列表中。

### Effect ordering 效果次序

内置效果是自动排序的，但是自定义效果呢?一旦你创建了一个新的效果或者将它导入到你的项目中，它就会被添加到你相机的后期处理Layer组件的自定义效果排序列表中。

它们将按插入点预先排序，但您可以随意重新排序。次序是每层的，这意味着您可以对每个相机使用不同的次序方案。

### Custom editor

默认情况下，设置类的编辑器会自动为您创建。但有时您需要对字段的显示方式有更多的控制。与经典的Unity组件一样，您也可以创建自定义编辑器。

**重要提示** :与经典编辑器一样，您必须将这些文件放入编辑器文件夹中。

如果我们复制灰度效果的默认编辑器，它会是这样的:

``` csharp
using UnityEngine.Rendering.PostProcessing;
using UnityEditor.Rendering.PostProcessing;

[PostProcessEditor(typeof(Grayscale))]
public sealed class GrayscaleEditor : PostProcessEffectEditor<Grayscale>
{
    SerializedParameterOverride m_Blend;

    public override void OnEnable()
    {
        m_Blend = FindParameterOverride(x => x.blend);
    }

    public override void OnInspectorGUI()
    {
        PropertyField(m_Blend);
    }
}
```

## 
