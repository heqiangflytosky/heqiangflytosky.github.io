---
title: Android -- 导航栏启动多任务
categories:  Android Launcher
comments: true
tags: [Launcher3]
description: 介绍导航栏启动多任务的流程
date: 2021-2-10 10:00:00
---

## 概述

在Android Ｏ以后的版本中，多任务（QuickStep）的逻辑都放在了Launcher中，代码放在　quickstep　目录中。
启动多任务，一般是通过底部导航栏或者手势来启动，它们的发起者都是 SystemUI，SystemUI(OverviewProxyService) 和 Launcher
(TouchInteractionService)通过进程间通信来启动多任务。
在Launcher中，LauncherRecentsView 是用来承载 TaskView 用来显示多任务的。

## 导航栏启动流程

### SystemUI

```
├── NavigationBarFragment.onRecentsClick
    ├── CommandQueue.toggleRecentApps
        ├── Recents.toggleRecentApps
            ├── OverviewProxyRecentsImpl.toggleRecentApps
                ├── OverviewProxyService.mOverviewProxy.onOverviewToggle
```

### Launcher

TouchInteractionService 收到指令后显示 LauncherRecentsView 并启动 Launcher。

```
TouchInteractionService.onOverviewToggle
    OverviewCommandHelper.onOverviewToggle
        RecentsActivityCommand.run
            handleCommand() // 多任务是否显示,连续点击判断和实现双击切换应用
            LauLauncherActivityControllerHelper.switchToRecentsIfVisible // 桌面中启动多任务
                launcher.getStateManager().goToState（OVERVIEW）  // 状态切换
            LauncherInitListener.createActivityInitListener
            LauncherInitListener.registerAndStartActivity //应用中启动多任务，或者启动独立Activity多任务RecentActivity
                ActivityInitListener.registerAndStartActivity
                    RemoteAnimationProvider.toActivityOptions
                        new LauncherAnimationRunner
                        ActivityOptionsCompat.makeRemoteAnimation(new RemoteAnimationAdapterCompat(
                    startActivity
```

registerAndStartActivity 可能会启动不同的activity，可能是 Launcher，也可能时 fallback 的 RecentActivity。Intent 在 OverviewComponentObserver.updateOverviewTargets() 方法中设置。

```
OverviewComponentObserver.getOverviewIntent
mOverviewIntent:
Intent { act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000000 pkg=com.meizu.flyme.launcher cmp=com.meizu.flyme.launcher/com.android.launcher3.uioverrides.QuickstepLauncher }
```

多任务显示：

```
Launcher.onNewIntent
    ActivityTracker.handleNewIntent
        ActivityInitListener.init
            LauncherInitListener.handleInit
                ActivityInitListener.handleInit
                    OverviewCommandHelper.RecentsActivityCommand.onActivityReady // 前面　createActivityInitListener　时注册
                        AppToOverviewAnimationProvider.onActivityReady
                            LauncherActivityInterface.prepareRecentsUI  // 返回AnimationFactory，提供显示多任务的动画
                                BaseActivityInterface.DefaultAnimationFactory.initUI
                                    StateManager().goToState
                                        RecentsViewStateController.setState
                                            getContentAlphaProperty().set()
                                                RecentsView.CONTENT_ALPHA.setValue
                                                    LauncherRecentsView.setContentAlpha
                                                        RecentsView.setContentAlpha
                                                            RecentsView.setVisibility
                    
```