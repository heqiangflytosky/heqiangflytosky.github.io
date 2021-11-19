---
title: SystemUI -- NotificationShadeWindowView
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍 NotificationShadeWindowView
date: 2021-11-6 12:00:00
---

## 概述

NotificationShadeWindowView 是承载锁屏和下拉通知的界面，NotificationShadeWindowViewController 作为 NotificationShadeWindowView 控制器主要是作为事件的分发以及执行锁屏切换下拉通知(锁屏下滑)的操作。
事件处理大概逻辑在前面文章中已经有了介绍，本文主要介绍一些方法的实现细节。
本文基于原生 Android S 代码。

## 相关类介绍

DragDownHelper 一个工具类，处理锁屏时下拉通知栏区域展开通知中心的操作。

## NotificationShadeWindowView

NotificationShadeWindowView 主要处理锁屏状态下的下拉(非状态栏下拉)事件的处理或者下拉面板可见情况下事件的传递。
在 NotificationShadeWindowView 中定义了一个 InteractionEventHandler，它的实现在 NotificationShadeWindowViewController，用于处理 NotificationShadeWindowView 对于事件的操作。

## dispatchTouchEvent

如果 mInteractionEventHandler 返回不为空，就返回 mInteractionEventHandler 的处理结果，true 表示处理该事件，中断分发，false 表示继续分发。如果 mInteractionEventHandler 为空，交给父类来处理。

```
// NotificationShadeWindowView.java
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Boolean result = mInteractionEventHandler.handleDispatchTouchEvent(ev);
        result = result != null ? result : super.dispatchTouchEvent(ev);
        mInteractionEventHandler.dispatchTouchEventComplete();
        return result;
    }
```

```
// NotificationShadeWindowViewController.java

            public Boolean handleDispatchTouchEvent(MotionEvent ev) {
                if (mStatusBarView == null) {
                    return false;
                }
                boolean isDown = ev.getActionMasked() == MotionEvent.ACTION_DOWN;
                boolean isUp = ev.getActionMasked() == MotionEvent.ACTION_UP;
                boolean isCancel = ev.getActionMasked() == MotionEvent.ACTION_CANCEL;

                boolean expandingBelowNotch = mExpandingBelowNotch;
                if (isUp || isCancel) {
                    mExpandingBelowNotch = false;
                }

                // Reset manual touch dispatch state here but make sure the UP/CANCEL event still
                // gets
                // delivered.

                if (!isCancel && mService.shouldIgnoreTouch()) {
                    return false;
                }
                if (isDown) {
                    setTouchActive(true);
                    mTouchCancelled = false;
                } else if (ev.getActionMasked() == MotionEvent.ACTION_UP
                        || ev.getActionMasked() == MotionEvent.ACTION_CANCEL) {
                    setTouchActive(false);
                }
                if (mTouchCancelled || mExpandAnimationRunning) {
                    return false;
                }
                mFalsingCollector.onTouchEvent(ev);
                mGestureDetector.onTouchEvent(ev);
                mStatusBarKeyguardViewManager.onTouch(ev);
                if (mBrightnessMirror != null
                        && mBrightnessMirror.getVisibility() == View.VISIBLE) {
                    // 正在调节亮度条时，有其他手指操作，这个情况不做处理。
                    if (ev.getActionMasked() == MotionEvent.ACTION_POINTER_DOWN) {
                        return false;
                    }
                }
                if (isDown) {
                    mNotificationStackScrollLayoutController.closeControlsIfOutsideTouch(ev);
                }
                if (mStatusBarStateController.isDozing()) {
                    mService.mDozeScrimController.extendPulse();
                }
                // In case we start outside of the view bounds (below the status bar), we need to
                // dispatch
                // the touch manually as the view system can't accommodate for touches outside of
                // the
                // regular view bounds.
                if (isDown && ev.getY() >= mView.getBottom()) {
                    mExpandingBelowNotch = true;
                    expandingBelowNotch = true;
                }
                if (expandingBelowNotch) {
                    return mStatusBarView.dispatchTouchEvent(ev);
                }

                if (!mIsTrackingBarGesture && isDown
                        && mNotificationPanelViewController.isFullyCollapsed()) {
                    float x = ev.getRawX();
                    float y = ev.getRawY();
                    if (isIntersecting(mStatusBarView, x, y)) {
                        if (mService.isSameStatusBarState(WINDOW_STATE_SHOWING)) {
                            mIsTrackingBarGesture = true;
                            return mStatusBarView.dispatchTouchEvent(ev);
                        } else { // it's hidden or hiding, don't send to notification shade.
                            return true;
                        }
                    }
                } else if (mIsTrackingBarGesture) {
                    final boolean sendToNotification = mStatusBarView.dispatchTouchEvent(ev);
                    if (isUp || isCancel) {
                        mIsTrackingBarGesture = false;
                    }
                    return sendToNotification;
                }

                return null;
            }
```


## onInterceptTouchEvent

NotificationShadeWindowView 通过 InteractionEventHandler.shouldInterceptTouchEvent 判断是否需要拦截，拦截事件的场景为锁屏界面滑动屏幕下拉（不是状态栏下拉），这时会拦截Move事件，那么接下来的Move事件和UP事件就交给它的onTouchEvent来处理。来处理面板的整体滑动操作。
那么为什么从状态栏滑动时没有拦截呢？因为在 NotificationPanelViewController.onQsIntercept 处理Down事件时设置了 mView.getParent().requestDisallowInterceptTouchEvent(true)，后面的Move事件就不会走 NotificationShadeWindowView 的 onInterceptTouchEvent 了。来处理QS的滑动操作。

```
// NotificationShadeWindowView.java
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        boolean intercept = mInteractionEventHandler.shouldInterceptTouchEvent(ev);
        if (!intercept) {
            intercept = super.onInterceptTouchEvent(ev);
        }
        if (intercept) {
            mInteractionEventHandler.didIntercept(ev);
        }

        return intercept;
    }
```

```
// NotificationShadeWindowViewController.java
            @Override
            public boolean shouldInterceptTouchEvent(MotionEvent ev) {
                if (mStatusBarStateController.isDozing() && !mService.isPulsing()
                        && !mDockManager.isDocked()) {
                    // Capture all touch events in always-on.
                    return true;
                }
                boolean intercept = false;
                if (mNotificationPanelViewController.isFullyExpanded()
                        && mDragDownHelper.isDragDownEnabled()
                        && !mService.isBouncerShowing()
                        && !mStatusBarStateController.isDozing()) {
                    intercept = mDragDownHelper.onInterceptTouchEvent(ev);
                }

                return intercept;

            }
```

```
// DragDownHelper
    override fun onInterceptTouchEvent(event: MotionEvent): Boolean {
        val x = event.x
        val y = event.y
        when (event.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                draggedFarEnough = false
                isDraggingDown = false
                startingChild = null
                initialTouchY = y
                initialTouchX = x
            }
            MotionEvent.ACTION_MOVE -> {
                val h = y - initialTouchY
                val touchSlop = if (event.classification
                        == MotionEvent.CLASSIFICATION_AMBIGUOUS_GESTURE)
                    touchSlop * slopMultiplier
                else
                    touchSlop
                if (h > touchSlop && h > Math.abs(x - initialTouchX)) {
                    falsingCollector.onNotificationStartDraggingDown()
                    isDraggingDown = true
                    // 获取手指触摸点的 startingChild
                    captureStartingChild(initialTouchX, initialTouchY)
                    initialTouchY = y
                    initialTouchX = x
                    dragDownCallback.onDragDownStarted()
                    dragDownAmountOnStart = dragDownCallback.dragDownAmount
                    // 如果触摸点在通知中心区域内，或者使能 isDragDownAnywhereEnabled 那么就拦截move事件
                    return startingChild != null || dragDownCallback.isDragDownAnywhereEnabled
                }
            }
        }
        return false
    }
```

因此，在通常情况下的锁屏界面只会对Move事件拦截，条件时如果 isDragDownAnywhereEnabled 使能，那么所有的Move事件都会拦截，否则，只会拦截触摸点在通知位置的事件。

## onTouchEvent

```
// NotificationShadeWindowView.java
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        boolean handled = mInteractionEventHandler.handleTouchEvent(ev);

        if (!handled) {
            handled = super.onTouchEvent(ev);
        }

        if (!handled) {
            mInteractionEventHandler.didNotHandleTouchEvent(ev);
        }

        return handled;
    }
```

```
// NotificationShadeWindowViewController.java
            @Override
            public boolean handleTouchEvent(MotionEvent ev) {
                boolean handled = false;
                if (mStatusBarStateController.isDozing()) {
                    handled = !mService.isPulsing();
                }
                if ((mDragDownHelper.isDragDownEnabled() && !handled)
                        || mDragDownHelper.isDraggingDown()) {
                    // we still want to finish our drag down gesture when locking the screen
                    handled = mDragDownHelper.onTouchEvent(ev);
                }

                return handled;
            }
```

```
// DragDownHelper
    override fun onTouchEvent(event: MotionEvent): Boolean {
        if (!isDraggingDown) {
            return false
        }
        val x = event.x
        val y = event.y
        when (event.actionMasked) {
            MotionEvent.ACTION_MOVE -> {
                lastHeight = y - initialTouchY
                captureStartingChild(initialTouchX, initialTouchY)
                dragDownCallback.dragDownAmount = lastHeight + dragDownAmountOnStart
                if (startingChild != null) {
                    //展开触摸点所在的通知
                    handleExpansion(lastHeight, startingChild!!)
                }
                if (lastHeight > minDragDistance) {
                    // 如果滑动距离大于minDragDistance，设置 draggedFarEnough
                    if (!draggedFarEnough) {
                        draggedFarEnough = true
                        dragDownCallback.onCrossedThreshold(true)
                    }
                } else {
                    if (draggedFarEnough) {
                        draggedFarEnough = false
                        dragDownCallback.onCrossedThreshold(false)
                    }
                }
                return true
            }
            MotionEvent.ACTION_UP -> if (!falsingManager.isUnlockingDisabled && !isFalseTouch &&
                    dragDownCallback.canDragDown()) {
                // 这有个重要变量 isFalseTouch，来区分是否时误触还是正常的下拉操作
                dragDownCallback.onDraggedDown(startingChild, (y - initialTouchY).toInt())
                if (startingChild != null) {
                    expandCallback.setUserLockedChild(startingChild, false)
                    startingChild = null
                }
                isDraggingDown = false
            } else {
                stopDragging()
                return false
            }
            MotionEvent.ACTION_CANCEL -> {
                stopDragging()
                return false
            }
        }
        return false
    }
    
    private val isFalseTouch: Boolean
        get() {
            return if (!dragDownCallback.isFalsingCheckNeeded) {
                false
            } else {
                // 看下拉的距离是否足够远，draggedFarEnough在处理MOVE事件时计算
                falsingManager.isFalseTouch(Classifier.NOTIFICATION_DRAG_DOWN) || !draggedFarEnough
            }
        }
```

```
DragDownHelper.onTouchEvent
    ACTION_MOVE
        LockscreenShadeTransitionController.dragDownAmount // 设置滑动的距离
            NotificationPanelViewController.setTransitionToFullShadeAmount()
                NotificationPanelViewController.updateQsExpansion() // 设置QS高度，具体看NotificationPanelView一文
            QSFragment.setTransitionToFullShadeAmount()
                QSFragment.setQsExpansion()
        DragDownHelper.handleExpansion() //展开触摸点所在的通知
        LockscreenShadeTransitionController.onCrossedThreshold()
            NotificationStackScrollLayoutController.setDimmed()// 设置背景
                NotificationStackScrollLayout.setDimmed()
    ACTION_UP //1.展开面板，或者 2.回到原位置
        LockscreenShadeTransitionController.onDraggedDown()
            DragDownHelper.canDragDown()
            NotificationStackScrollLayoutController.isInLockedDownShade() //是否处在通知中心已经全部展示，但是通知内容还没有准备好的状态
            StatusBar.dismissKeyguardThenExecute()
            LockscreenShadeTransitionController.goToLockedShadeInternal()
                SysuiStatusBarStateController.setState(StatusBarState.SHADE_LOCKED) // 切换为 SHADE_LOCKED 状态
            animationHandler
                NotificationPanelViewController.animateToFullShade() // 展开通知中心
                    NotificationStackScrollLayoutController.goToFullShade()
                    View.requestLayout()//重新布局
                NotificationPanelViewController.setTransitionToFullShadeAmount()
        DragDownHelper.stopDragging() // 取消拖动
            LockscreenShadeTransitionController.onDragDownReset()
            
```

```
    internal fun onDraggedDown(startingChild: View?, dragLengthY: Int) {
        if (canDragDown()) {
            if (nsslController.isInLockedDownShade()) {
                statusBarStateController.setLeaveOpenOnKeyguardHide(true)
                statusbar.dismissKeyguardThenExecute(OnDismissAction {
                    nextHideKeyguardNeedsNoAnimation = true
                    false
                },
                        null /* cancelRunnable */, false /* afterKeyguardGone */)
            } else {
                lockscreenGestureLogger.write(
                        MetricsEvent.ACTION_LS_SHADE,
                        (dragLengthY / displayMetrics.density).toInt(),
                        0 /* velocityDp */)
                lockscreenGestureLogger.log(LockscreenUiEvent.LOCKSCREEN_PULL_SHADE_OPEN)
                if (!ambientState.isDozing() || startingChild != null) {
                    // go to locked shade while animating the drag down amount from its current
                    // value
                    val animationHandler = { delay: Long ->
                        if (startingChild is ExpandableNotificationRow) {
                            startingChild.onExpandedByGesture(
                                    true /* drag down is always an open */)
                        }
                        notificationPanelController.animateToFullShade(delay)
                        notificationPanelController.setTransitionToFullShadeAmount(0f,
                                true /* animated */, delay)

                        // Let's reset ourselves, ready for the next animation

                        // changing to shade locked will make isInLockDownShade true, so let's
                        // override that
                        forceApplyAmount = true
                        // Reset the behavior. At this point the animation is already started
                        dragDownAmount = 0f
                        forceApplyAmount = false
                    }
                    val cancelRunnable = Runnable { setDragDownAmountAnimated(0f) }
                    goToLockedShadeInternal(startingChild, animationHandler, cancelRunnable)
                }
            }
        } else {
            setDragDownAmountAnimated(0f)
        }
    }
```
