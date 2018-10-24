---
title: Android 多线程 -- ConditionVariable 用法
categories:  Android 线程
comments: true
tags: [ConditionVariable]
description: 介绍 ConditionVariable 的用法
date: 2017-7-10 10:00:00
---

## 概述

在 Java 开发中，如果要进行线程同步操作，经常会用到 `wait` 和 `notify`，但用起来可能会有点繁琐，于是在 Android 中就把它包装了一下，以 `ConditionVariable` 类的形式带给开发者使用。

## 简介

`ConditionVariable` 用一个 `boolean mCondition` 变量来作为状态控制 `wait` 和 `notify` 方法的执行。
`ConditionVariable` 类提供了下面几个方法：

 - ConditionVariable()：构造方法
 - ConditionVariable(boolean state)：根据给定的状态构造对象
 - open()：打开 `mCondition` 状态，并释放所有被阻塞的线程。任何线程在调用 `open()` 后调用 `block()` 都不会生效，除非先调用了 `close()` ，后调用 `block()`。 
 - close()：重置 `mCondition` 为关闭状态。后面任何调用 `block()` 方法的线程将会被阻塞，直到调用了  `open()` 方法。
 - block()：阻塞当前线程直到调用 `open()` 方法。如果当前的 `mCondition` 为打开状态，则会立即返回，不会阻塞线程。
 - boolean block(long timeout)：阻塞当前线程直到调用 `open()` 方法或者是设置的超时时间到。如果当前的 `mCondition` 为打开状态，则会立即返回，不会阻塞线程。
  - 如果返回 true，表示 `mCondition` 为打开状态。
  - 如果返回 false，表示超时返回。
 

