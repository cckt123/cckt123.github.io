---
title: 《CLRViaC#》第4章-类型基础
date: 2022-02-17 13:14:46
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 额外参考

+ [CLRviaC#个人笔记2-类与分配](https://codingcodingk.top/2021/11/06/Tech/CSharp/CLR-Via-CSharp/cp2/)

  写笔记的时候借用了人家的图，因为我的是实体书，所以这篇笔记上面的图都是来源于这里。她对书记内容的分析也有很大价值。

## 4.1 所有类型都从System.Object派生

CLR要求所有类从System.Object派生，也就是说，无论是否声明，所有类都继承于System.Object。相应的，所有类都可以调用继承自System.Object下的方法。

| 公共方法/public | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| Equals          | 判断两个对象值是否相等。有关更多同一性和相等性的内容在第五章。 |
| GetHashCode     | 返回对象的哈希码，如果需要在哈希表中作为键，那么需要覆写保证良好的分布。 |
| ToString        | 默认返回类型的完成名称(this.GetType().FullName)。经常会覆写来反映当前类的状态。 |
| GetType         | 非虚方法，反射获取当前类类型。                               |

| protected       | 说明                         |
| --------------- | ---------------------------- |
| MemberwiseClone | 浅拷贝实例返回。             |
| Finalize        | 被GC之前会调用一次的虚方法。 |

### 关于New

1. 计算类型及其所有基类型(一直到System.Object)中定义所有的实例字段需要的字节数。堆上每个对象都需要一些overhead成员，包括“类型对象指针”和“同步索引块.
2. 从托管堆中分配类型要求的字节数，从而分配对象内存，分配的所有字节设为0。
3. 初始化对象的"类型对象指针"和"同步块索引"。
4. 调用类型的实例构造器，传递在new调用中指定的实参。逐步向上调用，直到最终调用到System.Object。

## 4.2 类型转换

CLR重要的特性之一是类型安全，任何时候都可以调用GetType来获取具体的类型，又因为这个是非虚方法，所以不会有一种类型伪装成为另一种类型。CLR允许将对象转换成为她的基类型，然而需要转换到派生类型的时候需要显示转换。

### 使用C# is as

is 检测是否兼容与指定类型 对象应用为null is总是返回false

``` csharp
if(o is Employee) Employee e = (Employee)o;//检测两次
Employee = o as Employee;//检测一次
if(e!=null)//检测是否为null相对于检测是否为对象类型快
{
    ....
}
```

as 尝试转换为指定类型，失败返回null

is as 都不会丢出异常。

## 4.3 命名空间和程序集

using and namespace

``` csharp
using WintellectWidgt = Wintellect.Widget;//指定别名
```

## 4.4 运行时的相互关系

线程创建会分配到1MB的栈。栈空间用于向方法传递实参，方法内部定义的局部变量也在栈上。栈从高位内存地址向低位内存地址构建。

``` csharp
void M1()
{
    String name = "Joe";
    M2(name);
    ...
    return;
}

void M2(String name)
{
    Int32 length = s.Length;
    Int32 tally;
    ...
    return;
}
```

![](https://cdn.jsdelivr.net/gh/CodingCodingK/CodingCodingK.github.io.ImageHost/img/20211110185718.png)

1. 图1到图2的过程就是执行"序幕"代码，在工作开始之前对其进行初始化。
2. 图2到图3的过程是M1调用M2的代码，将name作为实参传递，name局部变量地址被压入栈。调用方法将返回地址也压入栈。
3. 图3到图4 M2方法开始执行，它的"序幕"代码在线程中为局部变量length和tally分配内存
4. M2代码执行，return，cpu指令指针被设置成栈中的返回地址，M2栈帧unwind，返回图2
5. M1继续执行之后的代码
6. ...

### 类内调用方法的流程

``` csharp
internal class Employee
{
    public			Int32		GetYearsEmployed(){...}
    public virtual	String		 GetProgressReport(){...}
    public static	Employee	 Lookup(string name){...}
}

internal sealed class Manager:Employee
{
    public override String GetProgressReport(){...}
}
```



![](https://cdn.jsdelivr.net/gh/CodingCodingK/CodingCodingK.github.io.ImageHost/img/20211110182516.png)

演示步骤到这将会执行M3方法，JIT编辑器将M3的IL代码转换成本机CPU指令时，会注意到M3内部引用的所有类型，CLR将会确认这些类型的程序集都已经被加载，而后创建一些数据结构来表示这些类。

堆上所有对象都包含两个额外的成员:类型对象指针(type object pointer)和同步块索引(sync block index)。定义类型时，在类型内部定义静态数据字段，为这些字段提供支援的字节在类型对象自身中分配。

![](https://cdn.jsdelivr.net/gh/CodingCodingK/CodingCodingK.github.io.ImageHost/img/20211111162931.png)

当CLR确认方法需要的所有类型都已经创建，M3的代码已经编译之后，就允许线程执行M3的本机代码。

![](https://cdn.jsdelivr.net/gh/CodingCodingK/CodingCodingK.github.io.ImageHost/img/20211111164343.png)

而后，M3执行代码构造一个Manager对象，这造成在托管堆创建Manager类型的一个实例。

![](https://cdn.jsdelivr.net/gh/CodingCodingK/CodingCodingK.github.io.ImageHost/img/20211111165618.png)

如图4-9，Manager对象含有类型对象指针，同步块索引，实例字段，以及其基类所定义的所有字段。

而后调用静态方法，e = Employee.Lookup("Joe");

调用静态方法时，CLR会定位和定义静态方法的类型对应的类型对象。然后JIT编译器在类型对象的方法表中查找与被调用方法对应的记录项，并进行JIT编译流程。

![](https://cdn.jsdelivr.net/gh/CodingCodingK/CodingCodingK.github.io.ImageHost/img/20211111170356.png)

Lookup函数有重新制作Mangaer返回的功能，因而可以看到上图中e的指向发生了变化。e指向了一个新的Manager对象。这里第一个Manager实际上并没有被任何变量所引用，他将会是GC的目标。

下一句调用类内的非虚实例方法，e.GetProgressReport();

![](https://cdn.jsdelivr.net/gh/CodingCodingK/CodingCodingK.github.io.ImageHost/img/20211111171309.png)

JIT编译器会找到与"发出调用的那个变量e的类型Employee"对应的类型对象。这时的变量e被定义成一个Employee。如果Employee类型没有定义，那么JIT会回溯类的层次结构，并在沿途检索有没有对其定义，直到抵达System.Object。

最后是虚实例方法e.GetProgressReport();

![](https://cdn.jsdelivr.net/gh/CodingCodingK/CodingCodingK.github.io.ImageHost/img/20211111181049.png)

JIT编译器要在方法中额外生成一些代码，每次方法执行的时候调用。首先检查发出调用的变量，并跟随地址来到发出调用的对象。然后代码检测对象内部的"类型对象指针"成员，该成员指向对象的实际类型。

方法结束，返回。

> System.Type类型对象本身也是对象，内部也有类型对象指针，指向他本身。System.Object的GetType方法开业返回存储在指定对象的类型对象指针中的成员，也就是说他可以用来判断任何对象的真实类型，包括类型对象本身。
