---
title: Java 内存管理 -- 变量的内存管理
categories: Java
comments: true
tags: [Java]
description: 介绍 java 的实例变量和静态变量的内存管理
date: 2014-2-18 10:00:00
---

## 概述

本节将主要介绍Java中的内存分配机制中对变量内存分配的控制。

## 变量分类

Java 程序的变量大体可以分为局部变量和成员变量。
局部变量可以分为下面3类：

 - 形参：在方法定义时的参数，由方法调用者负责为其赋值，随方法的结束而消亡
 - 方法内局部变量：在方法内定义的局部变量，必须在方法内对其进行显示初始化。这种类型的局部变量从初始化完成后开始生效，Java随方法的结束而消亡。
 - 代码块内局部变量：在代码块内定义的局部变量，必须在代码块内对其进行显式初始化。这种类型的局部变量从初始化完成后开始生效，随代码块的结束而消失。

局部变量的作用时间很短暂，它们都被存储在方法的栈内存中。

类体内定义的变量被称为成员变量。如果该变量被static修饰，该成员变量被称为静态变量或者类变量。如果没有被static修饰，该成员变量被称为非静态变量或者实例变量。
static只能修饰类里的成员，不能修饰外部类，不能修饰局部变量，局部内部类。

## 变量定义的顺序

Java 类里定义成员变量时没有先后顺序，但是实际上Java要求定义成员变量时采用合法的前向引用。

```
public class Test {
    int num1 = num2 + 1;
    int num2 = 2;
}
```

上面的代码定义num1成员变量的初始值时，需要根据num2变量的值进行计算。上面的写法编译器将会提示“非法的前向引用”错误。
另外两个静态变量也不允许采用这种非法的前向引用。

```
public class Test {
    static int num1 = num2 + 1;
    static int num2 = 2;
}
```

上面写法也是不合法的。
但是，如果一个是实例变量，一个是静态变量，实例变量总是可以引用静态变量

```
public class Test {
    int num1 = num2 + 1;
    static int num2 = 2;
}
```

上面这种写法是合法的。因为静态变量num2的初始化时间总是处于实例变量的初始化时机之前。所以虽然先定义num1，再定义num2，但是num2的初始化时间总是在num1之前的，因此num1变量的初始化可根据num2的值计算得到。

## 变量的属性

使用static修饰的成员变量是类变量，属于该类本身；没有使用static修饰的成员变量是实例变量，属于该类的实例。
在同一个JVM内，每个类只对应一个Class对象，但每个类可以创建多个Java对象。
由于同一个JVM内每个类只对应一个Class对象，因此同一个JVM内的一个类的静态变量只需一块内存空间；但对于实例变量而言，该类每创建一次实例，就需要为实例变量分配一块内存空间。
Java 还允许通过类的实例变量来访问类的静态变量。其实底层依然会转换为通过类来访问静态变量。操作效果和通过类访问静态变量是一样的。
大部分时候我们会把类和对象严格地区分开，但从另一个角度来看，类也是对象，所有的类都是Class的实例。每个类初始化完成之后，系统都会为该类创建一个对应的Class实例，程序可以通过反射来获取某个类对应的Class实例。

## 实例变量的初始化时机

对于实例变量而言，它属于Java对象本身，每次程序创建Java对象时都需要为实例变量分配内存空间，并执行初始化。
我们可以通过下面的方法对实例变量进行初始化：

 - 定义实例变量时指定初始值
 - 非静态初始化块中对实例变量指定初始值
 - 构造器中对实例变量指定初始值

第1，2种方式比第3种方式更早执行，但是第1，2种方式的执行顺序与它们在源程序中的排列顺序相同。

```
public class Price {
    {
        Log.e("Test","non static block");
        initPrice = 22;
    }
    double initPrice = 20;

    public Price() {
        Log.e("Test","Price Construct");
    }
}
```

输出结果：

```
E/Test: non static block
E/Test: Price Construct
E/Test: 20.0
```

定义实例变量时指定的初始值、初始化块中为实例变量指定初始值的语句的地位是平等的，当经过编译器处理后，它们都将被提取到构造器中。
对于类定义中的语句：

```
double initPrice = 20;
```

实际上被分为如下2次执行。

 1.double initPrice；：创建Java对象时系统根据该语句为该对象分配内存。
 2.initPrice = 20;：这条语句将会被提取到Java类的构造器中执行。
 
## 静态变量的初始化时机

静态变量属于Java类本身，只有当程序初始化该Java类时才会为该类的静态变量分配内存空间，并执行初始化。
程序可以在2个地方对静态变量执行初始化：

 - 定义静态变量时指定初始值
 - 静态初始化块中对静态变量指定初始值

这两种方式的执行顺序与它们在源程序中的排列顺序相同。

```
public class Price {
    static {
        Log.e("Test","non static block");
        initPrice = 22;
    }
    static double initPrice = 20;

    public Price() {
        Log.e("Test","Price Construct");
    }
}
```

上面的代码如果调用 `Price.initPrice`，结果会是 20；
每次运行程序时，系统会将对 Price 类执行初始化：先为所有静态变量分配内存空间，再按源代码中的排列顺序执行静态初始化块所指定的初始值和定义静态变量时所指定的初始值。

再看下面一个例子：

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

下面是测试代码：

```
        Log.e("Test",""+Price.INSTANCE.currentPrice);
        Log.e("Test",""+new Price(2.8).currentPrice);
```

结果会输出：-2.8和17.2。
下面我们从内存角度来分析一下为什么会有这个结果：第一次使用到 Price 这个类时，程序开始对 Price 进行初始化，初始化有下面两个阶段：

 1.系统为Price的两个静态变量分配内存空间
 2.按初始化代码的排列顺序对静态变量进行初始化

初始化的第一阶段，系统先为INSTANCE、initPrice 两个静态变量分配内存空间，此时 INSTANCE、initPrice 的值默认为 null和0.0。
接着进行第二阶段，程序按照顺序依次为 INSTANCE、initPrice 进行赋值。对INSTANCE赋值时要调用Price(2.8)，创建Price实例，此时立即对currentPrice 进行赋值，此时 initPrice 静态变量为0.0，因此赋值的结果是 currentPrice 等于 -2.8。接着，程序再次将initPrice赋值为20，但此时对 INSTANCE 的currentPrice实例变量已经不起作用了。
当 Price 类初始化完成后，INSTANCE 静态变量应用到一个currentPrice为-2.8 的Price实例，而initPrice静态变量为20，当再次创建Price实例时，该Price实例的currentPrice实例变量的值才等于20.0-discount。
如果想当前写法都输出17.2，可以把initPrice静态变量用final修饰。具体原因参考[Java 基础 -- 深入理解 final 修饰符 ](http://www.heqiangfly.com/2014/02/18/java-basic-final/)。

## 参考

《深入理解 Java 虚拟机》
《疯狂Java-突破程序员基本功的16课》