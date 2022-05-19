---
title: Java 设计模式 -- 单例模式的几种实现
categories: 设计模式
comments: true
tags: [设计模式]
description: 介绍单例模式的几种实现
date: 2014-11-5 10:00:00
---

## 概述

单例模式（Singleton）我们平时经常用到，本文总结一下单例模式几种常见的实现。

 - 饿汉模式
 - 懒汉模式
 - 双重锁懒汉模式
 - 静态内部类模式
 - 枚举模式

## 饿汉模式

```
public class Singleton {
    private static Singleton INSTANCE = new Singleton();
    private Singleton(){}
    public static Singleton getInstance(){
        return INSTANCE;
    }
}
```

我们都知道，静态代码是在类初始化时就被初始化的，饿汉模式在类被初始化时在内存中创建了对象，因此是线程安全的。
为什么呢？在《深入理解Java虚拟机》一文中有这样的介绍：
类的初始化过程就是执行类构造器`<clinit>()`的过程。
虚拟机会保证一个类的`<clinit>()`方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只有一个线程去执行这个类的`<clinit>()`方法，其他线程都需要阻塞等待，直到活动线程执行`<clinit>()`方法完毕。
饿汉模式无法向构造函数传递参数。
如果要传递非构造函数的参数，参考 静态内部类模式 这一节的方式。

## 懒汉模式

```
public class Singleton {
    private static Singleton INSTANCE = null;
    private Singleton(){}
    public static Singleton getInstance(){
        if (INSTANCE == null) {
            INSTANCE = new Singleton();
        }
        return INSTANCE;
    }
}
```

懒汉模式在方法第一次调用时创建单例对象，多线程环境下存在线程安全问题。

## 双重锁懒汉模式

```
public class Singleton {
    private static volatile Singleton INSTANCE = null;
    private Singleton(){}
    public static Singleton getInstance(){
        if (INSTANCE == null) {
            synchronized(Singleton.class){
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
```

双重锁懒汉模式在方法第一次调用时创建单例对象，上面的写法是线程安全的，但是一定要记得 INSTANCE 要加 volatile 禁止指令重排序，否则会有线程安全的问题。
第一次判断 INSTANCE == null为了避免非必要加锁，在实例化时 Singleton 时采取加锁，优化性能。

## 静态内部类模式

```
public class Singleton {
    private Singleton(){}
    private static class SingletonHolder{
        private static Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance(){
        return SingletonHolder.INSTANCE;
    }
}
```

静态内部类的优点是：外部类加载时并不需要立即加载内部类，内部类不被加载则不去初始化INSTANCE，故而不占内存。
但我们第一次调用 getInstance 方法时，虚拟机才会去初始化 SingletonHolder 类，另外，我们在饿汉模式里面介绍的 `<clinit>()`方法的执行时线程安全的。
因此，单例对象也是在第一次调用的时候去初始化的，而且是线程安全的。
静态内部类有一个问题就是传参问题：由于是静态内部类的形式去创建单例的，故外部无法传递参数到构造函数。
如果需要向构造函数传递参数，可以选择双重锁懒汉模式。
如果不是向构造函数传递参数，可以参考下面这款：

```
public class Singleton {
    private static String mStr;
    private Singleton(){}
    private static class SingletonHolder{
        private static Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance(String s){
        mStr = s;
        return SingletonHolder.INSTANCE;
    }
}
```

## 枚举模式

```
public enum Singleton {
    INSTANCE;
    Singleton() {
    }
    public void testMethod() {
        Log.e("Test","enum Singleton test mothod");
    }
}
```

枚举模式也是线程安全的。
枚举类型还可以防止反序列化创建新对象对单例模式的破坏。