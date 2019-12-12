---
title: Java 基础 -- 深入理解 try catch finally
categories: Java
comments: true
tags: [Java]
description: 介绍 try catch finally 在一些特殊情况下的执行顺序
date: 2014-11-8 10:00:00
---

                
## 概述

try catch块是我们编程过程中经常用到的，也是面试时经常遇到的，会问到一些执行顺序方面的问题，本文就将这些问题汇总并用实例验证解答。

## 几个知识点

### catch 的写法

当 try 块中需要 catch 多个异常时，可以采用下面的写法：

```
        try {
            a();
            b();
        }catch (NullPointerException | ClassNotFoundException |IOException e) {
            Log.e("Test","error",e);
        }
```

和常规写法是等同的：

```
        try {
            a();
            b();
        }catch (NullPointerException e) {
            Log.e("Test","error",e);
        } catch (ClassNotFoundException e) {
            Log.e("Test","error",e);
        } catch (IOException e) {
            Log.e("Test","error",e);
        }
```

### 日志

在 Android 中，异常日志最好使用下面方式打印：

```
Log.e("Test","error",e);
```
比传统的 

```
e.printStackTrace()
```
占用内存更少，更具有易读性。

但第一种写法更美观，简介。

## 问题描述

 1. 当try中没有出现异常时，finally块中代码是否会执行？
 2. 当try或者catch中有return时，finally是否会执行、以及执行顺序如何？
 3. 如果try或者catch中有return时，finally也有return时，会执行哪个return。


## 问题解答

### 问题1

这个问题比较容易解答，无论 try 中是否出现异常，finally 块中的代码都会执行。
finally中常用来用清尾工作。

### 问题2

现在来讨论一下当当try或者catch中有return时，finally是否会执行、以及执行顺序的问题。

```
    private int testTry() {
        try{
            return 1;
        } catch (Exception e) {

        } finally {
            Log.e("Test","finally");
        }
        return 0;
    }
```

通过代码测试，finally 中的打印是可以出现的。

下面来讨论一下finally是否对return值有影响的问题。

```
    private int testTry() {
        int a = 20;
        try{
            return a;
        } catch (Exception e) {

        } finally {
            Log.e("Test","finally");
            a = 30;
        }
        return 0;
    }
```

返回结果 20；

### 问题3

再来看一下当try或者catch中有return时，finally也有return时，会执行哪个return的问题。

```

    private int testTry() {
        int a = 20;
        try{
            return a;
        } catch (Exception e) {

        } finally {
            Log.e("Test","finally");
            a = 30;
            return a;
        }
    }
```

返回30；

上面介绍的是在finally中改变返回结果的情况，那如果改变返回结果的属性呢？会不会生效呢？

```
    private ArrayList testTry2() {
        ArrayList list = new ArrayList();
        try{
            return list;
        } catch (Exception e) {
            Log.e("Test","Exception");
            return null;
        } finally {
            Log.e("Test","finally");
            list.add("a");
        }
    }
```

这种情况下，finally里面对list的改变是起作用的，这时候打印返回值的size()，结果是1。
如果是返回引用类型的情况下修改返回结果同样也是不生效的。

```
    private ArrayList testTry2() {
        ArrayList list = new ArrayList();
        try{
            list.add("a");
            return list;
        } catch (Exception e) {
            Log.e("Test","Exception");
            return null;
        } finally {
            Log.e("Test","finally");
            list = new ArrayList();

        }
    }
```
这时候打印返回值的size()，结果是1。

当 catch 和 finally 都有返回时，看看执行情况。

```
    private int testTry() {
        int a = 20;
        try{
            a = a/0;
            return a;
        } catch (Exception e) {
            Log.e("Test","Exception");
            return 0;
        } finally {
            Log.e("Test","finally");
            a = 30;
            return a;
        }
    }
```

```
E/Test: Exception
E/Test: finally
E/Test: result = 30
```

返回30；