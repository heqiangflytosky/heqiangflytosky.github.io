---
title: SystemUI -- SystemUI 的几种状态
categories:  SystemUI
comments: true
tags: [SystemUI]
description: 介绍 SystemUI 的几种状态
date: 2021-10-20 10:00:00
---

## 概述

本文来介绍 SystemUI 的几种状态，基于 Android S。

## 状态定义

StatusBarState 类定义了状态栏的几中状态：

 - SHADE：解锁进入桌面之后的状态。
 - KEYGUARD：位于锁屏时的状态。
 - SHADE_LOCKED：在锁屏上，下拉锁屏通知，将面板和通知栏下拉出来，此时会处于SHADE_LOCKED状态。
但是如果手指下滑状态栏将面板和通知栏下拉出来 ，仍然属于KEYGUARD状态。
 - FULLSCREEN_USER_SWITCHER：用户选择界面，在手机上时通过 `config_enableFullscreenUserSwitcher` 来取消了这种状态的，比如在车机上，就有这种状态。可以通过 `UserSwitcherController.useFullscreenUserSwitcher()` 来获取是否使能这种状态。


StatusBarStateController 如果状态有更新，向 StateListener 发送更新的状态，提供了下面的方法：

 - getState()：当前通知栏处的状态。
 - isDozing()：设备处于AOD或者睡眠状态。
 - isExpanded()：面板是否展开。
 - isPulsing()：
 - addCallback()：添加StateListener

StatusBarStateControllerImpl：StatusBarStateController 的实现类，跟踪和报告 StatusBarState 的变化。

## 状态设置

StatusBarStateControllerImpl.setState()


## 状态判断

StatusBarStateControllerImpl.getState()

```
Dependency.get(StatusBarStateController.class).getState()
```