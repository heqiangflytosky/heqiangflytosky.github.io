---
title: Android 性能优化之旋转屏幕优化
categories: Android性能优化
comments: true
tags: [Android, Android性能优化]
description: 介绍开发过程中遇到的旋转屏幕卡顿问题的解决过程
date: 2017-4-26 10:00:00
---

## 问题背景

在桌面转屏发现响应不够迅速，对比其他产品有很大的提升空间，针对此问题进行了一些分析和优化。

## 问题分析和解决方法

首先简单介绍一下旋转屏幕的流程，首先各个界面要进行重绘，在重绘过程中要进行冻屏，只有所有`Window`都进行绘制完成了才进行转屏，因此这里面就有个木桶效应，转屏的时间取决于重绘最慢的那个。
首先分析Log，找出可以优化的点：

```
adb shell dumpsys window -d enable 10
adb logcat -v threadtime -s WindowManager | grep -E "Screen frozen for|Dismissing screen|Orientation start waiting for draw|Orientation not waiting for draw"
```

`adb shell dumpsys window -d enable 10`是使能`DEBUG_ORIENTATION`，开启打印转屏相关的Log。

```
WindowManager: Orientation start waiting for draw mDrawState=DRAW_PENDING in Window{249e41f u0 StatusBar}, surface Surface(name=StatusBar)
WindowManager: Orientation start waiting for draw mDrawState=DRAW_PENDING in Window{731dc6a u0 com.android.launcher/com.android.launcher.Launcher}, surface Surface(name=com.android.launcher/com.android.launcher.Launcher)
WindowManager: Orientation start waiting for draw mDrawState=DRAW_PENDING in Window{b22eda5 u0 com.android.systemui.ImageWallpaper}, surface Surface(name=com.android.systemui.ImageWallpaper)
WindowManager: Orientation start waiting for draw mDrawState=DRAW_PENDING in Window{731dc6a u0 com.android.launcher/com.android.launcher.Launcher}, surface Surface(name=com.android.launcher/com.android.launcher.Launcher)
WindowManager: Orientation start waiting for draw mDrawState=DRAW_PENDING in Window{731dc6a u0 com.android.launcher/com.android.launcher.Launcher}, surface Surface(name=com.android.launcher/com.android.launcher.Launcher)
WindowManager: Orientation start waiting for draw mDrawState=DRAW_PENDING in Window{94517ed u0 com.android.launcher/com.android.launcher.Launcher}, surface Surface(name=com.android.launcher/com.android.launcher.Launcher)
WindowManager: Orientation not waiting for draw in Window{94517ed u0 com.android.launcher/com.android.launcher.Launcher}, surface Surface(name=com.android.launcher/com.android.launcher.Launcher)
WindowManager: Orientation not waiting for draw in Window{b22eda5 u0 com.android.systemui.ImageWallpaper}, surface Surface(name=com.android.systemui.ImageWallpaper)
WindowManager: Orientation not waiting for draw in Window{249e41f u0 StatusBar}, surface Surface(name=StatusBar)
WindowManager: Screen frozen for +968ms due to Window{249e41f u0 StatusBar}
WindowManager: **** Dismissing screen rotation animation
```

通过Log发现，转屏是要等`StatusBar`，`Launcher`和`ImageWallpaper`绘制完才会开始的。
根据`Screen frozen for +968ms due to Window{249e41f u0 StatusBar}`发现目前的瓶颈在状态栏这里，首先分析一下状态栏的代码，可以借助于 TraceView 工具迅速定位到耗时较多方法的位置，发现在`NotificationPanelView`的`onConfigurationChanged`函数中有一项耗时的操作，先进行这部分优化。
然后看效果：

```
WindowManager: Screen frozen for +747ms due to Window{3aaa434 u0 com.android.launcher/com.android.launcher.Launcher}
```

发现这个时候瓶颈已经不再`StatusBar`这里了，接下面再优化`Launcher`就可以。
接下来看看`ImageWallpaper`是不是有优化的余地呢？
通过 TraceView 工具发现在`ImageWallpaper.drawFrame()`方法中，每次旋转屏幕都会在`updateWallpaperLocked()`中调用`mWallpaperManager.getBitmap()`进行解码图片，这个也是没有必要的，只初始化一次就可以了，可以进行如下的修改：

```
                // Load bitmap if it is not yet loaded or if it was loaded at a different size
                if (mBackground == null/* || surfaceDimensionsChanged*/) {
```

只在`mBackground`为`null`是加在壁纸图片。
第三步的任务就是优化`Launcher`了，此处的优化点各不相同，而且较多，就不一一介绍了。


