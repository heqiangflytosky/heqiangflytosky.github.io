---
title: SystemUI -- NotificationPanelView
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍NotificationPanelView
date: 2021-11-7 10:00:00
---


## 概述

NotificationPanelView 是Panel的子类，它继承自 FrameLayout，提供了一个承载下拉通知面板的容器。NotificationPanelView 主要作用就是用 PanelViewController.TouchHandler 来进行一些事件处理。
本文基于原生 Android S 代码。

## NotificationPanelView & PanelView 

TouchHandler 主要来处理 onInterceptTouchEvent() 和 onTouch()

```
public abstract class PanelView extends FrameLayout {

    public void setOnTouchListener(PanelViewController.TouchHandler touchHandler) {
        super.setOnTouchListener(touchHandler);
        mTouchHandler = touchHandler;
    }
    
    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        return mTouchHandler.onInterceptTouchEvent(event);
    }

}
```

NotificationPanelViewController 和 PanelViewController 都生成了自己的 TouchHandler。

## NotificationPanelViewController & PanelViewController

NotificationPanelViewController主要是进行QS面板的整体操作，比如显示，隐藏和整体滑动等等。 

mOverExpansion:QS可以回弹下拉的高度
mQsExpanded:QS是否可见，这时QS正在展开或者全部展开，此时展开高度大于 mQsMinExpansionHeight，也就是在只显示QQS到全部显示QS之间的状态变化时为true。
mQsMinExpansionHeight:QS展开的最小高度：如果是锁屏模式，就是0，全部隐藏了。如果不是就是 QuickStatusBarHeader 的高度。
mQsMaxExpansionHeight：QS展开的最大高度，通过 getDesiredHeight() 获取，一般是QSContainerImpl 的高度。
mQsExpansionHeight：QS的当前实时高度，一般通过 setQsExpansion() 方法设置。
mQsFullyExpanded：QS是否完全展开，展开高度等于mQsMaxExpansionHeight
mExpanding：面板是否在展开
mClosing：是否正在关闭面板
mQsTracking：是否在触摸快捷面板（展开、收起快捷面板部分逻辑）
mTracking：为true时表示 NotificationPanelViewController 正在做QS和通知中心的收起或者展示动画。
mExpandedHeight:整个下拉面板的高度。为初始位置和手势滑动距离之和。原生用的时渐隐渐现动画。
mExpandedFraction:下拉面板的幅度,展开高度除于整体高度。

getMaxPanelHeight():它的值有两种情况，一个是从只显示QQS到完全显示QS的状态，这个时候的值时经过calculatePanelHeightQsExpanded()计算，一般为mQsMaxExpansionHeight加上NotificationShelf的高度；其他状态时经过calculatePanelHeightShade()计算，其实时通知面板的高度;
calculateNotificationsTopPadding()：计算通知中心的最上面的通知距离顶部的距离
setQSClippingBounds()： 设置QS的绘制区域，下面介绍。
expand()
collapse():收起
positionClockAndNotifications()：计算锁屏上时钟和通知中心的位置
flingSettings()：做通知中心滑动松手后的动画，开始QS的展开或者收起动画，下面介绍。
fling():释放手指后的动画。
flingExpands():计算fling之后最终的状态，看是否时展开或者收起。下面介绍。
flingToHeight()：下拉面板整体滚动到指定高度，下面介绍。

## 方法详解

### flingSettings

通知中心做fling动画时调用，会展开QS或者折叠QS只显示QQS。

```
// NotificationPanelViewController.java
    protected void flingSettings(float vel, int type, final Runnable onFinishRunnable,
            boolean isClick) {
        float target;
        // 根据fling类型确定最终的高度
        switch (type) {
            case FLING_EXPAND: // 展开QS
                target = mQsMaxExpansionHeight;
                break;
            case FLING_COLLAPSE: // 折叠QS
                target = mQsMinExpansionHeight;
                break;
            case FLING_HIDE: // 直接隐藏
            default:
                if (mQs != null) {
                    mQs.closeDetail();
                }
                target = 0;
        }
        if (target == mQsExpansionHeight) {
            if (onFinishRunnable != null) {
                onFinishRunnable.run();
            }
            traceQsJank(false /* startTracing */, type != FLING_EXPAND /* wasCancelled */);
            return;
        }

        // If we move in the opposite direction, reset velocity and use a different duration.
        boolean oppositeDirection = false;
        boolean expanding = type == FLING_EXPAND;
        if (vel > 0 && !expanding || vel < 0 && expanding) {
            vel = 0;
            oppositeDirection = true;
        }
        ValueAnimator animator = ValueAnimator.ofFloat(mQsExpansionHeight, target);
        if (isClick) {
            animator.setInterpolator(Interpolators.TOUCH_RESPONSE);
            animator.setDuration(368);
        } else {
            mFlingAnimationUtils.apply(animator, mQsExpansionHeight, target, vel);
        }
        if (oppositeDirection) {
            animator.setDuration(350);
        }
        animator.addUpdateListener(animation -> {
            // 更新QS的高度以及通知中心位置，后面介绍
            setQsExpansion((Float) animation.getAnimatedValue());
        });
        animator.addListener(new AnimatorListenerAdapter() {
            private boolean mIsCanceled;
            @Override
            public void onAnimationStart(Animator animation) {
                notifyExpandingStarted();
            }

            @Override
            public void onAnimationCancel(Animator animation) {
                mIsCanceled = true;
            }

            @Override
            public void onAnimationEnd(Animator animation) {
                mQSAnimatingHiddenFromCollapsed = false;
                mAnimatingQS = false;
                notifyExpandingFinished();
                mNotificationStackScrollLayoutController.resetCheckSnoozeLeavebehind();
                mQsExpansionAnimator = null;
                if (onFinishRunnable != null) {
                    onFinishRunnable.run();
                }
                traceQsJank(false /* startTracing */, mIsCanceled /* wasCancelled */);
            }
        });
        // Let's note that we're animating QS. Moving the animator here will cancel it immediately,
        // so we need a separate flag.
        mAnimatingQS = true;
        // 开始动画
        animator.start();
        mQsExpansionAnimator = animator;
        mQsAnimatorExpand = expanding;
        mQSAnimatingHiddenFromCollapsed = computeQsExpansionFraction() == 0.0f && target == 0;
    }
```

NotificationPanelViewController.closeQs()
NotificationPanelViewController.collapse()
NotificationPanelViewController.expand()
PanelBar.collapsePanel()


### handleQsTouch()

用来更新 QS 的显示高度以及更新通知中心的位置。
处理触摸事件，在面板的子view没有处理的事件，都会在这里面进行处理。比如，通知中心的空白区域的滑动，在QS上上划呼出通知中心过程。

```
NotificationPanelViewController.handleQsTouch() // QS处理
    ACTION_DOWN
        mQsTracking = true
        NotificationPanelViewController.onQsExpansionStarted()
            NotificationPanelViewController.setQsExpansion() // 设置QS显示高度，更新QQS的可见性, 设置QS的绘制区域,更新通知中心的偏移,后面详细介绍
    NotificationPanelViewController.handleQsDown()
    NotificationPanelViewController.onQsTouch()
        MotionEvent.ACTION_DOWN
            mQsTracking = true
            NotificationPanelViewController.onQsExpansionStarted()
        MotionEvent.ACTION_MOVE
            NotificationPanelViewController.setQsExpansion() // 设置QS显示高度
        MotionEvent.ACTION_UP
            NotificationPanelViewController.flingQsWithCurrentVelocity()
            
```

### setQsExpansion()

设置QS高度以及通知中心位置
1.设置 QS 的绘制区域
2.执行QSTile的动画
3.设置通知中心的位置

```
NotificationPanelViewController.setQsExpansion()
    NotificationPanelViewController.updateQsExpansion() // 设置QS高度
        QSFragment.setQsExpansion()
            QSAnimator.startAlphaAnimation()
            QSContainerImpl.setExpansion()
            QSAnimator.setPosition()
                TouchAnimator.setPosition()
                    QSAnimator.onAnimationStarted()
                         QSAnimator.updateQQSVisibility() // 更新QQS的可见性
        ScrimController.setQsPosition() //设置ScrimView，比如通知中心背景透明度等
            ScrimController.applyAndDispatchState()
                ScrimController.setOrAdaptCurrentAnimation(mScrimBehind)
                    ScrimController.updateScrimColor()
                        ScrimView.setTint()
                        ScrimView.setViewAlpha()
        NotificationPanelViewController.setQSClippingBounds() // 设置QS的绘制区域，包括scrimView，QS 区域
            NotificationPanelViewController.applyQSClippingBounds()
                NotificationPanelViewController.applyQSClippingImmediately()
                    QSFragment.setFancyClipping()
                        QSContainerImpl.setFancyClipping()
                            QSContainerImpl.updateClippingPath()
                                View.invalidate()
    NotificationPanelViewController.requestScrollerTopPaddingUpdate() //更新通知中心的偏移
        NotificationStackScrollLayoutController.updateTopPadding()
            NotificationStackScrollLayout.updateTopPadding() // 更新通知中心的位置，后面详细介绍
```

### setExpandedHeightInternal()

设置QS展开的高度

```
PanelViewController.setExpandedHeightInternal()
    NotificationPanelViewController.onHeightUpdated()
        NotificationPanelViewController.updateExpandedHeight()
            NotificationStackScrollLayoutController.setExpandedHeight() // 更新通知栏高度
                NotificationStackScrollLayout.setExpandedHeight()
            NotificationPanelViewController.updateKeyguardBottomAreaAlpha()
            NotificationPanelViewController.updateBigClockAlpha()
            NotificationPanelViewController.updateStatusBarIcons()
        NotificationPanelViewController.updateHeader()
        NotificationPanelViewController.updateNotificationTranslucency()
        NotificationPanelViewController.updatePanelExpanded()
    PanelViewController.notifyBarPanelExpansionChanged()
        PhoneStatusBarView.panelExpansionChanged() // PhoneStatusBarView 更新状态
            PanelBar.panelExpansionChanged()
                if fullyOpened
                PanelBar.go(STATE_OPEN)
                PhoneStatusBarView.onPanelFullyOpened()
                else if fullyClosed
                PanelBar.go(STATE_CLOSED)
                PhoneStatusBarView.onPanelCollapsed()
                    post(mHideExpandedRunnable)
        NotificationShadeDepthController.onPanelExpansionChanged() //更新毛玻璃背景
            NotificationShadeDepthController.updateShadeBlur()
        StatusBarKeyguardViewManager.onPanelExpansionChanged() // 在锁屏界面时更新 KeyguardBouncer View
            KeyguardBouncer.setExpansion()
            KeyguardBouncer.show()
```

### setQSClippingBounds()

设置QS的绘制区域
更新 scrimView QS 区域，

```
NotificationPanelViewController.setQSClippingBounds()
    NotificationPanelViewController.applyQSClippingBounds()
        NotificationPanelViewController.applyQSClippingImmediately()
            NotificationPanelViewController.updateQsFrameTranslation() // 设置 R.id.qs_frame 的偏移
                View.setTranslationY()
            QSFragment.setFancyClipping() // QS 的剪切区域
                QSContainerImpl.setFancyClipping()
                    QSContainerImpl.updateClippingPath()
                        View.invalidate()
            ScrimController.setNotificationsBounds()
                mNotificationsScrim.setDrawableBounds() // 通知栏背景的剪切区域
                mScrimBehind.setBottomEdgePosition()
            ScrimController.setScrimCornerRadius()
            KeyguardStatusBarView.setTopClipping()
            NotificationStackScrollLayoutController.setRoundedClippingBounds()
```

### flingToHeight()

flingToHeight()：下拉面板整体滚动到指定高度，一般是UP事件后下拉面板的滚动动画。

```
NotificationPanelViewController.flingToHeight()
    PanelViewController.flingToHeight()
        PanelViewController.createHeightAnimator()
            PanelViewController.createHeightAnimator.onAnimationUpdate
                PanelViewController.setOverExpansionInternal()
                PanelViewController.setExpandedHeightInternal() //设置QS展开的高度
        PanelViewController.flingToHeight.onAnimationEnd()
            PanelViewController.onFlingEnd()
                PanelViewController.notifyBarPanelExpansionChanged()
                    PhoneStatusBarView.panelExpansionChanged()
                        PanelBar.panelExpansionChanged()
                            PanelBar.go(STATE_OPEN)
                            PhoneStatusBarView.onPanelFullyOpened() // 全部展开
                            PanelBar.go(STATE_CLOSED)
                            PhoneStatusBarView.onPanelCollapsed()
                                post(mHideExpandedRunnable) // 隐藏面板，处理一些隐藏面板后的其他操作，比如解锁等
```

### flingExpands()

计算下拉面板fling之后最终的状态，看是否是展开或者收起。

```
//NotificationPanelViewController.java

    protected boolean flingExpands(float vel, float vectorVel, float x, float y) {
        // 去 PanelViewController 做判断
        boolean expands = super.flingExpands(vel, vectorVel, x, y);

        // 如果正在做QS的扩展动画，就要保持最终状态时扩展
        if (mQsExpansionAnimator != null) {
            expands = true;
        }
        return expands;
    }
```

```
// PanelViewController.java
    protected boolean flingExpands(float vel, float vectorVel, float x, float y) {
        if (mFalsingManager.isUnlockingDisabled()) {
            return true;
        }

        @Classifier.InteractionType int interactionType = vel > 0
                ? QUICK_SETTINGS : (
                        mKeyguardStateController.canDismissLockScreen() ? UNLOCK : BOUNCER_UNLOCK);

        if (isFalseTouch(x, y, interactionType)) {
            return true;
        }
        //如果矢量速度来判断，如果小于某个定值，就再调用 shouldExpandWhenNotFlinging()
        if (Math.abs(vectorVel) < mFlingAnimationUtils.getMinVelocityPxPerSecond()) {
            return shouldExpandWhenNotFlinging();
        } else {
            // 此时判断Y方向滑动速度的方向，下划就是展开，上划收起
            return vel > 0;
        }
    }

```

```
//NotificationPanelViewController.java
    @Override
    protected boolean shouldExpandWhenNotFlinging() {
        if (super.shouldExpandWhenNotFlinging()) {
            return true;
        }
        if (mAllowExpandForSmallExpansion) {
            // When we get a touch that came over from launcher, the velocity isn't always correct
            // Let's err on expanding if the gesture has been reasonably slow
            long timeSinceDown = SystemClock.uptimeMillis() - mDownTime;
            return timeSinceDown <= MAX_TIME_TO_OPEN_WHEN_FLINGING_FROM_LAUNCHER;
        }
        return false;
    }
```

```
// PanelViewController.java
    protected boolean shouldExpandWhenNotFlinging() {
        return getExpandedFraction() > 0.5f;
    }
```


