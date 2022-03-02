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
关于事件分发流程和下拉交互部分参考前面介绍，本文只介绍一些方法实现的细节。本文基于原生 Android S 代码。

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

PanelViewController主要是进行QS面板的整体操作，比如显示，隐藏和整体滑动等等。 NotificationPanelViewController 可以处理通知中心的移动操作，此时 mTracking 为 true。

mOverExpansion:QS可以回弹下拉的高度
mQsExpanded:QS是否可见，这时QS正在展开或者全部展开，此时展开高度大于 mQsMinExpansionHeight，也就是在只显示QQS到全部显示QS之间的状态变化时为true。
mQsMinExpansionHeight:QS展开的最小高度：如果是锁屏模式，就是0，全部隐藏了。如果不是就是 QuickStatusBarHeader 的高度。
mQsMaxExpansionHeight：QS展开的最大高度，通过 getDesiredHeight() 获取，一般是QSContainerImpl 的高度。
mQsExpansionHeight：QS的当前实时高度，其实就是没有被通知中心覆盖住的QS部分。一般通过 setQsExpansion() 方法设置。一般为初始位置加手势移动的距离。
mQsFullyExpanded：QS是否完全展开，展开高度等于mQsMaxExpansionHeight
mExpanding：面板是否在展开
mClosing：是否正在关闭面板
mQsTracking：是否在触摸快捷面板（展开、收起快捷面板部分逻辑）
mTracking：为true时表示 NotificationPanelViewController 来处理手势移动，此时可能正在做QS和通知中心的收起或者展示动画。false时表示NotificationPanelViewController不处理手势移动，可能交给 PanelViewController处理下面面板整体操作。
mExpandedHeight:整个下拉面板的高度。为初始位置和手势滑动距离之和。原生用的时渐隐渐现动画。
mExpandedFraction:下拉面板的幅度,展开高度除于整体高度。
mQsExpandImmediate：下拉后是QS展开状态，用于双指操作
mTwoFingerQsExpandPossible：表示当前状态可以用于双指操作

getMaxPanelHeight():它的值有两种情况，一个是从只显示QQS到完全显示QS的状态，或者双指从状态栏下拉时，这个时候的值时经过calculatePanelHeightQsExpanded()计算，一般为mQsMaxExpansionHeight加上NotificationShelf的高度，因为它们的最终状态是全部显示QS；其他状态时经过calculatePanelHeightShade()计算，其实时通知面板的高度;
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

setQsExpansion() 是一个很关键的方法，用来设置QS高度以及通知中心位置
1.更新QS和QQS的可见性
2.设置 QS 的绘制区域
3.执行QSTile的动画
4.更新媒体控制器的位置
5.更新通知中心背景
6.设置通知中心的位置

```
NotificationPanelViewController.setQsExpansion()
    NotificationPanelViewController.setQsExpanded() // 设置QS是否展开,调用各个控制器更新UI
        NotificationPanelViewController.updateQsState()
            QSFragment.setExpanded()
                QSFragment.updateQsState()
                   QuickStatusBarHeader.setVisibility()
                   QuickStatusBarHeader.setExpanded()
                   QSFooter.setVisibility()
                   QSFooter.setExpanded()
                   QSPanelController.setVisibility() // 设置QS的可见性
                       QSPanel.setVisibility()
        PanelViewController.requestPanelHeightUpdate()
            PanelViewController.setExpandedHeight()
                PanelViewController.setExpandedHeightInternal() // 下面详解
        StatusBar.setQsExpanded()
        KeyguardBypassController.setQSExpanded()
        StatusBarKeyguardViewManager.setQsExpanded()
        LockIconViewController.setQsExpanded()
    mQsExpansionHeight = height // 更新 mQsExpansionHeight 变量
    NotificationPanelViewController.updateQsExpansion() // 设置QS高度
        QSFragment.setQsExpansion()
            QSAnimator.startAlphaAnimation()
            QSContainerImpl.setExpansion()
            QSAnimator.setPosition()
                TouchAnimator.setPosition()
                    QSAnimator.onAnimationStarted()
                         QSAnimator.updateQQSVisibility() // 更新QQS的可见性
        MediaHierarchyManager.setQsExpansion() //媒体控制器更新位置
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
        NotificationStackScrollLayoutController.setQsExpansionFraction() // 更新通知中心
        NotificationShadeDepthController.setQsPanelExpansion()
    NotificationPanelViewController.requestScrollerTopPaddingUpdate() //更新通知中心的偏移
        NotificationPanelViewController.calculateNotificationsTopPadding()
        NotificationStackScrollLayoutController.updateTopPadding()
            NotificationStackScrollLayout.updateTopPadding() // 更新通知中心的位置，后面详细介绍
```

调用的地方主要有下面几个场景：

根据初始位置+手势滑动距离来设置：

显示QQS时，

```
// NotificationPanelViewController.java
        public void onLayoutChange(

            } else if (!mQsExpanded && mQsExpansionAnimator == null) {
                setQsExpansion(mQsMinExpansionHeight + mLastOverscroll);
            }
        }
```

QQS切换QS时

```
// NotificationPanelViewController.java
            case MotionEvent.ACTION_MOVE:
                setQsExpansion(h + mInitialHeightOnTouch);
```


双指滑动：

```
// NotificationPanelViewController.java
    protected void onHeightUpdated(float expandedHeight) {
        if (mQsExpandImmediate || mQsExpanded && !mQsTracking && mQsExpansionAnimator == null
                && !mQsExpansionFromOverscroll) {
            ......
            float
                    targetHeight =
                    mQsMinExpansionHeight + t * (mQsMaxExpansionHeight - mQsMinExpansionHeight);
            setQsExpansion(targetHeight);
        }
    }
```

### setExpandedHeightInternal()

更新mExpandedHeight，整个下拉面板的高度

```
PanelViewController.setExpandedHeightInternal()
    mExpandedHeight = h
    mExpandedFraction = mExpandedHeight/maxPanelHeight
    NotificationPanelViewController.onHeightUpdated()
        NotificationPanelViewController.positionClockAndNotifications()：在QS没有展开的情况下，计算时钟(锁屏时)和通知中心的位置
            NotificationPanelViewController.requestScrollerTopPaddingUpdate()
                NotificationStackScrollLayoutController.updateTopPadding()
                    NotificationStackScrollLayout.updateTopPadding() // 更新通知中心的位置，后面详细介绍
        NotificationPanelViewController.setQsExpansion() // 计算QS展开高度，在
        NotificationPanelViewController.updateExpandedHeight()
            NotificationStackScrollLayoutController.setExpandedHeight() // 更新通知栏高度
                NotificationStackScrollLayout.setExpandedHeight()
            NotificationPanelViewController.updateKeyguardBottomAreaAlpha()
            NotificationPanelViewController.updateBigClockAlpha()
            NotificationPanelViewController.updateStatusBarIcons()
        NotificationPanelViewController.updateHeader()
        NotificationPanelViewController.updateNotificationTranslucency()
        NotificationPanelViewController.updatePanelExpanded()
    PanelViewController.notifyBarPanelExpansionChanged() // 通知 PhoneStatusBarView NotificationShadeDepthController StatusBarKeyguardViewManager 做对应更新，看下面介绍
        PhoneStatusBarView.panelExpansionChanged() // PhoneStatusBarView 更新状态
            PanelBar.panelExpansionChanged()
                PanelBar.updateVisibility()
                    NotificationPanelView.setVisibility()//设置 NotificationPanelView 可见性
                PanelBar.go(STATE_OPENING)
                    StatusBar.makeExpandedVisible() //设置Shade可见，然后可以接收事件
                        NotificationShadeWindowControllerImpl.setPanelVisible()
                            NotificationShadeWindowControllerImpl.apply()
                                NotificationShadeWindowControllerImpl.applyVisibility()
                                    NotificationShadeView.setVisibility() // 设置NotificationShadeView可见性
                if fullyOpened
                PanelBar.go(STATE_OPEN)
                PhoneStatusBarView.onPanelFullyOpened() // 全部展开
                else if fullyClosed
                PanelBar.go(STATE_CLOSED)
                PhoneStatusBarView.onPanelCollapsed()
                    post(mHideExpandedRunnable) // 隐藏面板，处理一些隐藏面板后的其他操作，比如解锁等
                        StatusBar.makeExpandedInvisible()
                            NotificationShadeWindowControllerImpl.setPanelVisible(false) // 隐藏面板
                                NotificationShadeWindowControllerImpl.apply()
                                    NotificationShadeWindowControllerImpl.applyVisibility()
                                        NotificationShadeWindowView.setVisibility() // 更新面板可见性
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
            ScrimController.setNotificationsBounds() // 设置通知中心的背景区域
                mNotificationsScrim.setDrawableBounds() // 通知中心背景的剪切区域
                mScrimBehind.setBottomEdgePosition()
            ScrimController.setScrimCornerRadius()
            KeyguardStatusBarView.setTopClipping()
            NotificationStackScrollLayoutController.setRoundedClippingBounds() //更新通知中心的显示区域
```

### flingToHeight()

flingToHeight()：下拉面板整体滚动到指定高度，一般是UP事件后下拉面板的滚动动画。可以是展开或者折叠下拉面板。

```
NotificationPanelViewController.flingToHeight()
    PanelViewController.flingToHeight()
        PanelViewController.createHeightAnimator()
            PanelViewController.createHeightAnimator.onAnimationUpdate
                PanelViewController.setOverExpansionInternal()
                PanelViewController.setExpandedHeightInternal() //设置QS展开的高度
        PanelViewController.flingToHeight.onAnimationEnd()
            PanelViewController.onFlingEnd()
                PanelViewController.notifyBarPanelExpansionChanged()  // 参考上面 setExpandedHeightInternal() 的介绍
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

### notifyBarPanelExpansionChanged()

通知 PhoneStatusBarView NotificationShadeDepthController StatusBarKeyguardViewManager 做对应更新。

```
PanelViewController.notifyBarPanelExpansionChanged() 
    PhoneStatusBarView.panelExpansionChanged() // PhoneStatusBarView 更新状态
        PanelBar.panelExpansionChanged()
            PanelBar.updateVisibility()
                NotificationPanelView.setVisibility()//设置 NotificationPanelView 可见
            PanelBar.go(STATE_OPENING) //设置正在打开状态
            PhoneStatusBarView.onPanelPeeked()
                StatusBar.makeExpandedVisible() //设置Shade可见，然后可以接收事件
                    NotificationShadeWindowControllerImpl.setPanelVisible()
                        NotificationShadeWindowControllerImpl.apply()
                            NotificationShadeWindowControllerImpl.applyVisibility()
                                NotificationShadeView.setVisibility() // 设置NotificationShadeView可见
            if fullyOpened
            PanelBar.go(STATE_OPEN)
            PhoneStatusBarView.onPanelFullyOpened() // 全部展开
            else if fullyClosed
            PanelBar.go(STATE_CLOSED)
            PhoneStatusBarView.onPanelCollapsed()
                post(mHideExpandedRunnable) // 隐藏面板，处理一些隐藏面板后的其他操作，比如解锁等
                    StatusBar.makeExpandedInvisible()
                        NotificationShadeWindowControllerImpl.setPanelVisible(false) // 隐藏面板
                            NotificationShadeWindowControllerImpl.apply()
                                NotificationShadeWindowControllerImpl.applyVisibility()
                                    NotificationShadeWindowView.setVisibility() // 更新面板可见性
    NotificationShadeDepthController.onPanelExpansionChanged() //更新毛玻璃背景
        NotificationShadeDepthController.updateShadeBlur()
    StatusBarKeyguardViewManager.onPanelExpansionChanged() // 在锁屏界面时更新 KeyguardBouncer View
        KeyguardBouncer.setExpansion()
        KeyguardBouncer.show()
```

### calculateNotificationsTopPadding()

calculateNotificationsTopPadding() 方法比较简单，但是比较关键的方法，它是通过调用 requestScrollerTopPaddingUpdate() 方法来更新通知中心位置的时候调用的，用来计算通知中心距屏幕顶部的间距。

```
// NotificationPanelViewController.java
    private float calculateNotificationsTopPadding() {
        // 如果当前是QS和通知中心分割两列的情况（一般横屏时使用），而且不在锁屏的情况下
        // 通知中心的top padding是个定值 mSplitShadeNotificationsTopPadding
        if (mShouldUseSplitNotificationShade && !mKeyguardShowing) {
            return mSplitShadeNotificationsTopPadding;
        }
        if (mKeyguardShowing && (mQsExpandImmediate
                || mIsExpanding && mQsExpandedWhenExpandingStarted)) {

            // Either QS pushes the notifications down when fully expanded, or QS is fully above the
            // notifications (mostly on tablets). maxNotificationPadding denotes the normal top
            // padding on Keyguard, maxQsPadding denotes the top padding from the quick settings
            // panel. We need to take the maximum and linearly interpolate with the panel expansion
            // for a nice motion.
            int maxNotificationPadding = getKeyguardNotificationStaticPadding();
            int maxQsPadding = mQsMaxExpansionHeight;
            int max = mBarState == KEYGUARD ? Math.max(
                    maxNotificationPadding, maxQsPadding) : maxQsPadding;
            return (int) MathUtils.lerp((float) mQsMinExpansionHeight, (float) max,
                    getExpandedFraction());
        } else if (mQsSizeChangeAnimator != null) {
            return Math.max(
                    (int) mQsSizeChangeAnimator.getAnimatedValue(),
                    getKeyguardNotificationStaticPadding());
        } else if (mKeyguardShowing) {
            // 在锁屏的情况下，通过 qs 的展开程度获取通知中心的静态padding到qs最大展开高度的差值
            // 如果最终状态是显示QQS，那么 computeQsExpansionFraction() 为0，那么就返回静态padding
            // 最终状态是显示QS，那么computeQsExpansionFraction() 为0到1之间的数，
            // 此时返回静态padding到qs最大展开高度之间的数
            return MathUtils.lerp((float) getKeyguardNotificationStaticPadding(),
                    (float) (mQsMaxExpansionHeight),
                    computeQsExpansionFraction());
        } else {
            // 其他情况就直接返回 mQsExpansionHeight 这个值通过 setQsExpansion 计算。
            return mQsExpansionHeight;
        }
    }
```
