---
title: 《CLRViaC#》第23章-程序集加载和反射
date: 2022-10-17 17:51:14
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"

---

## 23.1 程序集加载

在内部，CLR使用System.Reflection.Assembly类的静态Load方法尝试加载这个程序集。该方法在.Net Framework SDK文档中是公开的，可调用他显式地将程序集加载到AppDomain中。

```csharp
CSHARP
public class Assembly
{
    public static Assembly Load(AssemblyName assemblyRef);
    public static Assembly Load(String assemblyString);
}
```

内部，Load导致CLR向程序集应用一个版本绑定重定向策略（这是什么呢？），并在GAC（全局程序集缓存）中查找程序集。如果没找到，就接着去程序基目录，私有路径子目录和codebase位置查找。

如果调用Load时传递的是弱命名程序集，Load就不会向程序集应用版本绑定重定向策略，CLR也不会去GAC查找程序集。如果Load找到程序集，会返回对代表已加载的那个程序集的一个Assembly对象的引用。

大多数动态可拓展应用程序中，Assembly的Load方法是将AppDomain之中的首选方式。但是使用它需要了解程序集各个标识的各个部分，开发人员经常需要写一些工作或实用工具来操作程序集。这需要获取程序集文件路径。

```csharp
CSHARP
public class Assembly
{
    public static Assembly LoadFrom(String path);
}
```

LoadFrom内部调用System.Reflection.AssemblyName类的静态GetAssemblyName方法。该方法打开指定的文件，找到AssemblyRef元数据表的记录项，提取程序集标识信息，然后以一个System.Reflection.AssemblyName对象的形式返回这些信息。然后LoadFrom内部调用Load方法，传入AssemblyName。

LoadFrom允许传入URL，可以从网络上下载某些文件。

VS和其他工具一般使用Assemby.LoadFile。这个方法可以将具有相同标识的程序集多次加载到一个AppDomain中。通过LoadFile加载程序时，CLR不会自动解析任何依赖性问题，代码必须向AppDomain的AssemblyResolve事件登记，并让事件回调方法显式加载依赖的程序集。

如果你的工具只是想通过反射来分析程序集的元数据，并希望确保程序集中的任何代码都不会执行，那么加载程序集的最佳方式是使用Assembly的ReflectionOnlyLoadFrom方法或是使用Assembly的ReflectionOnlyLoad方法。

```csharp
CSHARP
public class Assembly
{
    public static Assembly ReflectionOnlyLoadFrom(String assemblyFile);
    public static Assembly ReflectionOnlyLoad(String assemblyString);
}
```

ReflectionOnlyLoadFrom方法加载由路径指定的文件，文件的强名称标识不会获取，也不会在GAC和其他位置搜索文件。ReflectionOnlyLoad则会在上述位置进行查找。和Load方法不同的是，ReflectionOnlyLoad方法不会应用版本控制策略。

用ReflectionOnlyLoadFrom或ReflectionOnlyLoad方法加载程序集时，CLR禁止程序集中的任何代码执行，试图执行程序集中的代码将会抛出错误。这两个方法允许加载延迟签名的程序集，这种程序集在正常情况下会因为安全权限问题无法加载。

利用反射来分析由这两个方法之一加载的程序集时，代码经常需要向AppDomain的ReflectionOnlyAssemblyResovle事件注册回调，以便手动加载引用的程序集。

## 23.2 使用反射构建动态可拓展应用程序

反射为利用System.Reflection命名空间中包含的类型，使用代码来反射/解析元数据表。这个命名空间中的类型为元数据提供了一个对象模型。利用对象模型中的类型，可以轻松枚举类型定义元数据表中的所有类型，而针对每个类型都可获取它的基类型，接口，和flag。解析元数据表来查询类型的字段、方法、属性和时间。

## 23.3 反射的性能

- 反射造成编译时无法确保类型安全性，由于反射严重依赖字符串，所以会丧失编译时的类型安全性。
- 反射速度慢，类型及其成员的名称在编译阶段未知，在运行阶段发现他们，因而需要不断的执行字符串搜索。

因而，最好避免使用反射来访问字段或调用方法/属性。应该利用以下两种技术之一开发。

- 让类型从编译时已知的基类型派生，在运行时构造派生类型的实例，将对它的引用放到基类型的变量中，将其放入基类型的变量中，调用基类型定义的虚方法。
- 让类型实现编译已知的接口，在运行时构造类型的实例，将对它的引用放到接口类型的变量当中，在调用接口方法。

### 23.3.1 发现程序集中定义的类型

最常用的API是Assembly的ExportedTypes

```csharp
CSHARP

using System;
using System.Reflection;

public static class Program
{
    public static void Main()
    {
        String dataAssembly = "System.Data,version=4.0.0.0,"+"...";
    	LoadAssemAndShowPublicTypes(dataAssembly);
    }
    
    private static void LoadAssemAndShowPublicTypes(String assemId)
    {
        Assembly a = Assembly.Load(assemId);
        foreach(Type t in a.ExportedTypes)
            Console.WriteLine(t.FullName);
    }
}
```

### 23.3.2 类型对象的准确含义

上述代码遍历了System.Type对象构成的数组。System.Type类型是执行类型和对象操作的起点。System.Type对象代表一个类型引用。

众所周知，System.Object定义了公共非虚实例方法GetType。调用这个方法时，CLR会判断指定对象的类型，并返回对该类型的Type对象的引用。由于在一个AppDomain中，每个类型只有一个Type对象，所以可以使用==或者!=来判断。

- System.Type 类型提供了静态GetType方法的几个重载版本。所有版本都接受一个String参数。字符串必须指定类型的全名。注意不允许使用编译器支持的基元类型。
- System.Type 类型提供了静态ReflectionOnlyGetType方法。该方法与上一条提起的GetType方法在行为上相似，只是类型会以仅反射的方式进行加载，而无法执行。
- System.TypeInfo 类型提供了实例成员
- System.Reflection.Assembly

许多编程语言允许使用一个操作符来根据编译时已知的类型名称来获得Type对象。尽量用这个操作符来获得Type引用，而不是上述方法。C#的操作符为typeof.

需要注意的是，Type对象是轻量级的对象引用，需要更多地了解类型本身，必须获取一个TypeInfo对象，后者代表类型定义。

```csharp
CSHARP
Type typeReference = ...;
TypeInfo typeDefinition = typeRefinition.AsType();

//另外还可以
typeReference = typeDefinition.AsType();
```

获取TypeInfo对象会强迫CLR确保已加载类型的定义程序集，从而对类型进行解析。

### 23.3.3 构建Exception派生类型的层次结构

暂略

### 23.3.4 构建类型的实例

- System.Activator.CreateInstance

  这个方法并不会返回新对象的引用，而是会返回System.Runtime.Remoteing.ObjectHandle对象。ObjectHandle类型允许将一个AppDomain中创建的对象传值其他AppDomain,期间不强迫对象具体化。需要的时候可以调用其Unwarp方法。

- System.Activator的CreateInstanceFrom

  Activator类提供了一组静态CreateInstanceFrom方法。与CreateInstance行为相似，只是必须通过字符串参数来指定类型及其程序集。

- System.AppDomain

  这里定义了一系列方法，他们的共同点是都是实例方法，允许指定在那个AppDomain之中定义对象。

- System.Reflection.ConstructorInfo的Invoke实例方法

  使用一个Type对象引用，绑定到特定的构造器，获取对构造器的ConstructorInfo对象的引用。

利用前面机制，可为除去数组和委托之外的所有类型创建对象。创建数组需要调用Array的静态CreateInstance方法。所有版本的CreateInstance方法获取的第一个参数都是对数组元素Type的引用。其他参数允许指定数组的维度和上下限。创建委托则要调用MethodInfo的静态CreateDelegate方法。所有版本的CreateDelegate方法获取的第一个参数都是对委托Type的引用。CreateDelegate方法的其他参数允许指定在调用实例方法时应将哪个对象作为this参数传递。

构造泛型类型的实例首先要获取对开销类型的引用，然后调用Type的MakeGenericType方法，并将其传递给一个数组。

```csharp
CSHARP

using System;
using System.Reflection;

internal sealed class Dictionary<TKey,TValue>{}

public static class Program
{
    public static void Main()
    {
        Type openType = typeof(Dictionary<,>);
        
        Type closedType = openType.MakeGenericType(typeof(String),typeof(Int32));
        
        object o = Activator.CreateInstance(closedType);
        
        Console.WriteLine(o.GetType());
    }
}
```

## 23.4 设计支持加载项的应用程序

构建可拓展应用程序时，接口是中心。可用基类代替接口，但接口通常是首选的，因为他允许加载项开发人员选择他们自己的基类。例如，如果要写一个应用程序无缝加载和使用别人创建的类型。

- 创建”宿主SDK”程序集，它定义了一个接口，接口的方法作为宿主应用程序和加载项之间的通信机制使用。为接口方法定义参数和返回类型时，尝试使用MSCorLib.dll中定义的其他接口或者类型。传递并返回自己的数据类型，也在宿主SDK之中定义。
- 加载项开发人员会在加载项程序之中定义自己的类型。这些程序将引用你“宿主”程序集中的类型。加载项开发人员可按自己的步调推出程序集新版本。
- 创建单独的“宿主应用程序”程序集，在其中包含你的应用程序类型。程序集显然要引用“宿主SDK”程序集，并使用其中定义的类型。

## 23.5 使用反射发现类型的成员

一般利用这个功能创建开发工具和应用程序，查找特定编程模式或对特定成员的使用，从而对程序集进行分析。

### 23.5.1 发现类型的成员

FCL包含抽象基类System.Reflection.MemberInfo，封装了所有类型成员都通用的一组属性。MemberInfo有许多派生类，每个都封装了与特定类型成员相关的更多属性。

```csharp
CSHARP

using System;
using System.Reflection;

public static class Program
{
    public static void Main()
    {
        Assembly[] assemblies = AppDomain.CurrentDomain.GetAssemblies();
        foreach(Assembly a in assemblies)
        {
            Show(0,"Assembly:{0}",a);
            foreach(MemberInfo mi in t.GetTypeInfo().DeclaredMembers)
            {
                String typeName = String.Empty;
                if(mi is Type)typeName = "(Nested)Type";
                if(mi is FieldInfo)typeName = "FieldInfo";
                if(mi is MethodInfo)typeName = "MethodInfo";
                if(mi is ConstructorInfo)typeName = "ConstructoInfo";
                if(mi is PropertyInfo)typeName = "PropertyInfo";
                if(mi is EventInfo)typeName = "EventInfo";
                Show(2,"{0},{1}",typeName,mi);
            }
        }
    }
}
```

其中，MemeberInfo是成员层次结构的根。

| 成员名称         | 成员类型                         | 说明                                                         |
| ---------------- | -------------------------------- | ------------------------------------------------------------ |
| Name             | String                           | 名称                                                         |
| DelaringType     | Type                             | Type                                                         |
| Module           | Module                           | Module                                                       |
| CustomAttributes | IEnumerable<CustomArrtibuteData> | 返回一个集合，其中每个元素都标识了应用于该成员的一个定制特性的实例。定制特效可应用于任何成员，虽然Assembly不从MemberInfo派生，但是他提供了可用于程序集的相同属性。 |

### 23.5.2 调用类型的成员

| 成员类型        | invoke成员需要call的方法                                     |
| --------------- | ------------------------------------------------------------ |
| FieldInfo       | 调用GetValue获取字段的值 调用SetValue设置字段的值            |
| ConstructorInfo | 调用Invoke构造类型的实例并调用构造器                         |
| MethodInfo      | 调用Invoke来调用类型方法                                     |
| PropertyInfo    | GetValue调用get访问器 SetValue set访问器                     |
| EventInfo       | 调用AddEventHandler来调用事件的add访问器方法 调用RemoveEventHandler来调用事件的remove访问器 |

### 23.5.3 使用绑定句柄减少进程的内存消耗

Type和MemberInfo派生对象需要使用大量的内存，如果系统容纳了过多该类对象，而只是偶尔调用，会对系统性能产生巨大的负面影响。

如果需要保存/缓存大量Type和MemberInfo派生对象，开发人员可以使用运行时句柄代替对象减少内存的消耗。

FCL包含了三个运行时句柄类型，RuntimeTypeHandler、RuntimeFieldHandler、RuntimeMethodHandle，三个都是值类型，只包含一个字段。

以下方法可以将Type或MemberInfo转换为句柄

- Type静态方法GetTypeHandle
- Type静态方法GetTypeFromHandle
- FieldInfo实例的制度属性FieldHandle
- FieldInfo静态方法GetFieldFromHandle
- MethodInfo实例MethodHandle
- MethodInfo静态方法GetMethodFromHandle