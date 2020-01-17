---
title: Kotlin -- 简介
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin 的特点以及一些学习资料
date: 2019-12-2 10:00:00
---

## 概述

Kotlin 是一种在 Java 虚拟机上运行的静态类型编程语言，Kotlin 可以编译成Java字节码，也可以编译成 JavaScript 字节码，方便在没有 JVM 的设备上运行。
Kotlin 和 Java 之间之所以能无缝对接就是因为无论是 Java 还是 Kotlin，最后都会被编译成 dex 字节码（android虚拟键最终执行的就是dex字节码），java经历 .java源文件-> .class java可执行文件-> dex字节码； Kotlin 经历 .kt源文件->dex字节码。
Kotlin 比 Java 语言更简洁、更安全、易扩展，能够静态检测常见陷阱，也可以应用于Android开发、JavaScript开发、服务器端开发的程序中。由于从实际使用效果来说，Kotlin语言比Java语言的开发效率高并且使用更安全，因此Kotlin语言的应用越来越广泛。
Kotlin 第一个官方 1.0 版发布于 2016 年 2 月，2017年谷歌I/O大会上 Android 团队宣布 Kotlin 成为其官方头等支持语言，这一举措，更进一步推动了 Kotlin 的应用。


## 学习资料

[Kotlin中文官网](https://www.kotlincn.net/)
[Kotlin中文官网文档：非常好的学习资料](http://www.kotlincn.net/docs/reference/)
[Kotlin中文官网教程](http://www.kotlincn.net/docs/tutorials/)
[菜鸟教程里面的Kotlin教程](https://www.runoob.com/kotlin/kotlin-tutorial.html)

## Kotlin 和 Java 的对比

[github中的from-java-to-kotlin：java和kotlin常用语法对比](https://github.com/MindorksOpenSource/from-java-to-kotlin)

笔者是 Android 开发出身，学习 Kotlin 时自然要和 Java 语言来做一番比较。
对比 Java，Kotlin 更简洁。粗略估计显示，代码行数减少约 40%。它也更安全，例如对不可空类型的支持使应用程 序不易发生 NPE。其他功能包括智能类型转换、高阶函数、扩展函数和带接收者的 lambda 表达式，提 供了编写富于表现力的代码的能力以及易于创建 DSL 的能力。

Kotlin 解决了一些 Java 中的问题：

 - 空引用由类型系统控制。
 - 无原始类型
 - Kotlin中数组是不型变的
 - 相对于 Java 的 SAM-转换，Kotlin 有更合适的 函数类型
 - 没有通配符的使用处型变
 - Kotlin没有受检异常

Kotlin 有而 Java 没有的特性：

 - Lambda表达式+内联函数=高性能自定义控制结构 — 扩展函数
 - 空安全
 - 智能类型转换
 - 字符串模板
 - 属性
 - 主构造函数
 - 委托
 - 变量与属性类型的类型推断
 - 单例
 - 声明处型变&类型投影
 - 区间表达式
 - 操作符重载
 - 伴生对象
 - 数据类
 - 分离用于只读与可变集合的接口
 - 协程


## 特性

### 空安全

这个可谓是 Kotlin 的标志性特性。Kotlin 的空安全设计对于声明可为空的参数，在使用时要进行空判断处理，有两种处理方式，字段后加 `!!` 像 Java 一样抛出空异常，另一种字段后加 `?` 可不做处理返回值为 null 或配合 `?:` 做空判断处理

```
//类型后面加?表示可为空
var age: String? = "23" 
//抛出空指针异常
val ages = age!!.toInt()
//不做处理返回 null
val ages1 = age?.toInt()
//age为空返回-1
val ages2 = age?.toInt() ?: -1
```

如果对一个可为空的变量直接使用 `.` 访问的话，编译器会发出错误提示：

```
Only safe (?.) or non-null asserted (!!.) calls are allowed on a nullable receiver of type Context?
```

如何为一个不可为空的变量赋值 `null` 的话，编译器同样会发出错误提示：

```
Null can not be a value of a non-null type String
```

使用 `?` ，声明可空类型，避免抛出 NPE。我们在声明变量、方法参数和返回值时都可以使用。
使用 `!!` 声明非空类型，为空则抛出NPE
三元操作符 `?:`

```
val name:String = student?.name ?: "defaultName"
```

当为空时，输出的是 `?:` 后面的默认字符串

```
fun foo(node: Node): String? {
  val parent = node.getParent() ?: return null
  val name = node.getName() ?: throw IllegalArgumentException("name expected")
  // ……
}
```

### 属性访问语法

Kotlin 提供一个很棒的语法糖叫做「属性访问语法」，它让你可以像访问 Kotlin 属性一样访问 JavaBeans 类型的 getters 和 setters 方法。例如，你可以这样 activity.context 调用 Activity.getContext()，而不用写整个方法名。如果你在 Kotlin 使用传统的方式调用，lint 会给你一个警告告诉你使用「属性调用语法」。
