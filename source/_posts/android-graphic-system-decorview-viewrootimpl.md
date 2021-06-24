---
title: Android 图形系统 --  Window，DecorView 和 ViewRootImpl
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 Android 图形系统 --  DecorView 和 ViewRootImpl
date: 2016-7-28 10:00:00
---


## 概述

本文是了解View绘制流程前的一些预备知识，本文会介绍 DecorView 是如何添加到PhoneWindow中，以及 ViewRootImpl 是如何连接 WindowManager 和 DecorView 的。
Activity 控制生命周期和处理事件，Window是视图的承载器，内部有个 DecorView 作为视图的根布局。在 Activity 中持有的是 Window 的子类 PhoneWindow。
DecorView 是一个顶层 View，是应用的整个界面。
ViewRootImpl 是实现 View 的绘制的类，里面有三个关键的方法来完成 View 的绘制流程。它是 DecorView 对应的 mParent 。普通子 View (非根 View) 对应的 mParent 是子 View 的父 View (即 ViewGroup)。
在 ActivityThread 中，当 Activity 对象被创建完毕后，会将 DecorView 添加到 Window 中，同时会创建ViewRootImpl 对象，并将 ViewRootImpl 对象和 DecorView 建立关联，然后从 ViewRootImpl 的performTraversals 方法开始 View 的绘制流程。

## PhoneWindow

PhoneWindow的初始化：

```
//Activity.java
    @UnsupportedAppUsage
    final void attach(Context context, .....) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        ......
    }
```

Activity 中创建了 Window，并且每个 Activity 都有一个与之对应的 Window ，具体的实现类，其实是一个 PhoneWindow。

## DecorView

DecorView继承自FrameLayout，其作为顶级View被加载到window中，主要包含titlebar和contentParent两个子元素，其中titlebar可以通过系统属性指定其状态。

我们来看一个应用的布局：

```
DecorView
    ActionBarOverlayLayout(R.id.decor_content_parent)
        (R.id.content)
        ActionBarContainer(R.id.action_bar_container)
            Toolbar(R.id.action_bar)
        View(R.id.statusBarBackground)
        View(R.id.navigationBarBackground)
```

```
Activity.setContentView
    PhoneWindow.setContentView
        PhoneWindow.installDecor()
            PhoneWindow.generateDecor()
                new DecorView()
            PhoneWindow.generateLayout()
                requestFeature(FEATURE_ACTION_BAR)//设置actionbar
                contentParent=findViewById(ID_ANDROID_CONTENT)//获取R.id.content
                DecorView.setWindowBackground()//设置背景
    LayoutInflater.inflate(layoutResID, mContentParent)
```

DecorView 是在 PhoneWindow 中生成的，然后把用户设置的布局添加到DecorView对应的子View上去。

## ViewRootImpl

ViewRootImpl 可以理解为一个 Window 中所有 View 的根 View 的管理者（但 ViewRootImpl 不是 View，只是实现了 ViewParent 接口），是 View 与 WindowManager 之间联系的桥梁，作用总结如下：

 - 将 DecorView 传递给 WindowManagerSerive
 - 完成 View 的绘制过程，包括 measure、layout、draw 过程
 - 向 DecorView 分发收到的用户发起的 event 事件，如按键，触屏等事件。

```
ActivityThread.handleResumeActivity()
　　Activity.getWindow() //获取activity的PhoneWindow
    PhoneWindow.getDecorView()//获取PhoneWindow的DecorView
    Activity.getWindowManager()//获取Activity的WindowManagerImpl
    WindowManagerImpl.addView(decor, l)//通过WindowManagerImpl将DecorView添加到Activity
        WindowManagerGlobal.addView(DecorView)
            new ViewRootImpl()//创建ViewRootImpl对象
            ViewRootImpl.setView()
                ViewRootImpl.requestLayout()
                    ViewRootImpl.scheduleTraversals()
                        Choreographer.postCallback(mTraversalRunnable)
                            ViewRootImpl.doTraversal()
                                ViewRootImpl.performTraversals()
                                    ViewRootImpl.performMeasure()
                                        DecorView.measure()
                                    ViewRootImpl.performLayout()
                                        DecorView.layout()
                                    ViewRootImpl.performDraw()
                                        DecorView.draw()
                WindowSession.addToDisplay()
                new WindowInputEventReceiver()
```

然后，DecorView的绘制流程就从ViewRoot的performTraversals方法开始了，后面就是遍历子view完成整个界面的测量，布局和绘制。
performTraversals()可能并不会执行完整的measure、layout和draw流程，内部有很复杂的判断条件，比如执行requestLayout()可能只会执行measure和layout，执行invalidate()只会执行draw方法。这方面的知识点后面详细介绍。
