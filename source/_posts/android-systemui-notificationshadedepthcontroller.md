---
title: SystemUI -- NotificationShadeDepthController
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍 NotificationShadeDepthController
date: 2021-11-15 12:00:00
---



## 概述

NotificationShadeDepthController 主要用来实现面板窗口的毛玻璃模糊效果。
Android 11 上我们的毛玻璃使用的是原生逻辑，通过设置 backgroundBlurRadius，可以使该窗口之下的 layer 通过算法模糊合成。
我们定制的通知面板毛玻璃由背景图 + 毛玻璃模糊组成，背景图为一张带有透明度的纯色图，通过 ScrimController 的 setScrimBehindDrawable 方法为 `id/scrim_behind` 设置，作为下拉面板的背景。透明度变换逻辑在 mzPanelViewHeightUpdateAnimation 方法中通过动画进行展现，通过 mScrimController 的 setBlurAlpha 进行设值。

本文基于 Android S 代码。

## 实现原理


下拉面板窗口模糊变换的的核心代码在 onPanelExpansionChanged 回调中，其通过 BlurUtils 工具类进行一个变换即可实现。

模糊的实现在 `updateBlurCallback = Choreographer.FrameCallback` 中：

```
NotificationShadeDepthController.updateBlurCallback
    BlurUtils.applyBlur()
        SurfaceControl.setBackgroundBlurRadius()
    NotificationShadeWindowController.setBackgroundBlurRadius()
```

```
    fun applyBlur(viewRootImpl: ViewRootImpl?, radius: Int) {
        if (viewRootImpl == null || !viewRootImpl.surfaceControl.isValid ||
                !supportsBlursOnWindows()) {
            return
        }
        createTransaction().use {
            it.setBackgroundBlurRadius(viewRootImpl.surfaceControl, radius)
            it.apply()
        }
    }
```

模糊效果的实现很简单，只需要调用 `SurfaceControl.setBackgroundBlurRadius()` 方法，设置一个模糊半径 backgroundBlurRadius 就行了。这个值最终会保存到 layer_state_t 中。当 Surfaceflinger 进行 GPU 合成的时候，通过 OpenGL 进行 sharder 进行渲染的时候实现模糊效果。

## 实现流程

当下拉面板位置变化时，会回调 `NotificationShadeDepthController.onPanelExpansionChanged()` 方法，然后根据位置计算当前的模糊半径，通过 SpringAnimation 去做模糊变换动画，在动画执行过程中，回调 `updateBlurCallback` 去执行设置模糊半径的方法，最终实现模糊效果。

```
NotificationShadeDepthController.onPanelExpansionChanged() // 前面的调用流程参考前面文章介绍的 PanelViewController.setExpandedHeightInternal() 方法
    NotificationShadeDepthController.updateShadeAnimationBlur()
        NotificationShadeDepthController.animateBlur()
            shadeAnimation.animateTo()
    NotificationShadeDepthController.updateShadeBlur()
        shadeSpring.animateTo()
            SpringAnimation.animateToFinalPosition()
                setValue()
                    radius = value.toInt() // 设置模糊半径
                    NotificationShadeDepthController.scheduleUpdate()
                        choreographer.postFrameCallback(updateBlurCallback) // 执行 updateBlurCallback 更新模糊效果
```

## 额外的话

另外，Android S 为我们提供了新的 Window.setBackgroundBlurRadius() API 为窗口背景创建雾面玻璃效果。这个 API 可以设置模糊半径，以调整雾面密度和范围，平台只会对您的应用窗口边框内的背景内容应用模糊效果。还可以使用 blurBehindRadius 来模糊窗口后面的所有内容，从而为浮动窗口营造出深度效果。
