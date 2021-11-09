---
title: Android -- NotificationStackScrollLayout
categories:  SystemUI
comments: true
tags: [SystemUI]
description: 介绍 NotificationStackScrollLayout
date: 2021-11-8 10:00:00
---

## 概述

NotificationStackScrollLayout 继承自 ViewGroup，它提供了一个动态添加和删除通知的可滚动的容器。
主要处理三种类型的事件：1.单个通知的展开和收缩手势，2.通知中心的滑动和滚动，3.左右滑动删除通知操作。
本文基于原生Android S代码。

## 常用变量和方法

1.NotificationStackScrollLayout
mQsExpansionFraction：通知栏展开比例，如果是0表示全部展开，这时只显示QQS，1表示通知中心全部隐藏。
mExpandHelper：处理通知的展开和收缩
mTopPadding：通知中心距离顶部的偏移量
mOverScrolledTopPixels
mOverScrolledBottomPixels
RUBBER_BAND_FACTOR_NORMAL:回弹系数
RUBBER_BAND_FACTOR_AFTER_EXPAND
RUBBER_BAND_FACTOR_ON_PANEL_EXPAND
mScrolledToTopOnFirstDown：它表示Down事件是是否是个滑动事件，如果时在通知满屏时在显示QQS场景到通知中心全部显示之间切换，那么它就是false。
mExpandedInThisMotion：是否是展开单个通知的事假
updateTopPadding():更新偏移量，来设置每条通知的位置。
updateEmptyShadeView()
updateFooterView()
setDismissAllInProgress()
fling()：处理通知中心放手后的惯性滚动，注意：不是回弹效果。
2.NotificationStackScrollLayoutController
mSwipeHelper:处理滑动删除通知逻辑
3.AmbientState: 为 StackScrollAlgorithm 保存一些全局状态。
mStackY：通知中心顶部距屏幕顶部的距离。
mTopPadding：通知中心距离顶部的偏移量
mScrollY:通知栏的滚动位置，通知中心由显示QQS到满屏显示通知场景过渡时来决定通知栏的位置
mOverScrollTopAmount：通知中心顶部回弹量，向下滑动时设置
mOverScrollBottomAmount：通知中心底部回弹量，通知中心满屏时向上滚动通知栏时设置
3.StackScrollAlgorithm：用来使 NotificationStackScrollLayout可以查询或者更新当前的 StackScrollAlgorithmState 状态。
mExpansionFraction ：通知栏展开的比例，场景2 -> 场景0 过渡时来决定通知的折叠比例，透明度和通知中心的位置。
mScrollY:AmbientState.mScrollY
4.ViewState：记录了一些View的属性值，translation，alpha，scale，visibility等。
5.ExpandableViewState：ViewState的子类，每个ExpandableView类都有个ExpandableViewState变量，记录该通知的一些属性信息。
yTranslation 通知的实际位置。

StackScrollAlgorithmState
scrollY:AmbientState.mScrollY
mCurrentYPosition:当前正在计算的通知的位置，累加值，以此计算各个通知的偏移。

## 通知栏的滑动

这部分主要是在显示 QQS 状态到通知中心完全隐藏状态之间的切换。这部分主要涉及到通知位置的计算和UI更新。
这部分介绍的仅仅时在通知中心的通知区域的手势操作，比如滑动通知中心的空白处的操作，这部分是在 NotificationPanelViewController.handleQsTouch() 处理。

### 事件处理

这部分事件处理主要在 NotificationStackScrollLayout.onScrollTouch() 中进行

```
// NotificationStackScrollLayout.java
    @ShadeViewRefactor(RefactorComponent.INPUT)
    boolean onScrollTouch(MotionEvent ev) {
        if (!isScrollingEnabled()) {
            return false;
        }
        if (isInsideQsContainer(ev) && !mIsBeingDragged) {
            return false;
        }
        mForcedScroll = null;
        initVelocityTrackerIfNotExists();
        mVelocityTracker.addMovement(ev);

        final int action = ev.getActionMasked();
        if (ev.findPointerIndex(mActivePointerId) == -1 && action != MotionEvent.ACTION_DOWN) {
            // Incomplete gesture, possibly due to window swap mid-gesture. Ignore until a new
            // one starts.
            Log.e(TAG, "Invalid pointerId=" + mActivePointerId + " in onTouchEvent "
                    + MotionEvent.actionToString(ev.getActionMasked()));
            return true;
        }

        switch (action) {
            case MotionEvent.ACTION_DOWN: {
                if (getChildCount() == 0 || !isInContentBounds(ev)) {
                    return false;
                }
                boolean isBeingDragged = !mScroller.isFinished();
                setIsBeingDragged(isBeingDragged);
                /*
                 * If being flinged and user touches, stop the fling. isFinished
                 * will be false if being flinged.
                 */
                if (!mScroller.isFinished()) {
                    mScroller.forceFinished(true);
                }

                // Remember where the motion event started
                mLastMotionY = (int) ev.getY();
                mDownX = (int) ev.getX();
                mActivePointerId = ev.getPointerId(0);
                break;
            }
            case MotionEvent.ACTION_MOVE:
                final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
                if (activePointerIndex == -1) {
                    Log.e(TAG, "Invalid pointerId=" + mActivePointerId + " in onTouchEvent");
                    break;
                }

                final int y = (int) ev.getY(activePointerIndex);
                final int x = (int) ev.getX(activePointerIndex);
                int deltaY = mLastMotionY - y;
                final int xDiff = Math.abs(x - mDownX);
                final int yDiff = Math.abs(deltaY);
                final float touchSlop = getTouchSlop(ev);
                if (!mIsBeingDragged && yDiff > touchSlop && yDiff > xDiff) {
                    setIsBeingDragged(true);
                    if (deltaY > 0) {
                        deltaY -= touchSlop;
                    } else {
                        deltaY += touchSlop;
                    }
                }
                if (mIsBeingDragged) {
                    // Scroll to follow the motion event
                    mLastMotionY = y;
                    float scrollAmount;
                    int range;
                    range = getScrollRange();
                    if (mExpandedInThisMotion) {
                        range = Math.min(range, mMaxScrollAfterExpand);
                    }
                    // 处理通知栏上下滑动
                    if (deltaY < 0) {
                        scrollAmount = overScrollDown(deltaY);
                    } else {
                        scrollAmount = overScrollUp(deltaY, range);
                    }
                    // 这里根据scrollAmount决定是否是要处理通知栏的滚动
                    // 处理通知栏滚动，关于这个scrollAmount时如何计算的，看下面的详细解释
                    if (scrollAmount != 0.0f) {
                        customOverScrollBy((int) scrollAmount, mOwnScrollY,
                                range, getHeight() / 2);
                        mController.checkSnoozeLeavebehind();
                    }
                }
                break;
            case MotionEvent.ACTION_UP:
                if (mIsBeingDragged) {
                    final VelocityTracker velocityTracker = mVelocityTracker;
                    velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                    int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);
                    // 看是否满足放手后全部展开QS的条件
                    // 主要有几个判断条件，1.滑动速度大于mMinimumVelocity 或者 2.速度大于0,滑动距离大于mMinTopOverScrollToEscape
                    // 3.mScrolledToTopOnFirstDown是否为true，它表示Down事件是是否是个滑动事件，如果时在通知满屏时在显示QQS场景到通知中心全部显示
                    // 之间切换，那么它就是false。
                    // 4.mExpandedInThisMotion为false，表示这个事件不是展开单个通知的事件
                    if (shouldOverScrollFling(initialVelocity)) {
                        //通知中心滚动，最终状态时展开QS
                        onOverScrollFling(true, initialVelocity);
                    } else {
                        if (getChildCount() > 0) {
                            if ((Math.abs(initialVelocity) > mMinimumVelocity)) {
                                float currentOverScrollTop = getCurrentOverScrollAmount(true);
                                if (currentOverScrollTop == 0.0f || initialVelocity > 0) {
                                    mFlingAfterUpEvent = true;
                                    setFinishScrollingCallback(() -> {
                                        mFlingAfterUpEvent = false;
                                        InteractionJankMonitor.getInstance()
                                                .end(CUJ_NOTIFICATION_SHADE_SCROLL_FLING);
                                        setFinishScrollingCallback(null);
                                    });
                                    // 处理通知中心放手后的惯性滚动，注意：不是回弹效果。
                                    fling(-initialVelocity);
                                } else {
                                    //通知中心滚动，最终状态时展开QS，最终状态时回到原位置
                                    onOverScrollFling(false, initialVelocity);
                                }
                            } else {
                                if (mScroller.springBack(mScrollX, mOwnScrollY, 0, 0, 0,
                                        getScrollRange())) {
                                    animateScroll();
                                }
                            }
                        }
                    }
                    mActivePointerId = INVALID_POINTER;
                    endDrag();
                }

                break;
            case MotionEvent.ACTION_CANCEL:
                if (mIsBeingDragged && getChildCount() > 0) {
                    if (mScroller.springBack(mScrollX, mOwnScrollY, 0, 0, 0,
                            getScrollRange())) {
                        animateScroll();
                    }
                    mActivePointerId = INVALID_POINTER;
                    endDrag();
                }
                break;
            case MotionEvent.ACTION_POINTER_DOWN: {
                final int index = ev.getActionIndex();
                mLastMotionY = (int) ev.getY(index);
                mDownX = (int) ev.getX(index);
                mActivePointerId = ev.getPointerId(index);
                break;
            }
            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                mLastMotionY = (int) ev.getY(ev.findPointerIndex(mActivePointerId));
                mDownX = (int) ev.getX(ev.findPointerIndex(mActivePointerId));
                break;
        }
        return true;
    }
```

下面来介绍一下啊 overScrollDown() 方法，顺便来介绍一下 overScrollDown 是如何计算的。

```
// NotificationStackScrollLayout.java
    private float overScrollDown(int deltaY) {
        // deltaY 表示手势滑动的距离，下滑就是小于0的数
        deltaY = Math.min(deltaY, 0);
        // 当前滑动的距离
        float currentBottomAmount = getCurrentOverScrollAmount(false);
        // 当前滑动的距离加上手势滑动距离，表示最新的滑动距离
        float newBottomAmount = currentBottomAmount + deltaY;
        if (currentBottomAmount > 0) {
            setOverScrollAmount(newBottomAmount, false /* onTop */,
                    false /* animate */);
        }
        // Bottom overScroll might not grab all scrolling motion,
        // we have to scroll as well.
        float scrollAmount = newBottomAmount < 0 ? newBottomAmount : 0.0f;
        // 原来的滚动距离加上滑动的距离得到新的滚动位置
        float newScrollY = mOwnScrollY + scrollAmount;
        // 如果新的滚动位置小于0，那么就是判断是个滑动操作
        if (newScrollY < 0) {
            // 距离通知中心顶部位置距离
            float currentTopPixels = getCurrentOverScrolledPixels(true);
            // 设置当前距离顶部的距离
            setOverScrolledPixels(currentTopPixels - newScrollY,
                    true /* onTop */,
                    false /* animate */);
            // 非滚动操作，设置mOwnScrollY为0
            setOwnScrollY(0);
            scrollAmount = 0.0f;
        }
        return scrollAmount;
    }
```

### 位置计算

通知中心位置的更新，都是通过 NotificationStackScrollLayout.updateTopPadding() 来计算的，最终都是更新了 AmbientState 的一些属性值，在后面 OnPreDraw 时应用到View上面。

```
NotificationStackScrollLayout.updateTopPadding()
    NotificationStackScrollLayout.getLayoutMinHeight()
    NotificationStackScrollLayout.setTopPadding()
        NotificationStackScrollLayout.mTopPadding // 更新 mTopPadding 变量
        NotificationStackScrollLayout.updateAlgorithmHeightAndPadding()
            AmbientState.setLayoutHeight()
            NotificationStackScrollLayout.updateAlgorithmLayoutMinHeight()
                AmbientState.setLayoutMinHeight()
            AmbientState.setTopPadding()
        NotificationStackScrollLayout.updateContentHeight()
            AmbientState.setContentHeight()
        NotificationStackScrollLayout.updateStackPosition()
        NotificationStackScrollLayout.requestChildrenUpdate()
            getViewTreeObserver().addOnPreDrawListener()// 添加OnPreDrawListener，后面更新视图位置
            invalidate()
        NotificationStackScrollLayout.notifyHeightChangeListener
    NotificationStackScrollLayout.setExpandedHeight()
        NotificationStackScrollLayout.updateStackPosition() // 更新通知的位置
            AmbientState.setStackY() // 设置Y坐标，在绘制时计算子view的位置
            AmbientState.setStackEndHeight()
            AmbientState.setStackHeight()
        NotificationStackScrollLayout.setStackTranslation()
            AmbientState.setStackTranslation()
```

```
    public void updateTopPadding(float qsHeight, boolean animate) {
        // 取qs高度作为top padding
        int topPadding = (int) qsHeight;
        //获取最小高度，通常在有通知过多无法显示时指NotificationShelf的高度，
        // 如果通知全部显示，就是0
        int minStackHeight = getLayoutMinHeight();
        //计算mTopPaddingOverflow，这个变量在NotificationPanelViewController计算QS整体高度时会用到
        if (topPadding + minStackHeight > getHeight()) {
            // 如果toppadding加上最小高度大于通知中心本身高度，设置一个溢出值
            // 这个时候表示通知中心最顶端的位置已经超过原来通知的显示范围的底部了
            mTopPaddingOverflow = topPadding + minStackHeight - getHeight();
        } else {
            mTopPaddingOverflow = 0;
        }
        setTopPadding(topPadding, animate && !mKeyguardBypassEnabledProvider.getBypassEnabled());
        // mExpandedHeight 为整个下拉面板的高度。
        setExpandedHeight(mExpandedHeight);
    }
```

### 通知位置视图更新

```

NotificationStackScrollLayout.OnPreDrawListener.onPreDraw()
    NotificationStackScrollLayout.updateChildren()
        StackScrollAlgorithm.resetViewStates() //重置状态
            StackScrollAlgorithm.resetChildViewStates()
                NotificationStackScrollLayout.getChildCount()
                ExpandableView.resetViewState() //重置所有子view状态
            StackScrollAlgorithm.initAlgorithmState()
            StackScrollAlgorithm.updatePositionsForState()
                StackScrollAlgorithm.StackScrollAlgorithmState.visibleChildren.size()
                StackScrollAlgorithm.updateChild() // 更新所有可见的子view的状态
                    viewState.alpha
                    algorithmState.mCurrentYPosition
                    algorithmState.mCurrentExpandedYPosition
                    viewState.yTranslation = algorithmState.mCurrentYPosition;// 首先设置为累加的偏移量
                    StackScrollAlgorithm.setLocation()
                    viewState.yTranslation += ambientState.getStackY()// 再加上距通知中心距QS顶部的距离就是实际的偏移
        NotificationStackScrollLayout.applyCurrentState()
            ExpandableView.applyViewState()
                ExpandableNotificationRow.NotificationViewState.applyToView()
                    ExpandableViewState.applyToView()
                        ViewState.applyToView()
                            view.setTranslationX
                            view.setTranslationY
                            view.setTranslationZ
                            view.setScaleX
                            view.setAlpha
                            view.setVisibility
                        expandableView.setActualHeight()
                        expandableView.setDimmed()
                        expandableView.setHideSensitive()
                        expandableView.setBelowSpeedBump()
                        expandableView.setClipTopAmount()
                        expandableView.setTransformingInShelf()
                        expandableView.setInShelf()
                        expandableView.setHeadsUpIsVisible()
```

```
//ViewState.java
//更新 TranslationX，TranslationY，TranslationZ，ScaleX，ScaleY，Alpha，visibility
    public void applyToView(View view) {
        ......
        // apply xTranslation
        boolean animatingX = isAnimating(view, TAG_ANIMATOR_TRANSLATION_X);
        if (animatingX) {
            updateAnimationX(view);
        } else if (view.getTranslationX() != this.xTranslation){
            view.setTranslationX(this.xTranslation);
        }

        // apply yTranslation
        boolean animatingY = isAnimating(view, TAG_ANIMATOR_TRANSLATION_Y);
        if (animatingY) {
            updateAnimationY(view);
        } else if (view.getTranslationY() != this.yTranslation) {
            view.setTranslationY(this.yTranslation);
        }

        // apply zTranslation
        boolean animatingZ = isAnimating(view, TAG_ANIMATOR_TRANSLATION_Z);
        if (animatingZ) {
            updateAnimationZ(view);
        } else if (view.getTranslationZ() != this.zTranslation) {
            view.setTranslationZ(this.zTranslation);
        }

        // apply scaleX
        boolean animatingScaleX = isAnimating(view, SCALE_X_PROPERTY);
        if (animatingScaleX) {
            updateAnimation(view, SCALE_X_PROPERTY, scaleX);
        } else if (view.getScaleX() != scaleX) {
            view.setScaleX(scaleX);
        }

        // apply scaleY
        boolean animatingScaleY = isAnimating(view, SCALE_Y_PROPERTY);
        if (animatingScaleY) {
            updateAnimation(view, SCALE_Y_PROPERTY, scaleY);
        } else if (view.getScaleY() != scaleY) {
            view.setScaleY(scaleY);
        }

        int oldVisibility = view.getVisibility();
        boolean becomesInvisible = this.alpha == 0.0f
                || (this.hidden && (!isAnimating(view) || oldVisibility != View.VISIBLE));
        boolean animatingAlpha = isAnimating(view, TAG_ANIMATOR_ALPHA);
        if (animatingAlpha) {
            updateAlphaAnimation(view);
        } else if (view.getAlpha() != this.alpha) {
            // apply layer type
            boolean becomesFullyVisible = this.alpha == 1.0f;
            boolean newLayerTypeIsHardware = !becomesInvisible && !becomesFullyVisible
                    && view.hasOverlappingRendering();
            int layerType = view.getLayerType();
            int newLayerType = newLayerTypeIsHardware
                    ? View.LAYER_TYPE_HARDWARE
                    : View.LAYER_TYPE_NONE;
            if (layerType != newLayerType) {
                view.setLayerType(newLayerType, null);
            }

            // apply alpha
            view.setAlpha(this.alpha);
        }

        // apply visibility
        int newVisibility = becomesInvisible ? View.INVISIBLE : View.VISIBLE;
        if (newVisibility != oldVisibility) {
            if (!(view instanceof ExpandableView) || !((ExpandableView) view).willBeGone()) {
                // We don't want views to change visibility when they are animating to GONE
                view.setVisibility(newVisibility);
            }
        }
    }
```

## 通知栏滚动

通知中心在显示QQS场景到满屏显示通知场景切换时。
这部分逻辑比较简单，就是为 AmbientState 设置一个 mScrollY，在 `NotificationStackScrollLayout.updateChildren()` 时为通知中心的偏移量加上这个滚动值。

```
NotificationStackScrollLayout.customOverScrollBy()
    NotificationStackScrollLayout.onCustomOverScrolled
        NotificationStackScrollLayout.springBack() // 回弹
        NotificationStackScrollLayout.setOwnScrollY() // 设置滚动量
            AmbientState.setScrollY()
```

那么，在计算通知中心位置时这两种场景时如何统一计算的呢？

```
    private void initAlgorithmState(StackScrollAlgorithmState state, AmbientState ambientState) {
        state.scrollY = ambientState.getScrollY();
        state.mCurrentYPosition = -state.scrollY;
        state.mCurrentExpandedYPosition = -state.scrollY;
        
    }
```

上面的 state.scrollY 在滑动场景下时为0的，在滚动场景下为大于0的数值，那么就为 state.mCurrentYPosition 赋了一个负数的初始值，这样在绘制时所有的通知就会上移 scrollY 的距离。
