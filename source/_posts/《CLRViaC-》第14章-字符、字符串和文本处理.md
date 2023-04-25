---
title: 《CLRViaC#》第14章-字符、字符串和文本处理
date: 2022-03-29 11:18:01
tags: [ C#, CLR]
categories: [ C#]
about:
description: "阅读笔记与总结，这本书的深度远高于我的理解范畴，因而笔记不免有所遗漏，如果有缺漏之处请查看原书。"
---

## 14.1 字符

.Net Framwork中，字节重视表示成16位Unicode代码值。每个字符都是System.Char结构的实例。System.Char类很简单，提供了两个公共只读常量字段:MinValue和MaxValue。

为Char的实例调用静态GetUnicodeCategory方法返回System.Globalization.UnicodeCategory枚举类型的一个值，表面该字符是由Unicode标准定义的控制字符、货币符号、小写字母、大写字母、标点符号、数字符号或者其他。

为简化开发，Char类型还提供了IsDigit等方法，大多数都在内部调用了GetUnicodeCategory方法，简单的返回true/false。注意，所有这些方法要么获取单个字符作为参数，要么获取一个String以及目标字符在这个String中的索引作为参数。

可调用静态方法ToLowerInvariant或者ToUpperInvariant，以忽略语言文化的方式将字符转换为大写形势或者小写。另一个方案是调用ToLower和ToUpper方法来转换大小写，但转换时会使用与调用线程关联的语言文化信息。

除了这些静态方法，Char类型还有自己的实例方法。其中，Equals方法在两个Char实例代表同一个16位Unicode码位的前提下返回true。CompareTo方法返回两个Char实例的忽略语言文化的比较结果。此处请查看微软文档或查看原书。

最后，可以使用三种技术实现各种数值类型与Char实例的相互转换。

+ 转型（强制类型转换）

  将Char转换成数值最简单的办法就是转型。这是在这三种技术中效率最高的，因为编译器会生成中间语言来执行转换，而不必调用方法。

+ 使用Convert类型

  System.Convert类型提供了几个静态方法来实现Char和数值类型的相互转换。所有这些转换都以checked方法执行，发现转换将造成数据丢失就抛出OverflowException异常。

+ 使用IConvertible接口

  Char类型和FCL中的所有数值类型都实现了IConvertibe接口。该接口定义了像ToUInt16和ToChar这样的方法。这种技术执行效率最差，因为在值类型上调用接口方法要求对实例进行装箱。

  注意，很多类型都将IConvertible的方法实现为显式接口成员。

``` csharp
using System;
public static class Program
{
    public static void Main()
    {
        Char c;
        Int32 n;
        
        c = (Char)65;
        Console.WriteLine(c);//A
        
        n = (Int32)c;
        Console.WriteLine(n);//65
        
        c = unchecked((Char)(65536+65));
        Console.WriteLine(c);//A
        
        // 使用Convert实现数字与字符的相互转换
        c = Convert.ToChar(65);
        Console.WriteLine(c);//A
        
        n = Convert.ToInt32(c);
        Console.WriteLine(n);//65
        
        try
        {
            c = Convert.ToChar(70000);//对16位来说过大
            Console.WriteLine(c);
        }
        catch(OverflowException)
        {
            Console.WriteLine("Can't convert 70000 to a Char.");
        }
        
        //使用IConvert实现数字与字符的相互转换
        c = ((IConvertible)65).ToChar(null);
        Console.WriteLine(c);//显示A
        
        n = ((IConvertible)c).ToInt32(null);
        Console.WriteLine(n);//65
    }
}
```

## 14.2 System.String 类型

String直接派生与Object，是引用类型，所以永远不会在线程栈上运行。

### 14.2.1 构造字符串

C#将String类型视为基元类型，也就是说，编译器允许源代码中直接使用字面值literal字符串。

C#不允许使用new操作符从字面值字符串中构造String对象。

``` csharp
using System;

public static class Program
{
    public static void Main()
    {
        String s = new String("Hi there.");//error
        Console.WriteLine(s);
    }
}
```

这里是正确的展示

``` csharp
using System;

public static class Program
{
    public static void Main()
    {
        String s = "Hi there";
        Console.WriteLine(s);
    }
}
```

用于调用构造创建新对象实例的IL指令是newobj。但是创建字符串的IL代码并没有使用这个指令，它使用从元数据获得的字面值构造String对象，使用Idstr指令的特殊方式构造了字面值String对象。

如果使用不安全的unsafe代码，可以从一个Char* 或 Sbyte* 构造一个String。这时要使用C#的new操作符，并调用由String提供，能接受Char\*或Sbyte\*参数的某个构造器。这些构造器将创建String对象，根据由Char实例或有符号字节构成的一个数组来初始化字符串。其他构造器则不允许接受任何指针参数，用任何托管编程语言写的安全代码都可以调用它们。

C#提供了一些特殊语法来帮助开发者在源代码中输入字面值字符串。对于换行符、回车符和退格符这样特殊的字符串，C#采用的是C/C++开发人员熟悉的转义机制

``` csharp
String s = "Hi\r\nthere.";
```

需要注意的是，一般不建议使用上述的格式。System.Environment类型定义了只读的NewLine属性。应用程序在Windows上运行时，该属性返回由回车符和换行符构成的字符串。NewLine只对平台敏感，会根据底层平台来返回恰当的字符串。

``` csharp
String s = "Hi" + Environment.NewLine + "there.";
```

可以使用C#的操作符将几个字符串链接成为一个。

``` csharp
String s = "Hi" + " " + "there.";
```

因为"Hi"" "等都是字面值，所以在这里使用+可以直接将其在编译阶段转换为一个字符串，而如果在使用非字面值的时候就会在运行时进行，在堆上创建多个字符串对象。因堆上的对象会GC，所以会对性能产生负面影响。应该使用String.Builder来处理非字面值类型字符串。

另外，C#提供了一种特殊的字符串声明方式，称之为"逐字字符串。"

``` csharp
String file = @"C:\~";
```

### 14.2.2 字符串是不可变的

String对象最重要的一点就是不可变性。字符串一旦创建便不可更改。这点允许在字符串上执行各种操作而不改变字符串本身。

``` csharp
if(s.ToUppeInvariant().SubString(10,21).EndsWith("EXE"))
{
    ...
}
```

字符串不可变还意味着在操纵和访问字符串时不会发生线程同步问题。此外，CLR还通过一个String对象共享多个完全一致的String内容。这样能减少系统中的字符串数量，节省内存，这就是字符串留用。

出于对性能的考虑，String类型与CLR精密集成。但是为了获取集成带来的性能与直接访问的好处，String类型只能是密封类，防止继承关系干扰CLR对String做出的种种预设。

### 14.2.3 比较字符串

判断字符串相等性或者排序的时候建议调用String类定义的以下方法之一。

``` csharp
//详细请参考原书或者String文档
Boolean Equals(String value,StringComparison comparisonType)
...
```

排序时执行区分大小写的比较排序，原因是加入只是大小写不同的两个字符串被视为相等，那么每次排序都可能按不同的顺序排序。

``` csharp
public enum StringComparison
{
    CurrentCulture = 0,
    CurrentCultureIgnoreCase = 1,
    InvariantCulture = 2,
    InvariantCultureIgnoreCase = 3,
    Ordinal = 4,
    OrdinalIgnoreCase = 5
}

[Flags]
public enum CompareOptions
{
    None = 0,
    IgnoreCase = 1,
    IgnoreNonSpace = 2,
    IgnoreSymbols = 4,
    IgnoreKeanaType = 8,
    IggnoreWidth = 0x00000010,
    Ordinal = 0x400000000,
    OrdinalIgnoreCase = 0x100000000,
    StringSort = 0x20000000
}
```

接受CompareOptions实参的方法是要求显示传递语言文化，传递Ordinal或者OrdinalIgoreCase标志，这些Compare方法忽略指定的语言文化。

许多程序内部，XML，路径，文件名等内容不需要向用户展示，可以直接忽略语言文化差异用于提升比较效率。

另一方面，以语言文化正确的方式来比较字符串，就应该使用StringComparison.CurrentCulture或者StringComarison.CurrentLgnoreCase。

StringComparison.InvariantCulture和StringComparison.InvariantCultureIgnoreCase平时最好不要用。虽然这两个值能保证比较时的语言文化的正确性，但用来比较内部变成所需的字符串，所需的时间远超出序号比较。此外，如果传递StringComparison.InvariantCulture，其实就是不使用任何的语言文化。所以在处理要求向用户显示的字符串时，选择它并不恰当。

要在序列比较前更改字符串中的字符的大小写，应该使用Sting的ToUpperInvariant或ToLowerInvariant方法。强烈建议用ToUpperInvariant方法对字符串进行正规化，而不要用ToLowerInvariant，因为微软对执行大写比较的代码进行了优化。事实上，执行不区分大小写的比较前，FCL会自动将字符串正规化为大写形势。之所以要用ToUpperInvariant和ToLowerInvariant方法，是因为String类没有提供ToUpperOrdinal和ToLowerOrdinal方法。ToUpper和ToLower方法对语言文化敏感。

在CLR中，每个线程都关联了两个特殊属性，每个属性都引用了一个CultureInfo对象。

+ CurrentUICulture属性 该属性获取要向用户显示的自愿，它在GUI或Web窗口应用程序中特别有用，因为它标识了在显示UI时应使用的语言。创建线程时，这个线程属性会被设置成为那个味一个默认的CultrueInfo对象，该对象标识了正在运行应用程序的Windows版本所用的语言。而这个语言是用Win32函数GetUserDefaultUIULanguage获取的。
+ CurrentCulture属性 不适合使用CurrentUICulture属性的场合就使用该属性，例如数字和日期格式化、字符串大小写转换以及字符串比较。格式化要同时用到CultrueInfo对象的“语言"和“国家”部分。创建线程时，这个线程属性被视为一个默认的CurrentCulture对象，其值通过调用GetUserDefaultLCID获取。

CultureInfo对象内部的一个字段引用了一个System.Globalization.CompareInfo对象，该对象封装了语言文化的字符排序表信息。以下代码演示了序号较和对语言文化敏感的比较的区别

``` csharp
using System;
using System.Globalization;

public static class Program
{
    public static void Main()
    {
        String s1 = "Strasse";
        String s2 = "StraPe";
        Boolean eq;
        
        //Compare返回非零值
        eq = String.Compare(s1,s2,StringComparison.Ordinal)==0;
        Console.WriteLine("Ordinal comparison:'{0}','{2}','{1}'",s1,s2,eq?"==":"!=");
        
        CultureInfo c1 = new CultureInfo("de—DE");
        
        //返回0
        eq = String.Compare(s1,s2,true,c1)==0;
        Console.WriteLine("Ordinal comparison:'{0}','{2}','{1}'",s1,s2,eq?"==":"!=");
    }
}
```

### 14.2.4 字符串留用

如果应用程序经常对字符串进行区分大小写的序号比较，或者事先指定许多字符串对象都有相同的值，就可利用CLR的字符串留用机制显著的提高性能。CLR初始化时会创建一个内部哈希表，在这个表中，Key是字符串，而Value是对托管堆中String对象的引用。	

``` csharp
public static String Intern(String str);
public static String IsInterned(String str);
```

Intern获取一个String，获取他的哈希码，在内部哈希表中检查是否有匹配的，如果存在完全相同的字符串，就返回对现有String对象的引用。如果不存在完全相同的字符串，就返回对现有String对象的引用。如果不存在完全相同的字符串，就创建字符串的副本，将副本添加到内部哈希表中，返回对该副本的引用。如果应用程序不在保持对原始String对象的引用，GC。内部哈希表中的字符串无法被删除。

IsInterned方法在哈希表中有匹配的字符串，IsInterned就返回对这个留用字符串对象的引用。但如果没有，IsInterned会返回null，不会将字符串添加到哈希表中。

程序集加载时，CLR会默认留用程序集的元数据中描述的所有字面值字符串。Microsoft知道可能因为额外的哈希表查找而显著的影响性能，所有现在禁用此功能。

即使程序集指定了这些特性和标志，CLR也可能选择对字符串进行留用，但不要依赖CLR的这个行为。事实上，除非显式调用String的Intern方法，否则永远都不要以“字符串已留用”的前提来写代码。

``` csharp
String s1 = "Hello";
String s2 = "Hello";
Console.WriteLine(object.ReferenceEquals(s1,s2));//False
s1 = String.Intern(s1);
s2 = String.Intern(s2);
Console.WriteLine(object.ReferenceEquals(s1,s2));//True
```

第一项在CLR4.5以上的版本运行，实际显示会是Ture。这个版本的CLR选择忽略C#编译器插入的特性和标志。

可以通过一个例子来理解字符串留用的功效。

``` csharp
private static Int32 NumTimesWordAppearsEquals(String word,String[] wordList)
{
    Int32 count = 0;
    for(Int32 wordnum = 0;wordnum<wordlist.Length;wordnum++)
    {
        if(word.Equals(wordlist[wordnum],StringComparison.Ordinal))
            count++;
    }
    return count;
}
```

该方法用于统计指定单词列表中出现了多少次并返回计数。

``` csharp
private static Int32 NumTimesWordAppearsIntern(String word,String[] wordlist)
{
    //这个方法假定wordlist中的所有数组元素都引用已留用的字符串
    word = String.Intern(word);
    Int32 count = 0;
    for(Int32 wordnum = 0;wordnum<wordlist.Length;wordnum++)
    {
        if(Object.ReferenceEquals(word,wordlist(wordnum)))
            count++;
    }
    return count;
}
```

这个使用比较引用，也就是比较地址，相对效率快于上面的方法。要注意的是，这个代码会将字符串存储内部留用的哈希表，实际上可能会影响效率。这也是C#编译器默认不启用字符串留用的原因。

### 14.2.5 字符串池

编译源代码时，编译器必须处理每个字面值字符串，并在托管模块的元数据中嵌入。同一个字符串在源代码中多次出现，把他们都嵌入元数据会使生成的文件无谓的增长。

为解决这个问题，许多编译器只在模块的元数据中只将字面值字符串写入一次。引用该字符串的所有代码都被修改成引用元数据中的同一个字符串。C/C++编译已使用该技术多年，称之为“字符串池”。

### 14.2.6 检查字符串中的字符和文本元素

System.Char实际上代表一个16位Unicode码值，而且该值不一定就等于一个抽象Unicode字符。例如有的抽象Unicode字符是两个码值的组合。U+0625和U+0650字符组合起来就构成了一个抽象字符或者 **文本元素**。

除此之外，有的Unicode文本元素要用两个16位值表示。第一个称为“高位代理项”，第二个称之为“低位代理项”。其中，高位代理项范围在U+D800到U+DBFF之间。低位代理项在U+DC00到U+DFFF之间。

### 14.2.7 其他字符串操作

| 成员名称  | 方法类型 | 说明                                                         |
| --------- | -------- | ------------------------------------------------------------ |
| Clone     | 实例     | 返回对同一个对象this的引用。能这样做是因为String对象不可变。该方法实现了String的ICloneable接口。 |
| Copy      | 静态     | 返回新的副本                                                 |
| CopyTo    | 实例     | 将字符串中部分字符复制到一个新字符串                         |
| Substring | 实例     | 返回代表原始字符串一部分的新字符串                           |
| ToString  | 实例     | 返回对同一个对象的引用                                       |

除此之外还有很多方法，请查看原书或者微软文档，需要注意的是他们都是会返回新的字符串引用，因为字符串本身是不可变的。

## 14.3 高效率构造字符串

StringBuilder对象包含一个字段，该字段引用了由Char结构构成的数组。可利用StringBuilder的各个成员来操纵该字符数组，高效率地缩短字符串中的字符。如果字符串变大，超过了事先分配的字符数组大小，StringBuilder会自动分配一个更大的数组。

### 14.3.1 构造StringBuilder对象

StringBuilder并不是基元类型。

``` csharp
StringBuilder sb = new StringBuilder();
```

StringBuilder提供了很多构造器，每个构造器的职责是分配和初始化由StringBuilder对象维护的状态。

+ 最大容量 

  一个Int32值，指定了能放到字符串中的最大字符数，默认值为Innt32.MaxValue。一般不需要更改，但有时需要指定较小的最大值来保证构造指定长度以下的字符串。设置后不能更改。

+ 容量

  Int32，指定StringBuilder中维护的字符数组的长度。默认为16。如果事先要知道在这个StringBuilder中放入多少个字符，那么构造StringBuilder的时候应该自己设置容量。

+ 字符数组

  一个由Char结构成的数组，负责维护“字符串”的字符内容。字符数总是小于或等于“容量”和“最大容量”。可在构造时向其传入一个String来初始化数组。

### 14.3.2 StringBuilder的成员

| 成员           | 成员类型            | 说明                                                         |
| -------------- | ------------------- | ------------------------------------------------------------ |
| MaxCapacity    | 只读属性            | 返回字符串能荣达的最大字符数。                               |
| Capacity       | 可读/可写属性       | 获取或设置字符数组的长度。将容量设得比字符串长度小或者比MaxCapacity大将抛出ArgumentOutOfRangeException。 |
| EnsureCapacity | 方法                | 保证字符数组至少具有指定的长度，如果传给方法的值大于目前的容量，将会扩大，小于则无用。 |
| Length         | 可读/可写属性       | 获取或设置当前字符串中字符数                                 |
| ToString       | 方法                | 无参版本返回String                                           |
| Chars          | 可读/可写索引器属性 | 获取或设置字符串数组中索引位置的字符。                       |
| Clear          | 方法                | 清除                                                         |
| Append         | 方法                | 添加                                                         |
| Insert         | 方法                | 插入                                                         |
| AppendFormat   | 方法                | 在字符串末尾追加指定的零个或者多个对象，如有必要，进行扩充。 |
| AppendLine     | 方法                | 在字符数组末尾追加一个行中止符或者一个带行中止符的字符串     |
| Replace        | 方法                | 将字符数组中的一个字符替换成另一个字符，或者将一个字符串替换成为另一个字符串 |
| Remove         | 方法                | 从字符数组中删除指定范围的字符                               |
| Equals         | 方法                | 略                                                           |
| CopyTo         | 方法                | 略                                                           |

使用StringBuilder的方法要注意，大多数方法返回的都是对同一个StringBuilder对象的引用，所以几个操作符可以连到一起完成。

``` csharp
StringBuilder sb = new StringBuilder();
String e = sb.AppendFormat("{0} {1}","Jeffrey","Richter").Replace(' ','-').Remove(4,3).ToString();
Console.WriteLine(e);
```

可以看到，有些String提供的方法StringBuilder并没提供，因而需要在String与StringBuilder之间不断的进行转换来完成具体的任务。

## 14.4 获取对象的字符串表示：ToString

System.Object实现的ToString只是返回对象所属类型的全名。任何类型想要提供合理的方式来获取当前对象值的字符串表示应该重写ToString方法。FCL内建的许多核心类型都重写了ToString方法。

### 14.4.1 指定具体的格式和语言文化

无参ToString方法有两个问题。首先，被调用者无法控制字符串的格式。其次，调用者不能方便地选择一种特有语言文化来格式化字符串。少数情况下，应用程序需要使用与调用线程不同的语言文化来格式化字符串。为了对字符串进行更多的控制，需要重写ToString方法允许具体的格式和语言文化信息。

``` csharp
public interface IFormattable
{
    String ToString(String format,IFomarProvider formatProvider);
}
```

FCL的所有基类型都实现了这个接口。此外，还有另一些类型也实现了它。每个枚举类型定义都自动实现IFormattable接口，以便从枚举类型的实例获取一个有意义的字符串符号。

IFormattable的ToString方法获取两个参数。第一个是format，这个特殊字符串告诉方法应该如何格式化对象。第二个是formatProvider，是实现了System.IFormatProvider接口的一个类型的实例。该类型为ToString方法提供具体的语言文化信息。

实现IFormattable接口的ToString方法的类型觉得那些格式字符串能被识别。如果传递的格式字符串无法识别，类型应抛出Sytsem.FormatException异常。

格式化数字适合应用对语言文化敏感的信息。Guid类型的ToString方法只返回代表GUID值的字符串。生成GUID的字符串时不必考虑语言文化，因为GUID只适用于编程。

调用IFormattable的ToString方法可以不传递null，而是传递一个对象引用，该对象的类型实现了IFormatProvider接口：

``` csharp
public interface IFormatProvider
{
    Object GetFormat(Type formatType);
}
```

IFormattable接口的基本思路是：当一个类型实现了该接口，就认为该类型的实例能提供对语言文化敏感的格式信息，与调用线程关联的语言文化应被忽略。

FCL的格式只有少数实现了IFormatProvider接口。System.Globalization.CultureInfo类型就是其中之一。

``` csharp
Decimal price = 123.54M;
String s = price.ToString("C",new CultureInfo("vi-VN"));
MessageBox.Show(s);
```

### 14.4.2 将多个对象格式化为一个字符串

有时需要构造由多个已格式化对象构成的字符串。

``` csharp
String s = String.Format("On {0},{1} is {2} years old.",new DateTime(2012,4,22,14,35,5),"Aidan",9);
Console.WriteLine(s);
```

在内部，Format方法会调用每个对象的ToString方法来获取对象的字符串表示。返回的字符串依次连接在一起，并返回最后的字符串。看起来不错，但是它意味着素有对象要使用它们的常规格式和调用线程的语言文化信息来格式化。

在大括号内指定格式信息，可以更全面地控制对象格式化。

``` csharp
String s = String.Format("On {0:D},{1} is {2:E} years old.",new DateTime(2012,4,22,14,35,5),"Aidan",9);
Console.WriteLine(s);
//On Sunday, April 22,2021,Aidan is 9.00000E+000 years old.
```

Format将会在解析过程中，将D与null传递给对应数据的IFormattable接口的ToString方法，并为该方法的两个参数分别传递“D”和null。如果对应类型并没有实现IFormattable接口，Format会调用最终从Object继承的ToString()。

String类提供了Format的数个重载版本，可用于特定语言文化信息，这里不再展开。

如果使用StringBuilder而不是String来构造字符串，可以调用StringBuilder的AppendFormat方法。它的原理和String类似，只是会格式化字符串并将其添加到StringBuilder字符数组当中。

System.Console的Write和WriteLine方法也能获取格式字符串和可替换参数。但Console的Write和WriteLine方法没有重载版本能获取一个IFormatProvider。

### 14.4.3 提供定制格式化器

可定义一个方法，在任何对象需要格式化字符串的时候由StringBuilder的AppendFormat方法调用该方法，String.Format同理。

``` csharp
using System;
using System.Text;
using System.Threading;

public static class Program
{
    public static void Main()
    {
        StringBuilder sb = new StringBuilder();
        sb.AppendFormat(new BoldInt32s(),"{0}{1}{2:M}","Jeff",123,DateTime.Now);
        Console.WriteLine(sb);
    }
}

internal sealed class BoldInt32s : IFormatProvider,ICustomFormatter
{
    public Object GetFormat(Type formatType)
    {
        if(formatType == typeof(ICustomFormatter))return this;
        return Thread.CurrentThread.CurrentCulture.GetFormat(formatType);
    }
    
    public String Format(String format,Object arg,IFormatProvider formatProvider)
    {
        String s;
        IFormattable formattable = arg as IFormattable;
        
        if(formattable == null) s = arg.ToString();//如果arg并没有实现IFormattable接口，调用ToString
        else s = formattable.ToString(format,formatProvider);
        
        if(arg.GetType() == typeof(Int32))//如果类型为Int32 那么加粗
            return "<B>" + s + "</B>";
        return s;
    }
}
```

## 14.5 解析字符串来获取对象：Parse

本节讨论如何获取字符串并得到它的对象表达。

能解析字符串的任何类型都提供了静态方法Parse。方法获取一个String并返回类型的实例。FCL中，所有数值类型、DateTime、TimeSpan以及其他一些类型均提供了Parse方法。

``` csharp
public static Int32 Parse(String s,NumberStyles style,IFormatProvider provider);
```

其中System.Globalization.NumberStyles参数style是位标志集合，标识了Parse应在字符串查找的字符，有不存在的样式会丢出异常。

``` csharp
Int32 x = Int32.Parse(" 123",NumberStyles.None,null);//error 解析字符串前有空白字符
Int32 x = Int32.Parse(" 123",NumberStyles.AllowLeadingWhite,null);//略过空白字符，详细请参照参考文档
```

需要注意的是，当应用程序频繁调用Parse，且Parse频繁抛出异常的时候，程序性能会显著下降。为此，微软提供了

``` csharp
public static Boolean TryParse(String s,NumberStyles style,IFormatProvider provider,out Int32 result);
```

## 14.6 编码：字符和字节的互相转换

在进行网络传输等情况下，大量的字符不利于传输效率。因而需要对字符进行编码，在需要的使用时进行解码。

用System.IO.BinaryWrite或System.IO.SystemWriter类型将字符串发送给文件或者网络流时，通常要进行编码。对应使用System.IO.BinaryReader或System.IO.StreamReader的时候需要从文件或网络中读取字符串时，通常要进行解码。默认使用的编码方式为UTF-8.

其他常见编码方式如下

+ UTF-16 每16字符编码为2字节，不对字符造成影响，不发生压缩，性能优秀，也被称之为“Unicode”编码。
+ UTF-8 字符编码为1、2、3、4个字节都有可能，以因对不同的语言条件。UTF-8编码非常流行，但是当大量使用0x0800或以上，也就是非英文字符时，效率反而不如UTF-16。
+ UTF-32 4字节编码所有字符。需要使用简单算法来遍历所有字符，同时不愿意花费额外精力应对字节数可变的字符可使用该方式。但显然网络传输和内存使用率并不高，常在程序内部使用。
+ UTF-7 使用七位值表示，已淘汰。
+ ASCII 编码 16位字符编码成ASCII字符，假设字符串完全由ASCII范围内字符组成，数据将会被压缩到原来的一半，速度快。

要编码或者解码一组字符时，应获取从System.Text.Encoding派生的一个类的实例。抽象类Encoding提供了几个静态只读属性，每个属性都返回从Encoding派生的一个类的实例。

``` csharp
using System;
using System.Text;

public static class Program
{
    public static void Main()
    {
        String s = "Hi there";
        Encoding encodeBytes = Encoding.UTF8;
        Byte[] encodedBytes = encodingUTF8.GetBytes(s);
        Console.WriteLine("Emcoded bytes:"+BitConverter.ToString(encodedBytes));
        String decodedString = encodingUTF8.GetString(encodedBytes);
        Console.WriteLine("Decoded string:"+ decodedString);
    }
}
```

Encoding还提供了其他的解码方式，其中Default属性返回的对象是依据用户当前代码页的设置来进行解码，在控制面板设置该参数。不建议使用，这样可能会导致在不同的机器下程序的运作方式发生异常。

除此之外，Encoding还提供了静态GetEncoding方法，允许指定代码页，返回可使用指定代码页来编码/解码的对象。

首次请求一个编码对象时，Encoding类的属性或GetEncoding方法会为请求的编码方案构造对象，并返回该对象。假如请求的编码之前请求过，Encoding类会直接返回之前构造好的对象。

除此之外，还可以构造一下某个类的实例：System.Text.UnicodeEncoding,System.Text.UTF8Encoding,System.Text.UTF32Encoding,System.Text.UTF7Encoding或System.Text.ASCIIEncoding。需要注意的是，构造这些类都会在堆中产生新的对象。

从Encoding派生的所有类型都提供了GetByteCount方法，它能统计对一组字符进行编码所产生的字节数，同时不实际进行编码，另有GetCharCount，返回解码得到的字符数，同时不实际进行解码。

GetByteCount和GetCharCount方法速度一般，如果需要更快的速度而不是准确性，可以改调用GetMaxByteCount或GetMaxCharCount，获取最坏情况下的值。

| 方法名称    | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| GetPreamble | 返回一个字节数组，指出在写入任何已编码字节前，首先应在流中写入什么字节。这些字节经常被称为“前导码”或“字节顺序标记”字节。 |
| Convert     | 将字节数组从一种编码转换为另一种编码方式，内部调用来源编码的GetChars方法，将结果传递给目标编码的GetBytes方法。结果数据返回给调用者。 |
| Equals      | 如果从Encoding派生的两个对象代表相同的代码页和前导码设置，返回true。 |
| GetHashCode | 略                                                           |

### 14.6.1 字符和字节流的编码和解码

所有的Encoding派生都不维护多个方法调用之间的状态。要编码或解码以数据块形式传输的字符/字节，必须进行一些额外的工作来维护方法调用之间的状态，防止数据丢失。

字符块解码首先要获取一个Encoding派生对象引用，在调用其GetDecoder方法。返回一个从System.Text.Decoder类派生的新对象。需要注意的是Decoder在文档之中找不到任何具体实现了Decoder的类。他们在FCL内部使用。

Decoder的所有派生类都提供了两个重要的方法：GetChars和GetCharCount。显然，这些方法的作用是对字节数组进行解码。该方法会尽可能的对字节数组进行解码，当前数组包含的字节不足以完成一个字符，剩余的字节会尽可能保存到Decoder对象内部。下次调用的时候会和之前存储的字节进行拼接。

从Encoding派生的类型可用于无状态编码和解码。而从Decoder派生的类型只能用于解码。以成块的方式编码字符串需要调用GetEnocder方法，而不是调用Encoding对象的GetDecoder方法。GetEncoder返回一个新构造的对象，它的类型从抽象基类System.Text.Encoder派生。

从Encoder派生的所有类都提供了两个重要方法：GetBytes和GetByteCount。每次调用，从Encoder派生的对象都会维护余下数据的状态信息，以便成块对数据进行编码。

### 14.6.2 Base-64 字符串编码和解码

另一个方案是将字节序列编码成Base-64字符串。FCL专门提供了进行Base-64编码和解码的方法。他通过System.Convert类型提供的一些静态方法来进行。

将Base-64字符串编码成字节数组需要调用Convert的静态FromBase64String或FormBase64CharArray方法。类似地，将字节数组解码成Base-64字符串需要调用Convert的静态ToBase64String或ToBase64CharArray方法。

``` csharp
using System;

public static class Program
{
    public static void Main()
    {
        //获取一组随机字节
        Byte[] bytes = new Byte[10];
        new Random().NextBytes(bytes);
        
        //显示字节
        Console.WriteLine(BitConverter.ToString());
        
        //解码
        String s = Convert.ToBase64String(bytes);
        Console.WriteLine(s);
        
        //转回去
        bytes = Convert.FromBase64String(s);
        Console.WriteLine(BitConverter.ToString(bytes));
    }
}
```

## 14.7 安全字符串

String对象在内存中包含一个字符数组，如果允许执行不安全或非托管代码，这些代码就可以扫描进程地址空间，找到敏感数据的字符串，以非授权的方式使用。即使支持使用String，CLR也可能无法立即重用String对象的内存。

为安全考虑，FCL添加了一个更加安全的字符串类，即System.Security.SecureString。构造SecureString对象时，会在内部分配一个非拓转内存块，在其中包含一个字符数组。

这些字符串的字符经过加密处理，可以防范而言代码获取机密。利用AppendChar、InsertAt、RemoveAt、SetAt。可以在方法内部解密字符，执行指定操作，再重新加密字符。这意味着字符有一段时间处于未加密状态，且性能相对缓慢，应尽量减少该系列操作。

SecureSting类实现了IDisposable接口，允许以简单的方式确定性地摧毁字符串中的安全内容。和String对象不同，SecureString对象被回收之后，加密字符串的内容将不再存在于内存当中。

最后，可创建自己的方法来接收SecureString对象参数。方法内部必须允许SecureString对象创建一个非托管缓冲区，它将用于包含解密后的字符，之后才能让方法使用缓冲区。为降低风险，应该尽可能的减短字符串存在时间，结束使用字符串之后，代码应该立即清零并释放缓冲区。此外绝对不要将SecureString的内容放在String当中。

``` csharp
using System;
using System.Security;
using System.Runtime.InteropServices;

public static class Program
{
 	public static void Main()
    {
        using (SecureString ss = new SecureString())
        {
            while(true)
            {
                ConsoleKeyInfo cki = Console.ReadKey(true);
                if(cki.key == ConsoleKey.Enter)break;
                ss.AppendChar(cki.KeyChar);
                Console.Write("*");
            }
            Console.WriteLine();
            DisplaySecireString(ss);
        }
    }
    
    private unsafe static void DisplaySecireString(SecireString ss)
    {
        char* pc = null;
        try
        {
            pc = (Char*)Marshal.SecureStringToCoTaskMemUnicode(ss);
            for(Int32 index = 0;pc[index] !=0;index++)
                Console.Wriye(pc[index]);
        }
        finally
        {
            if(pc!=null)
                Marshal.ZeroFreeCoTaskMemUnicode((IntPtr)pc);
        }
    }
}
```

