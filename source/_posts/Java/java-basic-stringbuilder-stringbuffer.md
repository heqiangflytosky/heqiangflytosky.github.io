---
title: Java 基础 -- StringBuilder 和 StringBuffer
categories: Java
comments: true
tags: [Java]
description: 介绍 StringBuilder 和 StringBuffer 的用法
date: 2014-1-22 10:00:00
---

## 概述

StringBuilder 和 StringBuffer 这两个类是我们经常使用的类，在[Java 基础 -- 深入理解 String ](http://www.heqiangfly.com/2014/01/20/java-basic-knowledge-about-string/) 简单介绍过这两个类。与 String 类不同，StringBuffer 和 StringBuilder 类的对象能够被多次的修改，并且不产生新的未使用对象。因此，在某些场景下使用StringBuilder 和 StringBuffer能够优化我们程序的内存和性能。
StringBuilder 和 StringBuffer 这两个类都是被 final 修饰的，它们是不能被继承的。
首先来看一下他们的类关系图：

![效果图](/images/java-basic-stringbuilder-stringbuffer/class.png)

## 方法

 - append()：将指定的字符串追加到此字符序列。
 - appendCodePoint()：将 codePoint参数的字符串表示法附加到此序列。比如追加一些ascii码表示的字符或者表情符号emoji等。
 - capacity()：返回当前容量。
 - charAt(int index)：返回此序列中指定索引处的 char 值。
 - codePointAt(int index)：返回指定位置字符的Unicode code point。和String类似。
 - codePointBefore(int index)：返回指定位置之前字符的Unicode code point。和String类似。
 - codePointCount(int index)：返回指定位置之后字符的Unicode code point。和String类似。
 - delete()：移除此序列的子字符串中的字符。
 - deleteCharAt()：移除此序列某个指定位置的字符。
 - ensureCapacity(int minimumCapacity)：确保容量至少等于指定的最小值。
 - getChars(int srcBegin, int srcEnd, char[] dst, int dstBegin)：将字符从此序列复制到目标字符数组 dst。
 - indexOf()：返回第一次出现的指定子字符串在该字符串中的索引。如果不存在就返回 -1。
 - insert()：将指定数据的字符串表示形式插入此序列中。
 - lastIndexOf()：返回最右边出现的指定子字符串在此字符串中的索引。如果不存在就返回 -1。
 - length()：返回长度（字符数）。
 - offsetByCodePoints(int index, int codePointOffset)：返回指定index位置偏移 codePointOffset 处字符的Unicode code point。和String类似。
 - readObject(ObjectInputStream s)：将StringBuffer 或 StringBuffer对象 从ObjectInputStream 对象中恢复，反序列化。
 - writeObject(ObjectOutputStream s)：将当前StringBuffer 或 StringBuffer对象保存到 ObjectOutputStream 对象，序列化。
 - replace()：使用给定 String 中的字符替换此序列的子字符串中的字符。
 - reverse()：将此字符序列用其反转形式取代。
 - setCharAt(int index, char ch)：将给定索引处的字符设置为 ch。
 - setLength(int newLength)：设置字符序列的长度。
 - subSequence(int start, int end)：返回一个新的字符序列，该字符序列是此序列的子序列。
 - substring()：返回一个新的 String，它包含此字符序列当前所包含的字符子序列。
 - toString()：转换成 Sting 对象。
 - trimToSize()：将StringBuffer 或 StringBuffer 对象的中存储空间缩小到和字符串长度一样的长度，减少空间的浪费。


## 区别

两者的方法基本相同。区别就是 StringBulider 是非线程安全的，StringBuffer 是线程安全的。通过看源码发现 StringBuffer 的方法基本都使用了 synchronized 关键字来保证线程安全。
在单线程环境下，StringBulider 比 StringBuffer 效率更高，因为省去了加锁的开销。

## 实现原理

StringBulider 比 StringBuffer 都是继承自 AbstractStringBuilder 类，下面我们就分析一下 AbstractStringBuilder 的实现来看一下它们的原理。
AbstractStringBuilder 内部用 `char[] value` 变量类存储字符序列，`int count` 变量来标识实际字符串的长度。
在初始化 StringBulider 比 StringBuffer 时如果不指定容量值，那么 `char[]` 数组默认大小是16。那么在我们不断的 append 过程中，数组的容量肯定会有不够用的时候，这个时候就会进行扩容。

### 扩容

在 append 方法中，在进行字符串拼接之前都会调用 `ensureCapacityInternal` 方法检查当前数组是否需要扩容，如果需要扩容的就要调用 `expandCapacity` 方法进行扩容：

```
    void expandCapacity(int minimumCapacity) {
        int newCapacity = value.length * 2 + 2;
        if (newCapacity - minimumCapacity < 0)
            newCapacity = minimumCapacity;
        if (newCapacity < 0) {
            if (minimumCapacity < 0) // overflow
                throw new OutOfMemoryError();
            newCapacity = Integer.MAX_VALUE;
        }
        value = Arrays.copyOf(value, newCapacity);
    }
```

每次的扩容会调用Arrays.copyOf重新生成一个原来2倍+2大小的数组，然后把字符串赋值到原来的数组。这期间伴随着内存的申请和回收，如果字符串的长度较大，对性能会造成很大的应用，因此，我们在构造 StringBulider 比 StringBuffer 时要合理预估使用的大小，合理设置初始容量大小。

<!--

@startuml
Title "StringBuilder 和 StringBuffer类图" 
interface CharSequence
interface Serializable
interface Appendable
abstract class AbstractStringBuilder
class StringBuilder


CharSequence <|.. AbstractStringBuilder
CharSequence <|.. StringBuilder
Serializable <|.. StringBuilder
Appendable <|.. StringBuilder
AbstractStringBuilder <|-- StringBuilder
CharSequence <|.. StringBuffer
Serializable <|.. StringBuffer
Appendable <|.. StringBuffer
AbstractStringBuilder <|-- StringBuffer
@enduml
-->