---
title: 《CLRViaC#》第11章 事件
date: 2022-02-24 17:15:07
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 11.1 设计公开事件的类型

这里假设设计一个电子邮件系统。电子邮件抵达的时候，用户可能希望将邮件传递给传真机或者寻呼机。先设计名为MailManager的类型来接收传入的电子邮件，公开NewMail事件。其他类型的对象登记对该事件的关注。MailManager收到新邮件的同时会引发事件，分发邮件给每一个登记的对象。每个对象都用他们的方式来处理邮件。

应用程序初始化时只实例化一个MailManager实例，然后可以实例化任意数量的Fax（传真机）和Page（网页）对象。

1. Fax对象中的一个方法登记对MailManager的NewMail事件的关注。
2. Pager对象中的一个方法登记对MailManager的NewMail事件的关注。
3. 邮件抵达MailManager。
4. MailManager对象将事件通知发送给所有已登记的方法，这些方法以自己的方式处理邮件。

### 11.1.1 第一步 定义类型来容纳所有需要发送给事件通知接受者的附加信息

事件引发时，引发事件的对象可能希望向接收事件通知的对象传递一些附加信息。这些附加信息需要封装到它自己的类当中，该类通常包含一组私有字段，以及一些公开这些字段的公共属性。依照约定，该类从System.EventArgs派生，类名从EventArgs结束。

``` csharp
internal class NewMailEventArgs:EventArgs
{
    private readonly String m_from,m_to,m_subject;
    
    public NewMailEventArg(String from,String to,String subject)
    {
        m_from = from;m_to = to;m_subject = subject;
    }
    
    public String From{get{return m_from;}}
    public String To{get{return m_to;}}
    public String Subject{get{return m_subject;}}
}
//示例，该类声明了收件人，发件人，主题的信息
```

``` csharp
//FCL之中有EventArgs的示例
[ComVisible(true),Serializable]
public class EventArgs
{
    public static readonly EventArgs Empty = new EventArgs();
    public EventArgs(){}
}
```

### 11.1.2 第二步：定义事件成员

事件成员使用C#关键字event定义，每个事件成员都要包含:

+ 可访问性修饰符-几乎都是public
+ 委托类型-指出调用方法的原型
+ 名称

``` csharp
internal class MailManger
{
    public event EventHandler<NewMailEventArgs> NewMail;
    ...
}
```

事件成员的类型是EventHandler\<NewMailEventArgs>，意味着事件通知的所有接受者都必须提供一个原型和EventHandler\<NewMailEventArgs>委托类型匹配的回调方法。

```csharp
//泛型System.EventHandler委托类型的定义如下
public delegate void EventHandler<TEventArgs>(Object sender,TEventArgs e);
//所以方法原型必须具有以下形式
void MethodName(Object sender,NewMailEventArgs e);
```

### 11.1.3 第三步：定义负责引发事件的方法来通知事件的登记对象

按照约定，类要定义一个受保护的虚方法。引发事件时，类及派生类之中的代码会调用该方法。方法只需要获取一个参数，即一个NewMailEventArgs对象，其中包含了传给接收通知的对象的信息。方法的默认实现只是检查一下是否有对象登记了对事件的关注。如果有，就引发事件来通知事件的登记对象。

```csharp
internal class MailManager
{
    ...
    // 如果类是密封的，该方法要声明为私有和非虚
    protected virtual void OnNewMail(NewMailEventArgs e)
    {
        //出于线程安全的考虑，现在将对委托字段的引用复制到一个临时变量当中
        EventHandler<NewMailEventArgs> temp = Volatile.Read(ref NewMail);
        //如果任何方法登记了对事件的关注，通知他们
        if(temp != null)temp(this,e);
    }
    ...
}
```

上述的逻辑中有提到一个线程安全的概念。

``` csharp
//这里是最初的版本，并没有考虑线程安全的问题
protected virtual void OnNewMail(NewMailEventArgs e)
{
    if(NewMail!=null)NewMail(this,e);
}
/*
	他的问题在于在检查完成是否为null到执行的过程中，有可能存在其他线程将NewMail移除的可能性。
*/
protected virtual void OnNewMail(NewMailEventArgs e)
{
    EventHandler<NewMailEventArgs> temp = NewMail;
    if(temp!=null)temp(this,e);
}
//这里的处理方案是保存复制检测null时的引用，但是可能会遇到的问题是编译器自动将这段代码优化到上述第一段代码
//而最初的Volatile.Read代码则强调了一定要在调用之后读取
//而为了简化代码，将其写成拓展方法
public static class EventArgExtensions
{
    public static void Raise<TEventArgs>(this TEventArgs e,Object sender,ref EventHandler<TEventArgs> eventDelegate)
    {
        EventHandler<TEventArgs> temp = Volatile.Read(ref eventDelegate);
        if(temp!=null)temp(sender,e);
    }
}

protected virtual void OnNewMail(NewMailEventArgs e)
{
    e.Raise(this,ref m_NewMail);
}
```

### 11.1.4 第四步:定义方法将输入转化为期望事件

``` csharp
internal class MailManager
{
    public void SimulateNewMail(String from,String to,String subject)
    {
        NewMailEventARgs e = new NewMailEventArgs(from,to,subject);
        OnNewMail(e);
    }
}
```

## 11.2 编译器如何实现事件

``` csharp
public event EventHandle<NewMailEventArgs> NewMail;
//C#编译器在编译的过程中将其编译成为以下三个构造
//1. 一个被初始化为null的私有委托字段
private EventHandler<NewMailEventArgs> NewMail = null;
//2. 一个公共add_Xxx方法 
//允许方法登记对事件的关注
public void add_NewMail(EventHandler<NewMailEventArgs> value)
{
    //通过循环和对CompareExchange 的调用，可以以一种线程安全的方式对事件添加委托
    EventHandler<NewMailEventArgs> preHandler;
    EventHandler<NewMailEventArgs> newMail = this.NewMail;
    do
    {
        prevHandler = newMail;
        EventHandler<NewMailEventArgs> newHandler = (EventHandler<NewMailEventArgs>) Delegate.Combine(prevHandler,value);
        newMail = Interlocked.CompareExchange<EventHandler<NewMailEventArgs>>(ref this.NewMail,newHandler,prevHandler);
    }while(newMail!=prevHandler);
}

//3. 一个公共remove_Xxx方法
//允许方法注销对事件的关注
public void remove_NewMail(EventHandler<NewMailEventArgs> value)
{
    //通过循环和CompareExchange 的调用，可以以一种线程安全的方式从事件之中移除一个委托
    EventHandler<NewMailEventArgs> preHandler;
    EventHandler<NewMailEventArgs> newMail = this.NewMail;
    do
    {
        prevHandler = newMail;
        EventHandler<NewMailEventArgs> newHandler = (EventHandler<NewMailEventArgs>) Delegate.Remove(prevHandler,value);
        newMail = Interlocked.CompareExchange<EventHandler<NewMailEventArgs>>(ref this.NewMail,newHandler,prevHandler);
    }while(newMail!=prevHandler);
}
```

## 11.3 设计侦听事件的类型

这里演示如何定义一个类型来使用另一个类型提供的事件。

``` csharp
internal sealed class Fax
{
    //将MailManager对象传给构造器
    public Fax(MailManager mm)
    {
        //构造EventHandler<NewMailEventArgs>委托的一个实例
        //使他引用我们的FaxMsg回调方法
        //向MailManager的NewMail事件登记我们的回调方法
        mm.NewMail += FaxMsg;
    }
    
    //新电子邮件到达时，MailManager将调用这个方法
    private void FaxMsg(Object sender,NewMailEventArgs e)
    {
        //sender表示MailManager对象，便于将信息传回给他
        //e表示MailManager对象想传给我们的附加事件信息
        //这里的代码正常情况下应该传真电子邮件
        //但这个测试性的实现只是控制台上显示邮件
        Console.WriteLine("Faxing mail message:");
        Console.WriteLine("From = {0},To={1},Subject={2}",e.From,e.To,e.Subject);
    }
    
    //执行这个方法，Fax对象将向NewMail事件注销自己对它的关注
    //以后不在接收通知
    public void Unregister(MailManager mm)
    {
        //向MailManager的NewMail事件注销自己对这个事件的关注
        mm.NewMail -= FaxMsh;
    }
}
```

对象只要向事件登记了它的一个方法，便不能被垃圾回收。所以，如果你的类型要实现IDsiposable的Dispose方法，就应该在实现中注销对所有事件的关注，详见21章。

C#会要求使用者使用+=与-=而不是add与remove来增减委托，如果强行使用会报错。

## 11.4 显式实现事件

使用数据结构来存储已存在的事件，显式进行控制。

``` csharp
using System;
using System.Collections.Generic;
using System.Threading;

//这个类的目的是在使用EventSet时，提供多一点类型安全性与代码可维护性
public sealed class EventKey{}

public sealed class EventSet
{
    private readonly Dictionary<EventKey,Delegate> m_events = new Dictionary<EventKey,Delegate>();
    
    public void Add(EventKey eventKey,Delegate handler)
    {
        Monitor.Enter(m_events);
        Delegate d;
        m_events.TryGetValue(eventKey,out d);
        m_events[eventKey] = Delegaet.Combine(d,handler);
        Monitor.Exit(m_events);
    }
}
```

