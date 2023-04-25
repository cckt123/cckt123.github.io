---
title: 《CLRViaC#》第8章 方法
date: 2022-02-21 19:30:55
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 8.1 实例构造器和类（引用类型）

构造器是将类型的实例初始化为良好状态的特殊方法。创建引用类型的实例时，首先为实例的数据字段分配内存，然后初始化对象的附加字段（类型对象和同步块索引），最后调用类型实例构造器来设置对象的初始状态。

和其他方法不同，实例构造器永远不能被继承，也就是说，类只有类自己定义的实例构造器。如果类没有显式声明，那么C#编译器将会定义一个默认的无参数构造器。

如果类的修饰符为abstract，那么编译器生成的默认构造器的可访问性就是protected;否则，构造器会被赋予public可访问性。如果基类没有提供无参构造器，那么派生类必须显示显式调用一个基类构造器，否则编译器将会报错。如果类的修饰符为static，编译器根本不会在类的定义中生成构造器。

一个类可以定义多个实例构造器，每个构造器都有不同的签名，而且每个都可以有不同的访问等级。而为了使代码可验证，类的实例构造器在访问从基类继承的任何字段之前，必须先调用基类的构造器。如果派生类的构造器没有显式调用一个基类构造器，C#编译器会自动生成对默认的基类构造器的调用。

极少数情况下可以在不调用实例构造器的前提下创建类型的实例。一个典型的例子就是Object的MemberwiseClone方法。该方法的作用是分配内存，初始化对象的附加字段（类型对象指针和同步块索引），然后将源对象的字节数据复制到新对象之中。另外，用运行时序列化器反序列化对象时，通常也不需要调用构造器。

如果有多个已初始化的实例字段和许多重载的构造器方法，可考虑不是在定义字段时的初始化，而是创建单个构造器来执行这些公共的初始化。

``` csharp
using System;
internal sealed class SomeType
{
    private Int32	m_x;
    private String	m_s;
    private Double	m_d;
    private Byte	m_b;
    
    public SomeType(){
        m_x = 5;
        m_s = "Hi there";
        m_d = 3.1415926;
        m_b = 0xff;
    }
    
    public SomeType(Int32 x):this()
    {
        m_x = x;
    }
    
    public SomeType(String x):this()
    {
        m_s = x;
    }
    
    public SomeType(Int32 x,String s):this()
    {
        m_x = x;
        m_s = s;
    }
}
```



## 8.2 实例构造器和结构（值类型）

值类型的构造器工作方式与引用类型完全不同，他实际上并不需要定义构造器，C#也不会为其定义默认的无参构造器。默认会将值类型设置为0或者null。但CLR确实允许为值类型定义构造器，但必须显示调用才会执行。

``` csharp
internal struct Point
{
    public Int32 m_x,m_y;
    
    public Point(Int32 x,Int32 y)
    {
        m_x = x;
        m_y = y;
    }
}

internal sealed class Rectangle
{
    public Point m_topLeft,m_bottomRight;
    
    public Rectangle()
    {
        //这里是显示调用案例
        m_topLeft = new Point(1,2);
        m_bottomRight = new Point(100,200);
    }
}
```

需要注意的是，如果没有显式调用他，C#编译器根本不会调用默认无参的构造函数。事实上，C#根本不允许使用显示的无参构造器来初始化值类型。如果使用会提示错误 **error CS0568:结构不能包含显示的无参数构造器**

由于C#不允许使用无参数构造器，因而不能在值类型中内联实例字段的初始化。同时，为了生成可验证的代码，定义值类型构造器的时候必须定义所有字段。而如果需要解决这个问题，对单一类型定义构造器...

``` csharp
public SomeValType(Int32 x)
{
    this =  new SomeValType();
    m_x = x;
}
```

如上，这里实际上是对其他字段默认了null或者0,对m_x单一赋值覆盖了原本的0。在引用类型之中，this是只读的，因而无法对其进行赋值。

## 8.3 类型构造器

除了实例构造器，CLR还支持类型构造器，用于设置类型的初始状态。类型默认没有定义的类型构造器，如果定义也只能定义一个。类型构造器没有参数。

``` csharp
internal sealed class SomeRefType
{
    static SomeRefType()
    {
        //首次访问的时候执行这里的代码
    }
}

internal struct SomeValType
{
    static SomeValType()
    {
        //...
    }
}
```

类型构造器总是私有，也只能是私有，默认声明也是私有（private）。

虽然可以在值类型之中定义类型构造器，但是永远不要这么做，因为CLR有时不会调用值类型的静态类型构造器。

``` csharp
internal struct SomeValType
{
    static SomeValType()
    {
        Console.WriteLine("这句话不会被显示");
    }
    public Int32 m_x;
}

public sealed class Program
{
    public static void Main()
    {
        SomeValType[] a = new SomeValType[10];
        a[0].m_x = 123;
        Console.WriteLine(a[0].m_x);
    }
}
```

注意 由于CLR保证一个类型构造器在每个AppDomain中只执行一次，而且这种执行是线程安全的，所以非常适合在类型构造器之中初始化需要的任何单例类型。

类型构造器只可以访问类型之中的静态字段，而且他的作用就应该是用来访问初始化这些静态字段。

注意 虽然C#不允许为她的实例字段直接初始化，但是静态字段是被允许直接初始化的。

## 8.4 操作符重载方法

编译源代码时，编译器会生成一个标识操作符行为的方法。CLR规范要求操作符重载方法必须是public和static方法。另外，C#要求操作符方法至少有一个参数的类型与当前定义这个方法的类型相同。

``` csharp
public sealed class Complex{
    public static Complex operator+(Complex c1,Complex c2){...}
}
```

C#允许重载的一元操作符及其相容的与CLS的方法名

| C#操作符 | 特殊方法名        | 推荐的相容于CLS的方法名 |
| -------- | ----------------- | ----------------------- |
| +        | op_UnaryPlus      | Plus                    |
| -        | op_UnaryNegation  | Negate                  |
| !        | op_LogicalNot     | Not                     |
| ~        | op_OnesComplement | OnesComplement          |
| ++       | op_Increment      | Increment               |
| --       | op_Decrement      | Decrement               |
| 无       | op_True           | IsTrue{get;}            |
| 无       | op_False          | IsFalse{get;}           |

C#允许重载的二元操作符及其相容的与CLS的方法名

| C#操作符 | 特殊方法名            | 推荐的相容于CLS的方法名 |
| -------- | --------------------- | ----------------------- |
| +        | op_Addition           | Add                     |
| -        | op_Subtraction        | Subtract                |
| *        | op_Multiply           | Multiply                |
| /        | op_Division           | Divide                  |
| %        | op_Modulus            | Mod                     |
| &        | op_BitwiseAnd         | BitwiseAnd              |
| \|       | op_BitwiseOr          | BitwiseOr               |
| ^        | op_ExclusiveOr        | Xor                     |
| <<       | op_LeftShift          | LeftShift               |
| >>       | op_RightShift         | RightShift              |
| ==       | op_Equality           | Equals                  |
| !=       | op_Ineqality          | Equals                  |
| <        | op_LessThan           | Compare                 |
| >        | op_GreaterThan        | Compare                 |
| <=       | op_LessThanOrEqual    | Compare                 |
| >=       | op_GreaterThanOrEqual | Compare                 |

更多内容请查看原书和微软官方文档。

### 操作符与编程语言的互操作性

使用不支持操作符重载的语言时，语言应该会允许你进行直接调用希望的op_*方法。但这个类型依然可能有一个op_Addition之类的方法。不过毫无关系，使用其他语言用操作符是无法调用她的。

## 8.5 转换操作符方法

如果将对象从一种类型转向另一种类型，且两者都是基元类型的时候，编辑器自己就知道怎么做。如果两者之间有一者并不是基元类型，那么编译器会生成代码，要求CLR执行强制转换。这种转换只会检测两者之间是否相同或者是否是派生关系。但有时需要将一种类型转换到完全不相关的另一种类型。

假设FCL包含了一个Rational类型，那么如果能将Int32或者Single转换成Rational会显得更加方便。

为了进行这种转换，Rational类型应该定义只有一个参数的公共构造器，该参数要求是源类型的实例。还应该定义无参的公共实例方法，例如ToString()。

``` csharp
public sealed class Rational
{
    public Rational(Int32 num){...}
    
    public Rational(Single num){...}
    
    public Int32 ToInt32(){...}
    
    public Single ToSingle(){...}
}
```

事实上，有些编程语言，例如C#还提供了转换操作符重载，将对象从一种类型转换成另一种类型。CLR同样要求转换操作符的方法必须是public和static方法。此外C#要求参数类型和返回类型两者必有其一与定义转换方法的类型相同。

``` csharp
public sealed class Rational
{
    ... 
    //隐式转换
    public static implicit operator Rational(Int32 num)
    {
        return new Rational(num);
    }
    
    public static implicit operator Rational(Single num)
    {
        return new Rational(num);
    }
    
    //显式转换
    public static explicit operator Int32(Rational r)
    {
        return r.ToInt32();
    }
    
    public static explicit operator Single(Rational r)
    {
        return r.ToSingle();
    }
}
```

C#之中implictit关键字告诉编译器为了生成代码来调用方法，不需要再源代码中进行显式转换。相反，explicit关键字告诉编译器只有显式转型的时候才会调用该方法。

``` csharp
public sealed class Program
{
    public static void Main()
    {
        Rational r1 = 5;
        Rational r2 = 2.5f;
        Int32 x = (Int32)r1;
        Single s= (Single)r2;
    }
}
```

## 8.6 拓展方法

C#语法允许定义一个静态方法，并用实例方法的语法来调用，只需在第一个参数前添加this参数。

``` csharp
public static class StringBuilderExtensions
{
    //注意static this
    public static Int32 IndexOf(this StringBuilder sb,Char value)
    {
    	for(Int32 index = 0; index<sb.Length ; index++)
        {
            if(sb[index]==value)return index;
        }    
        return -1;
    }
}
```

### 8.6.1 规则和原则

+ C#只支持拓展方法，并不支持拓展属性，拓展事件，拓展操作等。
+ 拓展方法必须在非泛型的静态类中声明。拓展方法有且只能由第一个参数作为this声明。
+ C#编译器在静态类之中查找拓展方法时，要求静态类本身必须具有文件作用域。
+ 由于静态类可以取任何名字，所以C#编译器要花费一定时间来寻找拓展方法，为了增强性能，必须导入拓展方法。
+ 当多个静态类定义了重名的方法时，必须显示指定静态类的名称，消除二义性之后才可以继续。
+ 注意，使用这个方法的时候并不是所有人都熟悉他，谨慎使用。
+ 版本控制，当未来版本发生更新的时候，如有重名的方法，会优先使用类原本的方法而不是拓展方法。

### 8.6.2 用拓展方法拓展各种类型

由于拓展方法实质上是对一个静态方法的调用，所以CLR不会生成代码对调用方法的表达式的值进行null值检测。所以可能会对null进行方法调用。

还要注意 可以为接口定义拓展方法。

``` csharp
public static void ShowItem<T>(this IEnumerable<T> collection)
{
    foreach(var item in collection)
    {
        Console.WriteLine(item);
    }
}
```

任何只要继承了上述接口的表达式都可以使用该拓展方法。

除此之外，还可以拓展委托类型和利用委托来调用其他的静态方法。

``` csharp
public static void InvokeAndCatch<TException>(this Action<Object> d,Object o) where TException:Exception
{
    try
    {
        d(o);
    }
    catch(TException)
    {
    }
}

Action<Object> action = o => Console.WriteLine(o.GetType());
action.InvokeAndCatch<NullReferenceException>(null);
```

### 8.6.3 ExtensionAttribute类

为了提供给其他可支持拓展方法的语言。编译器会在保证运行速度的前提下做额外的工作。在C#之中，这个特性会自动调用，其实不必太多关注。一旦this关键字标记了某个静态方法的第一个参数，编译器就会在该方法内部应有一个特性，表示这个程序集最后会包含拓展方法。

## 8.7 分步方法

分步方法的存在是为了解决虚方法的缺陷问题。原作者在这里给出了一个例子

``` csharp
internal class Base
{
    private String m_name;
    
    protected virtual void OnNameChanging(String value){}
    
    public String Name
    {
        get {return m_name;}
        set 
        {
            OnNameChanging(value.ToUpper());
        	m_name = value;
        }
    }
} 

internal class Derived : Base
{
    protected override void OnNameChanging(string value)
    {
        if(String.IsNullOrEmpty(value))
            throw new ArgumentNullException("value");
    }
}
```

+ 类型必须是非密封的类，这种技术不能应用于密封类，也不能用于值类型，更不可能应用在静态方法。
+ 而且这样会导致少量的性能问题。 如果没有继承，那么就会调用一个空的虚函数。

``` csharp
internal sealed partial class Base
{
    private String m_name;
    
    partial void OnNameChanging(string value);
    
    public String Name
    {
        get {return m_name;}
        set {
            OnNameChanging(value.ToUpper());
            m_name = value;
        }
    }
}

internal sealed partial class Base
{
    partial void OnNameChanging(String value)
    {
         if(String.IsNullOrEmpty(value))
            throw new ArgumentNullException("value");
    }
}
```

+ 类现在密封，而且可以是值类型或者静态类型。

同时，如果没有实现partial部分，那么编译器将不会生成这部分IL方法的元数据。因而减少IL代码量，提升了系统的效率。

### 规则和原则

+ 他们只能在分部类或者结构之中声明。
+ 分部方法的返回类型始终是void，任何参数不能使用out修饰符来标记。之所以有这个限制是因为，在允许的时候这两参数有可能并不会存在，所以不能将变量初始化为方法也许会返回的对象。
+ 当然，分部方法的声明和实现必须具有完全一致的签名。如果两者都应用了定制特性，那么特性也会合并。
+ 如果没有实现部分，那么便不能在代码之中通过任何方式引用。
+ 分部方法总是被认为是private方法，而C#编译器禁止在分部方法之前添加private。
