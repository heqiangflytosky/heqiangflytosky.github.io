---
title: SystemUI -- 媒体控制器
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍媒体控制器相关
date: 2021-11-16 12:00:00
---



## 概述

在 Android S上面分别有三个位置可以承载媒体控制器，分别时QQS、QS和锁屏，另外还有个动画切换的场景，这四个位置动态添加 MediaScrollView 来实现切换场景时媒体控制器的位置的变换。
如图：

<img src="/images/android-systemui-panel-media-controller/image.png" width="676" height="1134"/>

媒体控制器容器布局在 R.layout.media_carousel，媒体控制器布局是 R.layout.media_view。

```
UniqueObjectHostView
    FrameLayout
        MediaScrollView(R.id.media_carousel_scroller)
            LinearLayout(R.id.media_carousel)
                TransitionLayout(R.id.qs_media_controls)
                    Guideline(R.id.center_vertical_guideline)
                    FrameLayout(R.id.notification_media_progress_time)
                    TextView(R.id.media_elapsed_time)
                    SeekBar(R.id.media_progress_bar)
                    ......
        PageIndicator()
        ImageView(R.id.settings_cog)
```

相关的类：

 - MediaHierarchyManager
 - MediaHost
 - UniqueObjectHostView 承载媒体控制器的容器，它是 MediaHost 的一个变量，一个 MediaHost 对应一个 UniqueObjectHostView
 - MediaCarouselController
 - MediaControlPanel

在 MediaHierarchyManager 中定义了媒体控制器的四个位置场景。

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
         * 媒体控制器在ViewGroupOverlay，这时正在做手势切换动画
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
        mediaHost.showsOnlyActiveMedia = true // 设置是否常显示
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
        mMediaHost.setShowsOnlyActiveMedia(false); //当媒体不Active时不显示
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

MediaHost 初始化：

```
MediaHost.init()
    MediaHierarchyManager.register()
        MediaHierarchyManager.createUniqueObjectHost() //生成 UniqueObjectHostView
            new UniqueObjectHostView()
        MediaHost.addVisibilityChangeListener() //注册监听器，当MediaHost.updateViewVisibility()时回调
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
                        //位置有改变时添加View
                        (mediaFrame.parent as ViewGroup?)?.removeView(mediaFrame) // 先把mediaFrame从原来的位置移除
                        rootOverlay!!.add() //如果在做动画，就添加到 ViewGroupOverlay
                        UniqueObjectHostView.addView() // 动画完成添加到新的Host
```

## 切换动画流程

切换动画是在 ViewGroupOverlay 上面做的，根据手势的移动位置来计算出 mediaFrame 的显示区域，然后通过不断更新显示区域来实现移动的动画效果。
媒体控制器内部的联动动画是通过 TransitionLayout 来实现的。

```
NotificationPanelViewController.updateQsExpansion()
    MediaHierarchyManager.setQsExpansion()
        MediaHierarchyManager.updateDesiredLocation() // 更新location
        MediaHierarchyManager.getQSTransformationProgress() // 获取当前位移的进度，如果返回值大于0，才会走下面的位置更新等逻辑。下面介绍
        MediaHierarchyManager.updateTargetState() 
            MediaHierarchyManager.getTransformationProgress() // 获取当前移动的位置
                MediaHierarchyManager.getQSTransformationProgress()
                    return qsExpansion
            MediaHierarchyManager.interpolateBounds() // 根据进度计算 mediaFrame 的区域
        MediaHierarchyManager.applyTargetStateIfNotAnimating()
            MediaHierarchyManager.applyState()
               MediaCarouselController.setCurrentState()
                   MediaCarouselController.updatePlayerToState()
                       MediaViewController.setCurrentState()
                           TransitionLayoutController.getInterpolatedState() //根据起始位置和最终位置以及进度信息差值计算当前View的位置，透明度，缩放信息。
                           TransitionLayoutController.setState()//将当前状态应用到视图上
                               TransitionLayoutController.applyStateToLayout()
                                   TransitionLayout.setState()
                                       TransitionLayout.applyCurrentState()//更新视图
               mediaFrame.setLeftTopRightBottom() // 将计算的区域应用给 mediaFrame
```

```
PlayerViewHolder.kt

        val controlsIds = setOf(
                R.id.icon,
                R.id.app_name,
                R.id.album_art,
                R.id.header_title,
                R.id.header_artist,
                R.id.media_seamless,
                R.id.media_seamless_fallback,
                R.id.notification_media_progress_time,
                R.id.media_progress_bar,
                R.id.action0,
                R.id.action1,
                R.id.action2,
                R.id.action3,
                R.id.action4,
                R.id.icon
        )
        val gutsIds = setOf(
                R.id.remove_text,
                R.id.cancel,
                R.id.dismiss,
                R.id.settings
        )
```

## 方法详解

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
            // 非锁屏下刚开始下拉时，位置设置为 LOCATION_QS
            qsExpansion > 0.0f && !onLockscreen -> LOCATION_QS
            // 锁屏界面下拉超过0.4时，修改位置为 LOCATION_QS，锁屏状态下状态栏下滑
            qsExpansion > 0.4f && onLockscreen -> LOCATION_QS
            // 如果 QQS host 不可见，返回QS
            !hasActiveMedia -> LOCATION_QS
            // 锁屏界面切换到QQS时，设置位置为 LOCATION_QQS，比如：锁屏状态下下滑通知栏
            onLockscreen && isTransformingToFullShadeAndInQQS() -> LOCATION_QQS
            // 锁屏界面下如果允许在锁屏显示通知，那么就设置位置为 LOCATION_LOCKSCREEN
            onLockscreen && allowedOnLockscreen -> LOCATION_LOCKSCREEN
            else -> LOCATION_QQS
        }
        // 如果在锁屏界面，且媒体控制器不显示，那么就要设置在 LOCATION_QS
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

```
    private fun updateViewVisibility() {
        // 根据 showsOnlyActiveMedia 来为 visible 赋值
        // showsOnlyActiveMedia 在前面 HostView 初始化时配置
        state.visible = if (showsOnlyActiveMedia) {
            mediaDataManager.hasActiveMedia()
        } else {
            mediaDataManager.hasAnyMedia()
        }
        val newVisibility = if (visible) View.VISIBLE else View.GONE
        // 如果 MediaHost 可见性状态和 UniqueObjectHostView 不一致，就调用 VisibilityChangeListener 回调
        if (newVisibility != hostView.visibility) {
            hostView.visibility = newVisibility
            visibleChangedListeners.forEach {
                it.invoke(visible)
            }
        }
    }
```

```
    private fun getQSTransformationProgress(): Float {
        val currentHost = getHost(desiredLocation)
        val previousHost = getHost(previousLocation)
        // 如果 QQS 的Host是可见的，满足 desiredLocation 和 previousLocation 条件时，才会去计算位移进度
        // 也就是说如果QQS的HostView是不可见的，那么就不会去做这个动画
        if (hasActiveMedia && currentHost?.location == LOCATION_QS) {
            if (previousHost?.location == LOCATION_QQS) {
                if (previousHost.visible || statusbarState != StatusBarState.KEYGUARD) {
                    return qsExpansion
                }
            }
        }
        return -1.0f
    }
```

```
    // 以LOCATION_QQS的可见性来判断是否有 ActiveMedia
    private var hasActiveMedia: Boolean = false
        get() = mediaHosts.get(LOCATION_QQS)?.visible == true
```
