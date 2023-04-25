---
title: 《CLRViaC#》第10章 属性
date: 2022-02-23 17:44:54
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 10.1 无参属性

面向对象设计和编程的重要原则之一就是数据封装，意味着类型的字段永远不应该公开，否则容易因为不恰当使用字段而破坏对象的状态。另外还有一些可能的原因促使我们封装对类型中的数据字段的访问。

+ 我们可能希望访问字段来执行一些额外的功能、缓存某些值或者推迟创建某些对象。
+ 可能希望以线程安全的方式访问某些字段
+ 字段可能是一个逻辑字段，他的值不由内存中的字节表示，而是通过计算获得

为了解决这个问题，我们应该使用**访问器**来访问私有字段，而使用访问器会导致额外代码的增多，以及无法直接引用字段的问题。因而编程语言和CLR提供一种名为属性的机制。定义属性是，取决于属性的定义，编译器在最后的托管程序集中生成以下两到三项。

+ 代表属性get访问器的方法。仅在属性定义了get访问器方法时生成。
+ 代表属性set访问器的方法。仅在属性定义了get访问器方法时生成。
+ 托管程序集元数据中的属性定义。这一项必然生成。

编译器在你指定的属性名之前自动附加get_或者set\_前缀来生成方法名。

### 10.1.1 自动实现的属性

**自动实现的属性**（Automatically Implemented Property AIP），声明属性而不提供get/set方法的实现，C#会自动为你声明一个私有字段。

``` csharp
public String name{get;set;}
```

要注意的是，如果使用AIP，属性必然是可读且可写的，任何访问器方法要显示实现，两个访问器方法都必须显示实现而不能使用AIP。

### 10.1.2 合理定义属性

属性和字段有一定不同，这里需要注意，属性本质上是在调用一个 **方法**。

+ 属性可以只读或者只写，但是字段往往是两者即可，除非使用readonly。
+ 属性方法可以抛出异常，而字段不能。
+ 属性不能作为out/ref参数传给方法，而字段可以。
+ 属性方法可能会花费较长的时间进行，而字段访问则总是立即完成。许多人使用属性是为了线程同步，而这可能会导致线程永远终止。
+ 连续多次调用，属性可能会返回不同的值，字段会每次都返回相同的值。
+ 属性方法可能会导致明显的副作用，而字段访问则用于不会有这种情况。类型的使用者应该能按照她选择的任何顺序设置类型定义的各个属性，而不会造成类型中出现不同的行为。
+ 属性方法可能需要额外的内存，或者返回的引用并非指向对象状态一部分，造成对返回对象的修改作用不到原始对象身上。

### 10.1.3 对象和集合初始化器

``` csharp
Employee e = new Employee(){Name = "Jeff",Age = 45};
```

对象初始化器语法真正的好处在于，它允许在表达式的上下文中编写代码，允许组合多个函数，进而增强代码的可读性。

``` csharp
String s = new Employee(){Name = "Jeff",Age = 45}.ToString().ToUpper();
```

作为补充，如果原本调用的是一个无参构造器，C#还允许省略起始大括号之前的圆括号。

``` csharp
String s = new Employee{Name = "Jeff",Age = 45}.ToString().ToUpper();
```

如果属性的类型实现了IEnumerable或IEnumerable\<T>接口，属性就会被认为是集合，使用集合的初始化将会被认为是一种相加操作，而非替换的操作。而如果属性的类型实现了IEnumerable或IEnumerable\<T>,但是没有提供Add方法，编译器就不允许使用集合初始化语法向集合之中添加数据项。

### 10.1.4 匿名方法

C#的匿名类型功能，自动声明不可变的元组(tuple)类型。

``` csharp
var o1 = new {Name = "Jeff",Year = 1964};
Console.WriteLine("Name={0},Year={1}",o1.Name,o1.Year);
```

第一行在new关键字之后没有指定类型名称，所以编译器会自动创建类型名称，这也是其为何称之为匿名。编译器遇到这行代码的同时，会推断每个表达式的类型，创建推断类型的私有字段，为每个字段创建公共只读属性，并创建一个构造器来接受所有这些表达式。在构造器的代码中，会用传给它的表达式的求职结果来初始化私有只读字段。

同时编译器会重写Equals和GetHashCode代码，因而匿名对象可以存放到哈希表当中。

``` csharp
//编译器支持用额外的两种语法声明匿名类型中的属性，能根据变量推断属性名和类型
String Name = "Grant";
DetaTime dt = DetaTime.Now;

var o2 = new {Name,dt.Year};
```

编译器会自动判断在匿名类型之中的同一类型，如果他们具有相同的结构，那么只会创建一个他们的定义，而后创建多个他们的实例。所谓相同的结构，就是指这些匿名类型之中每个属性有相同的类型和名称，而且这些属性的指定顺序相同。由于这种同一性，可以创建同一类型的数组，在其中包含一组匿名类型的变量。

``` csharp
var people = new []
{
    o1,
    new {Name = "K",Year = 1970},
    new {Name = "S",Year = 2020}
};

//注意，这里遍历的var是必须的
foreach(var person in people)
{
    Console.WriteLine("Person={0},Year={1}",person.Name,person.Year);
}
```

匿名类型的实例不能泄露到方法外部，方法原先不能接受匿名类型的参数，同样的，方法也不能返回匿名类型的参数。

### 10.1.5 System.Tuple 类型

和匿名类型相似，Tuple类型创建之后就无法更改。

``` csharp
//这里展示了如何利用Tuple传递返回两样信息
private static Tuple<Int32,Int32>MinMax(Int32 a,Int32 b)
{
    return new Tuple<Int32,Int32>(Math.Min(a,b),Math.Max(a,b));
}

//这里展示了如何调用
private static void TupleTypes()
{
    var minmax = MinMax(6,2);
    Console.WriteLine("Min={0},Max={1}",minmax.Item1,minmax.Item2);
}
```

需要注意的是，Tuple类型，属性一律被Microsoft称之为Item#，我们无法对其做出任何更改。切记要为其添加注释以标记每个Item究竟是什么含义。

编译器只能在调用泛型方法的时候推断泛型类型，调用构造器的时候则不能。因此，System命名空间还包含了一个非泛型静态类Tuple类，其中包含一组静态Create方法，能根据实参推断泛型类型。

``` csharp
private static Tuple<Int32,Int32>MinMax(Int32 a,Int32 b)
{
    return Tuple.Create(Math.Min(a,b),Math.Max(a,b));
}
```

## 10.2 有参属性

有参属性支持get访问器方法接受一个或者多个参数，set访问器方法接受两个或者更多参数。C#称其为索引器。

``` csharp
using System;
public sealed class BitArray
{
    private Byte[] m_byteArray;
    private Int32 m_numsBits;
    
    public BitArray(Int32 numBits)
    {
        if(numBits<=0)
            throw new ArgumentOutOfRangeException("numsBits must be>0");
        m_numsBits = numBits;
        m_byteArray = new Byte[(numBits+7)/8];
    }
}

//这里是索引器 有参属性
public Boolean this[Int32 bitPos]
{
    get
    {
        //验证实参合法性
        if((bitPos<0)||(bitPos>=m_numBits))
            throw new ArgumentOutOfRangeException("bitPos");
        return (m_byteArray[bitPos/8]&(1<(bitPos%8)))!=0;
    }
    set
    {
        //验证实参合法性
        if((bitPos<0)||(bitPos>=m_numBits))
            throw new ArgumentOutOfRangeException("bitPos",bitPos.ToString());
        if(value)
        {
            m_byteArray[bitPos/8] = (Byte)(m_byteArray[bitPos/8]|(1<<(bitPos%8)));
        }
        else
        {
            m_byteArray[bitPos/8] = (Byte)(m_byteArray[bitPos/8]&~(1<<(bitPos%8)));
        }
    }
}

//索引器使用 PS 这个举例也太长了...

//分配空间
BitArray ba = new BitArray(14);
for(Int32 x = 0;x<14;x++)
{
    //set
    ba[x] = (x%2==0);
}

for(Int32 x = 0;x<14;x++)
{
    //get
    Console.WriteLine(ba[x]);
}
```

CLR本身并不区分有参和无参属性，对于她来说，每个属性都是类型之中定义的一对方法和一些元数据。C#使用this[...]作为索引器的表达语法，也仅是C#的语法。因而，虽然CLR支持静态索引器属性，但是C#不支持。

编译器在索引器名称之前附加get\_/set_前戳，而有因为C#不允许开发人员指定索引器的名称，所以他们指定了一个默认的，Item。因而编译器最后生成的方法就是get\_Item和set\_Item。留意类型是否提供了名为Item属性，可以判断该类型是否提供了索引器。

C#允许一个类型定义多个索引器，只要索引器的参数集不同。

``` csharp
using System;
using System.Runtime.CompilerServices;
public sealed class SomeType
{
    public Int32 this[Boolean b]
    {
        get{return 0;}
    }
    
    public String this[Int32 a]
    {
        get{return null;}
    }
}
```

## 10.3 调用属性访问器方法时的性能

对于简单的get;set;方法，JIT编译器会将代码内联，并没有额外的性能开销。

## 10.4 属性访问器的可访问性

可为get与set设置不同的访问性。如果两个访问器需要不同的访问性，C#要求其属性本身必须指定限制最小的访问性。两个访问器中只能有一个访问等级高于属性。

``` csharp
public class SomeType
{
    private String m_name;
    public String Name
    {
        get{return m_name;}
        protected set{m_name = value;}
    }
}
```

## 10.5 泛型属性访问器方法

C#不允许属性引入自己的泛型类型参数。
