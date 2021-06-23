---
title: Android 图形系统 --  DecorView 和 ViewRootImpl
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 Android 图形系统 --  DecorView 和 ViewRootImpl
date: 2016-7-28 10:00:00
---


## 概述

本文是了解View绘制流程前的一些预备知识，本文会介绍DecorView是如何添加到PhoneWindow中，以及ViewRootImpl是如何连接WindowManager和DecorView的。
DecorView 是一个顶层 View，是应用的整个界面。
ViewRootImpl 是实现 View 的绘制的类，里面有三个关键的方法来完成View的绘制流程。
在ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl对象和DecorView建立关联，然后从ViewRootImpl的performTraversals方法开始View的绘制流程。

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

## ViewRootImpl

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
```

然后，DecorView的绘制流程就从ViewRoot的performTraversals方法开始了，后面就是遍历子view完成整个界面的测量，布局和绘制。
