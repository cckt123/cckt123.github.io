---
title: 《CLRViaC#》第9章 参数
date: 2022-02-23 14:21:40
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 9.1 可选参数和命名参数

设计方法参数的时候，可以为部分或者全部参数分配默认值。然后调用这些方法的参数可以选择不提供这些参数，使用其默认值。

### 9.1.1 规则和原则

如果为其设定了默认值，要注意

+ 可为方法、构造器方法和有参属性的参数指定默认值。还可为属于委托定义一部分的参数指定默认值。以后调用该委托类型的变量时可省略实参来接收默认值。

+ 有默认值的参数必须放在没有默认值的参数之后。

+ 默认值必须是编译时可以确定的常量。这里指所有基元类型，枚举类型，可以设置为null的引用类型。值类型的参数可以将默认值设置为值类型的实例。可以使用default或者new来达到这种意思。

+ 不要重命名参数变量，否则任何调用者以传参数名的方式传递实参，他们的代码也必须修改。

+ 如果方法从模块外部调用，更改模块的默认值有潜在威胁。如果以后更改了参数的默认值，但没有重新编译包含call site的代码，就会在调用旧的方法的时候重新传递旧的默认值。可以考虑使用0或者null来警戒这种情况。

  ``` csharp
  //不要这么做
  private static String MakePath(String filename = "Untitled")
  {
      return String.Format(@"C:\{0}.txt",filename);
  }
  
  //而是要这么做
  private static String MakePath(String filename = null)
  {
      return String.Format(@"C:\{0}.txt",filename??"Untitled");
  }
  ```

+ 如果参数用ref或out关键字进行了标识，就不能设置默认值。因为没有办法为这些参数传递有意义的默认值。

使用可选或者命名参数调用方法时，要注意

+ 实参可按任意顺序传递，但是命名实参只能出现在可选参数末尾。

+ 可按名称将参数传递给没有默认值的参数，但是所有必须的实参必须传递。

+ C#不允许省略逗号之间的实参。对于有默认值的实参，可按他们的参数名传递实参。

+ 如果参数要求使用ref/out

  ``` csharp
  private static void M(ref Int32 x){...}
  
  Int32 a = 5;
  M(x: ref a);
  ```

### 9.1.2 DefaultParameterValueAttribute和OptionalAttribute

在其他语言中调用可选参数的步骤，C#案例，内部自动调用。话说这个东西有什么用呢...详情请查看原书。

## 9.2 隐式类型的局部变量

``` csharp
private static void ImplicitlyTypedLocalVariables()
{
    var collection = new Dictionary<String,String>(){{"Grant",4.0f}};
    foreach(var item in collection)
    {
        ShowVariableType(item);
    }
}

private static void ShowVariableType<T>(T t)
{
    Console.WriteLine(typeof(T));
}
```

无法将null赋予隐式类型的局部变量，这是由于null能隐式转型为任何引用类型或可空值类型。

上方代码展示了var真正的功能，如果没有他，就不得不在foreach中显式声明item的类型，如果后来修改了集合类型或者任何泛型参数类型，这里的代码也需要发生变化。

不能用var声明方法的参数类型，不能用其声明类型中的字段。后者是考虑到一个字段可被多个方法访问，另一个原因是可能会导致匿名类型被泄露到方法之外。

## 9.3 以传引用的方式向方法传递参数

CLR允许传引用而非传值的方式传递参数。C#用关键字Ref和out来支持这个功能。

值得注意的是

``` csharp
public sealed class Point
{
    static void Add(Point p){...}
    static void Add(ref Point p){...}
}
```

这种方式编译是完全合法的，但是如果再定义一个

``` csharp
static void Add(out Point p){...}
```

是不可行的，因为out和ref的元数据签名完全相同，无法做出区分。

## 9.4 向方法传递可变数量的参数

``` csharp
static Int32 Add(params Int32[] values)
{
    Int32 sum = 0;
    if(values != null)
    {
        for(Int32 x = 0;x<values.Length;x++)
        {
            sum += values[x];
        }
    }
    return sum;
}
```

params只能应用于方法签名之中的最后一个参数。另外，这个参数只能标识一维数组。可为这个参数传递null值，或传递对包含零个元素的一个数组的引用。

注意 调用参数数量可变的方法对性能有一定影响。毕竟最终会在堆上分配内存。可以考虑重载几个简单类型的方法以减少Params参数的使用频率。

``` csharp
//举例说明
public sealed class String : Object, ...
{
    public static string Concat(object arg0);
    public static string Concat(object arg0,arg1);
    public static string Concat(object arg0,arg1,arg2);
    public static string Concat(object arg0,arg1,arg2,arg3);
    public static string Concat(params object[] arg0);
}
```

## 9.5 参数和返回类型的设计规范

声明方法的参数类型时，尽量指定最弱的类型，宁愿选择接口也不选择基类。例如要写方法来处理一组数据项，最好使用接口（IEnumerable\<T>）声明参数，而不要用强数据类型(List\<T>)或者更强的接口类型(ICollection\<T>或者IList\<T>)

一般将方法的返回类型设置为最强的类型(防止受限于特定类型)。如果想保持一定灵活性，在将来更改方法返回的对象，请选择一个较弱的返回类型。

## 9.6 常量值

有的语言，例如C++允许将方法或参数声明为常量，从而禁止实例方法中的代码更改对象的任何字段，或者更改传给方法的任何对象。CLR显然没有这个功能。

