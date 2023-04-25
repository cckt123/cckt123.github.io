---
title: 《CLRViaC#》第3章 共享程序集和强命名程序集
date: 2022-02-16 16:35:16
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 3.1 两种程序集，两种部署

CLR支持两种程序集

+ 弱命名程序集
+ 强命名程序集 使用发布者的私钥/公钥进行了签名，对程序集进行唯一性表示。保护与版本控制，并允许程序集部署到用户机器的任何地点，甚至可以部署到Internet上。且因其被唯一性表示，CLR可以应用一些安全策略。

私有部署与全局部署

私有部署和上一章提到过的部署方式一样，只可以在应用程序基目录下使用。全局部署可以部署到一些公认位置，CLR在检查程序集的同时会检测这些位置。

| 程序集种类 | 可以私有部署 | 可以全局部署 |
| ---------- | ------------ | ------------ |
| 弱命名     | 是           | 否           |
| 强命名     | 是           | 是           |

## 3.2 为程序集分配强名称

假如两个程序集都复制到相同的公认目录，最后一个安装的就是"老大"。（这也是DLL hell的由来，共享DLL全部复制到System32下）

为程序集进行唯一性表示，强命名程序集有四个重要特征。

1. 文件名 不计拓展名
2. 版本号
3. 语言文化
4. 公钥 由于公钥数字很大，所以经常使用从公钥派生的小哈希值，称为**公钥标记**。

微软采用的是标准的公钥/私钥加密技术，而没有选择其他的唯一标志性技术，如GUID、URL和URN。

创建强命名程序集第一步是用.NET Framwork SDK和Microsoft Visual Studio携带的Strong Name实用程序获取密匙。SN.exe允许通过多个命令行开关来使用一套功能。

``` sh
SN -k MyCompany.snk #创建MyCompany.snk 文件 包含二进制公钥私钥
SN -p MyCompany.snk MyCompany.PublicKey sha256 #创建只含公钥的文件
SN -tp MyCompany.PublicKey #传递只含公钥的文件
```

SN.exe未提供任何方式查看私钥。

``` sh
csc /keyfile:MyCompany.snk Program.cs #如果这个方案没有被淘汰，那么你可以用它来生成强命名程序集
```

或者可以从VS新建公钥/私钥，显示项目属性，签名，勾选为程序集签名....

## 3.3 全局程序集缓存

由多个应用程序访问的程序集必须放在公认的目录，也就是**全局程序集缓存**。

``` shell
%SystemRoot%\Microsoft.NET\Assembly #通常在这里
```

这东西是工具自动生成的，不要手动操作。

常见开发和测试时在GAV之中安装强命名程序集最常用的工具是**GACUtil.exe**。\i 安装 \u 卸载 无法传递弱命名程序集。详细内容请查看原书或者相关内容。

值得注意的是，将程序集安装到GAC破坏了我们想要达到的基本目标，也就是简单的安装，备份，还原，移动和卸载应用程序。因而尽量使用私有部署而不是全局部署。

## 3.4 在生成的程序中引用强命名程序集

我们生成的任何程序集都会包含对其他强命名程序集的引用，因为System.Object在MSCorLib.dll中定义，后者就是强命名程序集。至于如何引用这些程序集，可以参照上一章的内容。

## 3.5 强命名程序能防篡改

签名之后，将公钥和签名嵌入程序集，CLR就会验证程序集是否发生过修改或者破坏。应用程序在安装到GAC的时刻，系统将会对包含清单的那个文件进行哈希处理，将哈希值与PE文件中嵌入的RSA签名进行比较。如果这个或者其他任何一个哈希值不匹配，系统将会拒绝在GAC之中安装文件。

如果程序在GAC之外的地方加载程序集，那么每次都会进行哈希匹配以确保程序并没有遭到修改。

## 3.6 延迟签名

SN.exe等工具生成公钥/私钥对。生成密钥的同时会将密钥存储到文件或者其他硬件设备之上。有些公司会把私钥存储在硬件设备上，然后将硬件设备锁在保险柜当中。然而开发过程之中访问这些私钥就有点费劲，为了解决这个问题，.NET提供了**延迟签名**，也就是**部分签名**。他允许只使用公钥生成程序集，而暂时不使用私钥。因为有公钥，所以其他程序集会嵌入正常的公钥值，且可以存储到GAC之中。

如果是C#编译器，需要指定/delaysign编译器开关。如果是VS，打开项目属性，签名选项卡勾选“仅延迟签名”。

![](/images/DelayedSigning.png)

如果需要使用混淆器之类生成后执行的程序，也需要使用延迟签名技术，否则将会拒绝对程序集进行更改。

## 3.7 私有部署强命名程序集

虽然强命名程序集能安装到GAC，但绝非必须。只有由多个应用程序共享的程序集才应部署到GAC。强命名程序集除了部署到GAC和私有部署，还可以部署到只有少数应用知道的目录。使用XML文件配置共享程序集的位置，但要注意的是，这会导致所有程序集都无法独立决定是否卸载程序集的文件。

配置文件的codeBase元素实际标记了一个URL。URL可引用用户机器上的目录，也可以引用Web地址自动下载。

## 3.8 "运行时"如何解析类型引用

解析类型引用类型时，CLR可在以下三个地方找到类型。

+ 相同文件

  编译时便可以发现，称之为**早起绑定**。类型加载，继续执行。

+ 不同文件，相同程序集

  "运行时"确保被引用的文件在当前程序集元数据的FileDef表中，检测加载程序集清单文件的目录，加载被引用的文件，检测哈希值。

+ 不同文件，不同程序集

  如果引用的类型在其他程序集的文件中，“运行时”会加载被引用程序集的清单文件。如果需要的类型不在该文件中，就继续加载包含了类型的文件。

ModuleDef，ModuleRef和FileDef元数据表在引用文件时使用了文件名和拓展名。但AssemblyRef元数据表只使用文件名，无拓展名。

System.AppDomain的AssemblyResolve，ReflectionOnly-AssemblyReslove和TypeResolve时间注册回调函数，可以用于解决绑定异常，使程序运作。

详情请参照原书p72.

## 3.9 高级管理控制

``` xml
<?xml version="1.0"?>
<configuration>
	<runtime>
		<assemblyBinding xmlns ="urn:schemas-microsoft-com:asm.vl">
			<probing privatePath ="AuxFiles;bin\subdir"/>
			<dependentAssembly>
				<assemblyIdentity name ="SomeClassLibrary" publicKeyToken="32ab4ba12e0a69a1" culture="neutral"/>
				
				<bindingRedirect oldVersion="1.0.0.0" newVersion="2.0.0.0"/>
				
				<codeBase version="2.0.0.0" href="http://www.WIntellect.com/SomeClassLibrary.dll"/>
			</dependentAssembly>
			<dependentAssembly>
				<assemblyIdentity name ="TypeLib" publicKeyToken="1fe74e897abbcfe" culture="neutral"/>

				<bindingRedirect oldVersion="3.0.0.0-3.5.0.0" newVersion="4.0.0.0"/>
				
				<publisherPolicy apply ="no"/>
			</dependentAssembly>
		</assemblyBinding>
	</runtime>
</configuration>
```

| 属性             | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| configuration    | 公共语言运行库和 .NET Framework 应用程序所使用的每个配置文件中的根元素。 |
| runtime          | 包含有关程序集绑定和垃圾回收的信息。                         |
| assemblyBinding  | 包含有关程序集版本重定向和程序集位置的信息。                 |
| probing          | 指定加载程序集时公共语言运行库要搜索的应用程序基子目录。     |
| privatePath      | 必选 指定可能包含程序集的应用程序基目录的子目录，用分号分割。 |
| assemblyIdentity | 封装每个程序集的绑定策略和程序集位置。dependentAssembly`为每个程序集使用一个元素。 |
| codeBase         | 指定运行时在计算机上未安装共享程序集的情况下可以找到该程序集的位置。 |
| bindingRedirect  | 将一个程序集版本重定向到另一个版本。                         |
| publisherPolicy  | 指定运行时是否应用此程序集的发布服务器策略。                 |

### 示例说明

+ probing 

  查找弱命名程序集时，检测应用程序基目录下的AuxFiles和bin\subdir子目录。强命名程序集会在GAC和codeBase下查找，没有指定codeBase的情况下在私有路径下查找。

+ 第一个dependentAssembly，assemblyIdentity和bindingRedirect元素

  查找有控制着公钥标记32ab4ba12e0a69a1的组织发布、语言文化为中心的SomeClassLibrary程序集的1.0.0.0.版本时，改为定位同一程序集的2.0.0.0版本。

+ codeBase元素

  尝试查找上述的2.0.0.0版本时尝试在指定URL处发现她。也就是http://www.WIntellect.com/SomeClassLibrary.dll。codeBase也可以应用于弱命名程序集。

+ 第一个dependentAssembly，assemblyIdentity和bindingRedirect元素

  同理

+ publisherPolicy

  如果生成TypeLib程序的组织部署了发布者策略文件，CLR应该忽略该文件。

## 额外参考资料

+ [probing元素](https://www.cnblogs.com/rash/p/6054303.html)

+ [微软文档dependentAssembly](https://docs.microsoft.com/zh-cn/dotnet/framework/configure-apps/file-schema/runtime/dependentassembly-element)
