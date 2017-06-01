---
title: Android实用技巧之adb命令：dumpsys命令的使用 
categories: Android实用技巧
comments: true
tags: [Android, dumpsys]
description: 介绍dumpsys的使用
date: 2014-10-15 10:00:00
---
Android提供的`dumpsys`工具可以用于查看手机中的应用程序和系统服务信息与状态，手机连接电脑后可以直接命令行执行`adb shell dumpsy`查看所有支持的`Service`。但是这样输出的太多，可以通过`dumpsys | grep "DUMP OF SERVICE"` 仅显示主要的`Service`的信息。
关于这个命令的使用方法在这里做一下记录，以备使用。
## dumpsys支持的所有命令
输入：
```
adb shell dumpsys | grep DUMP
```
或
```
adb shell dumpsys | grep "DUMP OF SERVICE" 
```
就列出了当前手机支持的所有`dumpsys`参数：
```
DUMP OF SERVICE Exynos.HWCService:
DUMP OF SERVICE SurfaceFlinger:
DUMP OF SERVICE access_control:
DUMP OF SERVICE accessibility:
DUMP OF SERVICE account:
DUMP OF SERVICE activity:
DUMP OF SERVICE alarm:
DUMP OF SERVICE android.security.keystore:
DUMP OF SERVICE appops:
DUMP OF SERVICE appwidget:
DUMP OF SERVICE audio:
DUMP OF SERVICE backup:
DUMP OF SERVICE battery:
DUMP OF SERVICE batterypropreg:
DUMP OF SERVICE batterystats:
DUMP OF SERVICE bluetooth_manager:
DUMP OF SERVICE blurglassinfo:
DUMP OF SERVICE clipboard:
DUMP OF SERVICE com.meizu.nfc.NxpExt:
DUMP OF SERVICE commontime_management:
DUMP OF SERVICE connectivity:
DUMP OF SERVICE consumer_ir:
DUMP OF SERVICE content:
DUMP OF SERVICE country_detector:
DUMP OF SERVICE cpuinfo:
DUMP OF SERVICE dbinfo:
DUMP OF SERVICE deivce_states:
DUMP OF SERVICE device_policy:
DUMP OF SERVICE devicestoragemonitor:
DUMP OF SERVICE diskstats:
DUMP OF SERVICE display:
DUMP OF SERVICE dreams:
DUMP OF SERVICE drm.drmManager:
DUMP OF SERVICE dropbox:
DUMP OF SERVICE entropy:
DUMP OF SERVICE gesture_manager:
DUMP OF SERVICE gfxinfo:
DUMP OF SERVICE hardware:
DUMP OF SERVICE input:
DUMP OF SERVICE input_method:
DUMP OF SERVICE iphonesubinfo:
DUMP OF SERVICE isms:
DUMP OF SERVICE location:
DUMP OF SERVICE lock_settings:
DUMP OF SERVICE media.audio_flinger:
DUMP OF SERVICE media.audio_policy:
DUMP OF SERVICE media.camera:
DUMP OF SERVICE media.player:
DUMP OF SERVICE media_router:
DUMP OF SERVICE meizu.camera:
DUMP OF SERVICE meminfo:
DUMP OF SERVICE mount:
DUMP OF SERVICE netpolicy:
DUMP OF SERVICE netstats:
DUMP OF SERVICE network_management:
DUMP OF SERVICE nfc:
DUMP OF SERVICE notification:
DUMP OF SERVICE package:
DUMP OF SERVICE permission:
DUMP OF SERVICE phone:
DUMP OF SERVICE phone_ext:
DUMP OF SERVICE power:
DUMP OF SERVICE pppoe:
DUMP OF SERVICE print:
DUMP OF SERVICE procstats:
DUMP OF SERVICE samba_client:
DUMP OF SERVICE samba_server:
DUMP OF SERVICE samplingprofiler:
DUMP OF SERVICE scheduling_policy:
DUMP OF SERVICE search:
DUMP OF SERVICE secloader:
DUMP OF SERVICE secloader2:
DUMP OF SERVICE secsystemserver:
DUMP OF SERVICE sensorservice:
DUMP OF SERVICE serial:
DUMP OF SERVICE servicediscovery:
DUMP OF SERVICE simphonebook:
DUMP OF SERVICE sip:
DUMP OF SERVICE statusbar:
DUMP OF SERVICE telephony.registry:
DUMP OF SERVICE textservices:
DUMP OF SERVICE uimode:
DUMP OF SERVICE updatelock:
DUMP OF SERVICE usagestats:
DUMP OF SERVICE usb:
DUMP OF SERVICE user:
DUMP OF SERVICE vibrator:
DUMP OF SERVICE voicesense:
DUMP OF SERVICE wallpaper:
DUMP OF SERVICE wifi:
DUMP OF SERVICE wifip2p:
DUMP OF SERVICE window:
```

## 具体命令如何查看帮助
从上面可以看出可以查看的`Service`非常多，`DUMP OF SERVICE`关键字后面的单词都可以直接通过 `dumpsys + 单词` 查看相关信息，具体每一个如何使用有一种通用的查看帮助的办法。
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

## 参考资料
http://www.open-open.com/lib/view/open1405061994872.html
https://source.android.com/devices/tech/input/dumpsys.html
