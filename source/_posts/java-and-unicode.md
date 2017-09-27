---
title: Java 与 Unicode
date: 2016-03-15 21:55
---

优化项目时，遇到了 Unicode 一些相关的问题，为此，需要了解 Java 中是如何支持 Unicode。以下是整理的笔记。

{% asset_img bg2016031503.jpg Java And Unicode %}

## 一、Unicode 是什么？

Unicode 简单说来就是对世界上大部分的文字系统进行了整理，然后为他们编号。

目前最新的版本是 2015 年 6 月 17 日公布的 8.0.0，收录了 120737 个字符。

<!-- more -->

Unicode 将这些字符分成 17 组编排，每组称为平面（Plane），每平面拥有 65536（即 $$2^{16}$$）个编号，每个编号称为码点。

U+0000 至 U+FFFF 构成的平面称为**基本多文种平面**，英文名称 Basic Multilingual Plane，简称 **BMP**。最常见的字符通常都在该平面内。剩余从 U+10000 至 U+10FFFF 的 16 个平面称为辅助平面。

在表示一个 Unicode 字符时，通常会用 “U+” 然后紧接着一组十六进制的数字来表示这一个字符。

> U+4E2D = 中  
> U+1F466 = {% asset_img bg2016031501.png %}

使用过 emoji 的可能知道，emoji 在不同的平台（比如苹果、安卓）上的表现形式是不同的

{% asset_img bg2016031502.png emoji %}

上面的都可以使用 U+1F601 表示，这也说明了 Unicode 的另一个特点，为每一个字符而非字形定义唯一的代码（码点）。

## 二、UTF-32、UTF-16、UTF-8

前面提到的 U+4E2D，最简单的存储方式是使用 32 位二进制，即 4 个字节来表示, 称为 UTF-32（或 UCS-4）。

假设我们现在使用 UTF-32 对字符进行存储、传输，那么像 A（U+0041）这样的原本仅需要一个字节来表示的字符，现在需要 4 个字节，这就造成了存储空间浪费、传输效率降低。

实际上我们绝大部分使用的是基本多文众平面中的字符，也就是说，我们使用的绝大部分字符 1-2 个字节就可以表示了。

除基本平面可以用 2 个字节表示外，其它的平面都至少需要 3 个字节，因为它们的范围是 U+10000 至 U+10FFFF

如果以 16 位长的[码元](https://zh.wikipedia.org/wiki/UTF-16)（即 2 个字节为单位）表示基本平面内的字符，一个 16 位的码元就够了，但是要表示辅助平面内的字符，需要一对，称为**代理对（surrogate pair）**。由于长度不固定，因此这是一个变长表示。

|平面| 编码范围     | 字节数量    |
|:---| :------------- | :------------- |
|基本多文种平面| U+0000---U+FFFF       | 2      |
|辅助平面| U+10000---U+10FFFF    | 4      |

如果用 8 位长的码元（即 1 个字节为单位），需要 1 到 4 个字节，称为 [UTF-8](https://zh.wikipedia.org/wiki/UTF-8)。这也是一个变长表示。

| 编码范围     | 字节数量    |
| :------------- | :------------- |
| 0x0000 - 0x007F | 1|
|0x0080 - 0x07FF|2 |
|0x0800 - 0xFFFF|3|
|0x010000 - 0x10FFFF|4|

## 三、Java 对 Unicode 的支持

Java 使用的是 UTF-16。char 类型的长度为 2 个字节，就代表一个长度为 16 的码元。因此对于基本平面内的字符，一个 char 就可以表示一个字符。

```
char ch = '中';
Integer.toHexString(ch); ==> 4e2d
```

这与 “中” 的码点 U+4E2D 是一致的。

Java 提供了将码点转换为 UTF-16 形式(即用 char 来表示)的函数， java.lang.Character.toChars，该函数声明如下:

```
public static char[] toChars(int codePoint);
```

使用该函数，我们看到辅助平面内的字符需要用两个char来表示：

```
Character.toChars(0x4E2D).length ==> 1
Character.toChars(0x1F466).length ==> 2
```

因此，原来我们统计字符个数的方法就不正确了。

```
new String(Character.toChars(0x1F466)).length() ==> 2
```

这时，可以使用

```
String s = new String(Character.toChars(0x1F466));
s.codePointCount(0, s.length()) ==> 1
```

Java 中关于使用 Character.toCodePoint 计算码点，计算方法为:

```
((high << 10) + low) + (0x010000 - ('\uD800' << 10) - '\uDC00')
```

还有其他的一些处理码点的函数（部分）：

```
String.codePointAt 返回指定位置的码点
String.offsetByCodePoints 从 index 开始，接下来第n个代码点的下标
String.Character.codePointBefore 返回指定位置处前面的一个码点
Character.isBmpCodePoint 判断一个码点是否为 BMP
Character.toCodePoint 给定一个代理对，转换为码点
Character.isSupplementaryCodePoint 判断代码点是否为辅助平面，即是否由两个 char 表示
Character.charCount 判断给定码点需要几个char表示
Character.isSurrogatePair 判断两个 char 是否为一个代理对
Character.isHighSurrogate 判断 char 是否为代理对中的高位
Character.isLowSurrogate 判断 char 是否为代理对中的低位
```

## 四、参考

https://zh.wikipedia.org/wiki/Unicode
https://zh.wikipedia.org/wiki/Unicode%E5%AD%97%E7%AC%A6%E5%B9%B3%E9%9D%A2%E6%98%A0%E5%B0%84
http://apps.timwhitlock.info/emoji/tables/unicode
http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html
http://www.ruanyifeng.com/blog/2014/12/unicode.html
