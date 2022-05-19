---
title: Android 组件 -- 前台 Service
categories: Android 组件
comments: true
tags: [Android]
description: 介绍前台 Service
date: 2015-12-22 10:00:00
---

## 概述

在前面的博客中介绍了前台服务和后台服务的区别，在本节中介绍如何启动一个前台服务和前台服务的特点。
前台服务一个经典使用场景就是音乐播放器。

## 前台服务特点

一般情况下 Service 都是运行在后台做一些默默无闻的工作，没有界面和用户交互，这样的服务优先级比较低，在系统内存不足时可能会优先被杀掉。
本节我们介绍一下前台服务，前台服务是可以被用户感知的（通知栏的信息），而且优先级较高，不会那么容易的被杀死。前台服务必须给状态栏提供一个通知。
前台服务的通知栏用户是无法手动移除的，只有在服务终止或者在通过 `stopForeground(true)` 主动移除通知栏时才会消失。
前台服务在进程杀掉后仍然会重新启动服务。除非使用设置中的“强行停止”，`forceStopPackage`。

## 相关方法

 - startForeground(int id, Notification notification)：开启前台，id 表示通知栏的标识，不能为 0。
 - stopForeground(boolean removeNotification)：取消前台，removeNotification 表示是否移除通知，false 情况下不会移除通知，但是此时通知栏编程可删除模式。
 - stopForeground(int flags)：两种模式：STOP_FOREGROUND_REMOVE 和 STOP_FOREGROUND_DETACH，上面的方法其实真正调用的是这个方法。

## 启动前台服务

在 Service 中有个 `startForeground(int id, Notification notification)` 方法，使用它可以设置一个服务为前台服务。
在 `onCreate()` 中添加下面代码，就可以启动一个前台服务，在通知栏里面显示通知信息。

```
    @Override
    public void onCreate() {
        Log.d(TAG, "RemoteService onCreate");
        super.onCreate();

        // 设置点击通知栏的跳转 intent
        Intent notificationIntent = new Intent(this,MainActivity.class);
        PendingIntent pendingIntent = PendingIntent.getActivity(this,0,notificationIntent,0);

        // 设置通知栏信息
        Notification.Builder builer = new Notification.Builder(this);
        builer.setContentTitle("前台 Service 标题");
        builer.setContentText("前台 Service 内容");
        builer.setSmallIcon(R.mipmap.ic_launcher);
        builer.setContentIntent(pendingIntent);

        Notification notification = builer.getNotification();
        //设置为前台Service，并显示通知栏
        startForeground(1, notification);
        // 如果把 id 设为 0 ，不会在通知栏显示通知
        //startForeground(0, notification);
    }
```

`startForeground` 里面的 id 表示一个前台通知栏的通知，id相同的通知栏会进行覆盖。