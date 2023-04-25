---
title: CommandBuffer and RenderTexture
date: 2022-05-16 08:55:18
tags: [ Shader, Unity]
categories: [ Shader]
about:
description: "后处理相关"
---

## RenderTexture

### 什么是RenderTexture?

RenderTexture，顾名思义，是可以进行渲染的纹理类型，继承于Texture接口。

可用于实现图像的渲染特效、动态阴影、投影、反射或者摄像机。

在某些情况下，例如加载新场景，系统进入屏幕保护模式，进退全屏等情况下会导致RenderTexture丢失。这时候要使用IsCreated函数来检查RenderTexture的情况。

需要注意的是，在使用完成之后，要手动调用Release来释放内存，RenderTexture并不会自动进行内存回收。

Texture对象分为两种，一种Server-side，创建在显存，由GPU访问，一种Client-side创建在内存，由CPU访问。CPU提交渲染请求的时候，会把内存中的纹理对象复制到GPU可访问显存，反之亦然。

### 如何使用RenderTexture?

1. 构造函数创建

   ``` csharp
   var RT = new RenderTexture(1024, 1024, 16, RenderTextureFormat.ARGB32, RenderTextureReadWrite.sRGB);
   //上述参数依次为纹理宽度，高度，深度，纹理颜色格式，RenderTexture颜色格式
   RT.Release();
   GameObject.Destory(RT);
   RT = null;
   ```

2. RenderTexture.GetTemporary

   ``` csharp
   //获取临时分配的纹理，使用完成之后需要立即调用ReleaseTemporary释放，以便下次调用
   RT = RenderTexture.GetTemporary(1024,1024,16,RenderTextureFormat.ARGB32, RenderTextureReadWrite.sRGB);
   RenderTexture.ReleaseTemporary(RT);
   RT = null;
   ```

3. CommandBuffer.GetTemporary

   ``` csharp
   //与方法2相似，第一个参数会获得指向RenderTexture的整型ID
   var cmd = new CommandBuffer();
   var texID = Shader.PropertyToID("_RTTex");
   cmd.GetTemporaryRT(texID, 1024, 1024, 16, FilterMode.Bilinear, RenderTextureFormat.ARGB32, RenderTextureReadWrite.sRGB);
   cmd.ReleaseTemoraryRT(texID);
   //应用CommandBuffer
   _currentCamera.AddCommandBuffer(CameraEvent.BeforeImageEffects,cmd);
   _currentCamera.RemoveCommandBuffer(CameraEvent.BeforeImageEffects, cmd);
   cmd.Release();
   cmd = null;
   ```

### RenderTexture的属性

1. ColorFormat

   ​	模式使用ARGB32，当前平台或者GPU可能并不支持特定的纹理格式，需要使用SystemInfo.SupportsRenderTextureFormat在使用时检测。

2. Size

   ​	纹理大小，在使用CommandBuffer时，如果输入参数为负数，表示当前关联相机像素1/X。

3. ZBuffer

   ​	用于设定Zbuffer的大小，包含两个类型内容，深度和模板。

   ​	有效参数有三个0，16，24。

   ​	0，不保存

   ​	16，depth

   ​	24，depth and stencil

4. RenderTextureReadWrite

   ​	Default 基于项目设定

   ​	Linear 线性，不进行颜色转换

   ​	sRGB 包含颜色数据

5. FilterMode

    + Point 从纹理是获取某个像素的颜色值作为采样结果输出
    + Bilinear 一次读取四个相邻的像素 平均后输出
    + Trilinear 从两个不同的mipmap层级上读取颜色，而后插值混合输出

6. Anti-Aliasing  反走样采样 消除锯齿 有效参数1 2 4 8


## CommandBuffer

要执行的图形命令列表。CommandBuffer可以保存渲染命令列表，将其设置为在摄像机渲染（Camera.AddCommandBuffer）、光源渲染期间(Light.AddCommandBuffer)各个点执行，或立即执行，用于拓展渲染管线的渲染效果。

1. 由CPU提交给GPU执行的一系列命令序列。
2. 可以更加灵活的方式来操纵渲染管线的流转
3. 可实现自定义渲染管线

``` csharp
//常见命令,详细请参照官方文档或CommandBuffer源码
CommandBuffer.Clear();//清除当前CommandBuffer之中所有命令

/*
	添加一个已激活的渲染目标命令
	参数:
		rt:同时设置目标的颜色与深度命令
		color:设置目标的颜色命令
		colors:设置目标们的颜色命令
		depth:设置目标深度命令
		mipLevel:设置目标的mip等级
		cubemapFace:要渲染到的立方体贴图渲染目标的立方体贴图面
		depthSlice:设置3D或数组渲染目标的切片
		loadAction:加载用于颜色和深度/模板缓冲的操作
		storeAction:存储用于颜色和深度/模板缓冲的动作
		colorLoadAction:用于颜色缓冲的加载操作
		colorStoreAction:...
		depthLoadAction:...
		depthStoreAction:...
*/
CommandBuffer.SetRenderTarget(RenderTargetIdentifier rt);
...//此处略过很多很多重载函数
    
/*
	添加一个清除渲染目标的命令
	参数：
		clearDepth:是否清理深度缓存
		clearColor:是否清理颜色缓存
		backgroundColor:背景颜色
		depth:清理深度 默认1.0
*/
CommandBuffer.ClearRenderTarget(bool clearDepth, bool clearColor, Color backgroundColor, float depth);
...//此处略过很多很多重载函数
    
/*
	添加一个绘制mesh命令
	参数：
		mesh:需要绘制的mehs
		matrix:变幻矩阵
		material:使用材质
		submeshIndex:渲染那个网格的子集
		shaderPass:使用的ShaderPass 默认-1 使用所有通道
		properties:之前被添加的额外材质属性
*/
CommandBuffer.DrawMesh(Mesh mesh, Matrix4x4 matrix, Material material, int submeshIndex, int shaderPass, MaterialPropertyBlock properties);

/*
	添加rt传输命令
	参数：
		source:源纹理
		dest:目标
		mat:使用材质球
		pass:通道
		scale:缩放
		offset:偏移
*/
CommandBuffer.Blit(RenderTargetIdentifier source, RenderTargetIdentifier dest, Material mat, int pass);

//清除当前摄像机上所有CommandBuffer
Camera.RemoveAllCommandBuffers();

//添加CommandBuffer
Camera.AddCommandBuffer(CameraEvent evt, CommandBuffer buffer);
```

这里可以用来获取相机纹理

``` csharp
//相机渲染时产生的内置临时渲染纹理
public enum BuiltinRenderTextureType
{
    ...
    //当前渲染相机的目标纹理。
    CameraTarget = 2
    ...
}
```

这里有对应渲染路线执行的相机事件

![CameraEvent](/images/CameraEvent.jpg)

## CommandBufferPool

``` csharp
public static CommandBuffer Get();
//返回一个新的CommandBuffer

public static CommandBuffer Get(string name);
//返回一个新的commandBuffer并为其赋予一个名字，被命名的commandBuffer将隐式的添加一个profiling

public static Release(CommandBuffer);
//释放
```

## ScriptableRenderContext

``` csharp
public void ExecuteCommandBuffer(CommandBuffer commandBuffer);
//调用指定的cmd
```

## 参考资料

+ [RenderTexture基础资料](https://blog.csdn.net/gzg_restart/article/details/121590654)
+ [CommandBufferHelloWorld](https://blog.csdn.net/gzg_restart/article/details/121062471?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-121062471-blog-88795640.pc_relevant_default&spm=1001.2101.3001.4242.2&utm_relevant_index=4)
+ [UnityCommandBuffer基础](https://blog.csdn.net/u012740992/article/details/88795640)
+ [Unity官方手册RenderTexture](https://docs.unity.cn/cn/current/ScriptReference/RenderTexture.html)
+ [Unity官方CommandBuffer文档](https://docs.unity3d.com/cn/current/ScriptReference/Rendering.CommandBuffer.html)
+ [Unity官方CommandBufferPool文档](https://docs.unity3d.com/Packages/com.unity.render-pipelines.core@11.0/api/UnityEngine.Rendering.CommandBufferPool.html)
+ [【技术美术百人计划】图形 3.72 command buffer及urp概述](https://www.bilibili.com/video/BV13F411e7Ai?p=2&vd_source=0ff5f2699eb1715de1a871b09a912a08)

