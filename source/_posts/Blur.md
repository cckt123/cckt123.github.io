---
title: Blur
date: 2022-06-21 16:49:52
tags: [ Shader, Unity]
categories: [ Shader]
about:
description: "模糊，模糊，还有模糊。"
password: Xm4399
---

## 前言

这篇博客应用于Unity模糊处理，具体使用算法为高斯模糊、表面模糊，应用于URP下的UGUI。

这里主要的问题是要模糊的强度。

高斯模糊，可以获取到_Size作为参数，但是控制\_Size的值并不会增加模糊的强度，而是会增加高斯模糊的采样半径，因而当\_Size变大的同时，图像将会出现多重重影，而不是预期的模糊增强效果。

``` c++
_Size("Size", Range(0, 255)) = 1
#define GRABPIXELH(weight,kernely) tex2Dproj(_MainTex, float4(i.uvgrab.x, i.uvgrab.y + _CameraOpaqueTexture_TexelSize.y * kernely*_Size*1.61, i.uvgrab.z, i.uvgrab.w)) * weight
```

网络上常见的方案为使用多次高斯模糊处理，反复迭代来达到增加强度的效果。但是这里的高斯模糊为截取屏幕而发生了改动，且网络上的方案多是适应Volume，因而，如何在现有条件下达到更好的模糊效果就成了问题。

## 更多模糊

高斯模糊有这个问题，那么不用高斯模糊会怎么样？这里参考[高品质后处理：十种图像模糊算法的总结与实现](https://zhuanlan.zhihu.com/p/125744132)给出下图。

![模糊](http://uwa-ducument-img.oss-cn-beijing.aliyuncs.com/Blog/USparkle_PostProcessing/6.png)

### 表面模糊

![SurfaceBlur](/images/Blur.png)

表面模糊是另一个议题，上图是PS中的菜单。这里写上他的原因是因为做过表面模糊的尝试，但实际上表面模糊非常耗时，不建议使用该方法进行实时模糊处理。参考来源于[图像算法---表面模糊算法](https://trent.blog.csdn.net/article/details/49864397?spm=1001.2101.3001.6650.7&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-7-49864397-blog-106872945.pc_relevant_multi_platform_whitelistv1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-7-49864397-blog-106872945.pc_relevant_multi_platform_whitelistv1&utm_relevant_index=14)。

![SurfaceBlur公式](/images/SurfaceBlur.jfif)

表面模糊对每个像素设置一个半径r，得到一个边长为(2r+1)的正方形窗口，窗口中心像素的像素值即为x，对于像素的RGB三个分量进行分别计算，得到结果。

首先，表面模糊的滤波矩阵大小不确定，这意味着他需要在每次计算的同时额外计算一次滤波器，其次，他会RGC三分量进行一次计算，这会导致计算的数量级*3。

``` c++
half4 frag(v2f i) : SV_Target
{

	half4 sum = half4(0,0,0,0);
	half4 w = half4(0, 0, 0, 0);
	half4 cur = half4(0, 0, 0, 0);			
	for (int h = -_Size; h < _Size; h++) 
	{
		for (int v = -_Size; v < _Size; v++) 
		{
			cur = 1 - (abs(tex2Dproj(_CameraOpaqueTexture, float4(i.uvgrab.x+v, i.uvgrab.y + h, i.uvgrab.z, i.uvgrab.w))))/(2.5* _Threshold);
			cur = max(cur, 0);
			sum += cur * tex2Dproj(_CameraOpaqueTexture, float4(i.uvgrab.x, i.uvgrab.y, i.uvgrab.z, i.uvgrab.w));
			w += cur;
		}
	}
	return sum / w;
}
```

### for and if

像是上面这么写是可以通过编译的，但是在实现的同时，会带来一份新的效率问题。UnityShader在GPU上运行，因而在使用if或者for语句时都会涉及到指令跳转，流水线停滞。

GPU进行计算的运算单元和执行指令的逻辑单元并不是一对一而是一对多的关系。

当执行指令时，并不是操作一个像素，而是同时操作一组像素，并一条一条向下执行。如果在代码里出现了if，在这组像素中，有可能返回true，也可能返回false，而不执行的像素就会陷入等待。

最后结论是

- if和for确实有效率问题（相比CPU下的情况），但是通常情况无法回避，也就无需处理。
- 用step代替if，大部分情况是无优化或者负优化，只在一些个别情况下有少量的提升。
- 可以用Shader变体来取代一些只和固定参数有关的if，但这只是为了减少if自身的成本（if和里面执行的表达式毕竟也是指令）

### 粒状模糊

实际运行中，上述的SurfaceBlur表现并不好。这里尝试使用粒状模糊代替高斯模糊和表面模糊使用，其可在单Pass下获得合适的模糊表现，同时性能出色。

![**Grainy Blur**](https://pic2.zhimg.com/80/v2-bcd34ff16a9328d604b0a4cd26fe163d_720w.jpg)

这里的粒度模糊实现参照了 [Grainy Blur](https://github.com/QianMo/X-PostProcessing-Library/tree/master/Assets/X-PostProcessing/Effects/GrainyBlur)。

``` c++
float Rand(float2 n)
{
	return sin(dot(n, half2(1233.224, 1743.335)));
}

half4 GrainyBlur(v2f i)
{
	half2 randomOffset = float2(0.0, 0.0);
	half4 finalColor = half4(0.0, 0.0, 0.0, 0.0);
	float random = Rand(i.uvgrab);

	for (int k = 0; k < int(_Iteration); k++)
	{
		random = frac(43758.5453 * random + 0.61432);
		randomOffset.x = (random - 0.5) * 2.0;
		random = frac(43758.5453 * random + 0.61432);
		randomOffset.y = (random - 0.5) * 2.0;

		finalColor += tex2Dproj(_CameraOpaqueTexture, float4(half2(i.uvgrab + randomOffset * _BlurRadius).x, half2(i.uvgrab + randomOffset * _BlurRadius).y, i.uvgrab.z, i.uvgrab.w));
	}
	return finalColor / _Iteration;
}
```

## 更多迭代

上面的方案看起来很不错，但是不行。原因在于粒度模糊提供的模糊效果过于粗糙。因而需要采取其他的模糊方案。

### URP相关

那为什么不直接用模糊质量好的高斯模糊或者XX模糊呢？因为是其他模糊效果大都需要多Pass，而URP不再支持多Pass。如何在URP下达成原有的多Pass渲染效果就成了问题。粒状模糊的好处就在于单Pass，简单易用。其次，在实际场景之中，因场景内后处理叠加，_CameraOpaqueTexture并不能很好的满足满足实际需求。

这里参考[在URP里写Shader](https://zhuanlan.zhihu.com/p/336428407)引用。

在Built-In管线里面，我们可以使用多Pass实现描边、毛发等等效果，但是在URP中就需要改变实现方式了。一般来说，Render Feature可以解决大部分问题，可以通过它实现大部分多Pass的效果。

那么问题变成了什么是Render Feature？如何使用Render Feature？直觉告诉我这里是个大坑。这里[另开一贴](https://crazyink47.xyz/2022/06/23/RenderFeature/)。

### 处理迭代

总而言之，到这里大概知道如何实现多Pass的模糊效果了，但是如何要进行多次迭代处理，让模糊效果明显呢？

这里参照[【unity URP】自定义后处理](https://zhuanlan.zhihu.com/p/385830661)

``` csharp
        for (int i = 0; i < testBlur.Iteration.value; i++)
        {
            cmd.GetTemporaryRT(destination, w / 2, h / 2, 0, FilterMode.Point, RenderTextureFormat.Default);
            cmd.Blit(destination, source, testBlurMaterial, shaderPass);
            cmd.Blit(source, destination);
            cmd.Blit(destination, source, testBlurMaterial, shaderPass + 1);
            cmd.Blit(source, destination);
        }
        for (int i = 0; i < testBlur.Iteration.value; i++)
        {
            cmd.GetTemporaryRT(destination, w * 2, h * 2, 0, FilterMode.Point, RenderTextureFormat.Default);
            cmd.Blit(destination, source, testBlurMaterial, shaderPass);
            cmd.Blit(source, destination);
            cmd.Blit(destination, source, testBlurMaterial, shaderPass + 1);
            cmd.Blit(source, destination);
        }

        cmd.Blit(destination, destination, testBlurMaterial, 0);
```



## 参考文献

+ [高品质后处理：十种图像模糊算法的总结与实现](https://blog.uwa4d.com/archives/USparkle_PostProcessing.html)
+ [图像算法---表面模糊算法](https://trent.blog.csdn.net/article/details/49864397?spm=1001.2101.3001.6650.7&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-7-49864397-blog-106872945.pc_relevant_multi_platform_whitelistv1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-7-49864397-blog-106872945.pc_relevant_multi_platform_whitelistv1&utm_relevant_index=14)
+ [shader中用for，if等条件语句为什么会使得帧率降低很多？](https://www.zhihu.com/question/27084107)
+ [Shader中if和for的效率问题以及使用策略](https://www.sohu.com/a/219886389_667928)
+ [在URP里写Shader](https://zhuanlan.zhihu.com/p/336428407)
+ [【Free Bird/URP教学】7.URP后处理--高斯模糊](https://zhuanlan.zhihu.com/p/483506609)
+ [【unity URP】自定义后处理](https://zhuanlan.zhihu.com/p/385830661)
+ [FullScreenBlurWithURP](https://forum.unity.com/threads/fullscreen-blur-with-urp.883987/)
