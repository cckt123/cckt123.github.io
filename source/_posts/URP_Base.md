---
title: URP_Base
date: 2022-07-05 15:07:20
tags: [ Shader, Unity]
categories: [ Shader]
about:
description: "Unity URP 学习记录"
---

## 前言

缅怀我那看了忘，忘了又看的URP基础，真是一点功夫都偷不得。这里大部分都是转载内容，转载来源在末尾参考文献。

## SRP Batcher

SRP Batcher工作原理不同于动态与静态合批，他通过ConstantBuffer将对象的信息储存到GPU上，从而避免重复提交数据带来的性能损失。

使用SRP Batcher，需要在URP Asset中勾选启用，并且有对应支持的Shader。如果是自定义的ShaderLab,需要将缓存到CBuffer的变量写到CBuffer_Start(UnityPerMaterial)和CBUFFER_END之间。

``` c++
CBUFFER_START(UnityPerMaterial)
    float4 _BaseMap_ST;
    float4 _BaseColor;
CBUFFER_END
```

## 单Pass

URP管线不再支持多个Pass，准确来说，他不再支持多个渲染Pass。传统前向渲染管线采用的ForwardBase Pass计算主光，ForwardAdd Pass计算其他光照叠加的实现方式。光源越多物体越多调用的Pass数量也就越多。而在URP中采用的是单Pass的光照计算，在一个Pass中遍历光源进行关照计算。

## URP Unlit Shader

``` csharp
Shader "URPCustom/Unlit"
{
    Properties
    {
        _BaseMap ("Base Texture",2D) = "white"{}
        _BaseColor("Base Color",Color)=(1,1,1,1)
    }
    SubShader
    {
        Tags
        {
            "RenderPipeline"="UniversalPipeline"//这是一个URP Shader！
            "Queue"="Geometry"
            "RenderType"="Opaque"
        }
        HLSLINCLUDE
         //CG中核心代码库 #include "UnityCG.cginc"
        #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
       
        //除了贴图外，要暴露在Inspector面板上的变量都需要缓存到CBUFFER中
        CBUFFER_START(UnityPerMaterial)
        float4 _BaseMap_ST;
        half4 _BaseColor;
        CBUFFER_END
        ENDHLSL

        Pass
        {
            Tags{"LightMode"="UniversalForward"}//这个Pass最终会输出到颜色缓冲里

            HLSLPROGRAM //CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            struct Attributes//这就是a2v
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD;
            };
            struct Varings//这就是v2f
            {
                float4 positionCS : SV_POSITION;
                float2 uv : TEXCOORD;
            };

            TEXTURE2D(_BaseMap);//在CG中会写成sampler2D _MainTex;
            SAMPLER(sampler_BaseMap);

            Varings vert(Attributes IN)
            {
                Varings OUT;
                //在CG里面，我们这样转换空间坐标 o.vertex = UnityObjectToClipPos(v.vertex);
                VertexPositionInputs positionInputs = GetVertexPositionInputs(IN.positionOS.xyz);
                OUT.positionCS = positionInputs.positionCS;

                OUT.uv=TRANSFORM_TEX(IN.uv,_BaseMap);
                return OUT;
            }

            float4 frag(Varings IN):SV_Target
            {
                //在CG里，我们这样对贴图采样 fixed4 col = tex2D(_MainTex, i.uv);
                half4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);                
                return baseMap * _BaseColor;
            }
            ENDHLSL  //ENDCG          
        }
    }
}
```

## URP Tag

专门为URP写Shader的话，应该打上对应的Tags让渲染管线识别到。

``` c++
SubShader
{
 Tags {
     "RenderPipeline"="UniversalPipeline"
    ……
  }
}
```

## 头文件

使用的底层代码也发生了变化，默认包含的头文件需要从CG的 **#include "UnityCG.cginc"**改为**#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"**

除了核心的工具库外，其他如光照、阴影等要包含的头文件也需要修改。

使用了不同的头文件后，自然一些使用各种工具函数也不一样，比如顶点着色器中转换空间的方法UnityObjectToClipPos(v.vertex);在URP中就使用GetVertexPositionInputs(IN.positionOS.xyz);来获取一个存储了各个空间Position坐标的结构体，再从中获取裁剪空间中的位置坐标。

## **CBUFFER**

为了支持SRP Batcher（具体看前一篇），Shader中要将所有暴露出的参数（贴图除外）给包含到**CBUFFER_START(UnityPerMaterial)**与**CBUFFER_END**之间。并且为了保证之后的每个Pass都能拥有一样的CBUFFER，这一段代码需要写在SubShader之内，其它Pass之前。

## **贴图采样**

在CG中我们一般使用sampler2D _Texture来采样贴图。在HLSL中，贴图采样需要声明为Sampler。具体的采样方法也从原来的tex2D(Texture tex,float uv)；变为SAMPLE_TEXTURE2D(Texture tex, Sampler sampler,float uv);

```c
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);
……
float4 frag(Varings IN):SV_Target
{
    ……
    half4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);     
    ……
}
```



## 参考文献

+ [【Unity Shader】在URP里写Shader（一）](https://zhuanlan.zhihu.com/p/336428407)
+ [【Unity Shader】在URP里写Shader（二）](https://zhuanlan.zhihu.com/p/336508199)
+ 
+ [Unity官方文档](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@7.4/manual/index.html)

