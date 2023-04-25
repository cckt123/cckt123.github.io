---
title: DepthOfFieldURP
date: 2022-10-19 17:23:45
tags: [ Unity]
categories: [ Unity]
about:
description: "Depth Of Field 景深效果使用。"

---

## 参考地址

- [Depth Of Field](https://docs.unity.cn/Packages/com.unity.render-pipelines.universal@11.0/manual/post-processing-depth-of-field.html)

## Depth Of Field

### Bokeh Depth of Field

[![img](https://docs.unity.cn/Packages/com.unity.render-pipelines.universal@11.0/manual/images/Inspectors/BokehDepthOfField.png)](https://docs.unity.cn/Packages/com.unity.render-pipelines.universal@11.0/manual/images/Inspectors/BokehDepthOfField.png)

景深_Bokeh模式效果非常接近真实相机。他的设置方式也趋近于真实的相机设置，提供了许多属性来调整相机上的光圈叶片。如果需要了解光圈叶片，以及它们如何影响相机输出的视觉效果，请参阅改进摄影指南 [Aperture Blades: How many is best?](https://improvephotography.com/29529/aperture-blades-many-best/)

| **属性**     | **介绍**                                                     |
| ------------ | ------------------------------------------------------------ |
| **对焦距离** | 设置相机与焦点之间的距离.                                    |
| **焦距**     | 设置Camera sensor与Camera lens之间的距离，单位为毫米. 这个值越大，景深的效果越浅. |
| **光圈**     | 设置光圈比例 (known as f-stop or f-number). 这个值越小，景深效果越浅。 |
| **叶片数**   | 设置相机形成光圈的光圈叶片数量. 使用的叶片越多，景深效果越圆. |
| **叶片曲率** | 设置相机形成光圈的光圈叶片曲率。这个值越小，光圈叶片越清晰，当这个值为1的时候，bokeh景深效果会变成完美的圆形。 |
| **叶片旋转** | 设置光圈叶片的旋转角度。                                     |

## [Aperture Blades: How many is best?](https://improvephotography.com/29529/aperture-blades-many-best/)

### 多少光圈叶片最佳？

简单来说，越多越好。只不过，这个简要的回答需要大量的定语辅助才能称得上完全正确。

### 初学背景

你可能已经知道镜头光圈会影响景深。如果你用大光圈进行拍摄，景深效果就会很浅。这种浅景深可以得到清晰的拍摄对象和相对模糊的背景，让你的拍摄对象和背景产生鲜明的对比，凸出其存在感。

其中照片失去焦距的部分被称为bokeh。bokeh效果可以平滑而自然，也可以粗暴直接。其中的差异就取决于构成镜头光圈的叶片数量。

镜头光圈是机械式打开和关闭的，运作起来就像你的瞳孔一样。它可以放大，也可以缩成一个小圆点。当你的光圈大张，光圈叶片将会缩回到镜头里，你的bokeh效果将永远会是圆形的。而只有当你关闭光圈的时候，光圈叶片才会弹出，发挥他的作用。

[![img](https://improvephotography.com/wp-content/uploads/2014/07/bokeh-comparison.jpg)](https://improvephotography.com/wp-content/uploads/2014/07/bokeh-comparison.jpg)

### 不同的镜头孔径

有些镜头的光圈上只有5~6片光圈叶片，而有些镜头的光圈上可能有9片甚至14片光圈叶片。在相对便宜的镜头里，光圈最常见的是6片叶片，而在专业级的镜头里，常见的是9片叶片。

除了叶片的数量之外，区别还在于它们是直的还是圆的。在下面的对比图中，你可以看到圆形叶片的孔比方形叶片的更接近圆形。虽然这对大多数人来说可能没有太大的区别，但摄影师通常更喜欢圆形叶片，这样散景看起来光滑而圆润。

[![img](https://improvephotography.com/wp-content/uploads/2014/07/different-apertures.jpg)](https://improvephotography.com/wp-content/uploads/2014/07/different-apertures.jpg)

如果你有更多的叶片，随着光圈的闭合，它会闭合成一个更接近完美圆形的形状。