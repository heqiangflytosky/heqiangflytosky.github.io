---
title: Android实用技巧之adb命令：dumpsys命令的使用 
categories: Android实用技巧
comments: true
tags: [Android, dumpsys]
description: 介绍dumpsys的使用
date: 2014-10-15 10:00:00
---

## dumpsys概述

Android提供的`dumpsys`工具可以用于查看手机中的应用程序和系统服务信息与状态，能够熟练使用`dumpsys`工具以及对它的输出内容进行理解是Android开发人员的必备技能，使用它我们不仅能够对方便的获取一些当前的平台的一些信息，而且能够对Android性能优化、Bug分析与调试带来很大的帮助。
手机连接电脑后可以直接命令行执行`adb shell dumpsy`查看所有支持的`Service`，但是这样输出的太多，可以通过`dumpsys | grep "DUMP OF SERVICE"` 仅显示主要的`Service`的信息。
关于这个命令的使用方法在这里做一下记录，以备使用。
关于`adb shell dumpsy` 命令的源码分析请参考我的博客[Android 深入理解 dumpsys ](http://www.heqiangfly.com/2017/06/13/android-source-code-analysis-dumpsys/)。

## dumpsys支持的所有命令

输入：

```
adb shell dumpsys -l
```

可以查看当前支持的所有系统服务列表：

```
  DockObserver
  GuiExtService
  SurfaceFlinger
  access_control
  accessibility
  account
  activity
  ......
  meminfo
  mount
  move_window
  netpolicy
  netstats
  network_management
  network_score
  networkmanagement_service_flyme
  notification
  package
  permission
  persistent_data_block
  power
  pppoe
  ......
  trust
  uimode
  updatelock
  usagestats
  usb
  user
  vibrator
  voiceinteraction
  wallpaper
  webviewupdate
  wifi
  wifip2p
  wifiscanner
  window
```

输入 `adb shell dumpsys` 命令可以输出所有的系统服务信息。可以通过`adb shell dumpsys | grep DUMP`过滤输出当前服务列表。

## 具体命令如何查看帮助

从上面可以看出可以查看的`Service`非常多，`adb shell dumpsys -l`所列举的服务都可以直接通过 `dumpsys + <service>` 查看相关信息，具体每一个如何使用有一种通用的查看帮助的办法。
查看每一个命令的使用帮助，以下以`activity`为例演示：
```
$ adb shell dumpsys activity -h

Activity manager dump options:
  [-a] [-c] [-h] [cmd] ...
  cmd may be one of:
    a[ctivities]: activity stack state
    b[roadcasts] [PACKAGE_NAME] [history [-s]]: broadcast state
    i[ntents] [PACKAGE_NAME]: pending intent state
    p[rocesses] [PACKAGE_NAME]: process state
    o[om]: out of memory management
    prov[iders] [COMP_SPEC ...]: content provider state
    provider [COMP_SPEC]: provider client-side state
    s[ervices] [COMP_SPEC ...]: service state
    service [COMP_SPEC]: service client-side state
    package [PACKAGE_NAME]: all state related to given package
    all: dump all activities
    top: dump the top activity
  cmd may also be a COMP_SPEC to dump activities.
  COMP_SPEC may be a component name (com.foo/.myApp),
    a partial substring in a component name, a
    hex object identifier.
  -a: include all available server state.
  -c: include client state.
```
这样就可以清楚每个命名的使用方法以及对应输出内容的信息查看方法。
以`window`为例：
```
$ adb shell dumpsys window -h

Window manager dump options:
  [-a] [-h] [cmd] ...
  cmd may be one of:
    i[input]: input subsystem state
    p[policy]: policy state
    s[essions]: active sessions
    surfaces: active surfaces (debugging enabled only)
    d[isplays]: active display contents
    t[okens]: token list
    w[indows]: window list
  cmd may also be a NAME to dump windows.  NAME may
    be a partial substring in a window name, a
    Window hex object identifier, or
    "all" for all windows, or
    "visible" for the visible windows.
  -a: include all available server state.
  -d list                     list the all of debug zones
  -d enable  <zone zone ...>  enable the debug zone
  -d disable <zone zone ...>  disable the debug zone
  zone usage : 
    0  : DEBUG
    1  : DEBUG_FOCUS
    2  : DEBUG_ANIM
    3  : DEBUG_LAYOUT
    4  : DEBUG_RESIZE
    5  : DEBUG_LAYERS
    6  : DEBUG_INPUT
    7  : DEBUG_INPUT_METHOD
    8  : DEBUG_VISIBILITY
    9  : DEBUG_WINDOW_MOVEMENT
    10 : DEBUG_ORIENTATION
    11 : DEBUG_CONFIGURATION
    12 : DEBUG_APP_TRANSITIONS
    13 : DEBUG_STARTING_WINDOW
    14 : DEBUG_REORDER
    15 : DEBUG_WALLPAPER
    16 : DEBUG_WALLPAPER_LIGHT
    17 : SHOW_TRANSCATIONS
    18 : HIDE_STACK_CRAWLS
    19 : PROFILE_ORIENTATION
    20 : DEBUG_TASK_MOVEMENT
    21 : DEBUG_ADD_REMOVE
    22 : DEBUG_TOKEN_MOVEMENT
    23 : DEBUG_APP_ORIENTATION
    24 : DEBUG_DRAG
    25 : DEBUG_SCREEN_ON
    26 : DEBUG_SCREENSHOT
    27 : DEBUG_BOOT
    28 : SHOW_SURFACE_ALLOC
    29 : SHOW_LIGHT_TRANSACTIONS
    30 : DEBUG_LAYOUT_REPEATS
    31 : DEBUG_SURFACE_TRACE
    32 : DEBUG_WINDOW_TRACE
    33 : DEBUG_WINDOW
    34 : DEBUG_STACK
    35 : DEBUG_DIM_LAYER
    36 : DEBUG_KEYGUARD
```
比如`adb shell dumpsys window -d enable 10`就是打开`DEBUG_ORIENTATION`，这样就把代码中的关于屏幕方向旋转相关的Log打印出来。

## 一些常用命令解释

 - adb shell dumpsys activity： 显示activity的相关信息，包括任务栈等
 - adb shell dumpsys activity top：查看当前应用的 activity 信息
 - adb shell dumpsys meminfo：查看各个进程内存使用情况。（`meminfo $package_name or $pid` 使用程序的包名或者进程id显示内存信息比如浏览器：`adb shell dumpsys meminfo com.android.browser`）
 - adb shell dumpsys SurfaceFlinger： 查看UI绘制的各个层级信息
 - adb shell dumpsys window： 显示键盘，窗口和它们的关系
 - adb shell dumpsys package <包名>： 查看该包的具体信息
 - adb shell dumpsys statusbar： 显示状态栏相关信息
 - adb shell dumpsys usagestats： 每个应用的启动次数和时间
 - adb shell dumpsys battery： 电池信息
 - adb shell dumpsys diskstats： 磁盘相关信息
 - adb shell dumpsys alarm： 显示Alarm信息
 - adb shell dumpsys wifi： 显示wifi信息
 - adb shell dumpsys user：查看当前的用户情况
 - adb shell dumpsys dbinfo <包名>：查看指定包名应用的数据库存储信息（包括存储的SQL语句）

## 参考资料

http://www.open-open.com/lib/view/open1405061994872.html
https://source.android.com/devices/tech/input/dumpsys.html
