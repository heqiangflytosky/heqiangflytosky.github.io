---
title: Java 内存管理 -- 构造方法
categories: Java
comments: true
tags: [Java]
description: 介绍 java 构造方法的初始化顺序
date: 2014-2-20 10:00:00
---


## 概述

本节将介绍一下构造方法在对象初始化时的执行顺序

## 构造方法调用

当创建任何Java对象时，程序总是先依次调用每个父类的非静态初始化块、父类构造器（总是从Object开始）执行初始化，最后才调用本类的非静态初始化块、构造方法执行初始化。
对非静态初始化块的调用，接着会调用父类的一个或多个构造方法执行初始化，这个调用即可以是通过super进行显示调用，也可以是隐式调用。
当所有的父类的非静态初始化块、构造器依次调用完成后，系统调用本类的非静态初始化块、构造器执行初始化，最后返回本类的实例。
看下一段代码：

```
public class Food {
    {
        Log.e("Test","Food code block");
    }
    public Food() {
        Log.e("Test","Food constructor");
    }
}

public class Fruit extends Food{
    {
        Log.e("Test","Fruit code block");
    }
    public Fruit() {
        Log.e("Test","Fruit constructor");
    }
}

public class Apple extends Fruit {
    {
        Log.e("Test","Apple code block");
    }
    public Apple() {
        Log.e("Test","Apple constructor");
    }
}
```

测试代码：

```
Apple apple = new Apple();
```

这里对父类构造器的调用为隐式调用。
执行步骤：

 1.执行Object类非静态初始化块（如果有的话）。
 2.隐式或显示调用Object类的一个或多个构造器执行初始化
 3.执行Food类非静态初始化块（如果有的话）。
 4.隐式或显示调用Food类的一个或多个构造器执行初始化
 5.执行Fruit类非静态初始化块（如果有的话）。
 6.隐式或显示调用Fruit类的一个或多个构造器执行初始化
 7.执行Apple类非静态初始化块（如果有的话）。
 8.隐式或显示调用Apple类的一个或多个构造器执行初始化

执行结果：

```
E Test    : Food code block
E Test    : Food constructor
E Test    : Fruit code block
E Test    : Fruit constructor
E Test    : Apple code block
E Test    : Apple constructor
```

只要在程序中创建Java对象，系统总是先调用最顶层父类的初始化操作，包括初始化块和构造器，然后依次向下调用所有父类的初始化操作，最终执行本类的初始化操作返回本类的实例。
至于调用父类的那个构造器执行初始化，分为下面几中情况：

 - 子类构造器执行体的第一行代码使用super显示调用父类构造器，系统将根据super调用里传入的实参列表来确定调用父类的哪个构造器。
 - 子类构造器执行体的第一行代码使用this显示调用本类中重载的构造器，系统将根据this调用里传入的实参列表来确定本类的另一个构造器（执行本类中另一个构造器时即进入第一种情况）。
 - 子类构造器执行体中既没有super调用，也没有this调用，系统将会在执行子类构造器之前，隐式调用父类无参数的构造器。

**注意**：super调用用于显示调用父类的构造器，this调用用于显式调用本类中另一个重载的构造器。super和this都只能在构造器中使用，而且super和this都必须作为构造器的第一行代码，否则编译器会报错，通过super调用父类的普通方法则没有限制。因此，构造器中的super和this调用最多只能使用其中之一，而且最多只能调用一次。

## 常见问题

关于构造函数的初始化顺序我们做下面的测试：

```
abstract public class Fruit{
    public Fruit() {
        super();
        Log.e("Test","Fruit constructor");
        setName();
    }

    protected String name;
    abstract public void setName();
    public String getName() {
        return name;
    }
}

public class Apple extends Fruit {
    public Apple() {
        super();
        Log.e("Test","Apple constructor");
    }

    String n = "Apple";
    @Override
    public void setName() {
        name = n;
    }
}

```

调用下面代码：

```
        Apple apple = new  Apple();
        Log.e("Test","name = "+apple.getName());
```

发现打印 name 为null。
因此抽象父类构造函数调用抽象方法 setName 时，调用的是子类 Apple 的 setName 方法，此时子类的变量 n 还没有初始化，因此 name 为 null。

## 参考

《深入理解 Java 虚拟机》
《疯狂Java-突破程序员基本功的16课》