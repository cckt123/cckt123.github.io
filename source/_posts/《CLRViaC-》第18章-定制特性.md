---
title: 《CLRViaC#》第18章-定制特性
date: 2022-07-01 09:15:23
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 18.1 使用定制特性

定制特性将附加信息与某个目标元素关联起来。编译器在托管模块的元数据中生成额外的信息，大多数特性对编译器来说没有意义，编译器只是检测到源代码特性，然后生成对应元数据。

FCL拥有数百个定制特性，例如

+ DIIImport特性应用于方法，告诉CLR该方法的实现位于指定DLL的非托管代码中。
+ Serializable应用于类型，告诉序列化格式器一个实例字段可序列化与反序列化。
+ AssemblyVersion特性应用于程序集，设置程序集的版号。
+ Flags应用于枚举类型。

CLR允许将特性应用于文件元数据中几乎所有的对象。但最常应用特性的还是以下定义表中的记录项：TypeDef、MethodDef、ParamDef、FieldDef、PropertyDef、EventDef、AssemblyDef和ModuleDef。更具体一点，C#允许特性应用于：程序集、模型、类型、字段、方法、方法参数、方法返回值、属性、事件、泛型类型参数。

应用特性时，C#允许使用一个前戳，指定特性要应用的目标元素。许多时候，即使省略前戳，也可以正常使用，但也有我们需要明确的向编译器指定目标的情况。

``` csharp
using System;
[assembly:SomeAttr]//应用于程序集
[module:SomeAttr]//应用于模块

[type:SomeAttr]//应用于类型
internal sealed class SomeType<[typevar:SomeAttr] T>//应用于泛型类型
{
    [field:SomeAttr]//应用于字段
    public Int32 SomeFiled = 0;
    
    [return:SomeAttr]//应用于返回值
    [method:SomeAttr]//应用于方法
    public Int32 SomeMethod(
    	[param:SomeAttr]//应用于参数
        Int32 SomeParam
    )
    {
        return SomeParam;
    }
    
    [event:SomeAttr]//应用于事件
    [field:SomeAttr]//应用于编译器生成的字段
    [method:SomeAttr]//应用于编译器生成的add&remove方法
    public event EventHandler SomeEvent;
}
```

定制特性其实是一个类型的实例，为了符合CLS的要求，定制特性必须要直接或间接从公共抽象类System.Attribute派生。C#只允许符合CLS规范的特性。

**注意** ：将特性应用于源代码中目标元素时，C#编译器允许省略Attribute后缀来减少打字量。例如可使用[DIIImport(...)]，而不是[DIIImportAttribute(...)]

特性是类的实例，类必须由公共构造器才能创建它的实例。所以，将特性应用于目标元素时，语法类似于调用类的某个实例构造器。除此之外，语言可能支持一些特殊的语法，允许设置与特性类关联的公共字段或属性。

``` csharp
[DllImport("Kernel32",CharSet = CharSet.Auto,SetLastError = true)]
```

如上，其构造器接收一个String参数，构造器的参数成为 **定位参数** ，且是强制的。

后面两位参数用于设置对象的公共字段或属性。 称之为 **命名参数**。这种参数是可选的。

可将多个特性应用于某一个目标元素，彼此之间顺序关系可随意设置，间隔可用逗号，或直接添加。

``` csharp
[Serializable][Flags]
[Serializable,Flags]
[SerializableAttribute,FlagsAttribute]
[SerializableAttribute()][FlagsAttribute()]
```

## 18.2 定义自己的特性类

 以FlagsAttribute为例。特性类需要从Attribute继承，以符合CLS规范。此外，类名有Attribute后缀，是为了保持标准，但实际上也可以不写。最后，所有非抽象特性都必须至少包含一个公共构造器。

然后，要将其限定为只允许应用于枚举类型。因而需要向其添加System.AttributeUsageAttribute类的实例。

``` csharp
namespace System
{
    [AttributeUsage(AttributeTargets.Enum,Inherited = false)]
    public class FlagsAttribute : System.Attribute
    {
    	public FlagsAttribute(){}    
    }
}
```

System.AttributeUsageAttribute允许像是传递位标志一样来指定特性的合法应用范围。

少数特性有必要应用于同一个目标。FCL特性类ConditionalAttribute允许向它的多个实例应用于同一个目标元素。但不讲AllowMultiple设置为true就无法应用多次。

AttributeUsageAttribute的另一个属性是Inherited，他表示特性在应用于基类的同时是否应用于派生类和重写的方法。

注意，.NetFormework只认为类、方法、属性、事件、字段、方法返回值和参数等目标元素是可继承的。所以定义特性类型的时候，只有在该特性应用于上述某个目标的前提下，才应该将Inherited设置为ture。可继承特性并不会在派生类型中生成额外的元数据。

**注意** ：如果不设置AttributeUsage特性，CLR和编译器会默认特性可以被所有目标元素使用，可被应用一次，且可继承。

## 18.3 特性构造器和字段/属性数据类型

定义特性类的实例构造器、字段和属性时，可供选择的数据类型并不多，确切的说只允许Boolean、Char、Byte、SByte、Int16、UInt16、Int32、UInt32、Int64、UInt64、Single、Double、String、Type、Object和枚举类型。此外还可以使用上述类型一维0基数组，但应尽量避免，隐如果定制特性获取数组作为参数，会失去与CLS的相容性。

应用特性是必须传递一个编译时常量表达式，它与特性类定义的类型匹配。在特性类定义了一个Type的时候，都必须使用C# typeof操作符。定义Object的时候，可以传递任意类型，包括null的常量表达式。如果常量表达式代表值类型，那么在运行时构建特性的实例会对值类型进行装箱。

``` csharp
using System;

internal enum Color{Red}

[AttributeUsage(AttributeTargets.All)]
internal sealed class SomeAttribute : Attribute
{
    public SomeAttribute(String name,Object o,Type[] types)
    {
        
    }
}
```

最后，编译器将特性的状态序列化到目标的元数据表记录项中。

## 18.4 检测定制特性

代码利用反射来检测自己是否存在特性。在定制特性的同时，必须实现一些代码来检测某些目标上是否存在该特性类的实例，然后执行一些逻辑分支代码。FCL提供了多种方式来检测特性的存在。如果通过System.Type对象来检测特性，可以使用IsDefined方法。但有时需要检测出类型外其他目标。

| 方法名称           | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| IsDefined          | 如果至少有一个指定的Attribute派生类的实例与目标关联，就返回true。这个方法效率很高，因为它不构造特性类的实例。 |
| GetCustomAttrbutes | 返回应用于目标的指定特性对象的集合。每个实例都使用编译时指定的参数、字段和属性来构造。如果目标没有应用知道特性类的实例，就返回一个空集合。该方法通常用于已将AllowMultiple设为ture的特性，或用于列出已应用的所有特性。 |
| GetCustomAttrbute  | 返回..实例。没有返回null。多个丢出异常。                     |

如果只是用来判断目标是否应用了一个特性，那么应该调用IsDefined，因为它比另两种方法更高效。但我们知道，将特性应用于目标时，可以为特性的构造器指定参数，并可选择设置字段和属性。使用IsDefined不会构造特性对象，不会调用构造器，也不会设置属性和字段。

要构造特性对象，必须使用GetCustomAttebute或者GetCustomAttebutes方法。

调用上述任何方法，内部都必须扫描托管模块的元数据，执行字符串比较来定位指定的定制特性类。如果对性能有要求，可以缓存这些方法的调用结果，而不是反复请求。

System.Reflection命名空间定义了几个类允许检查模块的元数据，这些类包括Assembly,Module,ParameterInfo,MemberInfo,Type,MethodInfo,ConstructorInfo，FieldInfo，EventInfo，PropertyInfo及其各自的*Builder类。所有类都提供了IsDefined和GetCustomAttributes方法。

反射类提供的GetCustomAttributes方法返回的是由Object实例构成的数组，而不是由Attribute实例构成的数组。这是由于反射类能返回不相容于CLS规范的特性类的对象。

只有Attribute，Type和MethodInfo类才实现了支持Booleaninherit参数的反射方法。其他能检查特性的所有反射方法都会忽略inherit参数，而且不会检查继承结构层级。

还要注意，将一个类传给IsDefined，GetCustomAttribute或者GetCustomAttributes方法时，这些方法会检测是否应用了指定的特性类或者它的派生类。如果只是想搜索一个具体的特性类，应针对返回值执行一次额外检测，确保方法返回的正是想要搜索的类，还可以考虑将特性类设计为sealed，减少可能的混淆，避免执行额外的检测。

``` csharp
//演示代码
using System;
using System.Diagnostics;
using System.Reflection;
using System.Linq;
[assembly:CLSCompliant(true)]//声明整个程序集需要符合CLS标准

[Serializable]//序列化
[DefalutMemberAttribute("Main")]//定义某类型的成员，该成员是 InvokeMember(String, BindingFlags, Binder, Object, Object[], ParameterModifier[], CultureInfo, String[]) 使用的默认成员。
[DebuggerDisplayAttribute("Richter",Name = "Jeff",Target = typeof(Program))]//确定类或字段在调试器变量窗口中的显示方式。
public sealed class Program
{
	[Conditional("Debug")]
    [Conditional("Release")]
    public void DoSomething(){}
    
    public Program(){}
    
    [CLSCompliant(true)]
    [STAThread]//指示应用程序的 COM 线程模型是单线程单元 (STA)。
    public static void Main()
    {
        //显示应用于这个类型的特性集
        ShowAttributes(typeof(Program));
        
        //获取与类型关联的方法集
        var members = from m in typeof(Program).GetTypeInfo().DeclaredMembers.OfType<MethodBase>() where m.IsPublic select m;
        
        foreach(MemberInfo member in members)
            ShowAttributes(member);
    }
    
    private static void ShowAttributes(MemberInfo attributeTarget)
    {
        var attributes = attributeTarget.GetCustomAttributes<Attribute>();
        
        Console.WriteLine("Attributes applied to {0}:{1}",attributeTarget.Name,(attributes.Cout()==0?"None":String.Empty));
        foreach(Attribute attribute in attributes)
        {
            Console.WriteLine("{0}",attribute.GetType().ToString());
            if(attribute is DefaultMemberAttribute)
                Console.WriteLine("MemberName = {0}",((DefaultMemberAttribute)attribute).MemberName);
            if(attribute is ConditionalAttribute)
                Console.WriteLine("MemberName = {0}",((ConditionalAttribute)attribute).ConditionalString);
            if(attribute is CLSCompliantAttribute)
                Console.WriteLine("MemberName = {0}",((DefaultMemberAttribute)attribute).IsCompliant);
            
            DebuggerDisplayAttribute dda = attribute as DebuggerDisplayAttribute;
            if(dda != null)
            {
                Console.WriteLine("Value = {0},Name = {1},Target = {2}",dda.Value,dda.Name,dda.Targer);
            }
        }
        Console.WriteLine();
    }
}
```

## 18.5 两个特性实例的相互匹配

System.Attribute重写了Object的Equals方法，会在内部比较两个对象的类型，不一致会返回false。如果一致，Equals会利用反射来比较两个特性对象中字段值，所有字段匹配返回true，否则false，可在自己的定制特性类中重写Equals来移除反射的使用。System.Attribute还公开了虚方法Match，可重写它来提供更丰富的语义。

``` csharp
using System;
[Flags]
internal enum Accounts
{
    Savings = 0x0001,
    Checking = 0x0002,
    Brokerage = 0x0004
}

[AttributeUsage(AttributeTargets.Class)]
internal sealed class AccountsAttribute : Atrribute
{
    private Accounts m_accounts;
    
    public AccountsAttribute(Accounts accounts)
    {
        m_accounts = accounts;
    }
    
    public override Boolean Match(Object obj)
    {
        //if(!base.Match(obj))return false;
        
        //如果基类已实现，且信任基类实现的Match方法，可忽略这两步
        if(obj==null)return false;
        if(this.GetType()!=obj.GetType())return false;
        
        AccountsAttribute other = (AccountsAttribute)obj;
        
        //比较字段是否相等
        //这里要只是个例子
        if((other.m_accounts & m_accounts)!=m_accounts)
            return false;
        
        return true;
    }
    
    public override Boolean Equals(Object obj)
    {
        if(obj==null)return false;
        if(this.GetType()!=obj.GetType())return false;
        AccountsAttribute other = (AccountsAttribute)obj;
        
        if(other.m_accounts != m_accounts)
            return false;
        
        return true;
    }
    
    public override Int32 GetHashConde()
    {
        return (Int32)m_accounts;
    }
}

[Accounts(Accounts.Savings)]
internal sealed class ChildAccount{}

[Accounts(Accounts.Savings|Accounts.Checking|Accouts.Brokerage)]
internal sealed class AdultAccount{}

public sealed class Program
{
    public static void Main()
    {
        CanWriteCheck(new childAccount());
        CanWriteCheck(new AdultAccount());
        CanWriteCheck(new Program());
    }
    
    private static void CanWriteCheck(Object obj)
    {
        Attribute checking = new AccountsAttribute(Accounts.Checking);
        Attribute validAccounts = Attribute.GetCustomAttribute(obj.GetType(),typeof(AccountsAttribute),false);
        
        if((validAccounts!=null)&&checking.Match(validAccounts))
        {
            Console.WriteLine("{0}types can write checks.",obj,GetType());
        }
        else 
        {
            Console.WriteLine("{0}types can not write checks.",obj,GetType());
        }
    }
}
```

## 18.6 检测定制特性时不创建从Attribute派生的对象

调用Attribute的GetCustomAttribute或GetCustomAttributes方法时，这些方法会在内部调用特性类的构造器。在构造器、set访问器方法以及类型构造器中，可能包含每次查找特性都要执行的代码。在安全性要求较高的场合，存在安全隐患。

可用System.Refilection.CustomAttributeData类在查找特性的同时禁止执行特性类中的代码。通常，先用Assembly的静态方法ReflectionOnlyLoad加载程序集，再用CustomAttributeData类分析这个程序集的元数据中的特性。简单来说，ReflectionOnlyLoad以特殊方式加载程序集，期间会禁止CLR执行程序集中的任何代码，包括类型构造器。

CustomAttributeData的GetCustomAttributes方法是一个工厂方法，也就是说，调用它会返回一个IList\<CustomAttributeData>类型的对象，其中包含了由CustomAttributeData对象构成的集合。集合中的每个元素都是应用于指定目标的一个定制特性。可查询每个CustomAttributeData对象的只读属性，判断特性对象如何构筑和初始化。具体来说，Constructor属性指出构造器方法如何调用。ConstructorArguments属性以一个IList\<CustomAttributeTypedArgement>实例的形式返回将传入这个构造器的实参。而NamedArguments属性已一个IList\<CustomAttributeNamedArgument>实例的形式，返回将设置的字段与属性。

## 18.7 条件特性类

应用了System.Diagnostics.ConditionalAttribute的特性类成为条件特性类。

``` csharp
#define VERIFY

using System;
using System.Diagnostics;

[Conditional("TEST")][Conditional("VERIFY")]
public sealed class CondAttribute : Attribute
{
    
}

[Cond]
public sealed class Program
{
    public static void Main()
    {
        Console.WriteLine("CondAttribute is {0} applied to Program type",
                         Attribute.IsDefined(typeof(Program)),
                          typeof(CondAttribute)?"":"Not "）;
        
    }
}
```

编译器如果发现向目标元素应用了CondAttribute的实例，那么当还有目标元素的代码编译时，只有定义TEST或VERIFY符号的前提下，编译器才会在元数据中生成特性信息。

## 额外参考

+ [控制程序集符合CLS规范](https://blog.csdn.net/weixin_30869099/article/details/94791465)
+ [DefaultMemberAttribute 类](https://docs.microsoft.com/zh-cn/dotnet/api/system.reflection.defaultmemberattribute?redirectedfrom=MSDN&view=net-6.0)
+ [DebuggerDisplayAttribute 类](https://docs.microsoft.com/zh-cn/dotnet/api/system.diagnostics.debuggerdisplayattribute?view=net-6.0)
+ [STAThreadAttribute 类](https://docs.microsoft.com/zh-cn/dotnet/api/system.stathreadattribute?view=net-6.0)
