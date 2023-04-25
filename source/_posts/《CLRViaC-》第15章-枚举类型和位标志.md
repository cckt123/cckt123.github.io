---
title: 《CLRViaC#》第15章-枚举类型和位标志
date: 2022-06-13 16:30:10
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 15.1 枚举类型

枚举类型定义了一组“符号名称/值”，比如说

``` csharp
internal enum Color
{
    White,
    Red,
    Green,
    Blue,
    Orange
}
```

+ 枚举类型易于维护查看，不必费心硬编码值的含义。
+ 枚举类型为强类型，例如将Color.Orange作为参数传递给要求Fruit枚举类型的方法会无法编译。

在Microsoft.NET framework中，枚举类型不只是在编译器所关心的符号，还是类型系统中的“一等公民”，可实现很多强大的操作。其他环境之中枚举类型并没有这个特点。

每个枚举类型直接从System.Enum派生，后者从System.ValueType派生，而System.ValueType又从System.Object派生。所以，枚举类型是值类型，可用未装箱和已装箱的形式来表达。有别于其他值类型，枚举类型不能定义任何方法、属性或事件。不过，可利用C#的“拓展方法”功能模拟向枚举类型添加方法。

编译枚举类型时，C#编译器把每个符号转换成类型的一个常量字段。

``` csharp
internal struct Color : System.Enum
{
    public const Color White = (Color)0;
    public const Color Red   = (Color)1;
    public const Color Greem = (Color)2;
    public const Color Blue  = (Color)3;
    public const Color Orange= (Color)4;
    
    //无法引用该字段
    public Int32 value__;
}
```

C#编译器不会实际编译上述代码，因为它禁止定义从特殊类型System.Enum派生类。

简单来说，枚举类型定义了一个包含一组常量字段和一个实例字段的结构体。常量字段会嵌入程序集的元数据中，通过反射来访问。这意味着可以在运行时获取与枚举类型关联的所有符号以及其值。

System.Enum基类型还提供了几个静态方法和实例方法，可利用他们操作枚举类型的实例，从而避免必须使用反射的问题。

``` csharp
public static Type GetUnderlyingType(Type enumType);//System.Enum中定义
public Type GetEnumUnderlyingType();//System.Type中定义
```

这些方法返回用于容纳一个枚举类型的值的基础类型，每个枚举类型都有一个基础类型，如byte,sbyte,short,ushort,int,uint,long,ulong。C#编译器为简化编译流程，要求只能指定基元类型名称。

``` csharp
internal enum Color: byte
{
    White,
    Red,
    Green,
    Blue,
    Orange
}
```

C#编译器将枚举类型视为基元类型，因而可应用基元类型的操作符来操作枚举类型的实例。这么操作符实际作用于每个枚举类型实例内部的value_实例字段，此外C#编译器允许将枚举类型转换为其他枚举类型或者数值类型。

``` csharp
Color c = Color.Blue;
Console.WriteLine(c);
Console.WirteLine(c.ToString());
Console.WriteLine(c.ToString("G"));
Console.WriteLine(c.ToString("D"));
Console.WriteLine(c.ToString("X"));

public static String Format(Type enumType,Object value,String format);
//可以在没有实例的情况下使用Format使用枚举类型输出
Console.WriteLine(Enum.Format(type(Color),3,"G"));//Blue
```

如果声明多个符号的枚举类型，所有符号可以由相同的数值，使用常规方法将数值转换为符号时，Enum的方法会返回其中一个符号，但不保证具体返回哪一个符号名称。另外，如果没有为要查找的数值定义符号，会返回包含该数值的字符串。

也可调用System.Enum的静态方法GetValues或者System.Type的实例方法GetEnumValues来返回数组，其中每个元素都对应枚举类型中的一个符号名称，每个元素都包含符号名称的数值。

``` csharp
public static Array GetValues(Type enumType);
public Array GetEnumValues();
```

除了GetValue方法，Sytsem.Enum和System.Type类型还提供了一下方法来返回枚举类型的符号。

``` csharp
public static String GetName(Type enumType,Object value);//System.Enum
public String GetEnumName(Object Value);//System.Type

public static String[] GetNames(Type enumType);//System.Enum
public String[] GetEnumNames();//System.Type
```

Enum的静态IsDefined方法和Type的IsEnumDefined方法

``` csharp
public static Boolean IsDefined(Type enumType,Object value);
public Boolean IsEnumDefined(Object value);

//可用IsDefined判断某枚举类型是否合法
Console.WriteLine(Enum.IsDefined(typeof(Color),1));
Console.WriteLine(Enum.IsDefined(typeof(Color),"White"));//True 检测大小写
Console.WriteLine(Enum.IsDefined(typeof(Color),"white"));//False
Console.WriteLine(Enum.IsDefined(typeof(Color),10));//False 没有对应的值
```

注意，使用IsDefined的时候总是执行大小写的查找，而且完全没有办法让他执行不区分大小写的查找。其次，因其内部使用了反射，导致其运行速度缓慢。且当枚举类型本身在同一程序集中时，才可以使用，因在另一程序中的扩充Enum类型会让其许可范围发生变化。

## 15.2 位标志

调用System.IO.File类型的GetAttributes方法，会返回FileAttibutes类型的一个实例。FileAttributes类似基本类型为Int32的枚举类型，其中每一位都反馈了文件的一个特性。

``` csharp
[Flags,Serializable]
public enum FileAttributes
{
    ReadOnly		= 0x0001,
    Hidder		    = 0x0002,
    System		    = 0x0004,
    Directory 	 	= 0x0010,
    Archive		    = 0x0020,
    Normal		    = 0x0040,
	...
}
```

判断文件是否隐藏可执行以下代码

``` csharp
String file = Assembly.GetEntryAssembly().Location;
FileAttributes attributes = File.GetAttributes(file);
Console.WriteLine("Is {0} hidden?{1}",file,(attributes&FileAttributes.Hidden)!=0);
```

值得注意的是，Enum定义了一个HasFlag方法

``` csharp
public Boolean HasFlag(Enum flag);
//但是使用它的时候会导致一次装箱，产生一次内存分配
```

如FileAttributes类型展示的那样，经常都要用枚举类型来表示一组可以组合的位标志。不过，虽然枚举类型和位标志相似，但是语义不同。前者表示单个的值，后者表示多个位的集合。

定义用于表示位标志的枚举类型时，应显示为每个符号分配一个数值。通常每个符号都有单独的一个位于on状态。此外，经常都要定义一个值为0的None符号，还可以定义一些符号来代表常见的位组合。另，强烈建议向枚举类型应用定制特性类型System.FlagsAttribute。

``` csharp
[Flags]
internal enum Actions
{
    None  = 0,
    Read  = 0x0001,
    Write = 0x0002,
    ReadWrite = Actions.Read|Actions.Write,
    ...
}
```

因Actions为枚举类型，因而在操作位标志枚举类型时，可以使用上一节描述的所有方法。

``` csharp
Actions actions  = Actions.Read | Actions.Delete;//0x0005
Console.WriteLine(actions.ToString());//Read,Delete
```

调用ToString时，他会试图将数值转换为对应的符号，但0x0005并无对应，因而ToString会检测到Flags特性，将生成字符串Read,Delete。应用Flags的特性之后，ToString的工作流程如下。

1. 获取枚举类型定义的数值集合，降序排列
2. 每个数值都和枚举实例中的值进行按位与计算，假如结果等于该值，那么将关联的字符串附加到输出字符串，对应的位被认为已被考虑，设置为0.
3. 检测完成之后，如果实例不为0，ToString返回枚举实例中的原始数值。
4. 如果枚举实例原始值不为0，返回符号之间已逗号分割。
5. 原始值为0，有定义0的符号，返回该符号。
6. return "0"

如果你需要将逗号分割的符号字符串转换为数值，可以尝试使用Parse和TryParse方法。这不再展开。

**注意**：不要对位标志枚举类型使用 **IsDefined** 方法。

+ 向 **IsDefined** 方法传递字符串，它不会将这个字符串拆分为单独的token进行查找，而是将逗号视为其中的一部分。由于不能在枚举类型之中定义包含逗号的符号，这个符号永远不会被找到。
+ 向 **IsDefined** 传递数值，会检测枚举类型是否定义了其数值和传入数值匹配的一个符号。

## 15.3 向枚举类型添加方法

使用拓展方法，假装添加。此处不再演示拓展方法的使用，详细请查看原书或参考微软文档。



