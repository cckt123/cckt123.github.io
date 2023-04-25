---
title: 《CLRViaC#》第21章-托管堆和垃圾回收
date: 2022-10-12 17:51:14
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"

---

## 20.1 托管堆基础

1. 调用IL指令 newobj，为代表资源的类型分配内存（C# new）
2. 初始化内存，设置资源的初始状态并使资源可用。类型的实例构造器负责设置初始状态。
3. 访问类型的成员来使用资源。
4. 摧毁资源的状态进行清理。
5. 释放内存。

现在，只要写的是可验证，类型安全的代码，应用程序就不可能出现内存被破坏的情况。内存依然有可能被泄露，但是不会是像之前一样是默认行为。现在内存泄露一般是因为集合内存储了对象，但不需要对象的时候没有进行删除。

为进一步简化编程，开发人员经常使用的大多数类型都不需要进行步骤4。

使用需要特殊清理的资源时，编程模型还是像刚才描述的那么简单。需要尽快清理资源的时候，可以实现Dispose方法。

### 20.1.1 从托管堆分配资源

CLR要求所有对象从托管堆分配，进程初始化时，CLR划出一个地址空间区域作为托管堆。CLR还要维护一个指针，称之为NextObjPtr。该指针指向下一个对象在堆中的分配位置。

一个区域被非垃圾对象填满的时候，CLR会分配更多区域。这个过程将会一直重复，直到整个进程空间都被填满。所以，应用程序的内存受进程的虚拟地址空间的限制。32位最多可分配1.5G，64位最多8TB。

C#的new 操作符导致CLR执行以下步骤。

1. 计算类型的字段所需的字节数。
2. 加上对象的开销所需的字节数。每个对象都有两个开销字节：类型对象指针和同步块索引。
3. CLR检测区域中是否有分配对象所需的字节数。如果托管堆有足够的可用空间，就在NextObjPtr指针的地址处放入对象，为对象分配的字节将会清零。而后调用类型的构造器，new操作符返回对象引用。

### 21.1.2 垃圾回收算法

应用程序在调用new操作符创建对象的同时，如果空间不足，CLR就执行垃圾回收。

对于对象生存周期的管理，某些系统，例如微软自己的组件对象模型用的就是引用计数。此类系统之中，堆上的每个对象都维护着一个内存字段来统计程序中对其的引用数量。引用计数的通病是其很难处理循环引用。

由于引用计数的问题，CLR使用一种引用跟踪算法。引用跟踪算法只关心引用类型的变量，因为只有这种变量才能引用堆上的对象；值类型变量直接包含值类型实例。将所有引用类型的变量称之为 **根**。

CLR开始GC时，首先暂停进程中的所有线程。这样可以防止线程在CLR检查期间访问对象并更改其状态。然后，CLR进入GC的标记阶段。在这个阶段，CLR遍历堆中的所有对象，将同步块索引字段中的一位设置为0。这表明所有对象都应该被删除。然后，CLR检查所有活动根，查看他们引用了哪些对象。这正是CLR的GC称之为引用跟踪GC的原因。

CLR知晓哪些对象保留之后，对已标记的对象进行压缩，保证内存连续性。

完成之后，NextOjectPtr指针将会指向最后一个幸存对象之后的位置。

### 21.1.3 垃圾回收和调试

一旦根离开作用域，它引用的对象就会不可达，GC会回收其内存。

```csharp
using System;
using System.Threading;

public static class Program
{
    public static void Main()
    {
    	Timer t = new Timer(TimerCallback,null,0,2000);
	    Console.ReadLine();    
    }
    
    private static void TimerCallback(Object o)
    {
        Console.WriteLine();
        
        GC.Collect();
    	//发现没有任何程序引用t，将其回收，只执行一次。
    }
}c
```

可以尝试修改。

```csharp
using System;
using System.Threading;

public static class Program
{
    public static void Main()
    {
    	Timer t = new Timer(TimerCallback,null,0,2000);
	    Console.ReadLine();    
        t = null;
    }
    
    private static void TimerCallback(Object o)
    {
        Console.WriteLine();
        
        GC.Collect();
    }
}
```

但是这样使用并不会有任何效果，因为JIT实际上是一个优化编译器，将局部变量或者参数变量设置为null，等价于根本不引用该变量。JIT会将这行代码直接优化掉。

```csharp
using System;
using System.Threading;

public static class Program
{
    public static void Main()
    {
    	Timer t = new Timer(TimerCallback,null,0,2000);
	    Console.ReadLine();    
        t.Dispose();
    }
    
    private static void TimerCallback(Object o)
    {
        Console.WriteLine();
        
        GC.Collect();
    }
}
```

## 21.2 代：提升性能

CLR的GC是基于代的垃圾回收器

- 对象越新，生存期越短
- 对象越老，生存期越长
- 回收堆的一部分，速度快于回收整个堆

CLR初始化时为第0代对象选择一个预算容量。如果分配一个新对象造成第0代超过预算，就必须启动一次垃圾回收。在垃圾回收器中存留的对象将会成为第一代对象。之后垃圾回收将会优先第0代对象，残存的对象将会被压缩到第一代。直到第一代的空间预算超过预期，才会进行压缩。产生第二代。这样的步骤最多到第二代。

### 21.2.1 垃圾回收触发条件

- 代码显式调用System.GC的静态方法Collect方法

  显式请求CLR执行回收。

- Windows报告低内存情况

  CLR内部使用Win32函数CreateMemoryResourceNotification和QueryMemoryResourceNotification监视系统的总体内存使用情况。

- CLR正在卸载AppDomain

  一个AppDomain卸载时，CLR认为进程中一切都不是根。对象有机会进行资源清理，但CLR不会试图压缩或释放内存。

### 21.2.2 大对象

CLR将对象分为大对象和小对象。目前认为85000字节或更大是大对象。

- 大对象不是在小对象的地址空间分配，而是在进程地址空间的其他地方分配。
- 目前版本的GC不压缩大对象，因为在内存中移动他们代价过高。
- 大对象总是第二代。

### 21.2.3 垃圾回收模式

CLR启动后选择一个GC模式，直到进程终止不会发生变化。

- 工作站

  针对客户端优化GC，造成延迟较低。

- 服务器

  该模式针对服务器应用程序优化GC。被优化的主要是吞吐量和资源利用。CG假定机器上没有运行其他应用程序，机器上的所有CPU都可以用于辅助完成GC。

应用程序默认以”工作站”GC模式运行。寄宿了CLR的服务器应用程序可请求CLR加载服务器GC。

### 21.2.4 强制垃圾回收

System.GC类型允许应用程序对垃圾回收器进行一些直接控制。

还可以直接调用GC类的Collect方法强制垃圾回收。

大多数时候都要避免直接调用Collect方法，让垃圾回收器自行斟酌。

如果发生某些非重复事件，导致大量对象死亡，需要进行回收，就可以考虑手动调用一次Collect方法。

### 21.2.5 监视应用程序的内存使用

GC类提供了以下静态方法，可以调用他们用于查看某代发生了多少次垃圾回收，或者托管堆中的对象当前使用了多少内存。

```CSHARP
Int32 CollectionCount(Int32 generation);
Int64 GetTotalMemory(Boolean forceFullCollection);
```

## 21.3 使用需要特殊清理的类型

有些类型除去需要内存之外，还需要使用本机资源。

包含本机类型的资源被GC时，GC会回收对象在托管堆中使用的内存。但这样会造成本机资源的泄露。为此，CLR提供了名为 终结 的机制。允许对象在被判定为垃圾之后，在对象内存被回收之前执行一些代码。

最终基类Object定义了受保护的虚方法Finalize。垃圾回收器判定对象是垃圾后，会调用对象的Finalize方法。

```csharp
internal sealed class SomeType
{
    ~SomeType()
    {
        
    }
}
```

被视为垃圾的对象在垃圾回收完毕之后才会调用Finalize方法，所以这些对象的内存不是马上被回收的，因为Finalize方法可能要执行访问字段的代码。可终结对象在回收时必须存活，造成它被提升到另一代，使对象存活周期加长。因而尽量避免使用这个语法。

另外要注意，Finalize方法的执行时间是控制不了的，应用程序请求更多内存时才可能发生GC，而只有GC完成之后才运行Finalize。另外，CLR不保证多个Finalize方法的调用顺序。因而访问在Finalize方法中访问定义其类型的类可能会产生诸多问题。

CLR用一个特殊的，高优先级的专用线程调用Finalize方法规避死锁问题。如果Finalize方法阻塞，那么该特殊线程就调用不了任何更多的Finalize方法。

综上所述，Finalize方法问题较多，使用需要谨慎。

而使用创建封装了本机资源的托管类型时，应该先从System.Runtime.InteropService.SafeHandle这个特殊基类派生出一个类。

详细暂略。

### 21.3.1 使用包装了本机资源的类型

举例而言，FileStream对象在构造时会调用Win32 CreateFile函数，函数返回的句柄保存到SafeFileHandle对象中，然后通过FileStream对象的一个私有字段来维护对该对象的引用。

```csharp
using System;
using System.IO;

public static class Program
{
    public static void Main()
    {
        Byte[] bytesToWrite = new Byte[]{1,2,3,4,5};
        FileStream fs = new FileStream("Temp.dat",FileMode.Create);
        fs.Write(bytesToWrite,0,bytesToWrite.Length);
        File.Delete("Temp.dat");
    }
}
```

例如上述代码，他可能会正常工作，但是大多数时候不能，问题在于其File的静态Delete方法要求Windows删除一个依然打开的文件。所以Delet会丢出异常。

但某些情况下，文件是可以被删除的。比如说另一线程不知为何造成了一次垃圾回收，节点恰好发生在调用Wirte之后，调用Delete之前，那么Finalize方法将会被调用，关闭文件。

类如果想允许使用者控制类锁包装的本机资源的生存期，就必须实现如下所示的IDisposable接口

```csharp
public interface IDisposable
{
    void Dispose();
}
```

FileStream实现了IDisposable接口。

```csharp
using System;
using System.IO;

public static class Program
{
    public static void Main()
    {
        Byte[] bytesToWrite = new Byte[]{1,2,3,4,5};
        FileStream fs = new FileStream("Temp.dat",FileMode.Create);
        fs.Write(bytesToWrite,0,bytesToWrite.Length);
        fs.Dispose();
        File.Delete("Temp.dat");
    }
}
```

需要注意的是，并非一定要调用Dispose才能保证本机资源得到清理。本机资源的清理最终总会发生，调用Dispose只是控制这个清理动作的发生时间。此外，调用Dispose只是控制这个清理动作的发生时间。另外，调用Dispose不会将托管对象从托管堆中删除。只有在GC后，才会回收内存。

如果显式调用Dispose，最好将其放于finally块中，保证其最终一定会被执行。

```csharp
using System;
using System.IO;

public static class Program
{
    public static void Main()
    {
        Byte[] bytesToWrite = new Byte[]{1,2,3,4,5};
        FileStream fs = new FileStream("Temp.dat",FileMode.Create);
        
        try
        {
       		fs.Write(bytesToWrite,0,bytesToWrite.Length);
        }
        finally
        {
            if(fs!=null)fs.Dispose();
        }
        File.Delete("Temp.dat");
    }
}
```

using简化语句。

```CSHARP
using System;
using System.IO;

public static class Program
{
    public static void Main()
    {
        Byte[] bytesToWrite = new Byte[]{1,2,3,4,5};
        using(FileStream fs = new FileStream("Temp.dat",FileMode.Create))
        {
            fs.Write(bytesToWrite,0,bytesToWrite.Length);
        }
        File.Delete("Temp.dat");
    }
}
```

### 21.3.2 有趣的依赖问题

FileStream只支持字节的写入，写入字符和字符串可以使用一个System.IO.StrreamWriter，如下所示。

```csharp
FileStream fs = new FileStream("DataFile.dat",FileMode.Create);
StreamWriter sw = new StreamWriter(fs);
sw.Write("Hi there");
sw.Dispose();
//StreamWriter.Dispose会调用FileStream.Dipose
```

如果没有显示调用Dispose，那么垃圾回收器会自动调用，但是不保证顺序，因而可能会导致StreamWriter无法写入，丢出异常。微软的方案是，StreamWriter不支持Finally，需要显式Dispose。

### 21.3.3 GC为本机资源提供的其他功能

本机资源有时会消耗大量内存，但是用于包装它的托管对象只占用很少的内存。

```csharp
public static void AddMemoryPressure(Int32 bytesAllocated);
public static void RemoveMemoryPressure(Int64 bytesAllocated);
```

GC提供如上两个静态方法，用于提示垃圾回收器实际需要消耗多少内存。

### 21.3.4 终结的内部工作原理

new操作符从堆中分配内存，如果对象的类型定义了Finalize方法，那么在该类的实例构造器调用之前，会指向该对象的指针放到了一个终结列表。

### 21.3.5 手动监控和控制对象的生存期