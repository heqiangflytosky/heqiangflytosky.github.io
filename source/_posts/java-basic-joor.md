---
title: Java 基础 -- 使用 JOOR 实现反射 
categories: Java
comments: true
tags: [Java]
description: 介绍JOOR的用法
date: 2017-1-20 10:00:00
---

## 概述

在前面的博客 [Java 基础 -- 反射 ](http://www.heqiangfly.com/2014/11/02/java-basic-reflect/) 列举了Java反射的一些使用，下面介绍开源反射工具JOOR的使用。
JOOR 封装了反射操作，实现了流式调用，只需几行代码就实现了前面博客中内部类的反射。
[GitHub 地址](https://github.com/jOOQ/jOOR)


## 使用方法

gradle中配置：

```
compile 'org.jooq:joor:0.9.5'
```

```
    public void testJoor(){
        Reflect.on("com.example.heqiang.testsomething.OuterClass")
                .create()
                .field("mInnerClass")
                .call("printInt");

        OuterClass outerClass = new OuterClass();
        Reflect.on(outerClass).field("mInnerClass").call("printInt");
        Reflect.on(outerClass).field("mInnerClass").set("mInt", 8);
        Reflect.on(outerClass).field("mInnerClass").call("printInt");
        
        Reflect.on(object).call("testMethod",1);
    }
```

```
String world = on("java.lang.String")  // Like Class.forName()
                .create("Hello World") // Call most specific matching constructor
                .call("substring", 6)  // Call most specific matching substring() method
                .call("toString")      // Call toString()
                .get();                // Get the wrapped object, in this case a String
```

## 代码解释

Reflect 提供了三个变量：classCache、classCache和classCache分别来做缓存使用。
静态方法：

 - on(String name)：根据类名创建 Reflect 对象。
 - on(String name, ClassLoader classLoader)：根据类名和ClassLoader创建 Reflect 对象。
 - on(Class<?> clazz)：根据Class创建 Reflect 对象。
 - on(Object object)：根据一个对象创建 Reflect 对象。
 - accessible(@NonNull T accessible)

on 方法都返回一个 Reflect 对象。

方法：

 - get()：获取一个包装对象
 - set(String name, Object value)：为某个Field设置值
 - get(String name)：获取某个Field的值
 - field(String name)：获取某个Field
 - fields()：获取所有Field
 - call(String name)：执行某个Method
 - call(String name, Object... args)：执行某个Method，可以添加参数
 - create()：调用构造方法
 - create(Object... args)：调用带参数的构造方法
 - `as(Class<P> proxyType)`


## 异常处理

JOOR 把反射相关的异常都统一转成了 ReflectException ，这个异常继承自 RuntimeException，不是强制捕获的异常，因此我们写代码时一定要注意对这个异常的处理。