---
title: Java 基础 -- 深入理解 String
categories:  Java
comments: true
tags: [String]
description: 深入理解 String
date: 2014-1-20 10:00:00
---

## 概述

`String` 类是我们在编程过程中经常用到的类，也是面试过程中经常涉及到的类。本文就我们平时遇到的关于 `String` 类的知识点做个归纳整理。
[String 源码](https://code.csdn.net/hty1053240123/jdk-source/tree/master/java/lang/String.java)

## final 类型

通过查看 `String` 源码我们可以看到，该类是 final 类型的，也即意味着 `String` 类不能被继承。

## 不可变性

`String` 对象一旦被创建就是固定不变的了，对 `String` 对象的任何改变都不影响到原对象。
`String` 对象是使用

```
    /** The value is used for character storage. */
    private final char value[];
```

来存储字符串的，而 `value[]` 是 `final` 类型的，一旦赋值后是无法改变的。我们通过查看源码可以发现，`String` 的 `substring`、`concat`、`replace` 等操作都不是在原有的字符串上进行的，而是重新生成了一个新的字符串对象。也就是说进行这些操作后，最原始的字符串并没有被改变。

## 字符串常量池

JVM 为了减少字符串对象的重复创建,其维护了一个特殊的内存,这段内存被成为字符串常量池。
可以通过下面的方法使用到常量池：

 - 当使用双引号声明 `String` 对象时，JVM 首先会对这个字面量进行检查，如果字符串常量池中存在相同内容的字符串对象的引用，则将这个引用返回，否则新的字符串对象被创建，然后将这个引用放入字符串常量池，并返回该引用。
 - 当使用 `String.intern()` 方法时，会从字符串常量池中查询当前字符串是否存在，如果存在， 就会直接返回当前字符串。如果常量池中没有此字符串， 会将此字符串放入常量池中后，再返回该常量池中的字符串。

在 Jdk6 以及以前的版本中，字符串的常量池是放在堆的Perm区的，Perm区是一个类静态的区域，主要存储一些加载类的信息，常量池，方法片段等内容，默认大小只有4m，一旦常量池中大量使用 `intern`  是会直接产生 `java.lang.OutOfMemoryError:PermGen space` 错误的。
在 jdk7 的版本中，字符串常量池已经从 Perm 区移到正常的 Java Heap 区域了。和普通的对象在同一个存储区域中。
下面来看哥例子：

```
        String str1 = "abc";
        String str2 = "abc";
        System.out.println(str1 == str2);
```

结果是打印了 true。
str1 和 str2 指向了同一个字符串常量池中的对象。

```
        String str1 = new String("abc");
        String str2 = new String("abc");
        System.out.println(str1 == str2);
```

上面的结果时打印了 false。
当使用 new 操作符时，总是会创建新的字符串对象。

## + 号操作符

```
        String str1 = "abcdef";
        String str2 = "abc" + "def";
        System.out.println(str1 == str2);
```

结果是打印了 true。
为什么是这个结果呢？
我们先来看一下字节码信息：

```
    Code:
       0: ldc           #2                  // String abcdef
       2: astore_1
       3: ldc           #2                  // String abcdef
       5: astore_2
       6: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       9: aload_1
      10: aload_2
```

可以看到 str2 在编译期优化成了 `str2 = "abcdef"`。因此和 str1 是字符串常量池中的同一对象。
再来看下面一段代码：

```
        String str1 = "abcdef";
        String abc = "abc";
        String def = "def";
        String str2 = abc + def;
        System.out.println(str1 == str2);
```

结果是打印了 false。
可见，字符串对象的相加和字符串常量的相加是不一样的。
先来看一下字节码：

```
    Code:
       0: ldc           #2                  // String abcdef
       2: astore_1
       3: ldc           #3                  // String abc
       5: astore_2
       6: ldc           #4                  // String def
       8: astore_3
       9: new           #5                  // class java/lang/StringBuilder
      12: dup
      13: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
      16: aload_2
      17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      20: aload_3
      21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      24: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
```

可见，字符串对象的相加实际上也是通过 `StringBuilder` 对象来完成的，首先以最左边的字符串为参数创建 `StringBuilder` 对象，然后依次对右边进行 `append` 操作，最后将 `StringBuilder` 对象通过 `toString()` 方法转换成 `String` 对象。
`String str2 = abc + def` 其实等价于 `String str2 = new StringBuilder().append(abc).append(def).toString()`。
再来看下面的例子：

```
        String e = "e";
        String str2 = "abc"+"def"+e+"h"+"g";
```

对应字节码：

```
    Code:
       0: ldc           #11                 // String e
       2: astore_1
       3: new           #5                  // class java/lang/StringBuilder
       6: dup
       7: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
      10: ldc           #2                  // String abcdef
      12: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      15: aload_1
      16: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      19: ldc           #12                 // String hg
      21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      24: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
```

这也是编译器进行优化的结果，等价于 `String str2 = new StringBuilder().append("abcdef").append(e).append("hg").toString()`。
每次这样的赋值操作，编译器都会创建一个 `StringBuilder` 对象来进行操作。
再来看下面一种情况：

```
        String s = null;
        for(int i = 0; i < 5; i++) {
            s += "a";
        }
```

这种情况下会创建几个 `StringBuilder` 对象呢？
看字节码：

```
    Code:
       0: aconst_null
       1: astore_1
       2: iconst_0
       3: istore_2
       4: iload_2
       5: iconst_5
       6: if_icmpge     35
       9: new           #5                  // class java/lang/StringBuilder
      12: dup
      13: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
      16: aload_1
      17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      20: ldc           #11                 // String a
      22: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      25: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      28: astore_1
      29: iinc          2, 1
      32: goto          4
      35: return
```

每做一次 + 就产生个 `StringBuilder` 对象，然后 `append` 后就扔掉。下次循环再到达时重新产生个 `StringBuilder` 对象，然后 `append` 字符串，如此循环直至结束。一共创建了 5 个`StringBuilder` 对象。
如果我们直接采用 `StringBuilder` 对象进行 `append` 的话，我们可以节省 4 次创建和销毁对象的时间。
所以对于在循环中要进行字符串连接的应用，一般都是用 `StringBuffer` 或 `StringBulider` 对象来进行 `append` 操作。 

## intern() 方法

前面也介绍过：当使用 `String.intern()` 方法时，会从字符串常量池中查询当前字符串是否存在，如果存在， 就会直接返回当前字符串。如果常量池中没有此字符串， 会将此字符串放入常量池中后，再返回该常量池中的字符串。
我们把前面的例子做一下修改：

```
        String str1 = new String("abc").intern();
        String str2 = new String("abc").intern();
        System.out.println(str1 == str2);
```

打印结果为：true。
他们返回的都是字符串常量池中对象的引用。
字符串常量池内部是用 Hashtable 来维护的，如果存放的字符串过多，就会造成Hash冲突严重，从而导致链表会很长，而链表长了后直接会造成的影响就是当调用 `String.intern` 时性能会大幅下降。因此，在一些数据量很大切易变的字符串场景中要谨慎使用，否则会适得其反。

## StringBulider 和 StringBuffer

他们的使用场景前面我们都由介绍，两者的方法基本相同。区别就是 `StringBulider` 是非线程安全的，`StringBuffer` 是线程安全的。


