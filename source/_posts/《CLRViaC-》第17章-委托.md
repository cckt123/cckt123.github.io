---
title: 《CLRViaC#》第17章-委托
date: 2022-06-29 10:27:26
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 17.1 初识委托

```csharp
using System;
using System.Windows.Forms;
using System.IO;

//声明委托参数
internal delegate void Feedback(Int32 value);

public sealed class Program
{
    public static void Main()
    {
        StaticDelegateDemo();
        InstanceDelegateDemo();
        ChainDelegateDemo1(new Program());
        ChainDelegateDemo2(new Program());
    }
    
    public static void StaticDelegateDemo()
	{
    	Console.WriteLine("---- Static Delegate Demo ----");
        Counter(1,3,null);
        Counter(1,3,new Feedback(Program.FeedbackToConsole));
        Counter(1,3,new Feedback(FeedbackToMsgBox));
        Console.WriteLine();
	}
    
    private static void InstanceDelegateDemo()
    {
    	Console.WriteLine("---- Instance Delegate Demo ----");
        Program p = new Program();
        Counter(1,3,new Feedback(p.FeedbackToFile));
        Console.WriteLine();
    }
    
    private static void ChainDelegateDemo1(Program p)
    {
    	Console.WriteLine("---- Chain Delegate Demo1 ----");
        Feedback fb1 = new Feedback(FeedbackToConsole);
        Feedback fb2 = new Feedback(FeedbackToMsgBox);
        Feedback fb3 = new Feedback(p.FeedbackToFile);
        
        Feedback fbChain = null;
        fbChain = (Feedback)Delegate.Combine(fbChain,fb1);
        fbChain = (Feedback)Delegate.Combine(fbChain,fb2);
        fbChain = (Feedback)Delegate.Combine(fbChain,fb3);
        Counter(1,2,fbChain);
        
        Console.WriteLine();
        fbChain = (Feedback)Delegate.Remove(fbChain,new Feedback(FeedbackToMsgBox));
        Counter(1,2,fbChain);
    }
    
    private static void ChainDelegateDemo2(Program p)
    {
    	Console.WriteLine("---- Chain Delegate Demo2 ----");
        Feedback fb1 = new Feedback(FeedbackToConsole);
        Feedback fb2 = new Feedback(FeedbackToMsgBox);
        Feedback fb3 = new Feedback(p.FeedbackToFile);
        
        Feedback fbChain = null;
        fbChain += (Feedback)Delegate.Combine(fbChain,fb1);
        fbChain += (Feedback)Delegate.Combine(fbChain,fb2);
        fbChain += (Feedback)Delegate.Combine(fbChain,fb3);
        Counter(1,2,fbChain);
        
        Console.WriteLine();
        fbChain -= new Feedback(FeedbackToMsgBox);
        Counter(1,2,fbChain);
    }
    
    private static void Counter(Int32 from,Int32 to,Feedback fb)
    {
        for(Int32 val = from;val<=to;val++)
            if(fb!=null)fb(val);
    }
    
    private static void FeedbackToConsole(Int32 value)
    {
        Console.WriteLine("Item="+value);
    }
    
    private static void FeedbackToMsgBox(Int32 value)
    {
        MessageBox.Show("Item="+value);
    }
    
    private void FeedbackToFile(Int32 value)
    {
        using(StreamWriter sw = new StreamWriter("Status",true))
        {
            sw.WriteLine("Item="+value);
        }
    }
}
```

## 17.2 用委托回调静态方法

参照上一节的StaticDelegateDemo方法。

委托对象是方法的包装器，使方法能通过包装器来间接回调。

**注意** : FeedbackToConsole方法被定义成Program类型内部的私有方法，但Counter方法能调用Program的私有方法。这样没有问题，即使是Counter的方法在另一个类型中定义，也不会有问题。在一个类型中通过委托来调用另一个类型的私有成员，只要委托对象是具有足够安全/可访问性的代码创建的，就是没有问题的。

将方法绑定到委托时，C#和CLR都允许引用类型的 **协变性** 和 **逆变性** 。协变性是指方法能返回从委托类型派生的一个类型，逆变性是指方法获取的参数可以是委托的类型的基类。

**注意** ：只有引用类型才支持协变性和逆变性，值类型或Void不支持。

## 17.3 用委托回调实例方法

可参照第一节的InstanceDelegateDemo方法。

实例方法，委托需要知道方法具体操作的是哪个实例对象。包装实例方法很有用，因为对象内部的代码可以访问对象的实例成员。这意味着对象可以维护一些状态，并在回档反方执行期间利用这些状态信息。

## 17.4 委托揭秘

本节解释CLR和编译器如何协同工作来实现委托，介绍一些通过委托来实现的附加功能。

``` csharp
internal delegate void Feedback(Int32 value);
//编译器会将其装换为
internal class Feedback : System.MulticastDelegate
{
    public Feedback(Object @object,IntPtr method);
    
    public virtual void Invoke(Int32 value);
    
    public virtual IAsyncResult BeginInvoke(Int32 value,AsyncCallback callback,Object @object);
    
    public virtual void EndInvoke(IAsyncResult result);
}
```

在本例中，编译器定义了Feedback类，它派生自FCL定义的System.MulticastDelegate类型。（所有委托类型都派生于System.MulticastDelegate）

**注意** ：System.MulticastDelegate派生自System.Delegate，后者又派生自System.Object。是历史原因造成有两个委托类。

委托类型既可以嵌套在一个类型中定义，也可以在全局范围中定义，简单来说，由于委托是类，所以凡是能够定义类的地方，都能定义委托。

由于所有委托类型都派生自MulticastDelegate，所有它们都继承了MultciastDelegate的字段，属性和方法，其中需要注意

| 字段            | 类型          | 说明                                                         |
| --------------- | ------------- | ------------------------------------------------------------ |
| _target         | System.Object | 当委托对象包装一个静态方法时，这个字段为null，当委托对象包装一个实例方法时，这个字段引用的是回调方法要操作的对象。 |
| _methodPtr      | System.IntPtr | 一个内部的整数值，CLR用它表示要回调的方法。                  |
| _invocationList | System.Object | 这个字段通常为null，构造委托链时它引用一个委托数组。         |

构造委托时，C#编译器会为委托类型分析源代码来确定引用的是哪个对象和方法。对象引用被传给构造器的object参数，标识了方法的一个特殊IntPtr值被传给构造器的method参数。分别保存在_target和\_mthodPtr私有字段内。

调用回调方法时，编译器会自动调用Invoke方法

``` csharp
//如上述第一节的范例
fb(val);
//实质上和
fb.invoke(val);
//一样
```

## 17.5 用委托回调多个方法

委托链是委托对象的集合，可利用委托链调用集合中的委托所代表的所有方法。

如上述参考ChainDelegateDemo1方法，可使用Delegate的Combine方法将委托添加到链中。

``` csharp
fbChain = (Feedback)Delegate.Combine(fbChain,fb1);
fbChain = (Feedback)Delegate.Combine(fbChain,fb2);
fbChain = (Feedback)Delegate.Combine(fbChain,fb3);
```

当调用第二次调用Combine方法时，发现已引用一个委托对象，所以Combine会构造一个新的委托对象。新委托对象对他的_target和\_method方法进行初始化，\_invocationList字段被初始化为引用一个委托对象数组。

当第三次调用时，重新构筑一个新的数组，将原有_invocationList数组的第1、2项传入新数组，注意，原有的委托及其字段引用的数组会被gc。

当调用时，委托发现_invocationList不为null，所以执行循环依次遍历包装的每个方法。

Remove方法可以删除其中的委托，可见上述参考。

### 17.5.1 C#对委托链的支持

C#编译器自动重载了+=与-=操作符，自动调用Combine和Remove方法。

### 17.5.2 取得对委托链调用的更多控制

顺序调用委托有时候并不能完成所有的要求。MulticastDelegate类提供了一个实例方法GetInvocationList，用于显示调用链中的每一个委托，并允许你使用需要的任何算法。

``` csharp
public abstract class MulaticastDelegate : Delegate
{
    public sealed override Delegate[] GetInvocationList();
}
```

GetInvocationList方法通过操作从MulaticastDelegate派生的对象，返回包含Delegate引用的一个数组，其中每个引用都指向链中的一个委托对象。

这里有一个显式调用数组中每个对象的实例。

``` csharp
using System;
using System.Reflection;
using System.Text;

internal sealed class Light
{
    public String SwitchPosition()
    {
        return "The Light is off";
    }
}

internal sealed class Fan
{
    public String Speed()
    {
        throw new InvalidOperationException("The fan broke due to overheating");
    }
}

internal sealed class Speaker
{
 	public String Volume()
    {
        return "The volume is loud";
    }   
}

public sealed class Program
{
    private delegate String GetStatus();
    
    public static void Main()
    {
        GetStatus getStatus = null;
        
        getStatus += new GetStatus(new Lisght().SwitchPosition;
        getStatus += new GetStatus(new Fan().Speed);
        getStatus += new GetStatus(new Speaker().Volume);
                                   
        Console.WriteLine(GetComponentStatusReport(getStatus));
    }
                                   
 	private static String GetComponentStatusReport(GetStatus status)
    {
        if(status == null)return null;
        StringBuilder report = new StringBuilder();
        //获得一个数组，其中每个元素都是链中的委托
        Delegate[] arrayOfDelegates = new status.GetInvocationList();
        
        foreach(GetStatus getStatus in arrayOfDelegates)
        {
         	try
            {
                report.AppendFormat("{0}{1}{1}",getStatus(),Environment.NewLine);
            }   
            catch(InvalidOperationException e)
            {
                Object component = getStatus.Target;
                reprot.AppendFormat
                    (
                    	"Failed to get status from {1}{2}{0} Error:{3}{0}{0}",
                    	Environment.NewLine,
                    	((component==null)?"":component.GetType()+"."),
                    	getStatus.Method.Name,
                    	e.Message
                	);
            }
        }
        return report.ToString();
    }
}                               
```

## 17.6 委托定义不要太多/泛型委托

.Net Formwrok现在提供了17个Action委托，他们从无参数到最多16个参数。如需获取16个以上的参数，就必须定义自己的委托类型。

``` csharp
public delegate void Action();
public delegate void Action<T>(T obj);
...
public delegate void Action<T1,...,T16>(T1 arg1,...,T16 arg16);

public delegate TResult Func<TResult>();
..
public delegate TResult Func<T1,...,T16,TResult>(T1 arg1,...,T16 arg16);
```

尽量使用这些委托类型，而不是定义更多，这样可减少在系统中的类型数量，简化编码。

然而，如果委托要使用ref或out关键字以传引用的方式传递参数，就不得不定义自己的委托。同样，如果需要通过params关键词获取数量可变的参数，或为委托任何参数指定默认值，或者进行约束，都要自定义委托。

## 17.7 C# 为委托提供的简便语法

``` csharp
button1.Click += new EventHandler(button_Click);//为保证类型安全，构造新的委托
button1.Click += button_Click;
```

### 17.7.1 简化语法1 不需要构造委托对象

``` csharp
internal sealed class AClass
{
    public static void CallbackWithoutNewingAelegateObject()
    {
        TreadPool.QueueUserWorkItem(SomeAsyncTask,5);
    }
    
    private static void SomeAsyncTask(Object o)
    {
        Console.WriteLine(o);
    }
}
```

TreadPool.QueueUserWorkItem方法期待一个WaitCallBack **委托** 的引用，这里传递SomeAsyncTask，编译器自动推断，构造委托。

### 17.7.2 简化语法2 不需要定义回调方法/Lambda

C# 允许以内联方式直接书写回调代码。

``` csharp
internal sealed class AClass
{
    public static void CallbackWithoutNewingAelegateObject()
    {
        TreadPool.QueueUserWorkItem(obj=>Console.WriteLine(obj),5);
    }
}
```

编译器看到Lambda表达式之后，在类中定义个新的私有方法，编译器自动分配名称。

**注意** ：Lambda表达式没有办法向编译器生成的方法应用定制特性。不能使用方法修饰符，Lambda方法是否为静态取决于方法是否访问实例成员。不包含实例的时候默认生成静态方法，因其使用效率高于实例方法，不需要额外的this参数。

以下为lambda表达式示例。

``` csharp
Func<String> f = ()=>"Jeff";
//如果委托获取一个或更多参数，可以显示指定类型
Func<Int32,String> f2 = (Int32 n) => n.ToString();
Func<Int32,Int32,String> f3 = (Int32 n1,Int32 n2)=>(n1+n2).ToString();
//如果委托获取一个或更多参数，可由编译器推断类型
Func<Int32,String> f4 = (n)=>n.ToString();
Func<Int32,Int32,String> f3 = (n1,n2)=>(n1+n2).ToString();
//如果委托获取一个参数，可忽略（）
Func<Int32,String> f4 = n=>n.ToString();
//如果委托有ref/out参数，必须显式指定
Bar b = (out Int32 n)=>n=5;
//如果主题是两个或多个语句，那么必须用大括号封闭
Func<Int32,Int32,String> f7 = (n1,n2)=>{Int32 sum=n1+n2;return sum.ToString();};
```

### 17.7.3 简化语法3 局部变量不需要手动包装

有时希望回调代码引用存在于定义方法中的局部参数或变量。

``` csharp
internal sealed class AClass
{
    public static void UsingLocalVariablesInTheCallbackCode(Int32 numToDo)
    {
        //一些局部变量
        Int32[] squares = new Int32[numToDo];
        AutoResetEvent done = new AutoResetEvent(false);
        //在其他线程上执行的一系列任务
        for(Int32 n=0;n<squares.Length;n++)
        {
            ThreadPool.QueueUserWorkItem
            (
                obj=>
                {
                    Int32 num = (Int32)obj;
                    squares[num] = num*num;
                    if(Interlocked.Decrement(ref numToDo)==0)
                        done.Set();
                }
            ),n
        };
        
        done.WaitOne();
    }
}
```

编译器自动完成了定义一个新的辅助类，定义传入回调代码的每一个值对应的字段，在其中原有类中构筑辅助类实例，传入需要的数据，再在辅助类中定义lambda方法并调用。

**注意** lambda编译成类之后，参数/局部变量的生存时间被延长，现在只要包含字段的对象不被回收，那么原有的参数/局部变量就不会被回收。

## 17.8 委托和反射

某些情况下，并不确定回调方法需要多少参数，以及参数的具体类型。System.Delegate.MethodInfo提供了一个CreateDelegate方法，允许在编译时不知道委托的所有必要信息的前提下创建委托。

``` csharp
public abstract class MethodInfo : MethodBase
{
    //静态
    public virtual Delegate CreateDelegate(Type delegateType);
    //实例
    public virtual Delegate CreateDelegate(Type delegateType,Object target);
}
```

之后使用Delegate的DynamicInvoke方法调用他。

``` csharp
public abstract class Delegate
{
    //调用委托并传递参数
    public Object DynamicInvoke(params object[] args);
}
```

使用反射API首先必须获取引用了回调方法的一个MethodInfo对象，然后调用CreateDelegate方法来构造第一个参数delegateType所标识的Delegate派生类型的对象。如果委托包装了实例方法，还需要向CreateDelegate传递一个target参数，指定作为this参数传给实例方法的对象。

System.Delegate的Dynamic方法允许调用委托对象的回调方法，传递一组在运行时确定的参数，调用DynamicInvoke方法时，包装内部传递的参数与回调方法期望的参数兼容。不兼容丢出异常。
