---
title: HDR_ToneMapping_Bloom
date: 2022-02-09 15:18:37
tags: [ Shader, Unity]
categories: [ Shader]
about:
description: "HDR_ToneMapping_Bloom联系与介绍"
---

## HDR?

常见的图片格式，例如jpg/png可以在广义上被划分为LDR，低动态范围图像。LDR这个概念为8位图片。有RGB三通道，每个通道2^8种灰度值。但LDR这个范围有它的局限性。LDR可以表现出我们大部分常见的图片色彩，但在人类的视觉的范畴，许多色彩在LDR之中并没有呈现，相应的就有了HDR，单通道位数超过8位，常见有12/16位，相应可呈现的图片色彩呈指数级上升，计算方式为2^16^3。

### 如何设置HDR？

![HDRSet](/images/HDRSET.png)

如图，默认Camera使用HDR是Use Graphics Settings，Graphics在默认平台下的HDR不一定是打开的，具体而言可以在其面板自行设置，也可以直接修改Camera下的HDR设置用于关闭HDR选项。

![](/images/GraphicsSet.png)



## ToneMapping?

我们有了HDR，但是使用的时候涉及到问题，HDR是一个相对宽泛的概念，她**常见**可能就有12/16位两种，**即使**高端的显示器材可以呈现HDR，也需要考虑是哪一类型的HDR。因而如何将复杂的**HDR图片转换为LDR图片**格式显示成为了课题。这种转换方式被称之为**色调映射（ToneMapping）**。 而对于Unity而言，Unity（2019.4.8f1）当前**不支持**HDR**显示**。假定所有显示均为LDR sRGB。和上述HDR的设置有细微的差别，这里的支持显示是指不支持直接显示，但是可以通过HDR数据进行计算，利用HDR我们可以进行更高质量的LDR/sRGB显示。

![HDR2LDR](/images/HDR2LDR.jpg)

常用的转换方式例如线性映射，可以将数据直接按比例讲HDR数据换算到LDR范畴。但显然，如果线性映射可以解决这个问题，就不需要这么长的篇幅介绍了。 

![LinearMap](/images/LinearMap.jpg)

这里借用一张图片，上方图片为线性映射之后得到的结果，可以很明显的观察到台灯的亮度超出合理的阈值，阴影遮蔽了不少有效信息。总结来说，图片中亮的地方太亮，暗的地方太暗。他需要更好的映射方式来处理高光和灰暗的部分。

到这里为止，我们对ToneMapping有了一个直观的概念，然后这里有一段对他相对书面而缜密的解释。

Tone Mapping原是摄影学中的一个术语，因为打印相片所能表现的亮度范围不足以表现现实世界中的亮度域，而如果简单的将真实世界的整个亮度域线性压缩到照片所能表现的亮度域内，则会在明暗两端同时丢失很多细节，这显然不是所希望的效果，Tone Mapping就是为了克服这一情况而存在的，既然相片所能呈现的亮度域有限则我们可以根据所拍摄场景内的整体亮度通过光圈与曝光时间的长短来控制一个合适的亮度域，这样既保证细节不丢失，也可以不使照片失真。人的眼睛也是相同的原理，这就是为什么当我们从一个明亮的环境突然到一个黑暗的环境时，可以从什么都看不见到慢慢可以适应周围的亮度，所不同的是人眼是通过瞳孔来调节亮度域的。

ToneMapping延伸到图形学领域，到目前为止，他的工作依然可以被描述为处理在明暗两端的信息丢失，只是原来所使用摄像曝光手段处理的部分被更换为了数学公式。

### Reinhard tone mapping

上张图片下方的方案显而易见的好很多，这种在图中被标记为New operator的方案实际上提出在2002年，被称之为Reinhard tone mapping。

``` c++
float3 ReinhardToneMapping(float3 color, float adapted_lum) 
{
    const float MIDDLE_GREY = 1;
    color *= MIDDLE_GREY / adapted_lum;
    return color / (1.0f + color);
}
//color是线性的HDR颜色，adapted_lum是根据整个画面统计出来的亮度
```

这种方式计算得出的结果可以观察到有效，但上面这段代码并没有更多原理解释。原文章称其为经验派，有种俺寻思能行的绿皮科技味道。

这里给出他的数学公式与函数曲线。

![RFormula](/images/ReinhardFormula.png)

Middlegrey为全屏幕或者部分屏幕的中间灰度，用于控制屏幕的亮度，在上方的代码之中可以看到作者直接将MiddleGrey设置为1，但实际上这个数值并没有固定的计算手段，而是由经验和效果来判断的。AvgLogLuminance就是全屏幕或部分屏幕的亮度的对数的平均值。

![](/images/AvgLogLuminanceFormula.png)

可以注意到，上述代码之中将adapted_lum也就是公式之中的avgLogLuminance作为传入参数处理。

<img src="/images/ReinhardCurve.png" alt="ReinhardCurve" style="zoom:75%;" />

实质上他的映射曲线如图，这种2002年的新方案将映射平滑，减少了最低和最高的占比部分，将极暗与极亮的部分所占比重降低，进而达到良好的映射效果。

### CEToneMapping

``` c++
float3 CEToneMapping(float3 color, float adapted_lum) 
{
    return 1 - exp(-adapted_lum * color);
}
```

2007年孤岛危机推陈出新，更干脆的直接模拟了S曲线。

exp是以e为底的指数函数计算，这么说可能比较模糊，但是上图之后就清晰了很多。

<img src="/images/exp.png" alt="exp" style="zoom:50%;" />

这个函数图像很亲切，我们可以看懂上面的代码单个是什么意思了，但是这串东西合起来是什么意思呢？

<img src="/images/CECure.png" alt="CECure" style="zoom:75%;" />

这个是上述函数的图像，可以看到他的映射关系如图。总体还是s曲线，但是换了一种表达形式占比。有种用MatLab画图的感觉，用数学手段强行模拟曲线关系。

### FilmicToneMapping

``` c++
float3 F(float3 x)
{
	const float A = 0.22f;
	const float B = 0.30f;
	const float C = 0.10f;
	const float D = 0.20f;
	const float E = 0.01f;
	const float F = 0.30f;
 
	return ((x * (A * x + C * B) + D * E) / (x * (A * x + B) + D * F)) - E / F;
}

float3 Uncharted2ToneMapping(float3 color, float adapted_lum)
{
	const float WHITE = 11.2f;
	return F(1.6f * adapted_lum * color) / F(WHITE);
}
```

这个方法的本质是把原图和让艺术家用专业照相软件模拟胶片的感觉，人肉tone mapping后的结果去使用数学手段做曲线拟合，得到一个高次曲线的表达式。

<img src="/images/FilmicToneMappingCure.png" alt="FilicToneMappingCure" style="zoom:75%;" />

如图，为其拟合映射曲线。

### [Academy Color Encoding System](https://link.zhihu.com/?target=http%3A//www.oscars.org/science-technology/sci-tech-projects/aces)

这种方案不同于以上三种，他单独创建了一套颜色编码系统，作为一种通用的数据交换方式，将不同的输入设备格式转换成为ACES，然后再在不同的设备上使用ACES来正常显示表达。我们只需要利用其中一小部分的转换逻辑，因为游戏渲染流程之中并没有图像输入设备的需求，我们可以直接得到需要渲染的图元数据，所以只要在ACES的基础上转到LDR或者另一个HDR就可以。

``` c++
float3 ACESToneMapping(float3 color, float adapted_lum)
{
	const float A = 2.51f;
	const float B = 0.03f;
	const float C = 2.43f;
	const float D = 0.59f;
	const float E = 0.14f;

	color *= adapted_lum;
	return (color * (A * color + B)) / (color * (C * color + D) + E);
}
```

![ACESCurve](/images/ACESCurve.png)



更好的地方是，按照前面说的，ACES为的是解决所有设备之间的颜色空间转换问题。所以这个tone mapper不但可以用于HDR到LDR的转换，还可以用于从一个HDR转到另一个HDR。

### 卡通游戏的色彩变幻问题

![CT](/images/CT.png)



## Bloom?

Bloom直译为绽放，在这里的含义为后期特效中的泛光/辉光。

![Bloom](/images/Bloom.jpg)

如图，就是经典的Bloom效果展示，在强光部分会产生向外扩散的光晕，这个就是Bloom。

Bloom的形状不一定是圆形，他和光圈的形状也有一定关联，具体他的形成原理请参照参考资料10[光学系统像差杂谈（3）：理想的尽头](https://zhuanlan.zhihu.com/p/27024300)。



### 如何制造Bloom？

被我找到的方式有三种，第一种是计算超出阈值的高光像素，将其向外扩散，第二种就是使用高斯模糊来模拟Bloom，然后与原图像相叠加，第三种是采用盒式，实质上也是采取和原图像叠加的方式实现，相对高斯而言，这种方案性能上会稍好，但效果稍差。这里给出第二种方式与第三种方式。高斯模糊实质上是对一个像素周边的想去分配一个权重，然后按权重采样混合，得到最终值。如需了解高斯模糊的更相关内容，请参照参考资料11[高斯模糊原理](https://www.cnblogs.com/invisible2/p/9177018.html)。

+ 高斯模糊的滤波器用于计算像素周围的权重，它是一种低通滤波器。之后高斯模糊会将当前的像素与周围的像素按一定权重混合，产生一定模糊效果。二维高斯公式如下图。σ为假定值。

  <img src="/images/高斯模糊公式.png" alt="高斯模糊公式" style="zoom:75%;" />

  ``` C++
  //显然我们可以通过这个公式用代码来计算出指定大小的滤波器
  double sigma = (double)radius / 3.0;  
  double sigma2 = 2.0 * sigma * sigma;  
  double sigmap = sigma2 * PI;  
    
  for(long n = 0, i = - radius; i <=radius; ++i)  
  {  
      long i2 = i * i;  
      for(long j = -radius; j <= radius; ++j, ++n)  
          kernel[n] = exp(-(double)(i2 + j * j) / sigma2) / sigmap;  
  }
  ```

  

  如图为5*5的滤波器，因为上述代码同样需要进行计算耗时，因而将其结果直接提出使用。

  ![55滤波器](/images/55滤波器.png)

+ 盒式过滤

  ```c++
  half3 Sample(float2 uv)
  {
        return tex2D(_MainTex,uv).rgb;
  }
  
  half3 BoxFilter(float2 uv)
  {
        half2 upL,upR,downL,downR;
                  
        upL =_MainTex_TexelSize.xy*half2(-1,1);
        upR =_MainTex_TexelSize.xy*half2(1,1);
        downL =_MainTex_TexelSize.xy*half2(-1,-1);
        downR =_MainTex_TexelSize.xy*half2(1,-1);
  
        half3 col =0;
  
        col+=Sample(uv+upL)*0.25;
        col+=Sample(uv+upR)*0.25;
        col+=Sample(uv+downL)*0.25;
        col+=Sample(uv+downR)*0.25;
  
        return col;
  }
  ```

  盒式过滤的流程相对于高斯来说简易很多，选取每一个像素的左上，右上，左下，右下四个方位，每个权重取0.25，然后将其结果相加。

### 降采样

在模糊图像之前，需要考虑一个问题，我们的计算方式是最终图像 = 原图像+模糊后的图像。而使用模糊是会逐像素进行计算的，这意味着大量的计算，而如果我们可以通过降低像素个数的方法来减少计算次数，无疑会提高我们的计算速度。

可以直接通过直接降低分辨率的方法解决问题，但参考资料12[从认识到实现：HDR Bloom in Unity](https://zhuanlan.zhihu.com/p/91390940)的作者给出了一种渐进式的采样方案，用于提升低分辨率的采样质量。有趣的是参考资料16[Bloom](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/)给出了一份同样的方案，也许是英雄所见略同。不过参考资料12的代码逻辑有所缺漏，阅读16会比较好。

这种方案是使用多次逐步降低分辨率来代替一次性降低，利用在每次迭代的过程中借由引擎的双线性过滤提升效果。

``` csharp
        int width = source.width;
        int height = source.height;
        int i = 1;
        for (; i < iterations; i++)
        {
            width /= 2;
            height /= 2;
            if (height < 2) break;
            currentDestination = textures[i] =
                RenderTexture.GetTemporary(width, height, 0, format);
            Graphics.Blit(currentSource, currentDestination,bloom,BoxDownPass);
            currentSource = currentDestination;
        }
        for (i -= 2; i >= 0; i--) 
        {
            currentDestination = textures[i];
            textures[i] = null;
            Graphics.Blit(currentSource, currentDestination,bloom,BoxUpPass);
            RenderTexture.ReleaseTemporary(currentSource);
            currentSource = currentDestination;
        }
```

## [实例](https://github.com/cckt123/HDR_ToneMapping_Bloom)

这个实例实质上来自于[Bloom](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/)，其他的文献之中并没有找到能帮助我推动效果的部分，~~即使是16也有部分问题待处理~~。

好吧，实质上这篇文章的作者提供了他的UnityPackage，不需要再次制作实例，这里给出他的[Package](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/creating-bloom/creating-bloom.unitypackage)。

## [Post ProcessingBloom](https://github.com/cckt123/HDR_ToneMapping_Bloom)

[HDR_ToneMapping_Bloom](https://crazyink47.xyz/2022/02/09/HDR-ToneMapping-Bloom/)这篇文章的后续，参考文献16[Bloom Blurring Light](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/)给了我很大帮助，大到我觉得翻译出来应该会更好，但继续研究的过程之中发生了一点意外，我找到了一篇更新的教程[Post Processing Bloom](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/)，同时确认到这篇教程拥有中文翻译[Unity通用渲染管线（URP）系列（十一）——后处理（Bloom）](https://zhuanlan.zhihu.com/p/339443207)。~~所以我补完了那篇翻译的同时尝试了一下这个崭新的方案，如果我做完了，那么你可以在相同的GITHUB地址找到他的实例。~~

同样，你可以找到原作者的[示例](https://bitbucket.org/catlikecodingunitytutorials/custom-srp-11-post-processing/)。

## 参考资料

1. [Tone mapping进化论 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/21983679)
2. [Tone Mapping – OWNSELF](http://www.ownself.org/blog/2011/tone-mapping.html)
3. [Unity3d HDR和Bloom效果（高动态范围图像和泛光）_思月行云-CSDN博客](https://blog.csdn.net/kenkao/article/details/44940629)
4. [HDR,ToneMapping,Bloom之间的关系 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/80253409)
5. [关闭Typora拼写检查功能_枸空的博客-CSDN博客_typora关闭拼写检查](https://blog.csdn.net/weixin_42349817/article/details/119563536)
6. [exp（语言函数）_百度百科 (baidu.com)](https://baike.baidu.com/item/exp/10942130?fr=aladdin)
7. [Unity3d HDR和Bloom效果（高动态范围图像和泛光） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/47394772)
8. [4.15 tonemapper - is it possible to simulate old effects?](https://forums.unrealengine.com/t/4-15-tonemapper-is-it-possible-to-simulate-old-effects/85379)
9. [Bloom是什么？](https://zhuanlan.zhihu.com/p/76505536)
10. [光学系统像差杂谈（3）：理想的尽头](https://zhuanlan.zhihu.com/p/27024300)
11. [高斯模糊原理](https://www.cnblogs.com/invisible2/p/9177018.html)
12. [从认识到实现：HDR Bloom in Unity](https://zhuanlan.zhihu.com/p/91390940)
13. [Unity通用渲染管线（URP）系列（十二）—— HDR](https://zhuanlan.zhihu.com/p/340145796)
14. [Unity用户手册](https://docs.unity3d.com/cn/current/Manual/HDR.html)
15. [在unity引擎中的工程如何适配HDR电视？](https://www.zhihu.com/question/61321220/answer/186706077)
16. [Bloom](https://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/)
17. [Bloom](https://zhuanlan.zhihu.com/p/339443207)

