---
title: ZString
date: 2022-06-13 09:28:25
tags: [ C#, Unity]
categories: [ C#]
about: 
description: "阅读笔记.NETCore和Unity的零分配StringBuilder"
---

## 前言

如果可以阅读英文ReadMe和源码的话，还是建议去直接看原作者的 [GitHub](https://github.com/Cysharp/ZString#getting-started)。这篇博客的翻译和学习内容因为个人水准问题难免所有出入。

如果对内容理解有所困难，请尝试查看这篇博客[CLRViaC# 5 值类型拆装箱](https://crazyink47.xyz/2022/02/18/%E3%80%8ACLRViaC-%E3%80%8B%E7%AC%AC5%E7%AB%A0-%E5%9F%BA%E5%85%83%E7%B1%BB%E5%9E%8B%E3%80%81%E5%BC%95%E7%94%A8%E7%B1%BB%E5%9E%8B%E5%92%8C%E5%80%BC%E7%B1%BB%E5%9E%8B/#%E5%80%BC%E7%B1%BB%E5%9E%8B%E4%B8%8E%E5%BC%95%E7%94%A8%E7%B1%BB%E5%9E%8B%E7%9A%84%E4%B8%80%E4%BA%9B%E5%8C%BA%E5%88%AB) 和这个[CLRViaC# 14 字符、字符串和文本处理](https://crazyink47.xyz/2022/03/29/%E3%80%8ACLRViaC-%E3%80%8B%E7%AC%AC14%E7%AB%A0-%E5%AD%97%E7%AC%A6%E3%80%81%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%92%8C%E6%96%87%E6%9C%AC%E5%A4%84%E7%90%86/)。

如果只是需要快速上手使用，那这篇博客还可以。

## ZString

.NETCore和Unity的零分配StringBuilder。

+ 结构化StringBuilder以回避构筑它自身而带来的内存分配。
+ 从 **TreadStatic** 或者 **ArrayPool** 租用并重写缓存。
+ 所有拼接通过泛型方法Append\<T>(T value)来直接重写缓存来避免value.ToString()
+ T1~T16 AppendFormat(AppendFormat<T1,...,T16>(string format, T1 arg1, ..., T16 arg16)避免结构体装箱。
+ T1~T16 Concat(Concat<T1,...,T16>(T1 arg1, ..., T16 arg16))  同理，规避装箱和值类型ToString带来的内存分配。
+ ZString.Format/Concat/Join可以很轻松的代替String.Format/Concat/Join
+ 可同时构筑Utf16(Span\<char>)和Utf8(Span\<byte>)directly
+ 可以通过缓冲区来最后分配字符串
+ 与Unity TextMeshPro集成以避免字符串分配

![ZString](https://user-images.githubusercontent.com/46207/74473217-9061e200-4ee6-11ea-9a77-14d740886faa.png)

这个图表比对了以下代码

+ “x:” + x + " y:" + y + " z:" +z
+ ZString.Concat("x:", x, " y:", y, " z:", z)
+ string.Format("x:{0} y:{1} z:{2}", x, y, z)
+ ZString.Format("x:{0} y:{1} z:{2}", x, y, z)
+ new StringBuilder(), Append(), .ToString()
+ ZString.CreateStringBuilder(), Append(), .ToString()

“x:” + x + " y:" + y + " z:" +z 被C#编译器转换为了 String.Concat(new[]{"x:",x.ToString(),"y:"y.ToString(), " z:", z.ToString() })。在每一次.ToString的过程之中，都会分配每个参数的内存。string.Format通过调用String.Format(string,object,object,object)，因而每个参数都会导致值类型到Object的装箱。

所有ZString方法只在最后分配一次String返回，同样的，ZString已经写入了内部缓存，因而如果输出到同样原理减少string消耗的api，你可以实现完全的0gc。

原作者博客记录了更多有关细节: [medium@neuecc/ZString](https://medium.com/@neuecc/zstring-zero-allocation-stringbuilder-for-net-core-and-unity-f3163c88c887)

使用ZString的相关项目, [Cysharp/ZLogger](https://github.com/Cysharp/ZLogger) - 0分配Text记录器.

## Public API

[ZString补充方案](https://gitee.com/rehen2000/zstring)的作者在代码了写了大量的注释，直接看源代码也不错。

``` csharp
public int Length;//返回字符串长度
public string Intern();//将ZString放入intern缓存表中供外部使用
public static string Intern(string value);
public char this[int i]//下标取值
public override int GetHashCode();//获取哈希值
public override bool Equals(object obj);//字面值比较
public override string ToString();//转换为String
public unsafe zstring ToUpper();//转换为大写
public unsafe zstring ToLower();//转换为小写
public zstring Remove(int start, int count);//移除剪切
public unsafe zstring Replace(char old_value, char new_value);//子字符串替换
public unsafe zstring Substring(int start, int count);//剪切start起count长度子串
public bool Contains(string value);//判断包含
public zstring Concat(zstring value);//拼接
//...太多不写了，这里已经差不多够用了，需要更详细的内容建议查看源代码，原作者比我写的详细多了，我就不中译中了
```

## 参考文献

+ [有趣的阅读 - ZString, 用于.NETCore和Unity的零分配StringBuilder](https://zhuanlan.zhihu.com/p/157678593)
+ [ZString](https://github.com/Cysharp/ZString#getting-started)
+ [ZString补充方案](https://gitee.com/rehen2000/zstring)
