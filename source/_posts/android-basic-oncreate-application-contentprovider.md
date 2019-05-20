---
title: Android -- Application 和 ContentProvider 的 onCreate() 方法的执行顺序
categories: Android
comments: true
tags: [Android]
description: Application 和 ContentProvider 的 onCreate() 方法的执行顺序
date: 2018-8-6 10:00:00
---

## 概述

在上个版本对客户端代码进行了一些重构，结果线上发现了一些 Crash 问题。
问题是 ContentProvider 里面用到的一些变量的空指针异常。
问题是这样的，ContentProvider 增删改查接口用到的一些数据在 Application 里面做了初始化，结果偶现空指针异常。
看到这个问题，首先想到的是初始化时机的问题，顺便看看源码，了解问题的真相。

## 源码分析

这里先给出源码分析的结论：ContentProvider 的 onCreate() 方法先于 Application 的 onCreate() 方法执行。 

```
├── ActivityThread.handleBindApplication
    ├── LoadedApk.makeApplication
        ├── Instrumentation.newApplication
            ├── Instrumentation.newApplication
                ├── (Application)clazz.newInstance() //调用构造方法
                ├── Application.attach
                    ├── Application.attachBaseContext //调用attachBaseContext方法
    ├── ActivityThread.installContentProviders
        ├── ActivityThread.installProvider
            ├── ContentProvider.attachInfo
                ├── ContentProvider.attachInfo
                    ├── ContentProvider.onCreate  // 调用 ContentProvider的onCreate方法
    ├── Instrumentation.callApplicationOnCreate
        ├── Application.onCreate  //调用Application的onCreate方法
```


## 结论

调用顺序：
Application 构造方法 --> Application.attachBaseContext --> ContentProvider.onCreate --> Application.onCreate --> Activity.onCreate

ContentProvider.onCreate 方法先于 Application.onCreate，因此在调用 ContentProvider.query 方法调用时，Application.onCreate 方法还没有调用导致变量未初始化从而发生崩溃。
