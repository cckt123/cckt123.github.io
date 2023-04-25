---
title: 《CLRViaC#》第13章-接口
date: 2022-03-27 10:33:28
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 13.1 类和接口继承

略

## 13.2 定义接口

接口可以对方法，事件，无参属性和有参属性进行定义。这些所有方法的本质都是方法，它们只是语法上的简化表达。但是接口不可以定义任何构造器方法，也不能定义任何实例字段。

虽然CLR允许接口定义静态方法、静态字段、常量和静态构造器，但CLS标准版的接口并不允许。

C#要用interface关键字定义接口，要为接口指定名称和一组实例方法签名。

``` csharp
public interface IDispoable
{
    void Dispose();
}

public interface IEnumerable
{
    IEnumerator GetEnumerator();
}

public interface Ienumerable<T>:IEnumerable
{
    new IEnumerator<T> GetEnumerator();
}

public interface ICollection<T> : IEnumerable<T>,IEnumerable
{
    void ADd(T item);
    void Clear();
    Boolean Contains(T item);
    void CopyTo(T[] array,Int32 arrayIndex);
    Boolean Remove(T item);
    Int32 Count{get;}
    Boolean IsReadOnly{get;}
}
```

在CLR看来，接口定义就是类型定义，也就是说，CLR会为接口类型对象定义内部数据结构，同时可通过反射机制来查询接口类型的功能。和类型一样，接口可在文件范围中定义，也可嵌套在另一个类型中。定义接口类型时，可指定你希望的任何可访问性。

依据约定，接口类型名称以I开头。CLR支持泛型接口和接口中的泛型方法。

## 13.3 继承接口

``` csharp
public interface IComparable<in T>
{
    Int32 CompareTo(T other);
}
```

``` csharp
using System;

//Point从System.Object派生，并实现了IComparable<T>
public sealed class Point : IComparable<Point>
{
    private Int32 m_x,m_y;
    
    public Point(Int32 x,Int32 y)
    {
        m_x = x;
        m_y = y;
    }
    
    //该方法为Point实现IComparable<T>.CompareTo()
    public Int32 Comparable(Point other)
    {
        return Math.Sign(Math.Sqrt(m_x*m_x+m_y*m_y)-Math.Sqrt(other.m_x*other.m_x+other.m_y*other.m_y));
    }
    
    public override String ToString()
    {
        return String.Format("{0},{1}",m_x,m_y);
    }
}

public static class Program
{
    public static void Main()
    {
     	Point[] points = new Point[]
        {
            new Point(3,3),
            new Point(1,2),
        };
        
        //下面调用由Point实现的IComparable<T>的CompareTo方法
        if(points[0].CompareTo(points[1])>0)
        {
            Point tempPoint = points[0];
            points[0] = points[1];
            points[1] = tempPoint;
        }
        Console.WriteLine("Points from closest to (0,0) to farthest:");
        foreach(Point p in points)
            Console.WriteLine(p);
    }
}
```

c#编译器要求就将实现接口的方法标记为public。CLR要求将接口方法标记为virtual。不将方法显式标记为virtual，编译器会自动将方法标记为virtual和sealed，这会阻止派生类重写接口方法。将方法显式标记为virtual，编译器就会将该方法标记为virtual，使派生类可以重写它。

派生类不能重写sealed的接口方法，但派生类可以重新继承同一个接口，并为接口方法提供自己的实现。

``` csharp
using System;

public static class Program
{
    public static void Main()
    {
        Base b = new Base();
        b.Dispose();
        ((IDisposable)b).Dispose();
        //Base's Dispose
        
        Derived d = new Derived();
        d.Dispose();
        ((IDisposable)d).Dispose();
        //Derived's Dispose
        
        b = new Derived();
        b.Dispose();//Base's Dispose
        ((IDisposable)b).Dispose();//Derived's Dispose
    }
}

internal class Base:IDisposable
{
    public void Dispose()
    {
        Console.WriteLine("Base's Dispose");
    }
}

internal class Derived : Base,IDisposable
{
    new public void Dispose()
    {
        Console.WriteLine("Derived's Dispose");
    }
}
```

## 13.4 关于调用接口方法的更多探讨

CLR允许定义接口类型的字段、参数或局部变量。使用接口类型的变量可以调用该接口定义的方法。此外，CLR允许调用Object定义的方法，因为所有类都继承了Object的方法。

``` csharp
String s = "Jeffrey";
//可以使用s调用在String,Object，IComparable，ICloneable，IConvertible,IEnumerable中定义的任何方法
ICloneable cloneable = s;
//使用cloneable只能调用ICloneable接口声明的任何方法
....
```

在这段代码中，所有变量都引用一个String对象，该对象在托管堆中，所以使用其中任何变量时，调用的任何方法都会影响这个String对象

和引用类型相似，值类型可实现零个或多个接口，但值类型的实例在转换为接口类型时必须装箱。这是由于接口变量是引用，必须指向堆上的对象，使CLR能检查对象的类型对象指针，从而判断对象的确切类型。调用已装箱值类型的接口方法时，CLR会跟随对象的类型对象指针找到类型对象的方法表，从而调用正确的方法。

## 13.5 隐式和显式接口方法实现

类型加载到CLR中时，会为该类型创建并初始化一个方法表。在这个方法表中，类型引入的每个新方法都要相应的记录项；另外，还为该类型继承的所欲虚方法都添加了记录项。继承的虚方法既有继承层次结构中的各个基类型定义的，也有接口类型定义的。所以，对于下面这个简单的类型定义：

``` csharp
internal sealed class SimpleType : IDisposable
{
    public void Dispose(){Console.WriteLine("Dispose");}
}
```

+ Object 定义所有虚实例方法。
+ IDisposable 定义的所有接口方法，这里只有Dispose。
+ SimpleType 引入新方法Dispose。

为了简化编程，C#编译器假定SimpleType引入的Dispose方法是对IDisposable的Dispose方法的实现。C#编译器将新方法和接口方法匹配之后，就会生成元数据，指明SimpleType类型的方法表中的两个记录项应引用同一个实现。

``` csharp
public sealed class Program
{
    public static void Main()
    {
        SimpleType st = new SimpleType();
        st.Dispose();
        IDisposable d = st;
        d.Dispose();
    }
}
```

由于C#要求公共Dispose方法同时是IDisposeable的Dispose方法的实现，所以会执行相同的代码。所以上述两个方法调用的结果相同。

``` csharp
internal sealed class SimpleType : IDisposable
{
    public void Dispose(){Console.WriteLine("public Dispose");}
    void IDisposable.Dispose(){Console.WriteLine("IDisposable Dispose");}
}

/*
在不该懂Main方法的前提下，结果为
public Dispose
IDisposable Dispose
*/
```

在C#之中，将定义方法的那个接口名称作为方法的前缀（IDisposable.Dispose()），就会创建 **显式接口方法实现（EIMI）**。注意C#不允许在定义显式接口方法时指定可访问性。但是，编译器生成方法的元数据时，可访问性会自动设为private，防止其他代码在使用类的实例的同时直接调用接口方法。

还要注意，EIMI方法不能标记为virtual，所以不能被重写。这是由于EIMI方法并不是真的是类型的对象模型的一部分，而是将接口和其进行链接，同时避免公开行为和方法。

## 13.6 泛型接口

泛型接口提供了出色的编译时类型安全性，有时接口定义的方法使用了Object参数和Object返回类型。在代码中调用这些接口方法时，可传递对任何类型的实例的引用。

``` csharp
private void SomeMethod1()
{
    Int32 x = 1,y = 2;
    IComparable c = x;
    //ComparaTo期待Int32 传递y一个Int32
    c.ComparaTo(y);
    //ComparaTo期待Int32 传递2 String造成编译错误
    c.ComparaTo("2");
}
```

接口方法理想情况下应该使用强类型，这也是为什么FCL包含泛型IComparable\<in T>接口原因。

``` csharp
private void SomeMethod2()
{
    Int32 x = 1,y = 2;
    IComparable<Int32> c = x;
    //ComparaTo期待Int32 传递y一个Int32
    c.ComparaTo(y);
    //ComparaTo期待Int32 传递2 String造成编译错误
    c.ComparaTo("2");
}
```

泛型接口的第二个好处是处理值类型装箱的次数会少很多。

注意FCL定义了IComparable，ICollection，IList和IDictonary等接口的泛型和非泛型版本。定义类型时要实现其中任何接口，一般应实现泛型版本。FCL为了保留非泛型版本是为了向后兼容。

有的泛型接口继承了非泛型版本，所以必须同时实现接口的泛型和非泛型版本，例如，泛型IEnumerable\<out T>接口继承了非泛型IEnumerable接口，所以实现IEnumerable\<out T>就必须实现的IEnumerable。

泛型接口的第三个好处在于类可以实现同一个接口若干次，每次使用不同的类型参数。

``` csharp
using System;

public sealed class Number : IComparable<Int32>,IComparable<String>
{
    private Int32 m_val = 5;
    
    public Int32 CompareTo(Int32 n)
    {
        return m_val.CompareTo(n);
    }
    
    public Int32 CompareTo(String s)
    {
        return m_val.CompareTo(Int32.Parses(s));
    }
}

public static class Program
{
    public static void Main()
    {
        Number n = new Number();
        IComparable<Int32> cInt32 = n;
        Int32 result = cInt32.CompareTo(5);
        IComparable<String> cString = n;
        result = cString.CompareTo("5");
    }
}
```

## 13.7 泛型和接口约束

本节要讨论将泛型类型参数约束为接口的好处。

第一个好处在于，可将泛型类型参数约束为多个接口。这样一来，传递的参数的类型必须实现全部接口约束。

``` csharp
public static class SomeType()
{
    public static void Test()
	{
    	Int32 x = 5;
        Guid g = new Guid();
	   //对M的调用能通过编译，因为Int32实现了IComparable和IConvertible       
        M(x);
        //这个M调用导致编译错误，因为Guid虽然实现了IComparable但没有实现IConvertible
        M(g);
	}
    
    //M的类型参数T被约束为只支持同时实现了IComparable,IConvertible的类型
    private static Int32 M<T>(T t) where T : IComparable,IConvertible
    {
        ...
    }
}
```

定义方法参数时，参数的类型规定了传递的实参必须是该类型或它的派生类型。如果参数的类型是接口，那么实参可以是任意类类型，只要该类实现了接口。使用多个接口约束，实际是表示向方法传递的实参必须实现多个接口。

事实上，如果将T约束为一个类或者两个接口，就表示传递的实参类型必须是指定的基类，而且必须实现两个接口。这种灵活性使方法能细致地约束调用者能传递的内容。调用者不满足这些约束，就会产生编译错误。

接口约束的第二个好处是传递值类型的实例时减少装箱。上述代码向M方法传递了x。x传给M方法时不会发生装箱。如果M方法内部的代码调用t.CompareTo(...)，这个调用本身也不会引发装箱。

另一方面，如果M方法像下面这样声明

``` csharp
private static Int32 M(IComparable t)
{
    ...
}
```

那么x传递的过程中就必须要装箱。

C#编译器为接口约束生成特殊IL指令，导致直接在值类型上调用接口方法而不装箱。不用接口约束便没有其他办法让C#编译器生成这些IL指令，如此一来，在值类型上调用接口方法总会引发装箱。

一个例外是如果值类型实现了一个接口方法，那么在值类型的实力上调用这个方法不会造成值类型的实例装箱。

## 13.8 实现多个具有相同方法名和签名的接口

定义实现多个接口的类型时，这些接口可能定义了具有相同名称和签名的方法。例如，假定有如下两个接口：

``` csharp
public interface IWinodw
{
    Object GetMenu();
}

public interface IRestaurant
{
    Object GetMenu();
}
```

要实现同时继承这两个接口的类，必须使用显示接口方法实现来实现这个类型的成员。

``` csharp
public sealed class MarioPizzeria : IWindow,IRestaurant
{
    Object IWindow.GetMenu(){}
    
    Object IRestaurant.GetMenu(){}
    
    public Object GetMenu(){}
}
```

因为这个类之中实现了多个GetMenu()方法，所以必须要告诉编译器GetMenu方法对应的是那个接口的实现。

``` csharp
MarioPizzeria mp = new MarioPizzeria();

mp.GetMenu();

IWindow window = mp;
window.GetMenu();

IRestaurant restaurant = mp;
restaurant.GetMenu();
```

## 13.9 用显式接口方法实现来增强编译时类型安全性

本节讨论在泛型接口不存在的情况下，要求使用Object,作为参数或者返回值的情况下利用EIMI减少拆装箱次数。

``` csharp
public interface IComparable
{
    Int32 ComparaTo(Object other);
}

internal struct SomeValueType : IComparable
{
    private Int32 m_x;
    public SomeValueType(Int32 x){m_x = x;}
    public Int32 CompareTo(Object other)
    {
        return (m_x - ((SomeValueType) other).m_x);
    }
}

public static void Main()
{
    SomeValueType v = new SomeValueType(0);
    Object o = new Object();
    Int32 n = v.CompareTo(v); //不希望的装箱操作
    n = v.CompareTo(o);		 //InvalidCastException
}
```

上述代码之中存在两个问题

+ 不希望的装箱操作

  作为实参传给CompareTo方法时必须装箱，因为CompareTo期待的是一个Object

+ 缺乏类型安全性

  代码能够通过编译，但是CompareTo方法内部试图将o转换为SomeValueType时抛出InvalidCastException。

``` csharp
internal struct SomeValueType ： IComparable
{
    private Int32 m_x;
    public SomeValueType(Int32 x){m_x = x;}
    
    public Int32 CompareTo(SomeValueType other)
    {
        return (m_x-other.m_x);
    }
    
    Int32 IComparable.CompareTo(object other)
    {
        return CompareTo((SomeValueType)other);
    }
}
```

第一个CompareTo方法获取SomeValueType作为参数，这样就不必将other转换为SomeValueType，所以去除后续用于强制转换的代码。在满足类型安全之后，SomeValueType还必须实现一个CompareTo方法来满足IComparable的协定。

``` csharp
public static void Main()
{
    SomaValueType v = new SomeValueType(0);
    Object o = new Object();
    Int32 n = v.CompareTo(v);//不发生装箱
    n = v.CompareTo(o);//编译时错误
}
```

不过，定义接口类型的变量会再次失去编译时的类型安全性，而且会再次发生装箱。

``` csharp
public static void Main()
{
    SomeValueType v = new SomeValueType(0);
    IComparable c = v;
    Object o = new Object();
    Int32 n = c.CompareTo(v);
    n = c.CompareTo(o);
}
```

## 13.10 谨慎使用显式接口方法实现

使用EIMI也可能造成一些严重后果，所以应该尽量避免使用EIMI。

+ 没有文档解释类型具体如何实现一个EIMI方法，也没有VS智能感应支持。
+ 值类型的实例在转换成接口时装箱。
+ EIMI不能由派生类型调用。

首先，文档在列出EIMI的时候并不会提供类型特有的帮助，只有接口方法的常规帮助。例如，Int32的文档表示他实现了IConbertible接口的所有方法。但是开发人员并不能直接调用这个方法，这容易产生误解与困惑。

``` csharp
public static void Main()
{
    Int32 x = 5;
    Single s = x.ToSingle(null);//报错
}
```

而我们需要在一个Int32上调用ToSingle，必须先将其转换为IConvertible。

``` csharp
public static void Main()
{
	Int32 x = 5;
    Single s = ((IConvertible)x).ToSingle(null);
}
```

这里类型转换的要求并不明确，会让许多开发人员产生困惑，同时值类型转换为接口会引发装箱，浪费内存损害性能。

``` csharp
internal class Base ： IComparable
{
    Int32 IComparable.CompareTo(Object o)
    {
        Console.WriteLine("Base's Comparable");
        return 0;
    }
}

internal class Base ： IComparable
{
    public Int32 CompareTo(Object o)
    {
        Console.WriteLine("Derived's CompareTo");
        //error
        base.CompareTo();
        return 0;
    }
}
```

Base类并没有一个可被调用的public或者protected的CompareTo方法，他只是提供了一个能用IComparable类型的变量来调用的CompareTo方法。

将Derived CompareTo方法修改。

``` csharp
public Int32 CompareTo(Object o)
{
    Console.WriteLine("Derived's CompareTo");
    
    //试图调用基类的EIMI导致无穷递归
    IComparable c = this;
    c.CompareTo(o);
    return 0;
}
```

Derived的公共CompareTo方法充当了Derived的IComparable.CompareTo方法的实现，所以造成了无限递归。这可以通过不继承IComparable接口来解决。

``` csharp
internal sealed class Derived : Base
{...}
```

有时不能因为想在派生类之中实现接口方法就将接口从类型之中删除。解决这个问题的最佳方法就是在基类之中除了提供一个被选为显式实现的接口方法，还要提供一个虚方法。然后Derived类可以重写虚方法。

``` csharp
internal class Base ： IComparable
{
    Int32 IComparable.CompareTo(Object o)
    {
        Console.WriteLine("Base's IComparable.CompareTo");
        return CompareTo(o);
    }
    
    public virtual Int32 CompareTo(Object o)
    {
        Console.WriteLine("Base's virtual CompareTo");
        return 0;
    }
}

internal sealed class Derived : Base,IComparable
{
    public override Int32 CompareTo(Object o)
    {
        Console.WriteLine("Derived's CompareTo");
        return base.CompareTo(o);
    }
}
```

**谨慎**使用EIMI

## 13.11 设计：基类还是接口

+ IS-A对比CAN-DO关系

  类型只能继承一个实现，如果派生类和基类不能建立IS-A，就使用基类而使用接口。接口意味着CAN-DO关系。如果多种对象能做某事，就为他们创建接口。

+ 易用性

  基类提供大量了功能，而接口必须重新实现。

+ 一致性实现

  接口协议按开发者预期实现有困难。

+ 版本控制

  向基类型添加一个方法，派生类型将继承新方法。一开始使用的就是一个能正常工作的类型，用户的源代码甚至不需要重新编译。而向借口添加新成员，会强迫借口继承者更改其源码并重新编译。
