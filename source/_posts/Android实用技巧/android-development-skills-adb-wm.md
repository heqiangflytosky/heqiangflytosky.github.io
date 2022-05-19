---
title: Android实用技巧之adb命令：wm 命令的使用
categories: Android实用技巧
comments: true
tags: [Android, wm]
description: 介绍 wm 的使用
date: 2015-3-28 10:00:00
---

## 概述

wm 命令是和 Android WindowManagerService 相关联的，可以获取设备屏幕相关信息，比如：分辨率、像素密度等。甚至还可以修改这些参数，可以很方便地查看 APP 在不同像分辨率和素密度设备上的显示效果。

## 命令参数

使用 `adb shell wm` 可以查看wm支持的一些参数：

```
usage: wm [subcommand] [options]
       wm size [reset|WxH|WdpxHdp]
       wm density [reset|DENSITY]
       wm overscan [reset|LEFT,TOP,RIGHT,BOTTOM]
       wm scaling [off|auto]
       wm screen-capture [userId] [true|false]

wm size: return or override display size.
         width and height in pixels unless suffixed with 'dp'.

wm density: override display density.

wm overscan: set overscan area for display.

wm scaling: set display scaling mode.

wm screen-capture: enable/disable screen capture.

wm dismiss-keyguard: dismiss the keyguard, prompting the user for auth if necessary.
```

 - `wm size`：查看屏幕的分辨率, 单位: px。
 - `wm size <WxH|WdpxHdp>`：修改置屏幕的分辨率：wm size 720x1280（单位px），wm size 360dpx640dp（单位dp）。
 - `wm size reset`：重置屏幕的分辨率，撤销对屏幕分辨率的修改（改回真实的物理分辨率）。
 - `wm density`：查看屏幕的像素密度, 单位: dpi（dots per inch）。
 - `wm density <DENSITY>`：修改屏幕的像素密度：wm density 360。
 - `wm density reset`：重置屏幕的像素密度，撤销对屏幕像素密度的修改（改回真实的像素密度）
 - `wm overscan`：该命令用来设置、重置LCD的显示区域。四个参数分别是显示边缘距离LCD左、上、右、下的像素数。例如，对于分辨率为540x960的屏幕，通过执行 命令wm overscan 0,0,0,420可将显示区域限定在一个540x540的矩形框里。
 - `wm scaling`：设置缩放模式
 - `wm screen-capture`：屏幕截图
 - `wm dismiss-keyguard`：取消锁屏
