---
title: 《CLRViaC#》第26章-线程基础
date: 2022-11-07 17:51:14
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"


---

## 26.1 Windows为什么要支持线程

计算机早期，没有线程的概念，系统只运行一个执行线程，同时包括系统代码和应用程序代码。问题在于长时间运行的任务会影响其他任务的进行。

为解决这些问题，新的OS内核在进程之中运行每个应用程序的实例。进程实际上是应用程序的实例要使用的资源的合集。每个进程都被赋予了一个虚拟地址空间，确保在一个进程中使用的代码和数据无法由另一个进程访问。

线程的职责是对CPU进行虚拟化，Windows为每个进程都提供线程，而一个线程上的应用程序卡死不会影响其他线程。

## 26.2 线程开销

- 线程内核对象

  创建线程时分配初始化的数据结构，包含对线程进行描述的属性与线程上下文等。

- 线程环境块

  在应用程序代码能快速访问的地址空间中分配的内存块。包含异常处理链首。

- 用户模式栈

  存储传给方法的局部变量和实参。包含一个地址，支出当前方法返回，线程应该在什么位置继续执行。

- 内核模式栈

  向操作系统的内核函数传递代码的时候使用。

- DLL线程连接和线程分离通知

  任何时候创建线程，都会调用进程中加载的所有非托管DLL的DLLMain方法，向该方法传递DLL_THREAD_ATTACH标志，类似的，销毁线程的时候也会调用类似的方法。

### 上下文切换

单CPU计算机一次只能做一件事，所以Windows必须在系统中的所有线程之间共享物理CPU。

Windows任意时刻只将一个线程分配给一个CPU。那个线程能运行一个“时间片”的长度，时间片到期就进行上下文切换到另一个线程。而每次上下文切换，Windows会

1. 将CPU寄存器的值保存到当前正在运行的线程的内核对象内部的一个上下文结构中。
2. 从现有线程集合只中选出一个线程供调度。
3. 将上下文结构中的值加载到CPU寄存器中。

Windows大概需要30ms完成一次上下文切换。

上下文切换会影响很多性能，每次切换都会清空CPU高速缓存，但是用户体验会更稳定。

此外，GC的过程中，CLR将会挂起所有线程，遍历他们的栈来查找根以便对其中的对象进行标记，而后再遍历一次恢复所有线程。

简而言之，尽量避免上下文切换。

## 26.3 停止疯狂

任何机器最优的线程数就是那台机器的CPU数量。

## 26.4 CPU发展趋势

略

## 26.5 CLR线程和Windows线程

暂略

## 26.6 使用专门线程执行异步的计算限制操作

建议避免使用该技术，Thread类不可用，WindowsStore应用时根本无法使用该技术，尽量使用线程池等方式来执行异步的计算限制操作。

例如，满足一下条件，就可以显示创建自己的线程。

- 线程需要以非普通线程优先级运行。所有线程池线程都以普通优先级运行，虽然可以更改正优先级，但不建议。
- 需要线程表现为一个前台线程，防止应用程序在线程结束任务前终止。
- 计算限制的任务需要长时间运行。线程池为了判断是否需要创建一个额外的线程，所采用的逻辑是比较复杂的。直接为其创建一个线程，以规避长时间运算。
- 要启动线程，并有可能Thread.Abort提前终止他。

为创建专有线程，要构造System.Threading.Thread类的实例，向构造器传递一个方法名。

```csharp
CSHARP
public sealed class Thread : CriticalFinalizerObject,...
{
    public Thread(ParameterizedTreadStart start);
}
```

start参数标识专用线程要执行的方法，这个方法必须和ParameterizedThreadStart委托的签名匹配

```csharp
CSHARP
delegate void ParameterizedThreadStart(Object obj);
```

构造Thread对象是个轻量级的操作，它不实际上创建一个操作系统线程。而是实际创建并操作系统线程，执行回调，必须调用Thread的Start方法。

```csharp
CSHARP

using System;
using System.Threading;
public static class Program()
{
    public static void Main()
    {
        Thread dedicateThread = new Thread(ComputeBoundOp);
        dedicateThread.Start(5);
        Thread.Sleep(10000);
        dedicateThread.Join();//线程终止
    }
    
    private static void ComputeBoundOp(Object state)
    {
        Thread.Sleep(1000);
    }
}
```

## 26.7 使用线程的理由

- 可响应性
- 性能

略，不必多说。

## 26.8 线程调度和优先级

抢占式操作系统必须使用算法判断在什么时候调度线程多长时间。每个线程的内核对象都包含一个上下文结构，反应了线程上一次执行完毕后CPU寄存器的状态。在一个时间片之后，Windows检测所有现有线程的内核对象，寻找可调度的线程，切换上下文。

Windows之所以被称之为抢占式多线程操作系统，是因为线程可以再任意时刻停止并调度另一线程。

每个线程都分配了从0到31(最高)的优先级。

只要存在可调度等级为31的线程，系统就永远不会调度0~30的任意线程。这种情况称之为饥饿。

优先级较高的线程总是会抢占低优先级的线程，无论正在运行的是什么较低优先级的线程。较低线程的时间片将会被暂停，无论他是否被用完，而较高有限度的线程将会得到一个完整的时间片。

系统启动时会创建一个特殊的零页线程。该线程的优先级是0，而且是整个系统唯一优先级为0的线程。

设计应用程序时，你可以自己决定应用程序需要比机器上同时运行的其他应用程序有更大还是更小的优先级。WIndows支持以下进程优先级类：

- IDLE
- Below Normal
- Normal
- Above Normal
- High
- Realtime

在系统什么都不做的时候进行优先级为IDLE，例如屏保。

只有绝对必要的进程才应该使用High优先级类，Realtime要尽可能避免使用。因为其优先级相当高，有可能影响到操作系统任务。

然后是线程优先级

- Idle
- Lowest
- Below Normal
- Normal
- Above Normal
- Highest
- Time-Critical

线程的优先级相对于进程靠后，他们协作映射到0~31的优先级排列。

你的应用程序可更改其线程的相对线程优先级，这需要设置Thread的Priority属性，向其传递ThreadPriority枚举类型定义的值之一

- Lowest
- BelowNormal
- Normal
- AboveNormal
- Highest

## 26.9 前台线程和后台线程

CLR将所有线程划分为前台线程或者后台线程。一个进程的所有前台线程都被终止的时候，CLR将会终止他的所有后台线程。

所以，前台线程应该执行确实想要完成的任务，比如将数据从内存缓存区flush到磁盘。非关键线程则使用后台线程，比如重新计算电子表格的单元格，或者为记录建立索引等。

在线程的生存周期中，任何时候都可以从前台变成后台，或者从后台变成前台。应用程序的主线程以及通过构造一个Thread对象来显式创建的任何线程都默认为前台线程，相反，线程池线程默认为后台线程。