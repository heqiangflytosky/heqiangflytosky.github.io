---
title: SystemUI -- 媒体控制器
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍媒体控制器相关
date: 2021-11-16 12:00:00
---



## 概述

在 Android S上面分别有三个位置可以承载媒体控制器，分别时QQS、QS和锁屏，这三个位置动态添加 MediaScrollView 来实现切换场景时媒体控制器的位置的变换。
如图：

<img src="/images/android-systemui-panel-media-controller/lockscreen.png" width="540" height="1200"/>
<img src="/images/android-systemui-panel-media-controller/qqs.png" width="540" height="1200"/>
<img src="/images/android-systemui-panel-media-controller/qs.png" width="540" height="1200"/>
<img src="/images/android-systemui-panel-media-controller/animation.png" width="540" height="1200"/>

相关的类：

 - MediaHierarchyManager
 - MediaHost
 - UniqueObjectHostView 承载媒体控制器的容器，它是 MediaHost 的一个变量，一个 MediaHost 对应一个 UniqueObjectHostView

在 MediaHierarchyManager 中定义了媒体控制器的三个位置场景。

```
// MediaHierarchyManager.kt
    companion object {
        /**
         * Attached in expanded quick settings
         * 添加在QS面板下方
         */
        const val LOCATION_QS = 0

        /**
         * Attached in the collapsed QS
         * QS折叠时添加在QQS下方
         */
        const val LOCATION_QQS = 1

        /**
         * Attached on the lock screen
         * 锁屏时添加到通知面板上方
         */
        const val LOCATION_LOCKSCREEN = 2

        /**
         * Attached at the root of the hierarchy in an overlay
         */
        const val IN_OVERLAY = -1000
```

定义了一个数组 mediaHosts 来保存

```
private val mediaHosts = arrayOfNulls<MediaHost>(LOCATION_LOCKSCREEN + 1)
```

## 初始化

三个场景分别定义了三个 MediaHost，并且实例化了三个 UniqueObjectHostView View，这三个View是在 SystemUI初始化时就创建的，然后添加到各个场景的UI中去，不管此时有没有媒体创建的需求。

锁屏初始化：

```
KeyguardMediaController.kt

    init {
        // First let's set the desired state that we want for this host
        mediaHost.expansion = MediaHostState.COLLAPSED
        mediaHost.showsOnlyActiveMedia = true
        mediaHost.falsingProtectionNeeded = true

        // Let's now initialize this view, which also creates the host view for us.
        mediaHost.init(MediaHierarchyManager.LOCATION_LOCKSCREEN)    
    }

```

QS初始化：

```
QSPanelController.java

    @Override
    public void onInit() {
        mMediaHost.setExpansion(1);
        mMediaHost.setShowsOnlyActiveMedia(false);
        mMediaHost.init(MediaHierarchyManager.LOCATION_QS);
        ....
    }
```

QQS 初始化：

```
QuickQSPanelController.java
    @Override
    protected void onInit() {
        super.onInit();
        mMediaHost.setExpansion(0.0f);
        mMediaHost.setShowsOnlyActiveMedia(true);
        mMediaHost.init(MediaHierarchyManager.LOCATION_QQS);
    }
```

```
MediaHost.init()
    MediaHierarchyManager.register()
        MediaHierarchyManager.createUniqueObjectHost() //生成 UniqueObjectHostView
            new UniqueObjectHostView()
        MediaHost.addVisibilityChangeListener()
        mediaHosts[mediaObject.location] = mediaObject // 添加到数组里面
        MediaHierarchyManager.updateDesiredLocation() //更新位置
```

## 添加流程


```
MediaDataCombineLatest.onMediaDeviceChanged()
    MediaDataCombineLatest.update()
        MediaDataCombineLatest.Listener.onMediaDataLoaded()
            MediaDataFilter.onMediaDataLoaded()
                MediaHost.listener.onMediaDataLoaded()
                    MediaHost.updateViewVisibility()
                        MediaHierarchyManager.register.updateDesiredLocation()
```

```
MediaHierarchyManager.updateDesiredLocation()
    MediaHierarchyManager.calculateLocation() // 计算当前应该处的位置
    MediaHierarchyManager.performTransitionToNewLocation() //切换到新的位置
        MediaHierarchyManager.updateTargetState() //更新需要绑定的View
            MediaHierarchyManager.getHost() // 根据位置获取Host View
        MediaHierarchyManager.cancelAnimationAndApplyDesiredState() //开始应用到新位置，不用动画
            MediaHierarchyManager.applyState()
                MediaHierarchyManager.updateHostAttachment()
                    MediaHierarchyManager.updateHostAttachment()
                        UniqueObjectHostView.addView()
```

```
//MediaHierarchyManager.kt
    private fun calculateLocation(): Int {
        if (blockLocationChanges) {
            // Keep the current location until we're allowed to again
            return desiredLocation
        }
        val onLockscreen = (!bypassController.bypassEnabled &&
            (statusbarState == StatusBarState.KEYGUARD ||
                statusbarState == StatusBarState.FULLSCREEN_USER_SWITCHER))
        val allowedOnLockscreen = notifLockscreenUserManager.shouldShowLockscreenNotifications()
        val location = when {
            qsExpansion > 0.0f && !onLockscreen -> LOCATION_QS
            qsExpansion > 0.4f && onLockscreen -> LOCATION_QS
            !hasActiveMedia -> LOCATION_QS
            onLockscreen && isTransformingToFullShadeAndInQQS() -> LOCATION_QQS
            onLockscreen && allowedOnLockscreen -> LOCATION_LOCKSCREEN
            else -> LOCATION_QQS
        }
        // When we're on lock screen and the player is not active, we should keep it in QS.
        // Otherwise it will try to animate a transition that doesn't make sense.
        if (location == LOCATION_LOCKSCREEN && getHost(location)?.visible != true &&
            !statusBarStateController.isDozing) {
            return LOCATION_QS
        }
        if (location == LOCATION_LOCKSCREEN && desiredLocation == LOCATION_QS &&
            collapsingShadeFromQS) {
            // When collapsing on the lockscreen, we want to remain in QS
            return LOCATION_QS
        }
        if (location != LOCATION_LOCKSCREEN && desiredLocation == LOCATION_LOCKSCREEN &&
            !fullyAwake) {
            // When unlocking from dozing / while waking up, the media shouldn't be transitioning
            // in an animated way. Let's keep it in the lockscreen until we're fully awake and
            // reattach it without an animation
            return LOCATION_LOCKSCREEN
        }
        return location
    }
```

```
    private fun getHost(@MediaLocation location: Int): MediaHost? {
        if (location < 0) {
            return null
        }
        return mediaHosts[location]
    }
```

## 切换动画流程

```
NotificationPanelViewController.updateQsExpansion()
    MediaHierarchyManager.setQsExpansion()
        MediaHierarchyManager.updateDesiredLocation()
```
