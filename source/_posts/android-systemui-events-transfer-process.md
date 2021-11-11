---
title: SystemUI -- 下拉面板事件分发流程
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍下拉面板事件分发流程
date: 2021-11-6 10:00:00
---

## 概述

本文基于原生 Android S 代码。


## 事件分发流程

事件的处理主要有下面几个类：

PhoneStatusBarView 主要处理从通知栏下拉通知面板
OverviewProxyService 主要操作桌面下拉通知面板
NotificationShadeWindowViewController 主要操作锁屏切换下拉通知的操作。
NotificationPanelViewController 主要处理QS Panel的整体操作，比如显示，隐藏和整体滑动等等。
NotificationStackScrollLayoutController 主要处理通知中心的滑动，它处理事件时会通知 NotificationPanelViewController 更新 QS 高度。

### NotificationShadeWindowView


### NotificationPanelView

NotificationPanelView 的父类 PanelView 分别设置了事件拦截器和OnTouchListener，事件的处理工作主要由 NotificationPanelViewController 创建的 TouchHandler() 来处理。

```
    public void setOnTouchListener(PanelViewController.TouchHandler touchHandler) {
        super.setOnTouchListener(touchHandler);
        mTouchHandler = touchHandler;
    }
    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        return mTouchHandler.onInterceptTouchEvent(event);
    }
```

```
            public boolean onInterceptTouchEvent(MotionEvent event) {
                if (mBlockTouches || mQsFullyExpanded && mQs.disallowPanelTouches()) {
                    return false;
                }
                initDownStates(event);
                // Do not let touches go to shade or QS if the bouncer is visible,
                // but still let user swipe down to expand the panel, dismissing the bouncer.
                // bouncer view 显示时拦截处理事件
                if (mStatusBar.isBouncerShowing()) {
                    return true;
                }
                if (mBar.panelEnabled() && mHeadsUpTouchHelper.onInterceptTouchEvent(event)) {
                    mMetricsLogger.count(COUNTER_PANEL_OPEN, 1);
                    mMetricsLogger.count(COUNTER_PANEL_OPEN_PEEK, 1);
                    return true;
                }
                if (!shouldQuickSettingsIntercept(mDownX, mDownY, 0)
                        && mPulseExpansionHandler.onInterceptTouchEvent(event)) {
                    return true;
                }
                // QS展开的情况下(1,2,3场景之间的切换)，onQsIntercept来判断是否需要拦截
                if (!isFullyCollapsed() && onQsIntercept(event)) {
                    return true;
                }
                return super.onInterceptTouchEvent(event);
            }
            
            public boolean onTouch(View v, MotionEvent event) {
                if (mBlockTouches || (mQsFullyExpanded && mQs != null
                        && mQs.disallowPanelTouches())) {
                    return false;
                }

                // Do not allow panel expansion if bouncer is scrimmed, otherwise user would be able
                // to pull down QS or expand the shade.
                if (mStatusBar.isBouncerShowingScrimmed()) {
                    return false;
                }

                // Make sure the next touch won't the blocked after the current ends.
                if (event.getAction() == MotionEvent.ACTION_UP
                        || event.getAction() == MotionEvent.ACTION_CANCEL) {
                    mBlockingExpansionForCurrentTouch = false;
                }
                // When touch focus transfer happens, ACTION_DOWN->ACTION_UP may happen immediately
                // without any ACTION_MOVE event.
                // In such case, simply expand the panel instead of being stuck at the bottom bar.
                if (mLastEventSynthesizedDown && event.getAction() == MotionEvent.ACTION_UP) {
                    expand(true /* animate */);
                }
                initDownStates(event);

                // If pulse is expanding already, let's give it the touch. There are situations
                // where the panel starts expanding even though we're also pulsing
                boolean pulseShouldGetTouch = (!mIsExpanding
                        && !shouldQuickSettingsIntercept(mDownX, mDownY, 0))
                        || mPulseExpansionHandler.isExpanding();
                if (pulseShouldGetTouch && mPulseExpansionHandler.onTouchEvent(event)) {
                    // We're expanding all the other ones shouldn't get this anymore
                    return true;
                }
                if (mListenForHeadsUp && !mHeadsUpTouchHelper.isTrackingHeadsUp()
                        && mHeadsUpTouchHelper.onInterceptTouchEvent(event)) {
                    mMetricsLogger.count(COUNTER_PANEL_OPEN_PEEK, 1);
                }
                boolean handled = false;
                if ((!mIsExpanding || mHintAnimationRunning) && !mQsExpanded
                        && mBarState != StatusBarState.SHADE && !mDozing) {
                    handled |= mAffordanceHelper.onTouchEvent(event);
                }
                if (mOnlyAffordanceInThisMotion) {
                    return true;
                }
                handled |= mHeadsUpTouchHelper.onTouchEvent(event);

                if (!mHeadsUpTouchHelper.isTrackingHeadsUp() && handleQsTouch(event)) {
                    return true;
                }
                if (event.getActionMasked() == MotionEvent.ACTION_DOWN && isFullyCollapsed()) {
                    mMetricsLogger.count(COUNTER_PANEL_OPEN, 1);
                    updateHorizontalPanelPosition(event.getX());
                    handled = true;
                }

                if (event.getActionMasked() == MotionEvent.ACTION_DOWN && isFullyExpanded()
                        && mStatusBarKeyguardViewManager.isShowing()) {
                    mStatusBarKeyguardViewManager.updateKeyguardPosition(event.getX());
                }

                if (mLockIconViewController.onTouchEvent(event)) {
                    return true;
                }

                handled |= super.onTouch(v, event);
                return !mDozing || mPulsing || handled;
            }
```

```
    private boolean shouldQuickSettingsIntercept(float x, float y, float yDiff) {
        if (!isQsExpansionEnabled() || mCollapsedOnDown || (mKeyguardShowing
                && mKeyguardBypassController.getBypassEnabled())) {
            return false;
        }
        // 如果时锁屏界面，那么header区域就是状态栏，否则就是QuickStatusBarHeader的区域。
        View header = mKeyguardShowing || mQs == null ? mKeyguardStatusBar : mQs.getHeader();
        // 根据header设置拦截区域
        mQsInterceptRegion.set(
                /* left= */ (int) mQsFrame.getX(),
                /* top= */ header.getTop(),
                /* right= */ (int) mQsFrame.getX() + mQsFrame.getWidth(),
                /* bottom= */ header.getBottom());
        // Also allow QS to intercept if the touch is near the notch.
        mStatusBarTouchableRegionManager.updateRegionForNotch(mQsInterceptRegion);
        //判读事件是否在header区域
        final boolean onHeader = mQsInterceptRegion.contains((int) x, (int) y);

        if (mQsExpanded) {
            return onHeader || (yDiff < 0 && isInQsArea(x, y));
        } else {
            return onHeader;
        }
    }
```

```
//onQsIntercept调用的前提时qs在展开状态
    private boolean onQsIntercept(MotionEvent event) {
        int pointerIndex = event.findPointerIndex(mTrackingPointer);
        if (pointerIndex < 0) {
            pointerIndex = 0;
            mTrackingPointer = event.getPointerId(pointerIndex);
        }
        final float x = event.getX(pointerIndex);
        final float y = event.getY(pointerIndex);

        switch (event.getActionMasked()) {
            case MotionEvent.ACTION_DOWN:
                mInitialTouchY = y;
                mInitialTouchX = x;
                initVelocityTracker();
                trackMovement(event);
                if (mKeyguardShowing
                        && shouldQuickSettingsIntercept(mInitialTouchX, mInitialTouchY, 0)) {
                    // 在锁屏界面而且时在锁屏的状态栏开始滑动时，禁止父组件再拦截事件，保证move事件可以下发下来
                    mView.getParent().requestDisallowInterceptTouchEvent(true);
                }
                if (mQsExpansionAnimator != null) {
                    mInitialHeightOnTouch = mQsExpansionHeight;
                    // 开始跟踪事件
                    mQsTracking = true;
                    traceQsJank(true /* startTracing */, false /* wasCancelled */);
                    mNotificationStackScrollLayoutController.cancelLongPress();
                }
                break;
            case MotionEvent.ACTION_POINTER_UP:
                final int upPointer = event.getPointerId(event.getActionIndex());
                if (mTrackingPointer == upPointer) {
                    // gesture is ongoing, find a new pointer to track
                    final int newIndex = event.getPointerId(0) != upPointer ? 0 : 1;
                    mTrackingPointer = event.getPointerId(newIndex);
                    mInitialTouchX = event.getX(newIndex);
                    mInitialTouchY = event.getY(newIndex);
                }
                break;

            case MotionEvent.ACTION_MOVE:
                final float h = y - mInitialTouchY;
                trackMovement(event);
                if (mQsTracking) {
                    // 更新QS的高度以及通知中心位置
                    setQsExpansion(h + mInitialHeightOnTouch);
                    trackMovement(event);
                    // 开始处理Move事件
                    return true;
                }
                if ((h > getTouchSlop(event) || (h < -getTouchSlop(event) && mQsExpanded))
                        && Math.abs(h) > Math.abs(x - mInitialTouchX)
                        && shouldQuickSettingsIntercept(mInitialTouchX, mInitialTouchY, h)) {
                    mView.getParent().requestDisallowInterceptTouchEvent(true);
                    mQsTracking = true;
                    traceQsJank(true /* startTracing */, false /* wasCancelled */);
                    onQsExpansionStarted();
                    notifyExpandingFinished();
                    mInitialHeightOnTouch = mQsExpansionHeight;
                    mInitialTouchY = y;
                    mInitialTouchX = x;
                    mNotificationStackScrollLayoutController.cancelLongPress();
                    return true;
                }
                break;

            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                trackMovement(event);
                mQsTracking = false;
                break;
        }
        return false;
    }
```

### NotificationStackScrollLayout

处理三种类型的事件：1.单个通知的展开和收缩手势，2.通知中心的滑动和滚动，3.左右滑动删除通知操作

```
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (mTouchHandler != null && mTouchHandler.onInterceptTouchEvent(ev)) {
            return true;
        }
        return super.onInterceptTouchEvent(ev);
    }
```

```
    public boolean onTouchEvent(MotionEvent ev) {
        if (mTouchHandler != null && mTouchHandler.onTouchEvent(ev)) {
            return true;
        }

        return super.onTouchEvent(ev);
    }
```

```
        public boolean onInterceptTouchEvent(MotionEvent ev) {
            mView.initDownStates(ev);
            mView.handleEmptySpaceClick(ev);

            NotificationGuts guts = mNotificationGutsManager.getExposedGuts();
            boolean expandWantsIt = false;
            if (!mSwipeHelper.isSwiping()
                    && !mView.getOnlyScrollingInThisMotion() && guts == null) {
                // 处理单个通知的展开和收缩手势
                expandWantsIt = mView.getExpandHelper().onInterceptTouchEvent(ev);
            }
            boolean scrollWantsIt = false;
            if (!mSwipeHelper.isSwiping() && !mView.isExpandingNotification()) {
                // 处理通知中心的滚动
                scrollWantsIt = mView.onInterceptTouchEventScroll(ev);
            }
            boolean swipeWantsIt = false;
            if (!mView.isBeingDragged()
                    && !mView.isExpandingNotification()
                    && !mView.getExpandedInThisMotion()
                    && !mView.getOnlyScrollingInThisMotion()
                    && !mView.getDisallowDismissInThisMotion()) {
                // 处理左右滑动删除通知操作
                swipeWantsIt = mSwipeHelper.onInterceptTouchEvent(ev);
            }
            // Check if we need to clear any snooze leavebehinds
            boolean isUp = ev.getActionMasked() == MotionEvent.ACTION_UP;
            if (!NotificationSwipeHelper.isTouchInView(ev, guts) && isUp && !swipeWantsIt &&
                    !expandWantsIt && !scrollWantsIt) {
                mView.setCheckForLeaveBehind(false);
                mNotificationGutsManager.closeAndSaveGuts(true /* removeLeavebehind */,
                        false /* force */, false /* removeControls */, -1 /* x */, -1 /* y */,
                        false /* resetMenu */);
            }
            if (ev.getActionMasked() == MotionEvent.ACTION_UP) {
                mView.setCheckForLeaveBehind(true);
            }

            if (scrollWantsIt && ev.getActionMasked() != MotionEvent.ACTION_DOWN) {
                InteractionJankMonitor.getInstance().begin(mView,
                        CUJ_NOTIFICATION_SHADE_SCROLL_FLING);
            }
            return swipeWantsIt || scrollWantsIt || expandWantsIt;
        }
        
```

```
NotificationStackScrollLayout.onInterceptTouchEvent()
    NotificationStackScrollLayoutController.TouchHandler.onInterceptTouchEvent()
        ExpandHelper.onInterceptTouchEvent() // 处理单个通知的展开和收缩手势
        NotificationStackScrollLayout.onInterceptTouchEventScroll() // 处理通知中心的滚动
            MotionEvent.ACTION_MOVE
                NotificationStackScrollLayout.setIsBeingDragged()
                    requestDisallowInterceptTouchEvent(true) // 阻止父组件拦截事件
        SwipeHelper.onInterceptTouchEvent() // 处理左右滑动删除通知操作
            MotionEvent.ACTION_MOVE
                NotificationStackScrollLayoutController.onBeginDrag()
                    NotificationStackScrollLayout.onSwipeBegin()
                        requestDisallowInterceptTouchEvent(true)
```

```
//NotificationStackScrollLayoutController.java
        @Override
        public boolean onTouchEvent(MotionEvent ev) {
            NotificationGuts guts = mNotificationGutsManager.getExposedGuts();
            boolean isCancelOrUp = ev.getActionMasked() == MotionEvent.ACTION_CANCEL
                    || ev.getActionMasked() == MotionEvent.ACTION_UP;
            mView.handleEmptySpaceClick(ev);
            boolean expandWantsIt = false;
            boolean onlyScrollingInThisMotion = mView.getOnlyScrollingInThisMotion();
            boolean expandingNotification = mView.isExpandingNotification();
            if (mView.getIsExpanded() && !mSwipeHelper.isSwiping() && !onlyScrollingInThisMotion
                    && guts == null) {
                ExpandHelper expandHelper = mView.getExpandHelper();
                if (isCancelOrUp) {
                    expandHelper.onlyObserveMovements(false);
                }
                boolean wasExpandingBefore = expandingNotification;
                // 处理单个通知的展开和收缩手势 
                expandWantsIt = expandHelper.onTouchEvent(ev);
                expandingNotification = mView.isExpandingNotification();
                if (mView.getExpandedInThisMotion() && !expandingNotification && wasExpandingBefore
                        && !mView.getDisallowScrollingInThisMotion()) {
                    mView.dispatchDownEventToScroller(ev);
                }
            }
            boolean scrollerWantsIt = false;
            if (mView.isExpanded() && !mSwipeHelper.isSwiping() && !expandingNotification
                    && !mView.getDisallowScrollingInThisMotion()) {
                // 处理通知中心的滚动
                scrollerWantsIt = mView.onScrollTouch(ev);
            }
            boolean horizontalSwipeWantsIt = false;
            if (!mView.isBeingDragged()
                    && !expandingNotification
                    && !mView.getExpandedInThisMotion()
                    && !onlyScrollingInThisMotion
                    && !mView.getDisallowDismissInThisMotion()) {
                // 处理左右滑动删除通知操作
                horizontalSwipeWantsIt = mSwipeHelper.onTouchEvent(ev);
            }

            // Check if we need to clear any snooze leavebehinds
            if (guts != null && !NotificationSwipeHelper.isTouchInView(ev, guts)
                    && guts.getGutsContent() instanceof NotificationSnooze) {
                NotificationSnooze ns = (NotificationSnooze) guts.getGutsContent();
                if ((ns.isExpanded() && isCancelOrUp)
                        || (!horizontalSwipeWantsIt && scrollerWantsIt)) {
                    // If the leavebehind is expanded we clear it on the next up event, otherwise we
                    // clear it on the next non-horizontal swipe or expand event.
                    checkSnoozeLeavebehind();
                }
            }
            if (ev.getActionMasked() == MotionEvent.ACTION_UP) {
                // Ensure the falsing manager records the touch. we don't do anything with it
                // at the moment.
                mFalsingManager.isFalseTouch(Classifier.SHADE_DRAG);
                mView.setCheckForLeaveBehind(true);
            }
            traceJankOnTouchEvent(ev.getActionMasked(), scrollerWantsIt);
            return horizontalSwipeWantsIt || scrollerWantsIt || expandWantsIt;
        }
    
```

```
NotificationStackScrollLayout.onTouchEvent()
    NotificationStackScrollLayoutController.TouchHandler.onTouchEvent()
        ExpandHelper.onTouchEvent() //处理单个通知的展开和收缩手势 
        NotificationStackScrollLayout.onScrollTouch() // 处理通知中心的滚动
            MotionEvent.ACTION_MOVE
                NotificationStackScrollLayout.overScrollDown() // 下滑
                NotificationStackScrollLayout.overScrollUp() //上滑
                NotificationStackScrollLayout.customOverScrollBy() //滚动
            MotionEvent.ACTION_UP
                NotificationStackScrollLayout.onOverScrollFling(true) // 展开QS
                NotificationStackScrollLayout.onOverScrollFling(false) // 通知中心滑动，回到原来位置
                NotificationStackScrollLayout.fling() // 处理通知中心放手后的惯性滚动，注意：不是回弹效果。
                OverScroller.springBack()
                NotificationStackScrollLayout.animateScroll()
        SwipeHelper.onTouchEvent() // 处理左右滑动删除通知操作
```

### 事件分发


NotificationPanelView 对事件拦截：

```
NotificationShadeWindowView.dispatchTouchEvent()
    PanelView.onInterceptTouchEvent()
        NotificationPanelViewController.TouchHandler.onInterceptTouchEvent()
            PhoneStatusBarView.panelEnabled() // 是否允许显示通知面板
                CommandQueue.panelsEnabled()
            NotificationPanelViewController.shouldQuickSettingsIntercept()
            PanelViewController.isFullyExpanded()
            NotificationPanelViewController.onQsIntercept() // QS 是否需要拦截，拦截后执行折叠动作
                MotionEvent.ACTION_MOVE
                    NotificationPanelViewController.setQsExpansion() // 设置QS显示高度
                    NotificationPanelViewController.onQsExpansionStarted() // 开始跟踪手势，实现通知中心上下滑
                    NotificationPanelViewController.notifyExpandingFinished()
            PanelViewController.TouchHandler.onInterceptTouchEvent() // PanelViewController 是否拦截
                NotificationPanelViewController.canCollapsePanelOnTouch() // 是否可以折叠 QSPanel
                    NotificationStackScrollLayoutController.isScrolledToBottom() // 通知中心有没有滑动到底部，滑动到底部表示不能滑动就拦截，不能再折叠了，move事件处理成QS整体操作
```

如果 `NotificationPanelViewController.TouchHandler.onInterceptTouchEvent()` 这里拦截了，就处理QSPanel的整体操作，比如显示和隐藏等。不拦截了就向下分发，处理通知中心折叠操作等。

滑动QS：
1.设置QS显示高度
2.更新QQS的可见性
3.设置QS的绘制区域
4.更新通知中心的偏移
5.更新面板可见性

```
PanelView.onTouchEvent()
    NotificationPanelViewController.TouchHandler.onTouch()
        NotificationPanelViewController.handleQsTouch() // QS处理
            NotificationPanelViewController.onQsExpansionStarted()
                NotificationPanelViewController.setQsExpansion() // 设置QS显示高度，更新QQS的可见性, 设置QS的绘制区域,更新通知中心的偏移，后面详细介绍
                    NotificationPanelViewController.updateQsExpansion()
                    NotificationPanelViewController.requestScrollerTopPaddingUpdate() // 更新通知中心的位置
        PanelViewController.TouchHandler.onTouch() // handleQsTouch 不处理，就走到这里，执行下拉通知整体操作
            MotionEvent.ACTION_MOVE
                NotificationPanelViewController.onTrackingStarted()
                    PanelViewController.onTrackingStarted()
                        PanelViewController.notifyBarPanelExpansionChanged()
                            PhoneStatusBarView.panelExpansionChanged()
                                PanelBar.panelExpansionChanged()
                                    PhoneStatusBarView.onPanelPeeked()
                                        StatusBar.makeExpandedVisible()
                                            CommandQueue.panelsEnabled() // 是否允许显示通知面板
                                            NotificationShadeWindowControllerImpl.setPanelVisible(true) // 设置面板可见
                                                NotificationShadeWindowControllerImpl.apply()
                                                    NotificationShadeWindowControllerImpl.applyVisibility()
                                                        NotificationShadeWindowView.setVisibility() // 更新面板可见性
                    PanelViewController.setExpandedHeightInternal()
                        NotificationPanelViewController.onHeightUpdated()
                            NotificationPanelViewController.positionClockAndNotifications()
                                NotificationPanelViewController.requestScrollerTopPaddingUpdate()
                                    NotificationStackScrollLayoutController.updateTopPadding() // 更新通知中心位置
            MotionEvent.ACTION_UP:
            MotionEvent.ACTION_CANCEL:
                PanelViewController.endMotionEvent()
                    PanelViewController.flingExpands() //判断是否expand，决定qspanel时小时还是展开。
                    PanelViewController.fling() // 开始 fling 动画
                        NotificationPanelViewController.flingToHeight()
                            PanelViewController.flingToHeight()
                                PanelViewController.createHeightAnimator()
                                    AnimatorUpdateListener
                                        PanelViewController.setExpandedHeightInternal() // 设置最终QS展开的高度
                                AnimatorListenerAdapter.onAnimationEnd()
                                    PanelViewController.onFlingEnd()
                                        PanelViewController.notifyBarPanelExpansionChanged()// 流程参考下面。
                                ValueAnimator.start()
                    NotificationPanelViewController.onTrackingStopped()
                        PanelViewController.onTrackingStopped()
                            PhoneStatusBarView.onTrackingStopped()
                                PanelBar.onTrackingStopped()
                                    StatusBarKeyguardViewManager.showBouncer(false) // 设置BouncerView可见性
                            PanelViewController.notifyBarPanelExpansionChanged()
                                PhoneStatusBarView.panelExpansionChanged()
                                    PanelBar.onPanelCollapsed()
                                        PhoneStatusBarView.onPanelCollapsed()
                                            post(mHideExpandedRunnable)
                                                StatusBar.makeExpandedInvisible()
                                                    NotificationShadeWindowControllerImpl.setPanelVisible(false) // 隐藏面板
                                                        NotificationShadeWindowControllerImpl.apply()
                                                            NotificationShadeWindowControllerImpl.applyVisibility()
                                                                NotificationShadeWindowView.setVisibility() // 更新面板可见性
```


滑动通知中心：

```
NotificationShadeWindowView.dispatchTouchEvent()
    NotificationStackScrollLayout.onInterceptTouchEvent()
        NotificationStackScrollLayoutController.onInterceptTouchEvent()
            NotificationStackScrollLayout.onInterceptTouchEventScroll()
                MotionEvent.ACTION_MOVE
                    NotificationStackScrollLayout.setIsBeingDragged()
                        requestDisallowInterceptTouchEvent(true) // 阻止父组件拦截事件
    NotificationStackScrollLayout.onTouchEvent()
        NotificationStackScrollLayout.TouchHandler.onTouchEvent()
            NotificationStackScrollLayout.onScrollTouch()
                MotionEvent.ACTION_MOVE
                    NotificationStackScrollLayout.overScrollUp()
                    NotificationStackScrollLayout.overScrollDown()
                        NotificationStackScrollLayout.setOverScrolledPixels()
                            NotificationStackScrollLayout.setOverScrollAmount()
                                NotificationStackScrollLayout.setOverScrollAmountInternal
                                    NotificationStackScrollLayout.notifyOverscrollTopListener()
                                        NotificationPanelViewController.OnOverscrollTopChangedListener.onOverscrollTopChanged()
                                            NotificationPanelViewController.setQsExpansion() // 设置QS可见高度以及通知中心位置
                    NotificationStackScrollLayout.customOverScrollBy() // 通知栏展示到顶部时
                MotionEvent.ACTION_UP
                    NotificationStackScrollLayout.shouldOverScrollFling()
                    NotificationStackScrollLayout.onOverScrollFling(true) // 通知中心弹开，全部展开QS
                        NotificationPanelViewController.OnOverscrollTopChangedListener.flingTopOverscroll()
                            NotificationPanelViewController.flingSettings() // QS 或者 QQS的动画，看下面方法详解
                                ValueAnimator.start()
                                    AnimatorUpdateListener.onAnimationUpdate()
                                        NotificationPanelViewController.setQsExpansion() // 设置QS可见高度以及通知中心位置
                    NotificationStackScrollLayout.fling() // 处理通知中心放手后的惯性滚动，注意：不是回弹效果。
                        OverScroller.fling() 
                    NotificationStackScrollLayout.onOverScrollFling(false) // 通知中心滚动，回到原来位置
                    NotificationStackScrollLayout.animateScroll() // 通知中心列表中通知的滚动
```


## QSPanel 的几种显示场景

NotificationStackScrollLayout 的三级折叠

0.QSPanel隐藏
1.通知中心全部显示
2.显示QQS和通知中心
3.显示QS，不显示通知中心

2到1的切换有个条件就是通知中心在2状态下时满屏的，可以滚动，这时的通知栏滑动处理成ScrollView的滚动问题。
一个比较重要的方法就是 `NotificationPanelViewController.canCollapsePanelOnTouch()`

```
    protected boolean canCollapsePanelOnTouch() {
        if (!isInSettings() && mBarState == KEYGUARD) {
            return true;
        }
        // 判断通知中心是否可以滚动
        if (mNotificationStackScrollLayoutController.isScrolledToBottom()) {
            return true;
        }

        return !mShouldUseSplitNotificationShade && (isInSettings() || mIsPanelCollapseOnQQS);
    }
```

在 `PanelViewController.TouchHandler.onInterceptTouchEvent()` 中相应 `MotionEvent.ACTION_MOVE` 做判断，如果通知中心不能滚动，就拦截 `MotionEvent.ACTION_MOVE` 事件，做 QSPanel 上移或者隐藏的动作。
如果通知中心可以滚动，那么就不拦截`MotionEvent.ACTION_MOVE` 事件，交给通知中心做滚动处理。

```
// PanelViewController.java
        public boolean onInterceptTouchEvent(MotionEvent event) {
                case MotionEvent.ACTION_MOVE:
                    final float h = y - mInitialTouchY;
                    addMovement(event);
                    if (canCollapsePanel || mTouchStartedInEmptyArea || mAnimatingOnDown) {
                        float hAbs = Math.abs(h);
                        float touchSlop = getTouchSlop(event);
                        if ((h < -touchSlop || (mAnimatingOnDown && hAbs > touchSlop))
                                && hAbs > Math.abs(x - mInitialTouchX)) {
                            cancelHeightAnimator();
                            startExpandMotion(x, y, true /* startTracking */, mExpandedHeight);
                            return true;
                        }
                    }
                    break;
```

NotificationStackScrollLayout.onInterceptTouchEventScroll 收到 `MotionEvent.ACTION_MOVE` 事件后，会做相应的判断，做事件的拦截：

```
    boolean onInterceptTouchEventScroll(MotionEvent ev) {
            case MotionEvent.ACTION_MOVE: {
                ......

                final int y = (int) ev.getY(pointerIndex);
                final int x = (int) ev.getX(pointerIndex);
                final int yDiff = Math.abs(y - mLastMotionY);
                final int xDiff = Math.abs(x - mDownX);
                if (yDiff > getTouchSlop(ev) && yDiff > xDiff) {
                    // 设置可以拖动 mIsBeingDragged 为true
                    setIsBeingDragged(true);
                    mLastMotionY = y;
                    mDownX = x;
                    initVelocityTrackerIfNotExists();
                    mVelocityTracker.addMovement(ev);
                }
                break;
            }
        return mIsBeingDragged;
    }
```

然后事件就交给 `NotificationStackScrollLayoutController.TouchHandler.onTouchEvent() -> NotificationStackScrollLayout.onScrollTouch() -> NotificationStackScrollLayout.customOverScrollBy()` 处理上划滚动。
2->3 通知中心下滑，这时 QSPanel不对 `MotionEvent.ACTION_MOVE` 做拦截，这时交给 NotificationStackScrollLayout 来处理向下滚动的 `MotionEvent.ACTION_MOVE` 和 `MotionEvent.ACTION_UP` 事件。


2->0 

`MotionEvent.ACTION_UP` 事件后包含两种场景，一个是QSPanel消失，另外一个时回弹回2场景。这两种情况都是由 `PanelViewController.TouchHandler.onTouch` 中对 `MotionEvent.ACTION_UP` 的处理。
然后在 `PanelViewController.endMotionEvent` 方法中通过 `flingExpands()` 来计算是否 expand，根据expand来计算最终的位置。如果expand为true就回原位，否则上划消失。

```
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
        if (Math.abs(vectorVel) < mFlingAnimationUtils.getMinVelocityPxPerSecond()) {
            // 根据滑动的距离来判断
            return shouldExpandWhenNotFlinging();
        } else {
            // 根据滑动速度来判断
            return vel > 0;
        }
    }
    
    protected boolean shouldExpandWhenNotFlinging() {
        return getExpandedFraction() > 0.5f;
    }
```

0->2 切换时满足 expand 条件，就会去做 expand 动画。

2->3 切换同样时由 NotificationStackScrollLayout 来处理 `MotionEvent.ACTION_MOVE` 和 `MotionEvent.ACTION_UP` 事件。


## 锁屏下拉通知栏

### 状态栏滑动
这时的最终的状态可以是(3)显示QS，不显示通知中心。处理QS的滑动。

NotificationPanelViewController 没有拦截事件，NotificationStackScrollLayout 也没有消费，那么Down事件还是给 NotificationPanelView 来消费，以及后面的 Move 事件也被NotificationPanelView 来消费。
虽然在 NotificationPanelViewController.onInterceptTouchEvent 没有拦截，但是在 NotificationPanelViewController.onQsIntercept 处理Down事件时设置了 `mView.getParent().requestDisallowInterceptTouchEvent(true)`，那么它的父组件们就不会再拦截了。
下面来看一下Down事件的分发流程

```
NotificationShadeWindowView.dispatchTouchEvent()
    PanelView.onInterceptTouchEvent()
        NotificationPanelViewController.TouchHandler.onInterceptTouchEvent()
            NotificationPanelViewController.onQsIntercept()
                NotificationPanelViewController.shouldQuickSettingsIntercept() // QS是否需要拦截，如果需要就进行下面的设置
                View.getParent().requestDisallowInterceptTouchEvent(true) // 阻止父组件拦截后面的事件
```

那么在什么情况下会拦截Down事件呢？

```
    private boolean shouldQuickSettingsIntercept(float x, float y, float yDiff) {
        if (!isQsExpansionEnabled() || mCollapsedOnDown || (mKeyguardShowing
                && mKeyguardBypassController.getBypassEnabled())) {
            return false;
        }
        // 如果时锁屏界面，那么header区域就是状态栏，否则就是QuickStatusBarHeader的区域。
        View header = mKeyguardShowing || mQs == null ? mKeyguardStatusBar : mQs.getHeader();
        // 根据header设置拦截区域
        mQsInterceptRegion.set(
                /* left= */ (int) mQsFrame.getX(),
                /* top= */ header.getTop(),
                /* right= */ (int) mQsFrame.getX() + mQsFrame.getWidth(),
                /* bottom= */ header.getBottom());
        // Also allow QS to intercept if the touch is near the notch.
        mStatusBarTouchableRegionManager.updateRegionForNotch(mQsInterceptRegion);
        //判读事件是否在header区域
        final boolean onHeader = mQsInterceptRegion.contains((int) x, (int) y);

        if (mQsExpanded) {
            return onHeader || (yDiff < 0 && isInQsArea(x, y));
        } else {
            return onHeader;
        }
    }

    private boolean onQsIntercept(MotionEvent event) {
        ......
        switch (event.getActionMasked()) {
            case MotionEvent.ACTION_DOWN:
                ......
                if (mKeyguardShowing
                        && shouldQuickSettingsIntercept(mInitialTouchX, mInitialTouchY, 0)) {
                    // Dragging down on the lockscreen statusbar should prohibit other interactions
                    // immediately, otherwise we'll wait on the touchslop. This is to allow
                    // dragging down to expanded quick settings directly on the lockscreen.
                    mView.getParent().requestDisallowInterceptTouchEvent(true);
                }
                ......
    }
```

此时 NotificationPanelViewController.TouchHandler.onTouch() 中的 handleQsTouch 是返回 true 的。
handleQsTouch() 方法来更新 QS 的显示高度以及更新通知中心的位置。

### 通知栏滑动

这时的最终的状态可以是(2)显示QQS和通知中心，处理面板的整体滑动。

这个时候 NotificationShadeWindowView 会拦截 `ACTION_MOVE` 事件，那么由 NotificationShadeWindowView.onTouchEvent() 来处理 `ACTION_MOVE` 事件。

```
NotificationShadeWindowView.onInterceptTouchEvent()
    NotificationShadeWindowViewController.InteractionEventHandler.shouldInterceptTouchEvent()
        LockscreenShadeTransitionController.DragDownHelper.onInterceptTouchEvent()
            LockscreenShadeTransitionController.isDragDownAnywhereEnabled // 这里返回true，拦截事件
```

```
    internal val isDragDownAnywhereEnabled: Boolean
        get() = (statusBarStateController.getState() == StatusBarState.KEYGUARD &&
                !keyguardBypassController.bypassEnabled &&
                qS.isFullyCollapsed)
```

```
StatusBar.getStatusBarWindowTouchListener
    NotificationShadeWindowView.onTouchEvent()
        NotificationShadeWindowViewController.setupExpandedStatusBar().handleTouchEvent()
            LockscreenShadeTransitionController.DragDownHelper.onTouchEvent()
                LockscreenShadeTransitionController.dragDownAmount
                    NotificationPanelViewController.setTransitionToFullShadeAmount()
                        NotificationPanelViewController.updateQsExpansion()
                            QSFragment.setQsExpansion()
                                QSContainerImpl.setTranslationY()
                                QSFragment.updateQsBounds()
                                    NonInterceptingScrollView.setClipBounds() // 设置绘制区域
                    QSFragment.setTransitionToFullShadeAmount()
                        QSFragment.updateShowCollapsedOnKeyguard()
                            QSFragment.updateQsState()
                                QuickStatusBarHeader.setVisibility()
                                QuickStatusBarHeader.setExpanded()
                                QSFooter.setVisibility()
                                QSFooter.setExpanded()
                                QSPanelController.setVisibility() // 设置QSPanel的可见性
                LockscreenShadeTransitionController.onCrossedThreshold()
                    NotificationStackScrollLayoutController.setDimmed() //设置通知中心透明度
                        NotificationStackScrollLayout.setDimmed()
                            NotificationStackScrollLayout.animateDimmed()
                            NotificationStackScrollLayout.setDimAmount()
```

## 桌面下拉

```
OverviewProxyService.onStatusBarMotionEvent()
    PanelViewController.startExpandLatencyTracking()
    StatusBar.onInputFocusTransfer()
        NotificationPanelViewController.stopWaitingForOpenPanelGesture()
            NotificationPanelViewController.collapse()
            NotificationPanelViewController.fling()
            NotificationPanelViewController.onTrackingStopped()
```

## 状态栏下拉

```
StatusBarWindowView.dispatchTouchEvent()
    PhoneStatusBarView.onTouchEvent()
        PanelBar.onTouchEvent()
            NotificationPanelView.dispatchTouchEvent()
                NotificationPanelViewController.TouchHandler.onInterceptTouchEvent()
                    PanelViewController.TouchHandler.onInterceptTouchEvent()
                NotificationPanelViewController.TouchHandler.onTouch()
                    PanelViewController.TouchHandler.onTouch()
```

## 锁屏上划通知栏解锁

这个时候 NotificationShadeWindowView 没有拦截也没有处理事件，交给NotificationPanelView去处理面板整体操作

```
NotificationShadeWindowView.dispatchTouchEvent()
    NotificationPanelViewController.TouchHandler.onTouch()
        PanelViewController.TouchHandler.onTouch()
            MotionEvent.ACTION_MOVE
                PanelViewController.setExpandedHeightInternal() //设置QS展开的高度
            MotionEvent.ACTION_UP
                PanelViewController.fling()
                    NotificationPanelViewController.flingToHeight() // 滚动到指定高度，后面文章详细介绍
                
```

## 点击导航栏收起

```
StatusBar.onReceive()
    Intent.ACTION_CLOSE_SYSTEM_DIALOGS
        ShadeControllerImpl.animateCollapsePanels()
            PanelBar.collapsePanel()
                NotificationPanelViewController.collapse()
```
