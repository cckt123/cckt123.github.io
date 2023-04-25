---
title: 《CLRViaC#》第19章-可空值类型
date: 2022-07-13 15:58:23
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

值类型的变量永远不为空的特性可能会导致某些问题，例如数据库可空类型映射到FCL。

为了解决这个问题，CLR引入了可空值的概念。

``` csharp
[Serializable,StructLayout(LayoutKind.Sequential)]
public struct Nullable<T> where T : struct
{
    private Boolean hasValue = false;//假定null
    internal T value = default(T);//假定所有位为0

    public Nullable(T value)
	{
    	this.value = value;
        this.hasValue = true;
	}
    
    public Boolean HasValue{get{return hasValue;}}
    
    public T Value
    {
        get
        {
            if(!hasValue)
            {
                throw new InvalidOperationException("Nullable object must have a value.");
            }
            return value;
        }
    }
    
 	public T GetValueOrDefault(){return value;}
    
    public T GetValueOrDefault(T defaultValue)
    {
     	if(!HasValue)return defaultValue;
        return value;
    }
    
    public override Boolean Equals(Object other)
    {
        if(!HasValue)return other==null;
        if(other == null)return false;
        return value.Equals(other);
    }
    
    public override int GetHashCode()
    {
        if(!HasValue)return 0;
        return value.GetHashCode();
    }
    
    public override string ToString()
    {
        if(!HasValue)return "";
        return value.ToString();
    }
    
    public static implicit operator Nullable<T>(T value)
    {
        return new Nullable<T>(value);
    }
    
    public static explicit operator T(Nullable<T> value)
    {
        return value.Value;
    }
}
```

## 19.1 C#对可空值类型的支持

``` csharp
Int32? x = 5;
Int32? y = null;
```

C# 允许向可空实例应用操作符，进行转换转型。

``` csharp
private static void ConversionsAndCasting()
{
    Int32? a = 5;
    Int32? b = null;
    Int32 c = (Int32)a;
    Double? d = 5;
    Double? e = b;
}    

private static void Operators()
{
    Int32? a = 5;
    Int32? b = null;
    a++;
    b = -b;
    a = a+3;
    ...//略
}
```

值得注意的是，操作可空实例会生成大量IL代码，且速度慢于非可空类型。

## 19.2 C#的可空接合操作符

C#提供了“可空接操作符”，??。假如左边的操作数为null，返回右边的值，否则返回左边。可应用于可空值类型与引用类型。

``` csharp
//语法糖 好吃
Func<String> f = ()=> SomeMethod() ?? "Untitled";
String s = SomeMethod() ?? SomeMethod2() ?? SomeMethod3(); 
```

## 19.3 CLR对可空值类型的特殊支持

### 19.3.1 可空值类型的装箱

具体来说，当CLR对可空值类型进行装箱的同时，会检测他是否为null，如果是，CLR不进行装箱，直接返回null。如果不是null，会装箱对应值类型。

### 19.3.2 可空值类型的拆箱

可将已装箱的类型T，拆箱为T或Nullable\<T>

### 19.3.3 通过可空值类型调用GetType

GetType会表示这个是T

``` csharp
Int32? x = 5;
Console.WriteLine(x.GetType());
```

### 19.3.4 通过可空值类型调用接口方法

可以直接调用对应值类型的接口，而不必拆箱。

```csharp
Int32? x= 5;
Int32 result = ((IComparable)n).CompareTo(5);
```

