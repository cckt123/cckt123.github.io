---
title: 《CLRViaC#》第5章-基元类型、引用类型和值类型
date: 2022-02-18 16:24:53
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 5.1 编程语言的基元类型

编译器直接支持的数据类型称为**基元类型**。他会直接映射到Framwork类库中存在的类型。

| C#基元类型 | FCL类型        | 符合CLS | 说明                                        |
| ---------- | -------------- | ------- | ------------------------------------------- |
| sbyte      | System.SByte   | 否      | 有符号8位值                                 |
| byte       | System.Byte    | 是      | 无符号8位值                                 |
| short      | System.Int16   | 是      | 有符号16位值                                |
| ushort     | System.UInt16  | 否      | 无符号16位值                                |
| int        | System.int32   | 是      | 有符号32位值                                |
| uint       | System.UInt32  | 否      | 无符号32位值                                |
| long       | System.int64   | 是      | 有符号64位值                                |
| ulong      | System.UInt64  | 否      | 无符号64位值                                |
| char       | System.Char    | 是      | 16位Unicode字符                             |
| float      | System.Single  | 是      | IEEE32位浮点数                              |
| double     | System.Double  | 是      | IEEE64位浮点数                              |
| bool       | System.Boolean | 是      | true/false                                  |
| decimal    | System.Decimal | 是      | 128位高精度值，通常用于不允许舍入的金融计算 |
| string     | System.String  | 是      | 字符数组                                    |
| object     | System.Object  | 是      | 略                                          |
| dynamic    | System.Object  | 是      | CLR中一致，C#中dynamic参与动态调度。        |

当转换安全的时候，C#允许基元类型的隐式转换。

不安全的转换意味着可能会丢失精度或者数量。不同的编译器可能会生成不同的截取方式来处理这种转换，而C#总是对结果进行截断而不是向上取整。

### checked unchecked

这部分内容已经看过很多次了，所以不做总结...

## 5.2 引用类型和值类型

C#的new操作符返回对象内存地址——即指向对象数据的内存地址。

1. 内存必须从托管堆分配。
2. 堆上分配的每个对象都有一些额外成员，这些成员必须初始化。
3. 对象中的其它字节(为字段而设)总是为0。
4. 从托管堆分配对象时，可能强制执行一次垃圾回收。

而值类型的实例一般在线程栈上分配，不包含指向实例的指针，因而相对轻量级。

### 划分引用类型与值类型

任何可以被称之为类的类型都是引用类型，而所有值类型都可以被称之为枚举或者结构。

所有值类型都从抽象类System.ValueType的直接派生，System.ValueType又是直接派生于System.Object。

所有枚举类型都从System.Enum抽象类型派生，后者又从System.ValueType派生。CLR和所有托管环境下的编程语言都给与了枚举类型特殊待遇。

虽然不能在定制值类型的时候为其选择基类型，但是如果愿意，可以令其实现一个或者多个接口。而且所有值类型都隐式密封。

在C#中Struct是值类型，而Class是引用类型。

### 设计自己的类型

除非满足以下条件，否则不应该设计为值类型。

+ 类型具有基元类型行为。也就是说，是十分简单的类型，没有成员会修改类型任何实例字段。如果类型没有提供会改变其字段的成员，就说该类型是不可变的。
+ 类型不需要从其他任何类型继承。
+ 类型也不派生出其他任何类型。

因为方法传递的同时会默认以传值的方式传入，这会导致值类型的复制，为了减缓这部分开支，所以在满足上述条件之后，应该

+ 类型的实例较小 16字节或更低
+ 类型的实例较大，但不作为方法实参传递，也不作为方法返回

### 值类型与引用类型的一些区别

+ 值类型对象有两种表示形式：**未装箱**和**已装箱**，详见下一节，引用类型总是处于已装箱的状态。
+ 值类型从**System.ValueType**派生。该类型提供了与**System.Object**相同的方法。但**System.ValueType**重写了**Equals**方法，能在两个对象的字段值完全相等的情况下返回true。同时他也重写了**GetHashCode**方法，以在计算哈希值的同时将值类型作为参考。这两个方法默认实现存在性能问题，所以在定义自己的值类型的时候需要重新考虑覆写他们。
+ 由于不能将值类型作为基类型来定义新的值类型或者新的引用类型，所以不应该在值类型中定义虚方法。所有方法都不能是抽象的，所有方法都隐式密封。
+ 引用类型的变量包含堆中的对象地址，引用类型的变量创建是默认初始化为null，表明当前不指向任何有效对象。值类型的变量总是包含其基础类型的一个值，而且值类型的素有成员都初始化为0。
+ 将值类型变量赋值给另一个值类型变量，会执行逐字段的复制。引用类型只会复制地址。
+ 两个或者多个引用类型可以同时引用堆中的同一个对象，所以对一个变量的操作可能会影响到另一个变量引用的对象。值类型不会有这种情况。
+ 未装箱的值类型不在堆上分配，因而定义其的方法不在活动之中值类型将会被释放。

## 5.3 值类型的拆箱与装箱

许多时候需要获取对值类型的实例的引用，这就涉及到了装箱的机制。下面是装箱步骤

1. 在托管堆中分配内存，分配的内存量是值类型各字段所需的内存量，还要加上托管堆所有对象都有的两个额外成员所需的内存量。
2. 值类型的字段复制到新分配的堆内存。
3. 返回对象地址，转换成了引用类型。

拆箱的过程是将引用类型转换成为值类型，但并不会完全复刻他的反序逻辑，拆箱实际只是获取指针的过程，指针指向一个对象中的原始值类型。他本身不会导致字段复制，而是在他完成之后往往会进行字段的复制。

1. 如果包含"对已装箱值类型实例的引用"的变量为null，抛出NullReferenceException异常。
2. 如果引用的对象不是所需值类型的已装箱实例，抛出InvalidCastExeption异常。注意，即使是隐式转换可行也会抛出该异常。

调用方法并传递值类型时，如果不存在与值类型对应的重载版本，那么调用的肯定是获取一个Object参数的重载版本，将值类型实例作为Object传递会导致装箱，从而对性能造成不利的影响。定义自己的类的时候，可以将类中的方法定义为泛型，而不必装箱。

未装箱的值类型相对比引用类型来说更加轻，因为

+ 不在托管堆上分配
+ 没有堆上的每个对象都有额外成员:"类型对象指针"和"同步块索引"。

由于未装箱值类型没有同步块索引，所以不能使用System.Threading.Monitor类型的方法让多个线程同步对实例访问。

虽然未装箱值类型没有类型对象指针，但是依然可以调用由类型继承或重写的虚方法。如果值类型重写了其中任何虚方法，那么CLR可以非虚的调用他们，因为值类型隐式密封，不可能有类型从他们中派生，而且这种方式不会导致值类型装箱。但如果虚方法需要调用基类中的实现，那么就需要对值类型进行装箱然后通过this指针将对一个堆对象的引用传递给基方法。

此外，将值类型的未装箱实例转向类型的某个接口时会对实例进行装箱，因为接口变量必须包含对堆对象的引用。

### 5.3.1 使用接口更改已装箱值类型中的字段

``` csharp

using System;
internal interface IChangeBoxedPoint
{
    void Change(Int32 x,Int32 y);
}

internal struct Point : IChangeBoxedPoint
{
    private Int32 m_x,m_y;
    
    public Point(Int32 x,Int32 y)
    {
        m_x = x;
        m_y = y;
    }
    
    public void Change(Int32 x,Int32 y)
    {
        m_x = x;
        m_y = y;
    }
    
    public override String ToString()
    {
        return String.Format("({0},{1})",m_x.ToString(),m_y.ToString());
    }
}

public sealed class Program
{
    public static void Main()
    {
        Point p = new Point(1,1);
        Object o = p;
        ...
        ((IChangeBoxedPoint)p).Change(4,4);
        Console.WriteLine(p);//1 1 
        ((IChangeBoxedPoint)o).Change(5,5);
        Console.WriteLine(o);//5 5 
    }
}
```

这里略过了一些内容，如果需要请查看原书。

在最后的两个阶段中，未装箱的Point p转型成为一个IChangeBoxedPoint。这个转型造成对p中的值进行装箱。然后再已装箱值上调用Change，这会导致其m_x，m_y发生变化，但是在Change返回之后，已装箱的对象将会立刻被GC，所以依然是11，而后者是已装箱的类型，所以会保留这次的修改。

需要注意的是，值类型应该是"不可变"的，也就是说，我们不应该定义任何会修改实例字段的成员。实际上我（原作者）建议将值类型的字段都标记为readonly，这样在尝试修改他们的时候编译器就会报错。

### 5.3.2 对象相等性和同一性

System.Object的实现如下

``` csharp
public class Object
{
    public virtual Boolean Equals(Object obj)
    {
        //如果两个引用指向同一个对象，他们肯定包含同一个值
        if(this == obj)return true;
        //假定对象不包含相同的值
        return false;
    }
}
```

显然这不合理，这会验证比较的引用是否是同一个，但是如果有两个不同的引用的值一样，就会返回错误的结果。原作者将这种比对方式称之**同一性**，而值也完全相同的比对方式称之为**相等性**。

因为Equals不合理的实现，所以实际上Equals的实现规则相对复杂。类型重写Equals方法的同时应该调用基类的Equals，除非他的基类就是Object。而且因为类型可以重写Object的Equals方法，所以不能再使用它来测试同一性。因而Object提供了静态方法ReferenceEquals

``` csharp
public class Object
{
    public static Boolean ReferenceEquals(Object objA,Object objB)
    {
        return (objA == objB);
    }
}
```

因而检测同一性的时候应该调用Reference方法，不应该使用==，因为某个操作数的类型可能重载了==，赋予了不同于验证同一性的语义。

System.ValueType就重写了Object的Equals方法，并进行了正确的实现来执行值的相等性检查。ValueType的Equals内部就是这么实现的

1. 如果obj实参为null,返回false。
2. 如果this和obj的实参指向不同类型的对象，那么返回false。
3. 针对类型定义的每个实例字段，都将this对象中的值与obj对象中的值进行比较。任何字段不相等返回false。
4. 返回true。ValueType的Equals方法不调用Object的Equals方法。

上述步骤三，CLR调用反射进行，因反射较慢，所以在定义自己的Equals函数的时候需要覆写实现。

定义自己的类型时，重现的Equals需要符合相等性的四个特征。

+ Equals必须自反，x.Equals(x)肯定返回true
+ Equals必须对称；x.Eqauls(y)和y.Equals(x)返回相同的值
+ Equals必须可传递;xEquals(y) y.Equals(z) x.Equals(y)
+ Equals必须一致。比较的两个值不变，Equals返回值也不能变

除此之外，还可能需要做

+ 让类型实现System.IEquatable<T>接口的Equals方法

  这个泛型接口允许定义类型安全的Equals方法。

+ 重载==和!=操作符方法

  通常应实现这些操作方法，在内部调用类型安全的Equals。

+ 如果需要排序，还应该实现System.IComparable的CompareTo方法和System.IComparable<T>的类型安全的CompareTo方法。

## 5.4 对象哈希码

选择算法来计算类型实例的哈希码时，应

+ 这个算法会提供良好的随机分步，是哈希表获得最佳的性能
+ 可以在算法之中调用基类的GetHashCode方法，并包含他的返回值，但一般不要调用Object或ValueType的GetHashCode方法。
+ 算法至少使用一个实例字段。
+ 理想情况下，算法使用的字段应该不可变，也就是说，字段应在对象构造时初始化，在对象生存期永不言变。
+ 算法执行速度尽量快
+ 包含相同值的不同对象应返回相同哈希码。

不要持久化哈希值，因为哈希算法或者其他原因，哈希值经常会发生变化。

## 5.5 dynamic基元类型

为了方便使用反射或者与其他组件进行通信，C#编译器允许将表达式的类型标记为dynamic。还可以将表达式的结果放到变量当中，并将变量类型标记为dynamic。代码使用dynamic表达式/变量调用成员时，编译器生成特殊的IL代码描述操作，这种特殊的代码成为payload有效载荷。

``` csharp
internal static class DynamicDemo
{
    public static void Main()
    {
        dynamic value;
        for(Int32 demo = 0;demo<2;demo++)
        {
            value = (demo == 0)?(dynamic)5:(dynamic)"A";
            value = value + value;
            M(Value);
        }
    }
    
    private static void M(Int32 n){Console.WriteLine("M(Int32):"+n);}
    private static void M(String s){Console.WriteLine("M(String):"+s);}
}

/*
M(Int32):10
M(String):AA
*/
```

如果字段，方法参数或方法返回值的类型是dynamic，编译器会将类型转换为System.Object，并在元数据中向字段、参数或返回类型应用System.Runtime.CompilerServices.DynmaicAttribute的实例。如果局部变量使用Dynamic，局部变量也会成为Object，但不会向局部变量应用DynamicAttribute，因为它限制在方法内部使用。由于dynamic其实就是Object，所以方法签名不能仅靠dynamic和Object的变化来区分。

编辑器允许dynamic隐式转型到其他类型，但是依然会验证类型安全性。

dynamic表达式其实是和System.Object一样的类型。编译器假定你在表达式上进行的任何操作都是合法的，所以不会生成任何警告或者错误。但如果在允许的时候进行无效的操作会丢出异常。

``` csharp
using Mincrosofr.Office.Interop.Excel;
...
public static void Main()
{
    Application excel = new Application();
    excel.Visable = true;
    excel.Workbooks.Add(Type.Missing);
    ((Range)excel.Cells[1,1].Value) = "Text in cell AI";
    //注意这里，如果excel.Cells[1,1]返回Object
    //那么需要转型才可以访问Value
}

//这里是对照组
using Mincrosofr.Office.Interop.Excel;
...
public static void Main()
{
    Application excel = new Application();
    excel.Visable = true;
    excel.Workbooks.Add(Type.Missing);
    excel.Cells[1,1].Value = "Text in cell AI";
    //dynamic类型不必显示的转换
}
```

下面这段代码展示如何利用反射在String目标上调用Contains方法，传递string实参，并存储结果。

``` csharp
Object target = "Jeffrey Richter";
Object arg = "ff";
//在目标上查找和希望的实参类型匹配的方法
Type[] argTypes = new Type[]{ arg.GetType() };
MethodInfo method = target.GetType().GetMethod("Contains",argTypes);
//在目标上调用方法，希望传递的实参
Object[] arguments = new Object[]{arg};
Boolean result = Convert.ToBoolean(method.Invoke(target,arguments));


//dynamic重写
dynamic target = "Jeffrey Richter";
dynamic arg = "ff";
Boolean result = target.Contains(arg);
```

C#编译器会生成payload代码，在允许时会根据对象的实际类型判断要执行什么操作，这些payload代码使用了称为**运行时绑定器**的类，不同编程语言定义了不同的运行时绑定器来封装自己的规则。C#“运行时绑定器”的代码在Microsfot.CSharp.dll程序集中，生成使用dynamic关键字的项目必须引用该程序集。编译器的默认响应文件CSC.rsp中已引用了该程序集。

C#在运行时会将一定量程序集加载到AppDomain中，详情请查看原书。但总而言之，这个步骤会产生额外的性能开销，加载所有的这些程序集以及额外的内存消耗，会对性能造成额外影响。如果程序中只有一两处需要动态行为，传统做法或许更加有效。
