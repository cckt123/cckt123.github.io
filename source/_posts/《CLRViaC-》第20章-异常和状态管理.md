---
title: 《CLRViaC#》第20章-异常和状态管理
date: 2022-07-13 17:51:14
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 20.1 定义“异常”

行动成员代表类本身或者类型实例能执行的行动，当行动成员不能完成任务时，应抛出异常。

``` csharp
Boolean f = "Jeff".Substring(1,1).ToUpper().EndsWith("E");
```

例如上述代码，有很好的阅读性，但前提是中间没有操作失败。

## 20.2 异常处理机制

本章旨在提供何时以及如何使用异常处理的设计规范，而更多详细内容请参照微软的Structured Exception Handling，SEH文档。

``` csharp
private void SomeMethod()
{
    try
    {
        //需要得体地进行恢复/清理的代码
    }
    catch(InvalidOperationException)
    {
        //从InvalidOperationException恢复的代码
    }
    catch(IOException)
    {
        //IOException恢复的代码
    }
    catch
    {
 		//除上所述
        throw;
    }
    finally
    {
        //无论如何，最终都会执行finally
    }
}
```

### 20.2.1 try块

如果代码需要执行一般性的资源清理操作，从异常中恢复，可以放到try块中，负责清理的代码放到finally块中。

**提示** : 如果在一个try块中执行多个可能抛出同一个异常的代码，且不同的操作有不同的异常恢复措施，就应该将每一个操作放到自己的try块中。

### 20.2.2 catch块

catch关键字后圆括号所带的表达式称为捕捉类型。C#要求捕捉类型必须是System.Exception或者它的派生类型。

**注意** ：VS调试catch块时，可在监视窗口添加特殊变量名称$exception来查看当前抛出的异常对象。

CLR从上到下搜索匹配的catch块，所有应将较具体的异常放到顶部。也就是说，首先出现的是派生程度最大的异常类型，接着是他们的基类型，最后是System.Exception。事实上，如果顺序搞反，会导致报错。

在try块的代码中抛出异常，CLR将搜索捕捉类型与抛出的异常相同的catch块，如果没有任何捕捉类型与抛出的异常匹配，CLR会去调用栈更改的一层搜索与异常匹配的捕捉类型。

一旦CLR找到了匹配的catch块，会执行内层所有finally块中的代码，即从抛出异常的try块开始，到匹配异常的catch块之间的所有finally块。注意，匹配异常的那个catch块所关联的finally尚未执行，该finally快中的代码一直要等到这个catch块中的代码执行完成之后才会执行。

所有内层finally块执行完毕之后，匹配异常的那个catch块中的代码才开始执行。catch块中的代码通常执行一些对异常进行处理的操作。在其末尾，通常

+ 重新抛出相同的异常，向调用栈更高一层的代码通知异常的发生
+ 抛出一个新的异常，提供更丰富的异常信息
+ 让线程在catch块底部退出

当线程在catch底部退出后，他将立刻执行包含在finally块中的代码，finally块的所有代码执行完毕之后，线程退出finally块，执行之后的语句。

C# 允许在捕捉类型后指定一个变量。捕捉到异常时，该变量将引用抛出的System.Exception派生对象。catch块的代码可以通过引用该变量来访问异常的具体信息。

### 20.2.3 finally块

finally块中是保证会执行的代码。

``` csharp
private void ReadData(String pathname)
{
    FileStream fs = null;
    try
    {
        fs = new FileStream(pathname,FileMode.Open);
        //处理文件中数据...
    }
    catch(IOException)
    {
        //再次添加从IOException恢复的代码
    }
    finally
    {
        //确保文件被关闭
        if(fs!=null)fs.Close();
    }
}
```

线程在执行完成finally代码之后，会执行紧跟在finally块之后的语句。

**注意**：finally块中的代码应该是清理代码，这些代码只需对try块中发起的操作进行清理。catch和finally块中的代码应该非常短，且不会抛出异常。

当然，这种情况也是很有可能的。如果发生在catch和finally块中丢出异常的情况，CLR会正常运作，但不会记录对应try块中抛出的第一个异常，关于第一个异常的信息将会丢失，CLR会终止你的进程。

**注意**：CLR允许抛出任何对象来报告异常，但是C#编译器只允许代码抛出从Exception派生的对象，而其他语言不一定，有些语言允许抛出非Exception派生异常。

## 20.3 System.Exception类

CLR允许抛出任何类型的实例，但微软并没有让所有编程语言都抛出和捕获任意类型的异常。因此，他们定义了System.Exception类型，并规定所有CLS相容的编程语言都必须抛出和捕捉派生自该类型的异常。派生自System.Exception的异常类型被认为是CLS相容的。

System.Exception类属性如下，但不应以任何形式访问查询这些属性，应在应用程序因未处理的异常终止时在调试器中查看对应代码。

| 属性名称       | 访问  |    类型     | 说明                                                         |
| -------------- | :---- | :---------: | :----------------------------------------------------------- |
| Message        | 只读  |   String    | 指出抛出异常的原因。如果抛出的异常未处理，消息应该被写入日志。 |
| Data           | 只读  | IDicitonary | 引用一个“键值对”集合，代码在抛出异常前在抛出异常前在该集合中添加记录项；捕捉异常的代码可在异常恢复中查询记录项并利用信息。 |
| Source         | 读/写 |   String    | 包含生成异常程序的名称                                       |
| StackTrace     | 只读  |   String    | 包含抛出异常之前调用过的所有方法的名称和签名，该属性有益于调试。 |
| TargetSite     | 只读  | MethodBase  | 包含抛出异常的方法                                           |
| HelpLink       | 只读  |   String    | 包含帮助用户理解异常的文档URL                                |
| InnerException | 只读  |  Exception  | 如果当前异常是在处理一个异常时抛出的，该属性就指出上一个异常是什么。这个只读属性通常为null。Exception类型还提供了公共方法GetBaseException来遍历由内层异常构成的链表，并返回最初抛出的异常。 |
| HResult        | 读/写 |    Int32    | 跨越托管和本机代码边界时使用的一个32位值。                   |

catch块可读取StackTrace属性来获取一个堆栈跟踪，叙述了异常发生前调用了哪些方法。一个异常抛出时，CLR在内部记录throw指令的位置。一个catch块捕捉到该异常时，CLR记录捕捉位置。在catch块内访问被抛出的异常对象的StackTrace属性，负责实现该属性的代码会调用CLR内部的代码，后者创建一个字符串来指出从异常抛出到异常捕捉位置的所有方法。

**注意**：抛出异常时，CLR会重置异常起点，也就是说，CLR会记录最新的异常对象的抛出位置。

``` csharp
private void SomeMethod()
{
    try{...}
    catch(Exception e)
    {
        ...
        throw e;//CLR认为这是异常的起点
    }
}
```

但如果仅仅使用throw关键字本身来重新抛出异常对象，CLR就不会重置堆栈的起点。

``` csharp
private void SomeMethod()
{
    try{...}
    catch(Exception e)
    {
        ...
        throw;//不影响CLR对异常起点的认知 
    }
}
```

这两段代码的唯一区别只有对抛出起点的认识，无论是throw还是throw e，CLR都会重置栈的起点。因而，如果一个异常成为未处理的异常，那么想要获取他的失败位置就会很难。而为了解决这一点。

``` csharp
private void SomeMethod()
{
    Boolean trySucceeds = false;
    try
    {
        ..
        trySucceeds = true;
    }
    finally
    {
        if(!trySucceeds)
        {
            ....//在这里捕捉
        }
    }
}
```

StackTrace属性返回的字符串不包含调用栈中比接收异常对象的那个catch块高的任何方法。要获得从线程起始出到异常处理程序之间的完整堆栈跟踪，需要使用System.Diagnostics.StackTrace类型。该类型定义了一些属性和方法，允许开发人员程序化地处理堆栈跟踪以及构成堆栈跟踪的栈帧。

可用多个不同的构造器来构造一个StackTrace对象，一些构造器构造从线程起始处到StackTrace对象的构造位置的栈帧。另一些使用作为参数传递的一个Exception的StackTrace属性或者System.Diagnostics.StackTrace的ToString方法返回的字符串中。

如果CLR能找到你的程序集的调试符号，那么在System.Exception的StackTrace对象的构造位置的属性或者System.Diagnostics.StackTrace方法返回的字符串中，将包括源代码文件路径和代码行号。

获得堆栈跟踪后，可能发现实际调用栈中的一些方法没有出现在堆栈跟踪字符串中。这可能有两部分原因，首先，调用栈记录的是线程的返回位置。其次，JIT编译器可能进行了优化，将一些方法内联，以避免调用单独的方法并从中返回的开销。许多编译器可以通过打开Debug开关来告诉编译器不要进行内联，让开发人员获取更详细的内部信息。

**注意** ：JIT编译器会检查应用于程序集的System.Diagnostics.Debuggabletrribute定制特性，如果指定了DisableOptimizations标志，那么JIT编译器就不会内联。

## 20.4 FCL定义的异常类

FCL定义了许多异常类型，他们最终都从System.Exception类型派生。只有两个类型，System.SystemException和System.AppliactionException是唯一直接从Exception派生的类型。CLR抛出的所有异常都从System.Exception派生，应用程序抛出的所有异常都从ApplicationException派生。

但是很可惜，这只是理论上的结果，大家并没有按照规则执行，有很多异常派生顺次出错，因而无法通过System.SystemException和System.AppliactionException来完成catch所有类型的操作。

## 20.5 抛出异常

抛出异常要考虑两个问题

1. 抛出什么类型的异常
2. 向异常类型的构造器传递什么字符串消息

第一个问题，需要选择一个有意义的类型。可以直接使用FCL中定义好的异常类型，也可以自己从System.Exception中派生一个。建议定义浅而广的基类，因为基类的主要作用就是将大量的错误当成一个错误，这是非常危险的。同理，抛出System.Exception也是危险的。
