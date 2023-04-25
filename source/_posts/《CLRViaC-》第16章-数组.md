---
title: 《CLRViaC#》第16章-数组
date: 2022-06-21 09:33:36
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

CLR支持一维、多维和交错数组。所有的数组类型都隐式的从System.Array抽象类派生，后者又派生自System.Object。这意味着数组始终是引用类型，是在托管堆上分配的。在应用程序的变量或字段之中，包含的是对数组的引用而不是包含数组本身的元素。

``` csharp
Int32[] myIntegers;
myIntegers = new Int32[100];
```

除了数组元素，数组对象占据的内存块还包含一个类型对象指针、一个同步块缩影和一些额外的成员。该数组的内存地址被返回并保存到myIntegers变量中。

``` csharp
Control[] myControls;
myControls = new Control[50];
```

为了符合CLS（Common Language Specification/公共语言规范）的要求，所有数组都必须是0基数组，即最小索引为0。这样就可以用C#的方法创建数组，并将该数组的引用传递给其他语言。此外因0基数组是最常用的数组，因而微软花了大力气优化。不过CLR确实支持非0基数组，但是不提倡使用。

每个数组都关联了一些额外的开销信息，包含数组的秩、每一维的下限、每一维的长度、数组的元素类型。

应尽量使用一维零基数组，因为他们的性能是最佳的，有时候也将这种数组称之为SZ数组或者向量（vector）。有必要时也可以使用多维数组。

``` csharp
//创建一个二维数组，由Double值构成
Double[,] myDoubles = new Double[10,20];
//创建一个三维数组，由String引用构成
String[,,] myStrings = new String[5,3,10];
```

CLR还支持交错数组，即数组构成的数组，0基一维交错数组的性能和普通向量一样好。不过，访问交错数组的元素一位置必须进行两次或更多次数组访问。

``` csharp
//创建由多个Point数组构成的一维数组
Point[][] myPolygons = new Point[3][];
Point[0] = new Point[10];
Point[1] = new Point[20];
Point[2] = new Point[30];
```

**注意** ：CLR会验证数组索引的有效值。非法访问将抛出异常，允许访问数组范围之外的内存会破坏类型安全性，而且会造成潜在的安全漏洞，所以CLR不允许可验证的代码这么做。通常而言，边界索引的检测对性能并无太大影响。但如果依然担心CLR索引性能的问题，可以使用unsafe代码进行访问。

## 16.1 初始化数组元素

C#允许你在创建数组对象的同时初始化数组中的元素。

``` csharp
String[] names = new String[]{"Aidan","Grant"};
```

大括号中以逗号分隔的数据项被称之为 **数组初始化器**。这个可以嵌套使用。

在方法中声明局部变量来引用初始化好的数组时，可利用C#的“隐式类型的局部变量”功能来简化一下下述代码。

``` csharp
var names = new String[]{"Aidan","Grant"};
```

可利用C#的隐式类型的数组功能让编译器推断数组元素的类型。

``` csharp
var names = new[]{"Aidan","Grant","null"};
```

编译器会检查数组中用于初始化数组元素的表达式类型，并选择所有元素最接近的共同基类来作为数组的类型。本例中，编译器发现String和null，由于null可以被隐式转型为任意引用类型，所以编译器推断可以使用String。

``` csharp
var names = new[]{"Aidan","Grant",123};//error
```

String和Int的共同基类是Object，也就是说123需要装箱，C#团队认为隐式对数组元素装箱的开销太大，所以编译时报错。

你还可以这么写。

``` csharp
String[] names = {"Aidan","Grant"};
```

需要注意的是，你不能使用隐式类型的局部变量。

``` csharp
var names =  {"Aidan","Grant"};//error
```

理论上是可以的，但是C#团队认为在这里编译器会为你做太多的工作。

+ error CS0820:无法用数组初始值设定项初始化隐式类型的局部变量。
+ error CS0622:只能使用数组初始值设定项表达式为数组类型赋值，请尝试改用new表达式。

``` csharp
var kids = new[]{new {Name="Adian"},new{Name="Grant"}};
for(var kid in kids)
    Console.WriteLine(kid.Name);
```

## 16.2 数组转型

对于元素为引用类型的数组，CLR允许将数组元素从一种类型转型为另一种类型。成功转型要求数组维数相同，而且必须存在从元素源类型到目标类型的隐式或显示转换。CLR不允许将值类型的元素转换为其他任何类型。

``` csharp
FileStream[,] fsdim = new FileStream[5,10];
Object[,] o2dim = fsdim;//隐式转换
Stream[] s1dim = (Stream[])o2dim;//Error 二维无法转一维
Stream[,] s2dim = (Stream[,])o2dim;//显式转换
String[,] st2dim = (String[,])o2dim;//编译通过，运行InvaildCastException
Int32[] ildim = new Int32[5];
Object[] oldim = (Object[])ildim;//error,无法转换值类型
Object[] obldim = new Object[ildim.length];
Array.Copy(ildim,obldim,ildim.Length);//创建一个新数组，将其转入
```

**Array.Copy** 的作用不仅仅是将元素从一个数组复制到另一个。Copy方法还能正确处理内存的重叠区域，就像C的 **memmove** 函数。有趣的是，C的 **memcpy**函数反而不能正确处理重叠的内存区域。Copy方法还能在复制每个数组元素时进行必要的类型转换。

+ 将值类型的元素装箱为引用类型的元素，比如将一个 **Int32[]** 复制到一个 **Object[]** 之中。
+ 将引用类型的元素拆箱为值类型的元素，比如将一个 **Object[]** 复制到一个 **Int32[]** 中。
+ 加宽CLR基元值类型，例如将一个 **Int32[]** 元素复制到一个 **Double[]**中。
+ 在两个数组之间复制时，如果仅从数组类型证明不了两者的兼容性，比如从 **Object[]** 转型为 **IFormattable[]**，就根据需要对元素进行向下类型转换。如果  **Object[]** 中每个对象都实现了 **IFormattable** 接口，**Copy** 方法就能正常运行。

``` csharp
internal struct MyValueType : IComparable
{
    public Int32 CompareTo(Object obj)
    {
        ...
    }
}

public static class Program
{
    public static void Main()
    {
        //创建含有100个值类型的数组
        MyValueType[] src = new MyValueType[100];
        
        //创建IComparable引用数组
        IComparable[] dest = new IComparable[src.Length];
        
        Array.Copy(src,dest,src.Length);
    }
}
```

有时需将数组从一种类型转换为另一种类型，这种功能被称之为 **数组的协变性** 。需要注意的是，在使用他的时候要注意性能的问题。

``` csharp
String[] sa = new String[100];
Object[] oa = sa;
oa[5] = "Jeff";//性能损失，CLR检测OA的元素类型是否为String
oa[3] = 5;	   //性能损失，CLR检测OA的元素类型是否为Int32 error
```

如果只是需要将数组的某些元素复制到另一个数组，可选择System.Buffer的BlockCopy方法，比Array的Copy方法更快，但Buffer的BlockCopy方法值支持基元类型，不提供转型能力。 方法中的Int32代表数组的字节偏移量，实质上是将数组类型复制到另一个按位兼容的数据类型。

要讲数组的元素可靠的复制到另一个数组，应该使用System.Array的ConstrainedCopy方法。该方法要么完成复制，要么丢出异常。另外，它不执行任何装箱拆箱或相加类型转换。

## 16.3 所有数组都隐式派生自System.Array

``` csharp
FileStream[] fsArray;
```

如上声明，CLR会自动从AppDomain创建一个FileStream[]类型。该类型隐式派生于System.Array类型。因而可使用System.Array的所有实例方法和属性。

需要请查看微软文档，此处不做摘抄。

## 16.4 所有数组都隐式实现IEnumerable,ICollection,IList

许多方法都能操作多种集合对象，因为他们声明允许获取IEnumerable,ICollection和IList等参数。可将数组也传递给这些方法，因为System.Array也实现了三个接口。System.Array之所以能实现这里非泛型接口，是因为这些接口将所有的元素都视为System.Object。然而，最好是让System.Array实现这些接口的泛用形式。

但是，由于设计多维数组和非0基数组的问题，CLR团队不希望System.Array实现IEnumerable\<T>，IList\<T>，ICollection\<T>。因为在System.Array上定义这个接口会导致所有数组类型都启用这些接口，所有CLR并没有这么做，而是创建一维0基数组的时候，CLR让数组自动实现了IEnumerable\<T>，IList\<T>，ICollection\<T>。

注意，如果数组包含值类型的元素，数组类型不会为元素的基类型实现接口。这是因为值类型的数组在内存中的布局与引用类型的数组不同。

## 16.5 数组的传递与返回

默认使用浅拷贝。如果不想这么做，应该构建一个新数组，并通过Array.Copy方法返回新数组的引用。 **注意** ：Array.Copy执行的是浅拷贝。

如果定义返回数组引用的方法，且数组中不包含元素，那么方法既可以返回null，也可以返回对包含零个对象的数组引用，强烈建议后者。

## 16.6 创建下限非零的数组

 创建和操作下限非0的数组是被允许的。可以调用数组的静态CreateInstance方法来动态创建自己的数组。此方法有若干重载版本，用于指定数组元素的类型、维数、每一维的下限和每一维的元素数目。CreateInstance为数组分配内存，将参数信息保存到数组的内存块的开销部分。

如果数组的维数是2或者2以上，那么可以把CreateInstance返回的引用类型转型为一个ElementType[]变量。如果只有一维，那么C#要求必须使用GetValue和SetValue进行访问。

## 16.7 数组的内部工作原理

CLR内部支持两种数组

+ 下限为0的一维数组
+ 下限位置的一维或多维数组

``` csharp
using System;

public sealed class Program
{
    public static void Main()
    {
        Array a;
        a = new String[0];
        Console.WriteLine(a.GetType());//"System.String[]"
        
        a = Array.CreateInstance(typeof(string),new Int32[]{0},new Int32[]{0});
        Console.WriteLine(a.GetType());//"System.String[]"
    
        //创建一维1基数组，其中不包含任何元素
        a = Array.CreateInstance(typeof(string),new Int32[]{0},new Int32[]{1});
        Console.WriteLine(a.GetType());//"System.String[*]"
        
        a = new String[0,0];
        Console.WriteLine(a.GetType());//"System.String[,]"
        
        a = Array.CreateInstance(typeof(string),new Int32[]{0,0},new Int32[]{0,0});
        Console.WriteLine(a.GetType());//"System.String[,]"
    
        //创建一维1基数组，其中不包含任何元素
        a = Array.CreateInstance(typeof(string),new Int32[]{0,0},new Int32[]{1,1});
        Console.WriteLine(a.GetType());//"System.String[,]"
    }
}
```

需要注意的是，对于1基数组，显示的是System.String[\*]。*表示CLR知道数组不是0基的。但是C#不允许声明System.String[\*]类型的变量，因此不能使用C#语法来访问一维非0基数组。Array的GetValue和SetValue方法可以访问，但是有额外的开销。

访问速度的差异的多方面原因造成的，比如说特殊的IL指令，让JIT生成优化代码。

``` csharp
using System;

public static class Program
{
    public static void Main()
    {
        Int32[] a = new Int32[5];
        for(Int32 index = 0;index<a.Length;index++)
            ...
    }
}

//例如上述代码，JIT会知道Length是Array的一个属性，在生成的代码中会只调用该属性一次，储存到临时变量当中
//不要写一点“高明”的代码的重复这个步骤
```

CLR和C#还允许使用unsafe代码访问数组，这种技术实际能在访问数组时关闭索引上下限检查。可适用于SByte、Byte、Int16、UInt16、Int32、UInt32、Int64、UInt64、Char、Single、Double、Decimal、Bealean、枚举或字段为上述任何类型的值类型结构。因为它直接访问内存，使用应 **谨慎**，更多有关unsafe的内容在第2章已经介绍，这里不再记录，详细请查看原书。

+ 相较于其他技术，unsafe处理数组元素的代码更复杂，要使用C# fixed进行内存地址的计算。
+ 计算出错会破坏内存数据，损坏类型安全性。
+ 因为潜在的问题，某些平台禁止使用unsafe代码。

## 16.8 不安全的数组访问和固定大小的数组

unsafe的数组访问功能强大，因为它允许访问

+ 堆上托管数组对象中的元素
+ 非托管堆上的数组中的元素
+ 线程栈上数组中的元素

在线程栈上分配数组是通过C#的stackalloc语句来完成的，这行语句只能创建一维0基、由值类型元素构成的数组，且值类型绝对不能包含任何引用类型的字段。使用该功能需要启动C#编译器的unsafe选项。

``` csharp
using System;

public static class Program
{
    public static void Main()
    {
    	StackallocDemo();
        InlineArrayDemo();
    }
    
    private static void StackallocDemo()
    {
        unsafe
        {
            const Int32 width = 20;
            Char* pc = stackalloc Char[width];
            String s = "Jeffrey Richter";
            for(Int32 index = 0;index<width;index++)
                pc[width-index-1] = (index<s.Length)?s[index]:'.';
        }
    }
    
    private static void InlineArrayDemo()
    {
        unsafe
        {
        	CharArray ca;
            Int32 widthInBytes = sizeof(CharArray);
            Int32 width = widthInBytes /2;
            String s = "Jeffery Richter";
            
            for(Int32 index = 0;index<width;index++)
            {
                ca.Characters[width-index-1] =  (index<s.Length)?s[index]:'.';
            }
        }
    }
}

internal unsafe struct CharArray
{
    public fixed Char Characters[20];
}
```

通常由于数组是引用类型，所以结构中定义的数组字段实际上只是指向数组的指针或引用，数组本身在结构的内存的外部，但也可以和CharArray一样嵌入结构。嵌入要求

+ 类型必须是结构，不能在类中嵌入数组
+ 字段或其定义结构必须是unsafe关键词
+ 数组字段必须用fixed关键字标记
+ 数组必须是一维0基数组
+ 元素类型必须是SByte、Byte、Int16、UInt16、Int32、UInt32、Int64、UInt64、Char、Single、Double、Decimal、Bealean

### fixed

`fixed` 语句可防止垃圾回收器重新定位可移动的变量。 `fixed` 语句仅允许存在于`fixed`上下文中。 还可以使用 `fixed` 关键字创建`fixed`。

仅可在 `fixed` 上下文中使用指向可移动托管变量的指针。 如果没有 `fixed` 上下文，垃圾回收可能会不可预测地重定位变量。 C# 编译器只允许将指针分配给 `fixed` 语句中的托管变量。

## 额外参考

+ [Microsoft_Fixed文档](https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/fixed-statement)
