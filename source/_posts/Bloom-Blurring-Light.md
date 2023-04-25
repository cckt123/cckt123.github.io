---
title: Bloom
date: 2022-02-11 14:14:42
tags: [ Shader, Unity]
categories: [ Shader]
about:
description: "catlikecoding Bloom翻译"
---

# 前述

这篇文章来源于[HDR_ToneMapping_Bloom](https://crazyink47.xyz/2022/02/09/HDR-ToneMapping-Bloom/)这篇文章的后续，参考文献16[Bloom Blurring Light](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/)给了我很大帮助，大到我觉得翻译出来应该会更好。

# Bloom

> 渲染到一个临时纹理（a temporary texture）
>
> 通过上下调整分辨率模糊
>
> 采样
>
> 盒式过滤
>
> 应用Bloom到原图

本篇教程介绍如何向Camera添加Bloom效果，且默认您已经了解[Rendering](https://catlikecoding.com/unity/tutorials/rendering/part-1/)系列教程相关内容。

本篇教程使用Unity2017.3.0p3制作

![*A little bloom makes bright things glow.*](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/tutorial-image.jpg)

*查看[Custom SRP / Post Processing](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/)得到更新的Bloom教程.*（PS：这里有个[中文翻译](https://zhuanlan.zhihu.com/p/339443207)）

# 1 设置场景

绝大多数设备可以显示的光源是有限的。从RGB三原色的值从0到1，从黑色到纯白，是就是低动态范围，LDR。纯白的亮度在不同的显示设备上有所不同，但是绝对不会亮到无法观察。

真实生活之中的亮度远不止于LDR。事实上并没有最大亮度，随着光子在同一时间的不断聚集，亮度只会越来越高，直到你盯着它都会感觉到眼睛被灼烧，就像是直视太阳一样。

为了表示异常明亮的色彩，我们制造了HDR，他允许亮度超过最大值1.Shader处理这个当然没有问题，只要我们允许他计算的颜色数值大于1，但是设备所能展示的亮度是有限的，我们最终的色彩还是会回到LDR上。

为了使HDR色彩可见，他们必须映射回到LDR，这个步骤称之为tonemapping色调映射。简单来说就是图像的非线性变暗，使其可以被显示。这有点类似于我们的眼睛如何适应过亮的环境，虽然tonemapping映射是不会发生系数变化的。还有自动曝光技术，可以动态调整图像亮度。两者可以同时使用。但我们的眼睛也并不能适应所有情况。有些场景太过明亮，让我们肉眼也很难观察。在限制LDR显示的情况下，我们要如何显示这种效果?

Bloom是一种通过使像素的颜色影响相邻像素来混合图像的效果。这就像模糊图像，但是基于亮度。这样我们就可以通过模糊来传达过亮的色彩。这有点类似于光线在我们眼睛里的漫反射，在高亮度的情况下Bloom会变得很明显，但毕竟是一个模拟效果。

许多人不喜欢Bloom，因为他会干扰原本清晰的图像制造出不真实的光源。这并不是Bloom的缺陷，而是他被过于频繁的实用的原因。如果你的设计目标是有真实感的场景，请适度使用他。Bloom相对适合使用在非真实艺术效果上。

## 1.1 Bloom Scene

我们将通过一个Camera的后处理组件创建我们自己的Bloom效果，类似于我们如何在[Rendering14,Fog](https://catlikecoding.com/unity/tutorials/rendering/part-14/)中创建延迟的雾效。你可以从一个新的项目开始或从那个教程继续，但我使用了之前的高级渲染教程，[Surface Displacement](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/)，作为这个项目的基础。

[Surface Displacement unitypackage](https://catlikecoding.com/unity/tutorials/advanced-rendering/surface-displacement/culling-triangles/culling-triangles.unitypackage)

创建一个使用默认照明的新场景。放一些明亮的物体进去，在深色的背景前。我使用了一个黑色的Plane和一些白、黄、绿、红的立方体或者球体。确保Camera设置为HDR。同时将项目设置为线性色彩空间以更好的观测效果。

![Bright objects](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/setting-the-scene/scene.png)

通常情况下，我们会将ToneMapping应用到场景中的线性与HDR渲染的情况下。你可以先做自动曝光，然后应用Bloom，最后使用ToneMapping。但在本教程中，我们将专注于Bloom，不会应用任何其他效果。这意味着所有超出LDR的颜色将被映射在最终的图像中。

## 1.2 Bloom Effect

创建一个新的BloomEffect脚本，就像是DeferredFogEffect一样，让他在编辑模式下运行，并给他一个**OnRenderImage**方法。最初他并不做任何额外的处理，只是进行源数据到目标渲染纹理的比特数据传输。

``` csharp
using UnityEngine;
using System;

[ExecuteInEditMode]
public class BloomEffect : MonoBehaviour {

	void OnRenderImage (RenderTexture source, RenderTexture destination) {
		Graphics.Blit(source, destination);
	}
}
```

让我们将这个效果同时添加到场景视图之中，这样我们可以更容易的观察到效果变化。这只需要添加**ImageEffectAllowedInSceneView**特性到类之前就可以。

``` csharp
[ExecuteInEditMode, ImageEffectAllowedInSceneView]
public class BloomEffect : MonoBehaviour {
	…
}
```

将这个脚本添加Camera下，也只添加这个脚本，这样我们的测试场景就完成了。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/setting-the-scene/camera.png)

[unitypackage](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/setting-the-scene/setting-the-scene.unitypackage)

# 2 Blurring 模糊

Bloom效果是创建模糊图像，然后与原图像重叠，因而我们需要先创建模糊图像再进行后续步骤。

## 2.1 Rendering to Another Texture 渲染到一个中间纹理

应用一个效果需要在从一个渲染纹理过渡到另一个的过程之中应用，如果我们能够在一次pass之中就将工作完成，那么我们就只需要进行一次简单的从源数据到渲染目标的数据传输。但是模糊需要很多步骤，因而我们需要一个中间的过渡。

获取临时纹理的方式最好是调用**RenderTexture.GetTemporary**。这个方法负责为我们管理临时纹理，根据Unity的需要创建、缓存和销毁它们。我们只需要指定指定纹理的尺寸即可。这里将源纹理的大小作为参数填入。

```csharp
	void OnRenderImage (RenderTexture source, RenderTexture destination) {
		RenderTexture r = RenderTexture.GetTemporary(
			source.width, source.height
		);
		
		Graphics.Blit(source, destination);
	}
```

当我们需要模糊图像时，我们不需要对深度缓存区做任何更改，所以第三个参数传入0。

``` csharp
		RenderTexture r = RenderTexture.GetTemporary(
			source.width, source.height, 0
		);
```

因为我们需要使用HDR，所以要确保使用适当的纹理格式。当Camera开启HDR时，源数据的纹理格式是正确的，所以我们可以直接使用他。他也有可能使用其他格式，比如说ARGBHalf。

``` csharp
		RenderTexture r = RenderTexture.GetTemporary(
			source.width, source.height, 0, source.format
		);
```

现在，不再直接从源数据到渲染目标进行比特传输，而是先从源纹理到临时纹理进行比特传输，然后再从临时纹理到目标纹理。

``` csharp
//		Graphics.Blit(source, destination);
		Graphics.Blit(source, r);
		Graphics.Blit(r, destination);
```

在此之后，如果我们不需要temporary texture。同时要再次使用他，必须调用**RenderTexture.ReleaseTemporary**释放内存。

``` csharp
		Graphics.Blit(source, r);
		Graphics.Blit(r, destination);
		RenderTexture.ReleaseTemporary(r);
```

尽管结果没有发生什么变化，但是我们使用了一个临时纹理来存储他了。

## 2.2 Downsampling 降低像素采样

模糊图像的运转机理是取周围像素均值。我们必须决定每一个像素选择周围哪些像素，被选中的这些像素在过滤器上的系数。可以预料，这个计算亮会随着像素数量增加而不断递增。因而我们为了减少这个计算量，要尽可能的减少传入的像素个数。最简单且最迅捷的方法是利用GPU内置的双线性滤波。如果我们将临时纹理的分辨率减半，那么原本的4个像素就会合成1个像素。低分辨率的像素将被精确地采样在最初的四个像素之间，取它们的平均值。我们甚至不需要使用自定义着色器来进行处理。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/bilinear-downsampling.png)

``` csharp
		int width = source.width / 2;
		int height = source.height / 2;
		RenderTextureFormat format = source.format;

		RenderTexture r =
			RenderTexture.GetTemporary(width, height, 0, format);
```

使用一半尺寸的纹理意味着我们将会降低原本图像一半的分辨率，而之后我们在从临时纹理传递到目标纹理的过程之中要再次将尺寸翻倍，让其返回源纹理大小，让分辨率返回正常的范畴。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/bilinear-upsampling.png)

这两步将会把四个像素混合在一起，作为一个像素返还。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/bilinear-weights.png)

最后的结果将会被原本的图像更加模糊。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/half-size.png)

我们可以进一步减少中间纹理的尺寸来达到降低更多分辨率的效果。

## 2.3 Progressive Downsamping 逐步向下采样

但是遗憾是的直接降低过多倍数的尺寸会导致较差的结果，引擎通常会放弃计算过多的像素，而是只会取中心的四个像素作为参考混合成为一个像素。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/double-downsample.png)

一个更好的处理方案是通过多次降低分辨率，直到抵达需要降低的目标，这样就可以混合每一个像素，得到更好的结果。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/progressive-downsample.png)

为了控制降低的次数，我们额外设置一个参数，同时添加[Range]方便控制。在1-16的范畴足够将65536^2大小的像素块缩减到1个像素大小，大概够了。

``` csharp
	[Range(1, 16)]
	public int iterations = 1;
```

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/iterations.png)

为了完成这项工作，我们首先要重构**r**为**currentDestination**。当完成第一次比特流传输（bilt）时，添加一个命名非常直接的**currentSource**值并且将**currentDestination**赋给他，之后最后使用一次将其传递给源纹理，同时释放。

``` csharp
		RenderTexture currentDestination =
			RenderTexture.GetTemporary(width, height, 0, format);
			
		Graphics.Blit(source, currentDestination);
		RenderTexture currentSource = currentDestination;
		Graphics.Blit(currentSource, destination);
		RenderTexture.ReleaseTemporary(currentSource);
```

现在我们可以把声明**currentSource**到最后的**Blit**之间的代码放入一个循环，控制纹理空间释放处理，交换模糊后的图像与原图像存储，让每一次循环将纹理的分辨率缩减一倍。

``` csharp
		RenderTexture currentSource = currentDestination;

		for (int i = 1; i < iterations; i++) {
			width /= 2;
			height /= 2;
			currentDestination =
				RenderTexture.GetTemporary(width, height, 0, format);
			Graphics.Blit(currentSource, currentDestination);
			RenderTexture.ReleaseTemporary(currentSource);
			currentSource = currentDestination;
		}

		Graphics.Blit(currentSource, destination);
```

当我们将width或者height除到0的时候我们就不应该继续了，需要跳出循环。这里只判断了height的条件，因为常规显示器的宽度是大于高度的，而且因为显示到单像素的同时并不会有实际意义，所以我们将处理的阈值提升到2.

``` csharp
			width /= 2;
			height /= 2;
			if (height < 2) {
				break;
			}
```

## 2.4 Progressive Upsampling  逐步向上采样

虽然逐步向下采样有所改善获得的像素质量，但是这个结果依然还是太方块了。让我们试试逐步提高采样的分辨率是否会有所帮助。

两个方向上的迭代意味着我们最终会对除了最小的尺寸之外的每个等级的尺寸进行两次渲染。与其在每次渲染中释放并声明相同的纹理两次，不如让我们在一个数组中记录它们。这里简单地使用一个大小固定为16的数组字段，应该足够了。

``` csharp
RenderTexture[] textures = new RenderTexture[16];
```

每次当我们获取一个临时纹理的同时将其加入数组。

``` csharp
		RenderTexture currentDestination = textures[0] =
			RenderTexture.GetTemporary(width, height, 0, format);
		…

		for (int i = 1; i < iterations; i++) {
			…
			currentDestination = textures[i] =
				RenderTexture.GetTemporary(width, height, 0, format);
			…
		}
```

之后添加第二个循环。这一次是从最底层的纹理开始的。我们可以将**iterations**从第一个循环中提出来，减去2，并将其作为另一个循环的起点。第二个循环往回走，将**iterations**一直减到0。这个循环是我们应该释放源纹理的地方，而不是在第一个循环中。同样的，我们应该清理一下这里的数组。

``` csharp
		int i = 1;
		for (; i < iterations; i++) {
			…
			Graphics.Blit(currentSource, currentDestination);
//			RenderTexture.ReleaseTemporary(currentSource);
			currentSource = currentDestination;

		}

		for (i -= 2; i >= 0; i--) {
			currentDestination = textures[i];
			textures[i] = null;
			Graphics.Blit(currentSource, currentDestination);
			RenderTexture.ReleaseTemporary(currentSource);
			currentSource = currentDestination;
		}
```

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/progressive-up-5.png)



这个结果好，但是不够好。

## 2.5 Custom Shading 

为了改善我们的模糊效果，我们必须换到一个更高级的滤波处理算法，而不是简单的双线性滤波。这需要我们使用一个自定义Shader，所以创建一个新的Bloom Shader。就像DeferredFogShader一样，从一个最简单的Shader开始，它有一个_MainTex属性，没有剔除，也不使用深度缓冲区。给他一个Pass通道和一个顶点和片元着色器。

``` c++
Shader "Custom/Bloom" {
	Properties {
		_MainTex ("Texture", 2D) = "white" {}
	}

	SubShader {
		Cull Off
		ZTest Always
		ZWrite Off

		Pass {
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram
			ENDCG
		}
	}
}
```

这个顶点着色器甚至比之前的雾效果还简单。他只需要传递uv坐标并将顶点坐标转换到裁剪空间。因为我们将会创建多个Pass，所以除了片元着色器之外的都可以在**CGINCLUDE**块中定义。

``` C++
	Properties {
		_MainTex ("Texture", 2D) = "white" {}
	}

	CGINCLUDE
		#include "UnityCG.cginc"

		sampler2D _MainTex;

		struct VertexData {
			float4 vertex : POSITION;
			float2 uv : TEXCOORD0;
		};

		struct Interpolators {
			float4 pos : SV_POSITION;
			float2 uv : TEXCOORD0;
		};

		Interpolators VertexProgram (VertexData v) {
			Interpolators i;
			i.pos = UnityObjectToClipPos(v.vertex);
			i.uv = v.uv;
			return i;
		}
	ENDCG

	SubShader {
		…
	}
```

我们将会定义**FragmentProgram**在Pass通道内。在这里，我们先只是普通的采样纹理，然后染成红色以确认我们在使用我们的custom shader渲染。通常HDR颜色以半精度格式(half-precision)存储，所以我们使用half而不是float，尽管这对于非移动平台没有区别。

``` C++
		Pass {
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					return tex2D(_MainTex, i.uv) * half4(1, 0, 0, 0);
				}
			ENDCG
		}
```

在我们的脚本中添加一个公共字段来保存对这个Shader的引用，并将其在inspector中设置。

``` C++
	public Shader bloomShader;
```

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/shader.png)

添加一个字段来保存将使用这个着色器的材质，它不需要序列化。在渲染之前，检查我们是否有这个材质，如果没有创建它。我们不需要在inspector中看到它，也不需要保存它，所以相应地设置它的hideFlags。

``` C++
	[NonSerialized]
	Material bloom;
	
	void OnRenderImage (RenderTexture source, RenderTexture destination) {
		if (bloom == null) {
			bloom = new Material(bloomShader);
			bloom.hideFlags = HideFlags.HideAndDontSave;
		}
		
		…
	}
```

每次我们调用Blit的时候都使用这个材质来处理，而不是使用默认的材质。

```csharp
	void OnRenderImage (RenderTexture source, RenderTexture destination) {
		…
		Graphics.Blit(source, currentDestination, bloom);
		…
			Graphics.Blit(currentSource, currentDestination, bloom);
		…
			Graphics.Blit(currentSource, currentDestination, bloom);
		…
		Graphics.Blit(currentSource, destination, bloom);
		…
	}
```

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/red.png)

## 2.6 Box Sampling

我们要调整着色器好让他使用不同的方法进行滤波。因为采样取决于像素大小，所以在**CGINCLUDE**块中添加float4 _MainTex_TexelSize变量。请记住，这对应于源纹理的texel大小，而不是目标纹理。

``` C++
		sampler2D _MainTex;
		float4 _MainTex_TexelSize;
```

因为我们只对主纹理进行采样，且只关心RGB通道，所以来创建一个简单的采样函数

``` C++
		half3 Sample (float2 uv) {
			return tex2D(_MainTex, uv).rgb;
		}
```

我们使用一个简单的盒式滤波器，而不是默认的双线性滤波器。盒式取4个而不是1个样本，对角放置，这样我们就得到了4个相邻的2×2像素块的平均值。将这些样本相加，然后除以4，最终得到4×4像素块的平均值，使我们的处理范围大小翻倍。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/box-downsampling.png)

``` c++
		half3 SampleBox (float2 uv) {
			float4 o = _MainTex_TexelSize.xyxy * float2(-1, 1).xxyy;
			half3 s =
				Sample(uv + o.xy) + Sample(uv + o.zy) +
				Sample(uv + o.xw) + Sample(uv + o.zw);
			return s * 0.25f;
		}
```

使用盒式过滤的代码代替之前片元着色器之中测试的代码。

```C++
				half4 FragmentProgram (Interpolators i) : SV_Target {
//					return tex2D(_MainTex, i.uv) * half4(1, 0, 0, 0);
					return half4(SampleBox(i.uv), 1);
				}
```

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/box-sampling.png)

## 2.7 Different Passes

结果更平滑，质量更高，但也更模糊。这主要是由于使用了新的4×4盒式滤波器。当我们决定最后具体的像素点时，我们会参考极大范畴的像素，而没有权重划分，因而画面失去了焦距。

<img src="https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/box-upsampling-1.png" style="zoom:50%;" />



<img src="https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/box-upsampling-1-weights.png" style="zoom:50%;" />

我们可以调整盒式滤波器的选择范畴来调整采样的结果，这里我们需要重新设置参数，而不是将范围固定设置为1。

``` c++
		half3 SampleBox (float2 uv, float delta) {
			float4 o = _MainTex_TexelSize.xyxy * float2(-delta, delta).xxyy;
			half3 s =
				Sample(uv + o.xy) + Sample(uv + o.zy) +
				Sample(uv + o.xw) + Sample(uv + o.zw);
			return s * 0.25f;
		}
```

复制一次我们的着色器，第一个设置为pass0，用于向下采样，使用原先的数值1，第二个设置为pass1,用于向上采样，使用间距设置为0.5。

``` C++
		Pass { // 0
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					return half4(SampleBox(i.uv, 1), 1);
				}
			ENDCG
		}

		Pass { // 1
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					return half4(SampleBox(i.uv, 0.5), 1);
				}
			ENDCG
		}
```

当UV值为0.5时，我们会将重叠的采样点覆盖在3×3盒子上。因此，一些像素对结果的贡献不止一次，增加了它们的权重。所有样本中包含的中间像素，对角线像素只使用一次，其他像素出现两次。结果是一个更集中的向上采样。

接下来，我们必须指出在调用**Blit**应该使用哪个传递。为了简化这个过程，可以在BloomEffect中添加常量，这样我们就可以使用名称而不是索引来处理。

```csharp
	const int BoxDownPass = 0;
	const int BoxUpPass = 1;
```

前两次向下，后两次向上。

``` csharp
	void OnRenderImage (RenderTexture source, RenderTexture destination) {
		…
		Graphics.Blit(source, currentDestination, bloom, BoxDownPass);
		…
			Graphics.Blit(currentSource, currentDestination, bloom, BoxDownPass);
		…
			Graphics.Blit(currentSource, currentDestination, bloom, BoxUpPass);
		…
		Graphics.Blit(currentSource, destination, bloom, BoxUpPass);
		…
	}
```

在这一步上，我们有一个简单而有效的过滤方式获得模糊的图像。但同时我们有很多方式可以用来代替她，每一种方式都有她的优缺点，在实际运用的过程之中要视情况决定，不过我们在这里只使用这一种方式。

[package](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/blurring/blurring.unitypackage)

# 3 Creating Bloom

创建模糊图像是制作Bloom效果的第一步。而第二步自然是将模糊后的图像与原图像叠加，照亮他。然而我们并不会直接使用上面行为结束之后的模糊图片，她是一个相对整体的模糊结果。而实际光照过程中，距离物体较近的部分光源更亮，因而她需要一个权重划分，让靠近物体的地方更加明亮。我们可以通过叠加中间量来完成这一点，在逐步向上采样的过程中将结果叠加。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/additive-blurring.png)

## 3.1 Additive Blending 

在我们已经有的中间层级添加只需要添加Blend，而不需要再替换纹理。我们要做的只是在Pass1里添加**One One**。

``` c++
		Pass { // 1
			Blend One One

			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					return half4(SampleBox(i.uv, 0.5), 1);
				}
			ENDCG
		}
```

这个方法可以简单而有效的处理中间纹理，但是当我们渲染到最后的时候会遇到错误。因为我们还没有对他进行渲染，也就没有源纹理与其叠加。我们可能会在最后累积光照强度，叠加色素，或者其他事情，这取决于Unity将会如何做。为了解决这个问题，我们不得不创建一个单独的Pass通道用于处理最后一个图像，在那里我们将最后一个图像与原图像结合。为了达成这个目的，我们添加一个Shader变量来处理源纹理。

``` c++
		sampler2D _MainTex, _SourceTex;
```

添加第三次pass，这是第二次pass的拷贝，除了她使用了默认的混合模式，并将盒式过滤采样结果混合到源纹理的采样。

``` C++
		Pass { // 2
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					half4 c = tex2D(_SourceTex, i.uv);
					c.rgb += SampleBox(i.uv, 0.5);
					return c;
				}
			ENDCG
		}
```

为这个Pass定义一个常量，用于叠加最后的图像。

``` csharp
	const int BoxDownPass = 0;
	const int BoxUpPass = 1;
	const int ApplyBloomPass = 2;
```

最后次blit必须使用这个pass以得到正确的结果。

```csharp
//		Graphics.Blit(currentSource, destination, bloom, BoxUpPass);
		bloom.SetTexture("_SourceTex", source);
		Graphics.Blit(currentSource, destination, bloom, ApplyBloomPass);
		RenderTexture.ReleaseTemporary(currentSource);
```

## 3.2 Bloom Threshold

到现在为止，我们还在模糊整个图像。明亮的像素都会带有Bloom效果，但这不合理。Bloom应该只应用在最明亮的那些像素部分。为了实现这个一步，我们应该引入一个亮度阈值。添加一个public变量到我们的脚本，用于控制亮度范畴，默认为1，除外LDR像素。

``` csharp
	[Range(0, 10)]
	public float threshold = 1;
```

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/threshold.png)

这个值决定了哪些像素可以执行Bloom操作，如果他们不够明亮，他们就不应该包含在上下逐步采样的过程之中。为了完成这步操作，可以简单的将其设置为黑色，这一点必须由着色器Shader完成。所以我们要在Blit之前设置材质的_Threshold。

``` CS
		if (bloom == null) {
			…
		}

		bloom.SetFloat("_Threshold", threshold);
```

将这个变量也添加到着色器的**CGINCLUDE**块之中，并为她复制half值。

``` c++
		half _Threshold;
```

我们将会使用这个阈值来筛选掉我们不需要的像素。当我们在模糊阶段开始时期就这么处理的时候，将其称之为预处理阶段。创建一个函数专门用于处理这种情况，函数用于获取并返回过滤后的颜色。

``` c++
		half3 Prefilter (half3 c) {
			return c;
		}
```

我们将会使用颜色的最大分量来决定他的最大亮度，所以**b=Cr∨Cg∨Cbb=Cr∨Cg∨Cb**这里的v是用来表达计算最大值的公式。

我们可以通过将颜色分量减去阈值，然后除以亮度来确定最后颜色的贡献率。 c=(b−t)/b，这里的t是阈值。当t=0时，结果总是1，这意味着颜色将不会发生变化。当t的数值增加时，这条光亮曲线将会向下弯曲，直到在b=t时抵达零点。得益于曲线的形状，他被称之为膝盖。因为我们不需要负数，因而我们要确保b-t的值大于等于0，所以c=0V(b-t)/b。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/threshold-graph.png)

为了保证在Shader之中不会有0作为除数，我们要选取一个足够小的值作为最小值，像是0.000001，之后将其作为除数替换。

``` c++
		half3 Prefilter (half3 c) {
			half brightness = max(c.r, max(c.g, c.b));
			half contribution = max(0, brightness - _Threshold);
			contribution /= max(brightness, 0.00001);
			return c * contribution;
		}
```

这个过滤器只会在第一次过滤时使用，因而复制第一个pass通道，将其放置于Pass0之上，应用这个结果在盒式过滤之前。

``` c++
		Pass { // 0
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					return half4(Prefilter(SampleBox(i.uv, 1)), 1);
				}
			ENDCG
		}
```

添加一个新的常量用于表达新的Pass通道，之后的Pass通道向后+1。

``` csharp
	const int BoxDownPrefilterPass = 0;
	const int BoxDownPass = 1;
	const int BoxUpPass = 2;
	const int ApplyBloomPass = 3;
```

使用新的pass通道来进行第一次blit

``` csharp
		RenderTexture currentDestination = textures[0] =
			RenderTexture.GetTemporary(width, height, 0, format);
		Graphics.Blit(source, currentDestination, bloom, BoxDownPrefilterPass);
		RenderTexture currentSource = currentDestination;
```

这时将阈值设置为1，你可能几乎看不到，或者干脆就看不到Bloom效果，表示光线或者材质没有使用HDR的值。为了使Bloom效果出现，我们可以增加一点光线使得材质发光。举例而言，我使黄色的材质自发光，这样我们就可以将黄色像素推到HDR范畴进行发光。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/yellow-hdr.png)

## 3.3 Isolating Bloom

为了看到图像的哪些部分在进行Bloom，如果我们可以看到单独的Bloom效果就会方便很多。所以我们在脚本之中设置一个Debug的Bool值用于控制该效果。

``` csharp
public bool debug;
```

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/debug.png)

我们需要创建一个debug通道，因而需要一个额外的常量表示新的Pass通道。

``` csharp
	const int BoxDownPrefilterPass = 0;
	const int BoxDownPass = 1;
	const int BoxUpPass = 2;
	const int ApplyBloomPass = 3;
	const int DebugBloomPass = 4;
```

在Debug模式下，使用Debug-Blit将中间结果直接传递到最终目标渲染纹理，取代原本的代码程序。

``` csharp
		if (debug) {
			Graphics.Blit(currentSource, destination, bloom, DebugBloomPass);
		}
		else {
			bloom.SetTexture("_SourceTex", currentSource);
			Graphics.Blit(source, destination, bloom, ApplyBloomPass);
		}
		RenderTexture.ReleaseTemporary(currentSource);
```

新的Debug-Pass只执行最后一次简单的向上采样然后无条件合并。

``` c++
		Pass { // 4
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					return half4(SampleBox(i.uv, 0.5), 1);
				}
			ENDCG
		}
```

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/bloom-only.png)

在Debug模式之下，我们可以清晰的看到黄色的像素最后生成了Bloom。除此之外还有大量的白色像素，当他们反射了大量平行光光源的时候会显示出来。

## 3.4 Soft Threshold

我们制作的颜色曲线通过零点，这意味着Bloom将会在某一个值之后发生骤然的截断，这也是这条曲线被称之为hard knee的原因。我们产生Bloom的区域与不产生Bloom的区域将会急速过渡。可以在上图的白色球体之中明显的观察到，他有个不太明显的过渡，但是依然可以看到明显的过渡外轮廓线。为了使这条曲线更加光滑，让轮廓不在明显。我们重新制作了一个新的变量，变量为0时，我们采取现有的过渡方式，当值为1时，我们使用完全平滑曲线淡出得到的hard knee曲线。我们使用0.5作为默认值。

``` csharp
	[Range(0, 1)]
	public float softThreshold = 0.5f;
```

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/soft-threshold.png)

这种淡出过程也是由着色器来实现的，因而我们将这个数值传递到着色器当中。

``` csharp
		bloom.SetFloat("_Threshold", threshold);
		bloom.SetFloat("_SoftThreshold", softThreshold);
```

在Shader之中添加对应的参数。

``` c++
		half _Threshold, _SoftThreshold;
```

所以我们通过淡化的过程，将hard knee 转换为 soft knee。我们取b-t与软化曲线的最大值来代替原本的0.

**c = sV(b-t)/b**

软化曲线默认**s = (b-t+k)^2/4k**，当k = tts 且ts是软化阈值。

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/soft-knees.png)

我们要在这条曲线与0相交，与hard knee相交的同时截断他。要这么做，我们只需要...

**s = (0V(b-t+k)∧2k)^2/4k** 这里∧表示取最小值。

将其添加到预处理函数，然后再次将其除以0.

``` c++
		half3 Prefilter (half3 c) {
			half brightness = max(c.r, max(c.g, c.b));
			half knee = _Threshold * _SoftThreshold;
			half soft = brightness - _Threshold + knee;
			soft = clamp(soft, 0, 2 * knee);
			soft = soft * soft / (4 * knee + 0.00001);
			half contribution = max(soft, brightness - _Threshold);
			contribution /= max(brightness, 0.00001);
			return c * contribution;
		}
```

请注意，softknee函数在某些部分可以被单独抽出，他们只需要固定值，而不需要每次都将参数传入。我们可以预先处理好这些值，以矢量的方式将其传入Shader。在预处理阶段将其计算完成。

``` csharp
//		bloom.SetFloat("_Threshold", threshold);
//		bloom.SetFloat("_SoftThreshold", softThreshold);
		float knee = threshold * softThreshold;
		Vector4 filter;
		filter.x = threshold;
		filter.y = filter.x - knee;
		filter.z = 2f * knee;
		filter.w = 0.25f / (knee + 0.00001f);
		bloom.SetVector("_Filter", filter);
```

相应的调整Shader

``` C++
//		half _Threshold, _SoftTheshold;
		half4 _Filter;
		
		…
		
		half3 Prefilter (half3 c) {
			half brightness = max(c.r, max(c.g, c.b));
//			half knee = _Threshold * _SoftThreshold;
			half soft = brightness - _Filter.y;
			soft = clamp(soft, 0, _Filter.z);
			soft = soft * soft * _Filter.w;
			half contribution = max(soft, brightness - _Filter.x);
			contribution /= max(brightness, 0.00001);
			return c * contribution;
		}
```

## 3.5 Bloom Intensity

最后，让我们来调节Bloom效果的强度。这使淡入成为可能，也可以用它来创造出非常强烈的Bloom效果。

``` csharp
	[Range(0, 10)]
	public float intensity = 1;
```

![](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/intensity.png)

将这个强度值作为参数传给Shader。通常使用gamma space的数值来设置Bloom的强度，因而将其从linear space转换到gamma。

``` csharp
		bloom.SetVector("_Filter", filter);
		bloom.SetFloat("_Intensity", Mathf.GammaToLinearSpace(intensity));
```

在Shader之中添加对应值。

```c++
		half _Intensity;
```

在最后两道pass工序之中，将强度参数传入。

```c++
		Pass { // 3
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					half4 c = tex2D(_SourceTex, i.uv);
					c.rgb += _Intensity * SampleBox(i.uv, 0.5);
					return c;
				}
			ENDCG
		}

		Pass { // 4
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					return half4(_Intensity * SampleBox(i.uv, 0.5), 1);
				}
			ENDCG
		}
```

你现在有了一个基础的Bloom效果，他和Unity后处理技术栈中的版本2的Bloom效果非常相似。这个方法可以通过添加颜色、使采样间隔可配置、使用不同的过滤器等方式进一步扩展。或者你可以移步到[Depth of Field](https://catlikecoding.com/unity/tutorials/advanced-rendering/depth-of-field/)

[unitypackage](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/creating-bloom.unitypackage)
