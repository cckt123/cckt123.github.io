---
title: 《CLRViaC#》第12章-泛型
date: 2022-03-20 16:42:38
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

CLR允许创建泛型的引用类型和泛型值类型，但不允许创建泛型枚举类型。此外，CLR还允许创建泛型接口和泛型委托。

根据Microsoft的设计原则，泛型参数变量要么称为T，要么至少以T开头，例如TKey或者TValue。

+ 源代码保护

  使用泛型算法的开发人员不需要访问算法的源代码。然而，使用C++模板的泛型技术时，算法的源代码必须提供给准备使用算法的用户。

+ 类型安全

  将泛型算法应用于一个具体的类型时，编译器和CLR能理解开发人员的意图，并保证只有与指定数据类型兼容的对象才能用于算法。试图使用不兼容类型的对象会造成编译时错误，活在运行时抛出异常。

+ 更清晰的代码

  由于编译器强制类型安全性，所以减少了源代码中必须进行的强制类型转换次数，是代码更加容易编写和维护。

+ 更佳的性能

  没有泛型的时候，要想定义常规化的算法，它的所有成员都要定义成操作Object数据类型。使用其对值类型进行操作的过程中不可避免的会产生大量的拆装箱。而使用泛型可以直接以值类型的方式进行操作，从而减少拆装箱次数。

## 12.1 FCL中的泛型

新的泛型接口并不是要代替原有的非泛型接口。许多时候两者都要使用。例如，如果List\<T>类只实现了IList\<T>接口，代码就不能将List\<DateTime>对象当做IList来处理。

还要注意的是，System.Array类，也就是所有数组的基类提供了大量泛型静态方法，比如AsReadOnly，BinarySearch，ConvertAll，Exists等。

## 12.2 泛型基础结构

为了泛型可以正常工作，Microsoft必须完成以下工作。

+ 创建新的IL指令，使之能够识别类型实参。
+ 修改现有元数据的格式，以便表示具有泛型参数的类型名称和方法。
+ 修改编译器，使之能生成新的IL指令和修改的元数据格式。
+ 修改JIT编译器，以便处理新的支持类型实参的IL指令来生成正确的本机代码。
+ 创建新的反射成员，使开发人员能查询类型和成员，以判断它们是否具有泛型参数。另外，还必须定义新的反射成员，使开发人员能在运行时创建泛型类型和方法定义。
+ 修改调试器以显示和操纵泛型类型、成员、字段和局部变量。
+ 修改VS智能感应功能，将泛型类型或方法应用于特定数据类型时能显示成员的原型。

### 12.2.1 开放类型和封闭类型

泛型类型参数的类型称为开放类型，CLR禁止构造开放类型的任何实例。这类似于CLR禁止构造接口类型的实例。

代码引用泛型类型时可指定一组泛型类型实参。为所有类型参数都传递了实际的数据类型，类型就成为了封闭类型。CLR允许构造封闭类型的实例。然而，代码引用泛型类型的时候，可能留下一些泛型类型实参未指定。这会在CLR中创建新的开放类型对象，而且不能创建该类型的实例。

``` csharp
using System;
using System.Collection.Generic;

//一个部分指定的开放类型
internal sealed class DictionaryStringKey<TValue> : Dictionary<String,TValue>
{
}

public static class Program
{
 	public static void Main()
    {
        Object o = null;
        
        //Dictionary<,>是开放类型，有2个参数类型
        Type t = typeof(Dictionary<,>);
        
        //尝试创建该类型的实例（失败）
        o = CreateInstance(t);
        Console.WriteLine();
        
        //DictionaryStringKey<>是开放类型，有1个类型参数
        t = typef(DictionaryStringKey<>);
        
        //尝试创建该类型的实例
        o = CreateInstance(t);
        Console.WriteLine();
        
        //DictionaryStringKey<Guid>是封闭对象
        t = typeof(DicitonaryStringKey<Guid>);
        
        //尝试创建该类型的一个实例（成功）
        o = CreateInstance(t);
        
        //证明它确实能够工作
        Console.WriteLine(" 对象类型 = "+ o.GetType());
    }   
    
    private void object CreateInstance(Type t)
    {
        Object o = null;
        try
        {
            o = Activator.CreateInstance(t);
            Console.Write("已创建{0}的实例"，t.ToString());
        }
        catch(ArgumenException e)
        {
            Console.WriteLine(e.Message);
        }
        return o;
    }
}
```

Activator的CreateInstance方法会在试图构造开放类型的实例时抛出ArgumentException异常。注意，异常的字符串消息指明类型中仍然含有一些泛型参数。

还要注意，CLR会在类型对象内部分配类型的静态字段。因此，每个封闭类型都有自己的静态字段。换而言之，List\<T>之中定义了任何静态字段，这些字段不会再List\<DateTime>和List\<String>之间共享。另外，假如泛型类型定义了静态构造器，那么针对每个封闭类型，这个构造器都会执行一次。泛型类型定义静态构造器的目的是为了保证传递的类型实参满足特定条件。

例如

``` csharp
internal sealed class GenericTypeThatRequiresAnEnum<T>
{
    static GenericTypeThatRequiresAnEnum()
    {
        if(!typeof(T).IsEnum)
        {
            throw new ArgumentException("T must be an enumerated type");
        }
    }
}
```

CLR提供了名为约束的功能，可以更好地指定有效的类型实参。然而约束无法将类型实参限制为“仅枚举类型”。

### 12.2.2 泛型类型和继承

泛型类型也是类型，因而可以从其他任何类型之中派生。使用泛型类型并指定类型实参时，实际是在CLR中定义一个新的类型对象，新的类型对象从泛型类型派生自的那个类型派生。指定类型实参并不影响继承层次结构。

``` csharp
internal sealed class Node<T>
{
    public T m_data;
    public Node<T> m_next;
    
    public Node(T data):this(data,null)
    {
    }
    
    public Node(T data,Node<T> next)
    {
        m_data = data;
        m_next = next;
    }
    
    public override String ToString()
    {
        return m_data.ToString()+((m_next != null) ? m_next.ToString() : String.Empty);
    }
}
```

``` csharp
private static void SameDataLinkedList()
{
    Node<Char> head = new Node<Char>('C');
    head = new Node<Char>('B',head);
    head = new Node<Char>('A',head);
    Console.WriteLine(head.ToString());
}
```

在这个Node类之中，对于m_next字段引用的另一个节点来说，它的m_data字段必须包含相同的数据类型。这意味着在链表包含的节点中，所有数据项都必须具有相同的类型。例如，不能使用Node类来创建这样一个链表：其中一个元素包含Char值，另一个包含DateTime值，另一个包含String值。当然，如果到处都用Node\<Object>，那么确实可以做到，但是会丧失编译时的安全性，而且值类型会被装箱。

所以，更好的办法是定义非泛型Node基类，在定义泛型TypedNode类。这样就可以创建一个链表，其中每个节点都可以是一种具体的数据类型，同时获得编译时的类型安全性，并防止值类型装箱。

``` csharp
internal class Node
{
    protected Node m_next;
    
    public Node(Node next)
    {
        m_next = next;
    }
}

internal sealed class TypedNode<T>:Node
{
 	public T m_data;
    
    public TypedNode(T data):this(data,null)
    {
    }
    
    public TypeNode(T data,Node next):base(next)
    {
        m_data = data;
    }
    
    public override String ToString()
    {
        return m_data.ToString()+((m_next!=null)?m_next.ToString():String.Empty);
    }
}
```

``` csharp
private static void SameDataLinkedList()
{
    Node<Char> head = new Node<Char>('.');
    head = new TypedNode<DateTime>(DateTime.Now,head);
    head = new TypedNode<String>("Today is ",head);
    Console.WriteLine(head.ToString());
}
```

### 12.2.3 泛型类型同一性

源代码中散布大量\<>，有损可读性。为了增强可读性，给出了这样的方法。

``` csharp
List<DateTime> dtl = new List<DateTime>();

internal sealed class DateTimeList:List<DateTime>
{
    //这里无需任何代码
}

DateTimeList dtl = new DateTimeList();
```

但如果使用这种形势来表述会失去类型的同一性和相等性。

``` csharp
Boolean sameType = (typeof(List<DateTime>) == typeof(DateTime));
```

上述判断会是false，因为比较的是两个不同类型的对象。这也意味着如果方法的原型接受一个DateTimeList，那么不可以将一个List\<DateTime>作为参数传入。然而如果接受一个List\<DateTime>，那么将DateTimeList传入是合法的，因为他是前者的派生类。

``` csharp
//为了规避这种混乱的逻辑关系
using DateTimeList = System.Collections.Generic.List<System.DateTime>;
```

这种语法实质上只是在编译阶段将文本替换，因而不会影响同一性和相等性。

### 12.2.4 代码爆炸

在执行泛型的操作的同时，可能会导致CLR为每种不同的方法/类型组合生成本机代码。这种现象就称之为 **代码爆炸**。他可能会导致应用程序的工作集显著扩大。CLR内建了一些优化措施能缓解代码爆炸。假设为特定类型的实参调用了一个方法，以后再用相同的类型实参调用这个方法，CLR只会为这个方法/类型组合编译一次代码。另外CLR会认为所有引用类型的代码实质上是相同的，实际上也是指向堆的指针。

## 12.3 泛型接口

泛型接口的支持对CLR来说也很重要。没有泛型接口，每次用非泛型接口来操纵值类型都会发生装箱，而且会失去编译时的类型安全性。这将会严重制约泛型类型的应用范围。因而，CLR提供了泛型接口的支持，引用类型或值类型可指定类型实参实现泛型接口。也可保持类型实参的非指定状态来实现泛型接口。

``` csharp
public interface IEnumerator<T> : IDisposable,IEnumerator
{
    T Current{get;}
}

internal sealed class Triangle : IEnumerator<Point>
{
    private Point[] m_vertices;
    
    //IEnumerator<Point>的Current属性是Point类型
    public Point Current
    {
        get{...}
    }
    
    ...
}

internal sealed class ArrayEnumerator<T>:IEnumerator<T>
{
    private T[] m_array;
    
    public T Current{get{...}}
    
    ...
}
```

## 12.4 泛型委托

CLR支持泛型委托，目的是为了保证任何类型的对象都能以类型安全的方式传给回调方法。此外，泛型委托允许值类型实例在传给回调方法时不进行任何装箱。如果定义的委托类型指定了类型参数，编译器会定义委托类的方法，用指定的类型参数替换方法的参数类型和返回值类型。

``` csharp
public delegate TReturn CallMe<TReturn,TKey,TValue>(TKey key,TValue value);
//编译器会将其转换为以下类。
public sealed class CallMe<TReturn,TKey,TValue>:MulticastDelegate
{
    public CallMe(Object object,IntPtr method);
    public virtual TReturn Invoke(TKey key,TValue value);
    public virtual TAsyncResult BeginInvoke(TKey key,TValue value,AsyncCallback callback,Object object);
    public virtual TReturn EndInvoke(IAsyncResult result);
}
```

## 12.5 委托和接口的逆变和协变泛型类型实参

委托的每个泛型类型参数都可标记为协变量或逆变量。利用这个功能，可将泛型委托类型的变量转换为相同的委托类型。泛型类型参数可以是以下任何一种类型。

+ 不变量 意味着泛型类型参数不能更改。
+ 逆变量 一位置泛型类型参数可以从一个类更改为它的某个派生类。在C#是用in关键字标记逆变量形势的泛型类型参数。逆变量泛型类型参数只出现在输入的位置，比如说作为方法的参数。
+ 协变量 一位置泛型类型参数可以从一个类更改为它的某个基类。C#是用out关键字标记协变量形势的泛型类型参数。协变量泛型类型参数只能出现在输出位置，比如作为方法的返回类型。 

``` csharp
public delegate TResult Func<in T,out TResult>(T arg);
Func<Object,ArgumentException> fn1 = null;
Func<String,Exception> fn2 = fn1;
Exception e = fn2("");
```

注意，只有编译器能验证类型之间存在引用转换，这些可变性才有用。换而言之，由于需要装箱，所以值类型不具备可变性。

``` csharp
void ProcessCollection(IEnumerable<Object> collection){...}
```

上述方法无法向其传递一个List\<DateTime>对象引用，因为DateTime值类型和Object之间不存在引用转换。为了解决这个问题，可以如下声明ProcessCollection;

``` csharp
void ProcessCollection<T>(IEnumerable<T> collection){...}
```

另外，对于泛型类型参数，如果要将该类型的实参传给使用out或ref关键字的方法，便不允许其可变性。

而要使用获取泛型参数和返回值的委托时，建议尽量为逆变量和协变量指定in和out关键字。这样做不会有不良反应，而且可以让委托在更多情况之中得到应用。

## 12.6 泛型方法

CLR还允许方法指定他自己的类型参数，这些类型参数也可以作为参数、返回值或局部变量的类型使用。

``` csharp
internal sealed class GenericType<T>
{
    private T m_value;
    
    public GenericType(T value){m_value = value;}
    
    public TOutput Converter<TOutput>()
    {
        TOutput result = (TOutput)Convert.ChangeType(m_value,typeof(TOutput));
        return result;
    }
}
```

### 泛型方法和类型推断

编译器可以对泛型方法的类型进行自己推断。

``` csharp
private static void CallingSwapUsingInference()
{
    Int32 n1 = 1,n2 = 2;
    Swap(ref n1,ref n2);
    String s1  = "Aidan";
    Object s2 = "Grant";
    Swap(ref s1,ref s2);//error
}
```

在推断类型时，C#使用变量的数据类型，而不是变量引用的对象的实际类型。所以在第二个Swap调用中，C#发现s1是String，而s2是Object。由于s1和s2是不同数据类型的变量，编译器拿不准要为Swap传递什么类型实参。

``` csharp
private static void Display(String s)
{
    Console.WriteLine(s);
}

private static void Display<T>(T o)
{
    Display(o.ToString());
}
```

编辑器会优先调用较明确的匹配。

## 12.7 泛型和其他成员

C#之中，属性、索引器、事件、操作符方法、构造器和终结器本身不能有类型参数。但他们能在泛型中定义，而且可以使用类型的类型参数。

## 12.8 可验证性和约束

在编译泛型代码的同时，C#编译器会进行分析，确保代码适用于当前已有或将来可能定义的任何类型。

``` csharp
private static Boolean MethodTakingAnyType<T>(T o)
{
    T temp = o;
    Console.WriteLine(o.ToString());
    Boolean b = temp.Equals(o);
    return b;
}
```

因为所有类型都继承于Object，所有上述代码显然可以允许。

``` csharp
private static T Min<T>(T o1,T o2)
{
    if(o1.CompareTo(o2)<0)return o1;
    return o1;
}
```

这个例子里Min试图调用CompareTo方法。然而很多类型都没有提供CompareTo方法，所以C#编译器不能编译上述代码。他不能保证这个方法适用于所有类型。为此，编译器和CLR提供了 **约束** 机制。

约束的作用是为了限制能指定成为泛型的类型数量。通过这种方式，可以对某些特定约束下的类型进行更多操作。

``` csharp
public static T Min<T>(T o1,T o2)where T:IComparable<T>
{
    if(o1.CompareTo(o2)<0)return o1;
    return o2;
}
```

C#的where关键字告诉编译器，为T指定的任何类型都必须实现同类型的泛型IComparable接口。有了这个约束，就可以在方法中调用ComareTo，因为已知IComparable\<T>接口定义了CompareTo。而向该类泛型方法传入参数的同时，要确保其类型实参符合指定的约束。

约束可应有与泛型类型的类型参数，也可应用于泛型方法的类型参数。CLR不允许基于类型参数名称或约束来进行重载，只能基于元数（类型参数个数）对类型或方法进行重载。

``` csharp
internal sealed class AType{}
internal sealed class AType<T>{}
internal sealed class AType<T1,T2>{}
//错误，与没有约束的AType<T>产生冲突
internal sealed class AType<T> where T : IComparable<T>{}
//错误，与AType<T1,T2>冲突
internal sealed class AType<T3,T4>{}

internal sealed class AnotherType
{
    //可定义以下方法，参数个数不同
    private static void M(){}
    private static void M<T>(){}
    private static void M<T1,T2>(){}
    
    //错误
    private static void M<T>() where T : IComparable<T>{}
    
    //错误
    private static void M<T3,T4>(){}
}
```

重写虚泛型方法时，重写的方法必须指定相同数量的类型参数，而且这些类型参数会继承在基类方法上指定的约束。	

``` csharp
internal class Base
{
    public virtual void M<T1,T2>()where T1:struct where T2:class
    {       
    }
}

internal sealed class Derived : Base
{
    public override void M<T3,T4>()
        where T3:EventArgs
        where T4:class//错误
}
```

但是无法重写约束。

### 12.8.1 主要约束

类型参数可以指定0或1个主要约束，主要约束可以是代表非密封类的一个引用类型。不能指定为以下特殊引用类型：System.Object System.Array System.Delegate System.MulticasDelegate System.ValueType System.Enum System.Void。

指定引用类型约束时，指定的类型实参与约束类型相同，或者是从约束类型之中派生的子类。

``` csharp
internal sealed class PrimaryConstraintOfStream<T> where T :Stream
{
    public void M(T stream)
    {
        stream.Close();
    }
}
```

在这个类定义中，类型参数T设置了主要约束Stream。这意味着在指定类型时要求使用Stream或从Stream之中派生的类型。如果类型参数没有指定主要约束，就默认为System.Object。但不能显示声明为Object。

有两个特殊的主要约束:class和struct。其中，class约束向编译器承诺类型实参是引用类型。任何类类型、接口类型、委托类型或者数组类型都满足这个约束。

``` csharp
internal sealed class PrimaryConstraintOfClass<T> where T : class
{
    public void M()
    {
        T temp = null;
    }
}
```

struct 约束向编译器承诺类型实参是值类型。包括枚举在内的任何值类型都满足这个约束。但编译器和CLR将任何System.Nullable\<T>值类型视为特殊类型，不满足这个struct约束。//可空值类型见后

``` csharp
internal sealed class PrimaryConstraintOfStruct<T> where T:struct
{
    public static T Factory()
    {
        return new T();
    }
}
```

### 12.8.2 次要约束

类型参数可以指定0或多个 **次要约束** 。次要约束代表接口类型，这种约束向编译器承诺类型实参实现了接口。由于能指定多个接口约束，所以类型实参必须实现了所有接口约束。

还有一种次要约束称为 **类型参数约束** ，有时也称之为 **裸类型约束**。他允许一个泛型类型或方法规定：指定的类型实参要么是约束的类型。

``` csharp
private static List<TBase> ConvertIList<T,TBase>(IList<T> list) where T : TBase
{
    List<TBase> baseList = new List<TBase>(list.Count);
    for(Int32 index = 0;index< list.Count;index++)
        baseList.Add(list[index]);
    return baseList;
}

private static void CallingConvertILits()
{
    //构造并初始化一个List<String>
    IList<String> ls = new List<String>();
    Is.Add("A String");
    
    //1. 将IList<String>转换成一个IList<Object>
    IList<Object> lo = ConvertIList<String,Object>(ls);

    //2. 将IList<String>转换成一个IList<IComparable>
    IList<Object> lc = ConvertIList<String,IComparable>(ls);
    
 	//3. 将IList<String>转换成一个IList<IComparable<String>>
    IList<IComparable<String>> lcs = ConvertList<String,IComparable<String>>(ls);
    
    //4. 将IList<String>转换成一个IList<String>
    IList<String> ls2 = ConvertIList<String,String>(ls);
    
    //5. 将IList<String>转换成一个IList<Exception>
    IList<Exception> le = ConvertIList<String,Exception>(ls);//ERROR
}
```

### 12.8.3 构造器约束

类型参数可以指定0个或者1个构造器约束，它想编译器承诺类型实参是实现了 **公共无参构造器** 的非抽象类。需要注意的是，如果同时使用了构造器约束和struct约束，C#编译器会认为这是一个错误，因为他是多余的。所有值类型都隐式提供了公共无参构造器。

``` csharp
internal sealed class ConstructorConstraint<T> where T : new()
{
    public static T Factory()
    {
    	return new T();
    }
}
```

### 12.8.4 其他可验证性问题

1. 泛型类型变量的转型

   将泛型类型的变量转换为其他类型是非法的，除非转型为与约束兼容的类型

   ``` csharp
   private static void CastingAGenericTypeVariablel<T>(T obj)
   {
       Int32 x = (Int32)obj;//错误
       String s = (String)obj;//错误
   }
   
   private static void CastingAGenericTypeVariable2<T>(T obj)
   {
       Int32 x = (Int32)(Object) obj;//可行
       String s = (String)(Object) obj;
   }
   ```

2. 将泛型类型变量设为默认值

   将泛型类型变量设为null是非法的，除非将泛型类型约束成为引用类型。

   ``` csharp
   private static void SettingAGenericTypVariableToNull<T>()
   {
       T temp = null;//非法
   }
   ```

   微软的开发团队认为有必要允许开发人员将变量设为他的默认值，并专门为此提供default关键字。

   ``` csharp
   private static void SettingAGenericTypeVariableToDefaultValue<T>()
   {
       T temp = default(T);
   }
   ```

   这个关键字告诉编译器和CLR的JIT编译器，如果T是引用类型，就将temp设为null。

3. 将泛型类型变量与Null进行比较

   无论泛型类型是否被约束，使用==或者!=与null进行比较都是合法的。

   ``` csharp
   private static void ComparingAGenericTypeVariableWithNull<T>(T obj)
   {
       if(obj==null)
       {
           //值类型永远不会执行
       }
   }
   ```

   如果被约束为struct，则无法进行判断。

4. 两个泛型类型变量相互比较

   ``` csharp
   private static void ComparingTwoGenericTypeVariables<T>(T o1,T o2)
   {
       if(o1==o2){}
       //error
   }
   ```

   引用类型可以通过编译，而值类型在重写operator方法之前无法正确进行编译，基元类型有默认的==方法。

5. 泛型类型变量作为操作数使用

   不是很行。

