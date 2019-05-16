---
title: Android 性能优化工具篇 -- TraceView的使用
categories: Android性能优化
comments: true
tags: [Android性能优化, 开发工具]
description: 简单介绍TraceView工具的使用
date: 2016-2-25 10:00:00
---

## TraceView简介

工欲善其事，必先利其器。要想分析Android的性能问题，比如卡顿了之类的，那么就必需掌握TraceView工具的使用。
TraceView 是 Android SDK 中内置的一个工具，它可以加载 trace 文件，用图形的形式展示代码的执行时间、次数及调用栈，便于我们分析，以此来优化 App 运行效率。

## 生成trace文件

在进行分析以前，必需要生成trace文件，可以用下面的三种方法生成trace文件：

### 插入代码生成trace文件

```
Debug.startMethodTracing("test.trace")
……
Debug.stopMethodTracing();
```

然后把 trace 文件从手机导出来就可以进行分析了，这种方法的优点是可以精确的控制追踪的起点和终点，缺点是步骤繁琐了一些。

### 使用Android Monitor

使用Android Studio 内置的 Android Monitor 也可以很方便的生成 trace 文件到电脑。
在 CPU 监控的那栏会有一个类似秒表的按钮，未启动应用时是灰色，不可点击，应用启动后变成可以点击状态，点击之后开始追踪，再次点击结束追踪，并生成后缀为.trace的 tarce 文件。

![图片](/images/development-tool-traceview/android_monitor_start.png)

### 使用DDMS

DDMS 是 Android 调试监控工具，用来分析 trace 文件更为直观，菜单栏点击`Tools`->`Android`->`Android Device Monitor`，便可以打开DDMS，如图：

![图片](/images/development-tool-traceview/ddms_start.png)

Android Studio 3.0之后把 DDMS 从 Tools 里面去掉了，但是我们可以通过命令行打开：android-sdk/tools/monitor。

我们可以用 DDMS 打开前面方法生成的 trace 文件，`File` -> `Open File`，然后选择文件即可。
或者也可以生成 trace 文件。
选择需要调试的进程，在上面点击`Start Mothod Profiling`按钮：

![图片](/images/development-tool-traceview/trace_start.png)

此时会弹出一个选项框：

![图片](/images/development-tool-traceview/trace_profile_options.png)

 - Sample base profiling：抽样监听，以指定的频率进行抽样调查，一般不要超过5s，需要较长时间获取准确的样本数据。
 - Trace base profiling：整体监听，项目中所有方法都会监听，资源消耗比较大。

点击OK后，边开始了监听，再次点击mothod profiling按钮，结束监听。

![图片](/images/development-tool-traceview/trace_stop.png)

DDMS会打开trace文件，此时就可以对trace文件进行分析了。

## trace文件分析

如图：

![图片](/images/development-tool-traceview/trace_analysis.png)

通过界面图我们可以看到整个界面可以分为上下两个部分，上面是你测试的进程中每个线程的执行情况，每个线程占一行，下面是每个方法执行的各个指标的值。
上面一部分是测试进程的中每个线程运行的时间线，x轴表示时间，色块区域可放大，每个区域代表每个方法的执行时间，不同的颜色代表不同的方法，颜色长度代表占用时间。y轴表示每一个独立线程。
下面一部分Name为所选择的颜色区块所代表的性能分析。

属性介绍：

 - Incl cpu time：某方法占用CPU总时间（父+子）。
 - Excl cpu time：某方法本身占用cpu时间（父），即总时间减去子方法的时间。
 - Incl Real time：某方法真正执行总时间（父+子）。
 - Excl Real time：某方法自身执行时间（父），即总时间减去子方法的时间。
 - Calls+RecurCall： 调用次数+递归调用次数。
 - Cpu time/Call：平均每次调用占用CPU时间。
 - Real time/Call ：平均每次调用所执行的时间。

点击某一个方法，可以看到它的详细信息：

 - Parents：选中方法的调用处
 - Children：选中方法调用的方法

打开每个方法，会显示Paents和Children(即父方法和子方法)，以及分别所占用时间。
根据上图，很轻易的就找到了`createBitmp`方法占用了主线程中大量的时间，是可以优化的对象。

一些分析 trace 文件的方法：

 - 从上半部分图中先直观的分析那些函数运行的时间较长，点击色块区域下面会自动展开该函数。
 - 点击 TraceView 中的 Cpu Time/Call，按照占用 CPU 时间从高到低排序
 - 哪些方法调用次数非常频繁
 - 点击 TraceView 中的 Calls + Recur Calls/Total ，按照调用次数从高到底排序
