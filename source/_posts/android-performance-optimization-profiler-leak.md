---
title: Android 性能优化工具篇 -- Android Profiler 分析内存泄漏
categories: Android性能优化
comments: true
tags: [稳定性]
description: 介绍使用 Android Profiler 分析内存泄漏
date: 2018-10-12 10:00:00
---

## 概述

前面一篇文章 [Android 性能优化工具篇 -- MAT分析内存泄漏 ](http://www.heqiangfly.com/2016/02/20/android-performance-optimization-use-mat-to-analyse-leak/) 介绍了如何使用MAT来分析内存泄漏，但是 Android Studio 3.0 之后已经不再使用 Android Device Monitor了，取而代之的是 Android Profiler 工具，这是一款非常强大的工具，可以帮助我们了解应用的 CPU、内存、网络和电池资源使用情况。
本文着重介绍一下如何使用 Android Profiler 来解决 Android 内存泄漏的问题。

[Android 官方文档：利用 Android Profiler 测量应用性能](https://developer.android.com/studio/profile/android-profiler)

## 发现内存泄漏

[Android 性能优化工具篇 -- MAT分析内存泄漏 ](http://www.heqiangfly.com/2016/02/20/android-performance-optimization-use-mat-to-analyse-leak/) 一文中介绍了如何使用 `adb shell dumpsys meminfo <进程名称或者进程ID>` 命令来分析进程中是否存在内存泄漏。

## 工具准备

要打开 Profiler 窗口，请依次选择 View > Tool Windows > Profiler，或点击工具栏中的 Profile 图标 。

<img src="/images/android-performance-optimization-profiler-leak/profiler-button.png" width="528" height="74"/>

点击 Profile 图标可以自动将待分析的应用安装到设备上，也可自行安装，然后在 Profiler 的会话窗口中连接进程。

<img src="/images/android-performance-optimization-profiler-leak/select-process.png" width="2876" height="752"/>

<img src="/images/android-performance-optimization-profiler-leak/info.png" width="2836" height="724"/>

点击MEMORY进入内存详情，在这里可以实时查看内存的占用情况：

<img src="/images/android-performance-optimization-profiler-leak/memory-info.png" width="2834" height="748"/>

## 开始分析

退出我们前面写的 Demo 的 Activity，然后再点击一下 Profiler 面板上面的垃圾桶图标，进行一次垃圾回收，然后再点击垃圾桶图标右侧的图标，这时候 Android Studio 就开始分析进程的内存泄漏情况，并把分析结果呈现出来：

<img src="/images/android-performance-optimization-profiler-leak/arrange.png" width="1986" height="576"/>

前面我们已经操作过了使 Activity 退出了，如果你要分析的对象在这种情况下应该是消失的，但却在这里出现了，就说明你的对象无法被垃圾回收器回收，就发生了内存泄漏。
默认情况下，左侧的分配列表按类名称排列。在列表顶部，您可以使用右侧的下拉列表在以下排列方式之间进行切换：

 - Arrange by class： 基于类名称对所有分配进行分组。
 - Arrange by package： 基于软件包名称对所有分配进行分组。
 - Arrange by callstack： 将所有分配分组到其对应的调用堆栈。

在 HEAP DUMP 一栏中选择 Arrange by package，我们按照包名找到 Activity，发现内存中 MainActivity 实例仍然存在，发生了泄漏。
点击 MainActivity ，右侧出现 Instance View 窗口，点击 Instance View 中的对象，会出现 Reference 窗口，显示对选择对象的引用。

<img src="/images/android-performance-optimization-profiler-leak/result.png" width="1982" height="1158"/>

点击 MainActivity 对象，在 Reference 窗口会出现对它的引用情况，你会发现会有好多对 MainActivity 的引用。
mInnerClassInstance 的引用显示在第一个，对其他的引用一层层的点击下去，最终都是由我们创造的泄漏场景 mInnerClassInstance 对象对一些对象的引用导致的。

<img src="/images/android-performance-optimization-profiler-leak/results-1.png" width="662" height="310"/>

这样我们就最终发现了导致内存泄漏的元凶。

## 解决内存泄漏

