---
title: RenderFeature
date: 2022-06-23 10:49:46
tags: [ Shader, Unity]
categories: [ Shader]
about:
description: "Unity URP RenderFeature 学习记录"
password: Xm4399
---

## 前言

在Built-In管线里面，我们可以使用多Pass实现描边、毛发等等效果，但是在URP中就需要改变实现方式了。一般来说，Render Feature可以解决大部分问题，可以通过它实现大部分多Pass的效果。

## RenderFeature?

URP中，ScriptableRenderer类用来实现一套具体的渲染策略。它描述了渲染管线的工作细节，当前URP之中有ForwardRender和2dRender两种ScriptableRenderer。用户可自己创建类继承ScriptableRenderer来实现定制化的Renderer。

![自带的ForwardRender](https://pic1.zhimg.com/80/v2-0f0ec7ecde1aef3d5c73388148c10208_720w.jpg)

RenderFeature是依附于ForwardRenderer。ForwardRenderer其中可以添加自定义的RenderFeature。

![RenderFeature](https://pic2.zhimg.com/80/v2-ec4aa78cd97834cd8fb9d71ceb96b729_720w.jpg)

这里可以参照之前[ToneMapping博客](https://crazyink47.xyz/2022/05/18/ToneMapping-%E5%BA%94%E7%94%A8/#ScriptableRendererFeature)所做的笔记，或者阅读URP源代码来了解详细内容。为了阅读方便，我复制了一份过来。

### ScriptableRendererFeature

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

用户自定义的RenderFeature会填入到ScriptableRenderer中的一个列表中，即代码中的m_RendererFeatures。然后再ScriptableRenderer Setup的时候调用AddRenderPasses方法，将RenderFeature中的ScriptableRenderPass填入ScriptRenderer的m_ActiveRenderPassQueue中。这样当渲染时，ScriptableRenderPass中的Execute方法就会被调用。

![ScriptableRendererPath](/images/ScriptableRendererPath.png)

上图位置可以找到源代码，这里[参照文献](https://zhuanlan.zhihu.com/p/389510239)，截取部分相关内容。

``` csharp
	public abstract partial class ScriptableRenderer : IDisposable
    {        
        //这里是添加到的目标列表
        List<ScriptableRendererFeature> m_RendererFeatures = new List<ScriptableRendererFeature>(10);
        
        //提供封装
        protected List<ScriptableRendererFeature> rendererFeatures
        {
            get => m_RendererFeatures;
        }
		
        //在这里将RenderPass添加到上述列表
        protected void AddRenderPasses(ref RenderingData renderingData)
        {
            //ToDo:这一行是做什么呢？
            using var profScope = new ProfilingScope(null, Profiling.addRenderPasses);

            // 从自定义的rendererFeature添加renderPass
            for (int i = 0; i < rendererFeatures.Count; ++i)
            {
                //略过未激活的rendererFeature
                if (!rendererFeatures[i].isActive)
                {
                    continue;
                }
                //调用rendererFeature中的AddRenderPasses方法
                rendererFeatures[i].AddRenderPasses(this, ref renderingData);
            }

            // Remove any null render pass that might have been added by user by mistake
            int count = activeRenderPassQueue.Count;
            for (int i = count - 1; i >= 0; i--)
            {
                if (activeRenderPassQueue[i] == null)
                    activeRenderPassQueue.RemoveAt(i);
            }
        }
    }
```

### RenderObjects

这里是个栗子，同样来源于[URP-RenderFeature渲染流程及使用实例](https://zhuanlan.zhihu.com/p/389510239)

``` csharp
public class RenderObjects : ScriptableRendererFeature
{ 
    	//RenderObjectPass为继承ScriptableRenderPass的子类
        RenderObjectsPass renderObjectsPass;
		
		//被上述的ScriptableRenderer调用添加
        public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
        {
            renderer.EnqueuePass(renderObjectsPass);
        }
}
```

覆写AddRenderPasses方法，然后在方法中将RenderObjectsPass填入ScriptableRenderer中。

``` csharp
    public abstract partial class ScriptableRenderer : IDisposable
    {   
        ...    
        public void EnqueuePass(ScriptableRenderPass pass)
        {
            //将ScriptableRenderPass添加到m_ActiveRenderPassQueue
            m_ActiveRenderPassQueue.Add(pass);
        }
        ...
		
        void ExecuteRenderPass(ScriptableRenderContext context, ScriptableRenderPass renderPass, ref RenderingData renderingData)
        {
            using var profScope = new ProfilingScope(null, renderPass.profilingSampler);

            ref CameraData cameraData = ref renderingData.cameraData;

            CommandBuffer cmd = CommandBufferPool.Get();

            // Track CPU only as GPU markers for this scope were "too noisy".
            using (new ProfilingScope(cmd, Profiling.RenderPass.configure))
            {
                renderPass.Configure(cmd, cameraData.cameraTargetDescriptor);
                SetRenderPassAttachments(cmd, renderPass, ref cameraData);
            }

            // Also, we execute the commands recorded at this point to ensure SetRenderTarget is called before RenderPass.Execute
            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);

            renderPass.Execute(context, ref renderingData);
        }
        ...
    }
```

这里是执行环节。

## RendererFeature属性

![RendererFeature属性](https://pic2.zhimg.com/80/v2-0a1b7365c68d88c09974dc401b8f46c9_720w.jpg)

1. Name:Feature的名字。

2. Event:Unity在执行该RendererFeature时，该事件在URP中的执行顺序。

   ![RenderPassEvent](https://pic3.zhimg.com/80/v2-610b48f08eb9e15d8294caaf653fe18e_720w.jpg)

3. Filters：这个设置允许我们给这个Renderer Feature 去配置要渲染哪些对象；这里面有两个参数，一个是Queue，一个是Layer Mask。

   Queue：这个Feature选择渲染透明物体还是不透明物体；
   Layer Mask：这个Feature选择渲染哪个图层中的对象；

4.  **Shader Passes Name** ：如果shader中的一个pass具有 LightMode Pass 这个标记的话，那我们的Renderer Feature仅处理 LightMode Pass Tag 等于这个Shader Passes Name的Pass。

5. Overrides：使用这个Renderer Feature 进行渲染时，这部分的设置可让我们配置某些属性进行重写覆盖。

   Material：渲染对象时，Unity会使用该材质替换分配给它的材质。

   Depth：选择此选项可以指定这个Renderer Feature如何影响或使用深度缓冲区。此选项包含以下各项：

   Write Depth：写入深度，此选项定义渲染对象时这个Renderer Feature是否更新深度缓冲区。
   Depth Test：深度测试，决定renderer feature是否渲染object的片元。

   Stencil：选中此复选框后，Renderer将处理模板缓冲区值。

   Camera：选择此选项可让您覆盖以下“摄像机”属性：

   Field of View：渲染对象时，渲染器功能使用此Field of View（视场），而不是在相机上指定的值。

   Position Offset：渲染对象时，Renderer Feature将其移动此偏移量。Restore：选中此选项，在此Renderer Feature中执行渲染过程后，Renderer Feature将还原原始相机矩阵。

## 多Pass实现

这里注意上文的ShaderPassesName属性，当我们设置该参数之后，UnityURP将不会调用UniversalForwardPass。但如果我们将UniversalForwardPass加回该参数，那么就会同时有UniversalForwardPass和我们新添加的Pass。参照 [URP学习之七--RenderFeature](https://www.cnblogs.com/shenyibo/p/12554211.html)。上面是URP如何调用RenderFeature的过程，而下面将会以Unity自带的RenderObject为例，介绍如何自定义RendererFeature。

## RendererFeature实现

所有的Feature都要继承ScriptableRendererFeature。而实例RenderObjects中又有如下定义。

``` csharp
		public RenderObjectsSettings settings = new RenderObjectsSettings();//实例化RenderObjectsSettings
		[System.Serializable]
        public class RenderObjectsSettings
        {
            public string passTag = "RenderObjectsFeature";
            public RenderPassEvent Event = RenderPassEvent.AfterRenderingOpaques;
			//FilterSettings定义见下
            public FilterSettings filterSettings = new FilterSettings();
			//渲染对象时，Unity会使用该材质替换分配给它的材质。
            public Material overrideMaterial = null;
            public int overrideMaterialPassIndex = 0;
			//指定这个Renderer Feature如何影响或使用深度缓冲区
            public bool overrideDepthState = false;
            public CompareFunction depthCompareFunction = CompareFunction.LessEqual;
            public bool enableWrite = true;
			//UnityEngine.Rendering.Universal下定义
            public StencilStateData stencilSettings = new StencilStateData();
			//CustomCameraSettings定义见下
            public CustomCameraSettings cameraSettings = new CustomCameraSettings();
        }

        [System.Serializable]
        public class FilterSettings
        {
            // TODO: expose opaque, transparent, all ranges as drop down
            public RenderQueueType RenderQueueType;
            public LayerMask LayerMask;
            public string[] PassNames;

            public FilterSettings()
            {
                RenderQueueType = RenderQueueType.Opaque;
                LayerMask = 0;
            }
        }

        [System.Serializable]
        public class CustomCameraSettings
        {
            public bool overrideCamera = false;
            public bool restoreCamera = true;
            public Vector4 offset;
            public float cameraFieldOfView = 60.0f;
        }

```

这些结构就是我们看到的Renderer中Feature的参数界面，也就是说Feature的参数都是在这里定义的。然后需要实现基类的两个方法：

``` csharp
		public override void Create()
        {
            FilterSettings filter = settings.filterSettings;
            renderObjectsPass = new RenderObjectsPass(settings.passTag, settings.Event, filter.PassNames,
                filter.RenderQueueType, filter.LayerMask, settings.cameraSettings);

            renderObjectsPass.overrideMaterial = settings.overrideMaterial;
            renderObjectsPass.overrideMaterialPassIndex = settings.overrideMaterialPassIndex;

            if (settings.overrideDepthState)
                renderObjectsPass.SetDetphState(settings.enableWrite, settings.depthCompareFunction);

            if (settings.stencilSettings.overrideStencilState)
                renderObjectsPass.SetStencilState(settings.stencilSettings.stencilReference,
                    settings.stencilSettings.stencilCompareFunction, settings.stencilSettings.passOperation,
                    settings.stencilSettings.failOperation, settings.stencilSettings.zFailOperation);
        }

        public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
        {
            renderer.EnqueuePass(renderObjectsPass);
        }
```

可见，RenderObjects只是又执行了一次RenderObjectsPass。

## RendererPass实现

 

## 参考资料

+ [URP-RenderFeature渲染流程及使用实例](https://zhuanlan.zhihu.com/p/389510239)
+ [URP学习之七--RenderFeature](https://www.cnblogs.com/shenyibo/p/12554211.html)
+ [在URP里写Shader](https://zhuanlan.zhihu.com/p/336428407)
+ [ToneMapping](https://crazyink47.xyz/2022/05/18/ToneMapping-%E5%BA%94%E7%94%A8/#ScriptableRendererFeature)
+ [Unity Render Feature初探](https://zhuanlan.zhihu.com/p/146396120)
+ [fullscreen-blur-with-urp](https://forum.unity.com/threads/fullscreen-blur-with-urp.883987/)
