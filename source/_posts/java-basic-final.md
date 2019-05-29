---
title: Java 基础 -- 深入理解 final 修饰符
categories: Java
comments: true
tags: [Java]
description: 介绍 final 修饰符
date: 2014-2-24 10:00:00
---

## 概述

final 修饰符也是我们经常用到的一个简单修饰符，final修饰符的基本作用如下：

 - final 可以修饰变量，被final修饰的变量被赋值后，不能对它重新赋值
 - final 可以修饰方法，被final修饰的方法不能被重写
 - final 可以修饰类，被final修饰的类不能派生子类

但是，仅仅了解这些对于真正掌握final修饰符的用法是远远不够的，本文将尝试深入地了解final修饰符。

## final 修饰变量的初始化

### final 实例变量

被 final 修饰的实例变量必须显示指定初始值，而且只能在如下3个位置指定初始值。

 - 定义final实例变量时指定初始值
 - 在非静态初始化块中为final实例变量指定初始值
 - 在构造器中为final实例变量指定初始值

对于普通的实例变量，Java程序可以对它执行默认的初始化，也就是在初始类时JVM会将实例变量的值指定为默认的初始值。但对于final实例变量，则必须有程序员显示地指定初始值。

**温馨提示：**局部变量也必须显示地初始化。

看代码实现：

```
public class Test {
    // 定义final实例变量时赋初始值
    public final String str1 = "str1";
    public final String str2;
    public final String str3;

    {
        // 在初始化块中赋初始值
        str2 = "str2";
    }

    public Test(){
        // 在构造方法中赋初始值
        str3 = "str3";
    }
}
```

这里需要指出的是，其他经过编译器的处理，这3中方式都会被抽取到构造方法中进行赋初始值操作。
我们可以使用 javap 命令工具类分析一下该代码：
`javap -c Test.calss`

```
Compiled from "Test.java"
public class com.example.heqiang.testsomething.commontest.Test {
  public final java.lang.String str1;

  public final java.lang.String str2;

  public final java.lang.String str3;
  // 构造方法代码
  public com.example.heqiang.testsomething.commontest.Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: ldc           #2                  // String str1
       7: putfield      #3                  // Field str1:Ljava/lang/String;
      10: aload_0
      11: ldc           #4                  // String str2
      13: putfield      #5                  // Field str2:Ljava/lang/String;
      16: aload_0
      17: ldc           #6                  // String str3
      19: putfield      #7                  // Field str3:Ljava/lang/String;
      22: return
}
```

### final 类变量（静态变量）

对于类变量也就是静态变量而言，同样也必须显式指定初始值，不同的是final静态变量只能在两个地方指定初始值：

 - 定义final静态变量时指定初始值
 - 在静态初始化代码块中为final类变量指定初始值

```
public class Test {
    // 定义final静态变量时赋初始值
    public final static String str1 = "str1";
    public final static String str2;

    static {
        // 在静态初始化块中赋初始值
        str2 = "str2";
    }

    public Test(){
    }
}
```

### final 局部变量

Java 本来就要求普通的局部变量必须显式地赋初始值，final 局部变量一样也需要显式地赋初始值。与普通变量不同的是：final 被赋初始值后，再也不能对final局部变量进行赋值。

## 执行 “宏替换”

对于一个final变量而言，如果在定义final变量时指定了初始值，而且该初始值可以在编译时被确定下来，那么这个final变量本质已经不再是一个变量，而是一个直接量，编译器会在编译时做优化处理，在变量出现的地方直接用该初始值替换变量。类似宏的效果。
final 修饰符的一个重要用途就是定义“宏变量”。但是需要有两个条件：

 - 在定义final变量时指定了初始值
 - 该初始值可在编译时被确定下来

举例：

```
public class Test {
    final String str1 = "str1";

    public String getString(){
        return str1;
    }
}
```

这段代码其实已经被编译器优化成了：

```
public class Test {
    public String getString(){
        return "str1";
    }
}
```

我们写两个例子验证一下这个结论：

```
public class Price {
    final static Price INSTANCE = new Price(2.8);
    static double initPrice = 20;

    double currentPrice;
    public Price(double discount) {
        currentPrice = initPrice - discount;
    }
}
```

测试代码：

```
Log.e("Test","" + Price.INSTANCE.currentPrice);
```

这里输出：-2.8
原理在前面文章已经讲过。
如果这里把 initPrice 换成 final 修饰的，会输出什么结果呢？
答案是 17.2 。
原因就是这里执行了替换。把 initPrice 直接替换成了 20，就不会有初始化顺序引起的问题了。
下面通过反射来验证一下：

```
public class TestA {
    private final String mTestFinal = "testFinal";

    public String getFinalString() {
        return mTestFinal;
    }
}
```

下面是测试代码：

```
        TestA a = new TestA();
        try {
            Field field = TestA.class.getDeclaredField("mTestFinal");
            field.setAccessible(true);
            Object str = field.get(a);
            Log.e("Test","mTestFinal = "+str);
            Method method = TestA.class.getDeclaredMethod("getFinalString");
            Object ret = method.invoke(a);
            Log.e("Test","ret = "+ret);
            field.set(a, "test");
            str = field.get(a);
            Log.e("Test","mTestFinal = "+str);
            ret = method.invoke(a);
            Log.e("Test","ret = "+ret);
        } catch (Exception e) {
            e.printStackTrace();
        }
```

输出结果：

```
Test    : mTestFinal = testFinal
Test    : ret = testFinal
Test    : mTestFinal = test
Test    : ret = testFinal
```

可见修改 mTestFinal 变量的值，并没有改变 getFinalString 方法的返回值。
原因就是由于 mTestFinal 是final变量，getFinalString 已经被优化成：

```
    public String getFinalString() {
        return "testFinal";
    }
```

如果把 final 修饰符去掉，结果为：

```
Test    : mTestFinal = testFinal
Test    : ret = testFinal
Test    : mTestFinal = test
Test    : ret = test
```

下面再介绍一下其他情形的宏替换情况：

```
    // 会替换，编译器会优化成 "testFinalt"
    private final String mTestFinal = "testFinal"+"t";

    // 会替换，基本的算术运算
    private final int mTestFinal = 8+1;

    // 不会替换
    private final String mTestFinal = new String("testFinal");

    // 不会替换，调用了valueOf方法，无法在编译时确定
    private final String mTestFinal = String.valueOf("testFinal");

    // 不会替换，因为没有在定义时初始化，
    // 同样在构造方法中初始化也是不会替换的
    private final int mTestFinal;
    {
        mTestFinal = 9;
    }
```

我们在[Java 基础 -- 深入理解 String ](http://www.heqiangfly.com/2014/01/20/java-basic-knowledge-about-string/)一文中介绍过下面的例子：

```
String str1 = "abcdef";
String abc = "abc";
String def = "def";
String str2 = abc + def;
System.out.println(str1 == str2);
```

这个打印输出为false，如何才能输出true呢？

把abc和def改成final类型就行了，会执行宏替换，str2 仍然相当于执行 `"abc"+"def"`。

## 内部类中的局部变量

我们知道，如果程序需要在匿名内部类和局部内部类中使用局部变量，那么这个局部变量必须使用final修饰。
那么原因是什么呢？
对于普通局部变量而言，它的作用域就是停留在该方法内，当方法执行结束，该局部变量也随之消失；但是内部类则可能产生隐式的闭包，闭包使得局部变量脱离它所在的方法继续存在。比如创建一个局部 Thread 变量里面使用局部变量。也就是内部类会扩大局部变量的作用域，如果不被final修饰的话，该变量的值可能会被随意的改变，那么就会引起混乱，因此Java编译器要求被内部类访问的局部变量必须使用final修饰。