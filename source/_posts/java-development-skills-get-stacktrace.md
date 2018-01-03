---
title: Java 开发技巧 -- 获取 Java 的调用栈
categories: Java
comments: true
tags: [Java]
description: 本文介绍获取 Java 方法的调用栈的实现
date: 2016-11-1 10:00:00
---

## 概述

`StackTrace` 存放的就是方法调用栈的信息，异常处理中常用的 `printStackTrace()` 实质就是打印异常调用的堆栈信息。
有时候我们需要获取某个方法的调用栈信息，或者是根据某个方法调用的不同时机做一些不同的处理，那么就需要获取方法的调用栈。
本文介绍了一种获取 Java 方法的调用栈的实现。

## 实现

`StackTraceElement` 表示 StackTrace 中的一个方法对象，属性包括方法的类名、方法名、文件名以及调用的行数。
该类提供的方法如下：

```java
public final class StackTraceElement implements Serializable {
    String declaringClass;
    String methodName;
    String fileName;
    int lineNumber;
    @Override
    public boolean equals(Object obj) {}
    public String getClassName() {}
    public String getFileName() {}
    public int getLineNumber() {}
    public String getMethodName() {}
    public int hashCode() {}
    public boolean isNativeMethod() {}
    public String toString() {}
}
```

获取 `StackTraceElement` 的方法有两种，均返回 `StackTraceElement`数组，也就是这个栈的信息。

 - Thread.currentThread().getStackTrace()
 - new Throwable().getStackTrace()

```java
    private void getStackTrace() {
        StackTraceElement[] stackTrace = Thread.currentThread().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            String className = stackTraceElement.getClassName();
            String methodName = stackTraceElement.getMethodName();
            Log.e("Test","className = "+className+", methodName = "+methodName);
        }
    }
```

获取到调用的类名、方法名和文件名之后我们就可以做一些处理，比如日志打印、或者是在某个特殊类调用的时候做一些特殊设置。
我就是通过这中方法对 Android 中 `Context` 的 `getPackageName()` 方法做了一些特殊处理。
http://blog.csdn.net/lmj623565791/article/details/52506545 这篇文章通 `StackTraceElement` 过做了一些对Log的封装。

## 参考文章

http://blog.csdn.net/hp910315/article/details/52702199
http://blog.csdn.net/lmj623565791/article/details/52506545
