---
title: Android -- Launcher3 简介
categories:  Android Launcher
comments: true
tags: [Launcher3]
description: Launcher3 简介
date: 2021-1-2 10:00:00
---

## 概述

Android Launcher 就是我们系统的桌面，它和普通应用的区别就是多配置了Category
的`android.intent.category.HOME` 属性，当Android开机启动成功以后框架层会尝试启动包含上面属性配置的Activity，这样被启动的那个Activity就成了桌面。当我们按下设备的Home键时也会触发包含该属性的Activity。只不过当系统中只存在一个包含该属性的应用时，无论开机还是Home键触发都只会自动启动默认的；当存在多个时无论哪种触发都会弹出选择框进行选择设置。
那么从本文开始就用一系列的文章来简单的分析一下 Launcher3 的代码逻辑。代码时基于 Android R进行分析的。


## 代码结构

## 重要的类

Launcher:第三方桌面（不带多任务）多使用这个作为主界面Activity。
QuickstepLauncher：系统桌面主界面Activity，带多任务功能的Activity，桌面应用中最核心的 Activity，继承自 Launcher。
SecondaryDisplayLauncher：Launcher中的一个新的组件，在启动时会启动一个新的Android桌面界面。当用户连接到更大的显示屏或者在大屏幕中使用时，系统便会呈现一个横向的平板电脑界面，允许用户通过类似于Windows的多任务来使用设备。
RecentsActivity：普通模式下的多任务是和桌面以前显示在 QuickstepLauncher 中，此处这个独立的多任务 Activity 是为比如简易模式下启动多任务而设置的。
Fallback类，这是一些类的统称，当桌面是第三方桌面时由这个些类辅助处理一些功能，比如在简易桌面模式下。
RecentsView:承载多任务卡片的类，它有两个子类：LauncherRecentsView默认Launcher中的多任务，FallbackRecentsView在第三方桌面里面显示多任务。
OverviewComponentObserver 观察多任务组件的变化。当切换桌面时，会调用 TouchInteractionService.onConfigurationChanged 和 OverviewComponentObserver.updateOverviewTargets()


## 文章推荐

https://ericchows.github.io/Android-Launcher-Analysis/
https://www.pianshen.com/article/2581187899/
https://blog.csdn.net/qinhai1989/article/details/86706877