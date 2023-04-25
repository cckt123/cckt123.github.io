---
title: AssetBundle浅析
date: 2022-02-17 16:35:27
tags: [ Unity]
categories: [ Unity]
about:
description: "AssetBundle学习笔记。"
---

## 参考资料

+ [Unity手游实战：从0开始SLG——资源管理系统-基础篇（一）浅谈Asset(Unity资产映射)](https://zhuanlan.zhihu.com/p/96709802)
+ [Unity手游实战：从0开始SLG——资源管理系统-基础篇（三）AssetBundle原理](https://zhuanlan.zhihu.com/p/97551363)
+ [关于AssetBundle.LoadFromMemroy内存翻倍问题](https://answer.uwa4d.com/question/5e8ed8c6cd6a9b49fb4a46a3)
+ [AssetBundle](https://docs.unity3d.com/cn/current/Manual/AssetBundlesIntro.html)

## AssetBundle？

**AssetBundle** 是一个存档文件，包含可在运行时加载的特定于平台的资源（模型、纹理、预制件、音频剪辑甚至整个场景）。AssetBundle 可以表达彼此之间的依赖关系；例如 AssetBundle A 中的材质可以引用 AssetBundle B 中的纹理。为了通过网络进行有效传递，可以根据用例要求选用内置算法来压缩 AssetBundle（LZMA 和 LZ4）。

AssetBundle 可用于可下载内容（DLC），减小初始安装大小，加载针对最终用户平台优化的资源，以及减轻运行时内存压力。

## 加载AssetBundle?

1. AssetBundle.LoadFromMemory(Async)

   从一个字节数组创建 AssetBundle。

   Unity的建议是**不要使用这个API**。

   LoadFromMemory(Async) 是从托管代码的字节数组里加载AssetBundle。也就是说你要提前用其他的方式将资源的二进制数组加入到内存中。然后该接口会将源数据从托管代码字节数组复制到新分配的、连续的本机内存块中。

   ``` csharp
   using UnityEngine;
   using UnityEngine.Networking;
   using System.Collections;
   
   public class ExampleClass : MonoBehaviour
   {
       byte[] MyDecription(byte[] binary)
       {
           byte[] decrypted = new byte[1024];
           return decrypted;
       }
   
       IEnumerator Start()
       {
           var uwr = UnityWebRequest.Get("http://myserver/myBundle.unity3d");
           yield return uwr.SendWebRequest();
           byte[] decryptedBytes = MyDecription(uwr.downloadHandler.data);
           AssetBundle.LoadFromMemory(decryptedBytes);
       }
   }
   ```

   这里是官方给出的LoadFormMemory案例，可以看到她使用了一个UnityWebRequesy来获取Byte数据。

   如果AssetBundle使用了LZMA压缩类型，它将在复制时解压AssetBundle。而未压缩和LZ4压缩类型的AssetBundle将逐字节的完整复制。产生至少两倍的冗余。

   事实上我并没有找到官方关于这个的警告，不过多方参照之下大家都在警告这个问题，例如参考资料[Unity手游实战：从0开始SLG——资源管理系统-基础篇（三）AssetBundle原理](https://zhuanlan.zhihu.com/p/97551363)和[关于AssetBundle.LoadFromMemroy内存翻倍问题](https://answer.uwa4d.com/question/5e8ed8c6cd6a9b49fb4a46a3)，所以这里**没有**实际进行测试，默认了这件事。

2. AssetBundle.LoadFromFile(Async)

   LoadFromFile是一种高效的API，用于从本地存储(如硬盘或SD卡)加载未压缩或LZ4压缩格式的AssetBundle。

   ``` csharp
   //The function supports bundles of any compression type. In case of lzma compression, the data will be decompressed to the memory. Uncompressed and chunk-compressed bundles can be read directly from disk. 这里是Unity官方的介绍
   ```

   在桌面独立平台、控制台和移动平台上，API将只加载AssetBundle的头部，并将剩余的数据留在磁盘上。

   AssetBundle的Objects 会按需加载，比如加载方法(例如AssetBundle.Load)被调用或其InstanceID被间接引用的时候。在这种情况下，不会消耗过多的内存。

   但在Editor环境下中，API还是会把整个AssetBundle加载到内存中，就像读取磁盘上的字节和使用AssetBundle.LoadFromMemoryAsync一样。

   如果在Editor中对项目进行了分析，此API可能会导致在AssetBundle加载期间出现内存尖峰。但这不应影响设备上的性能，在做优化之前，这些尖峰应该在设备上重新再测试一一遍.

   要注意，这个API只针对未压缩或LZ4压缩格式，因为前面说过了，如果使用LZMA压缩的话，它是针对整个生成后的数据包进行压缩的，所以在未解压之前是无法拿到AssetBundle的头信息的。

   ``` csharp
   using UnityEngine;
   using System.Collections;
   using System.IO;
   
   public class LoadFromFileExample : MonoBehaviour
   {
       void Start()
       {
           var myLoadedAssetBundle = AssetBundle.LoadFromFile(Path.Combine(Application.streamingAssetsPath, "myassetBundle"));
           if (myLoadedAssetBundle == null)
           {
               Debug.Log("Failed to load AssetBundle!");
               return;
           }
   
           var prefab = myLoadedAssetBundle.LoadAsset<GameObject>("MyObject");
           Instantiate(prefab);
   
           myLoadedAssetBundle.Unload(false);
       }
   }
   ```

3. AssetBundleDownloadHandler//TODO

   DownloadHandlerAssetBundle的操作是通过UnityWebRequest的API来完成的。

   UnityWebRequest API允许开发人员精确地指定Unity 应如何处理下载的数据，并允许开发人员消除不必要的内存使用。使用UnityWebRequest下载AssetBundle 的最简单方法是调用UnityWebRequest.GetAssetBundle。

   就实战项目而言，最有意思的类是DownloadHandlerAssetBundle。它使用工作线程，将下载的数据流存储到一个固定大小的缓冲区中，然后根据下载处理程序的配置方式将缓冲数据放到临时存储或AssetBundle缓存中。

   所有这些操作都发生在非托管代码中，消除了增加堆内存的风险。此外，该下载处理程序并不会保留所有下载字节的栈内存副本，从而进一步减少了下载AssetBundle的内存开销。

   LZMA压缩的AssetBundles将在下载和缓存的时候更改为LZ4压缩。这个可以通过设置Caching.CompressionEnable属性来更改。

   如果将缓存信息提供给UnityWebRequest对象的话，一旦有请求的AssetBundle已经存在于Unity的缓存中，那么AssetBundle将立即可用，并且此API的行为将会与AssetBundle.LoadFromFile相同操作。

   在Unity5.6之前，UnityWebRequest系统使用了一个固定的工作线程池和一个内部作业系统来防止过多的并发下载，并且线程池的大小是不可配置的。在Unity5.6中，这些安全措施已经被删除，以便适应更现代化的硬件，并允许更快地访问HTTP响应代码和报头。

4. WWW.LoadFromCacheOrDownload

   古老的API，已放弃维护，不建议使用。

## 解决冲突

使用Unity版本为2019.4.17f1

优先级

1. 实际测试结果
1. Unity官方文档
1. 其他参考资料
