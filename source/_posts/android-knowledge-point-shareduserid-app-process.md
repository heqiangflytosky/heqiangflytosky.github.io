---
title: Android App Process 以及 sharedUserId的关系
categories: Android
comments: true
tags: [Android]
description: 介绍 sharedUserId App 和 Process 的关系
date: 2015-12-16 10:00:00
---

## 应用和进程之间的关系

### 一个应用内的多进程

Activity 可以通过 `android:process` 来运行在指定的进程。
指定方法：

 - 冒号开头的字符串：比如 `：remote`，那么该进程就是私有进程。
 - 其他字符串，必须包含一个 . ，否则安装 apk 时会报错“Invalid process name ** in package *.*.*.*: must have at least one '.' separator”。那么它就是公有进程，这样拥有相同 ShareUID 的不同应用可以跑在同一进程里。 

### 调用其他App中的 Activity

我们先来简单测试一下从应用 A 中启动应用 B 中的 Activity A1。
启动后 ps 进程关系：

```
u0_a267   2163  2669  1946936 110492 SyS_epoll_ 7da55ef488 S com.example.heqiang.myapplication
u0_a266   2198  2669  2048192 122872 SyS_epoll_ 7da55ef488 S com.example.heqiang.testsomething
```

它们位于不同的进程，其实这里不管启动模式是什么，他们都是在不同的进程。
即 A 启动的 Activity A1，它并不运行A自己的进程，而是运行在 A1 所属的应用 B 的进程。


### 通过 `android:process` 来指定进程

先来说一下场景：在应用 A 内为 Activity A1 指定进程为 `android:process="com.hq.test.process"`，在应用 B 内为 Activity B1 指定进程为 `android:process="com.hq.test.process"`，即和 A1 指定同一个进程。
我们先在 应用 A 内启动 A1，然后在应用 A 内启动 B1，最后在从桌面点击应用B的图标。
从上到下分别是 A1，B1，A 的 Launcher Acitvity，B的Launcher Acitvity所在的进程，可见他们都分别运行在不同的进程，虽然 A1 和 B1 的进程名称相同。

```
u0_a266   3264  2669  2086900 123736 do_signal_ 7da55ef488 T com.hq.test.process
u0_a267   3639  2669  2024264 138764 SyS_epoll_ 7da55ef488 S com.hq.test.process
u0_a267   3559  2669  1953996 126752 SyS_epoll_ 7da55ef488 S com.example.heqiang.myapplication
u0_a266   3781  2669  2216428 117764 SyS_epoll_ 7da55ef488 S com.example.heqiang.testsomething
```

下面来看同一个应用为两个不同的 Activity A1 和 A2 指定相同 `android:process` 的情况，这个时候只有一个名称为 com.hq.test.process 的进程：

```
u0_a268   7624  2669  2026660 141140 SyS_epoll_ 7da55ef488 S com.hq.test.process
```

通过dumpsys查看 Task 和进程：

```
    Task id #21375
    mFullscreen=true
    mBounds=null
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
      TaskRecord{9ea1cf3 #21375 A=com.example.heqiang.testsomething U=0 StackId=1 sz=2}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
        Hist #1: ActivityRecord{d23b29a u0 com.example.heqiang.testsomething/.event.EventActivity t21375}
          Intent { cmp=com.example.heqiang.testsomething/.event.EventActivity }
          ProcessRecord{86ad4dc 7624:com.hq.test.process/u0a268}
        Hist #0: ActivityRecord{de17cd2 u0 com.example.heqiang.testsomething/.MainActivity t21375}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
          ProcessRecord{7fd9e5 7587:com.example.heqiang.testsomething/u0a268}
    Task id #21376
    mFullscreen=true
    mBounds=null
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
      TaskRecord{2122429 #21376 A=com.hq.test.task U=0 StackId=1 sz=1}
      Intent { flg=0x10000000 cmp=com.example.heqiang.testsomething/.commontest.OtherTestActivity }
        Hist #0: ActivityRecord{73a26e8 u0 com.example.heqiang.testsomething/.commontest.OtherTestActivity t21376}
          Intent { flg=0x10000000 cmp=com.example.heqiang.testsomething/.commontest.OtherTestActivity }
          ProcessRecord{86ad4dc 7624:com.hq.test.process/u0a268}
```

他们虽然在不同的 Task 中，但是他们在相同的进程 7624 中。

## sharedUserId 和进程之间的关系

首先给出 sharedUserId 的设置方法：

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.heqiang.myapplication"
    android:sharedUserId="com.hq.test.shareduserid">
```

还要注意的是 sharedUserId 相同的两个 App 的签名必须相同。

### sharedUserId 相同，process 属性不同的两个App

配置两个包名不同的 App 为相同的 sharedUserId，它们的 MainActivity 的 `android:process` 也不一样。
看一下启动两个应用的进程：

```
:u0_a271   14565 2669  2220332 132320 SyS_epoll_ 7da55ef488 S com.example.heqiang.testsomething
:u0_a271   14662 2669  2051216 140792 SyS_epoll_ 7da55ef488 S com.example.heqiang.myapplication
```

可以看到，两个 Activity 仍然在不同的进程，但是 userId 确实相同的，u0_a271。

### sharedUserId 相同，process 属性相同的两个App

配置两个包名不同的 App 为相同的 sharedUserId，它们的 MainActivity 的 `android:process` 统一配置成 `com.hq.test.process`。
看一下ps结果：

```
u0_a271   14865 2669  2197988 156772 SyS_epoll_ 7da55ef488 S com.hq.test.process
```

两个 Activity 运行在同一个进程。

<!--  
https://blog.csdn.net/furuidelei123/article/details/7255188

https://www.cnblogs.com/scarecrow-blog/p/4876560.html

-->
