---
title: Android -- 手势导航启动多任务
categories:  Android Launcher
comments: true
tags: [Launcher3]
description: 介绍手势导航启动多任务的流程
date: 2021-2-12 10:00:00
---

## 相关类

 - `LauncherSwipeHandlerV2 <- BaseSwipeUpHandlerV2` 一个和手势相关的重要的类，它主要处理手势导航相关的操作，后面有一篇文章来专门介绍这个类。maybeUpdateRecentsAttachedState()方法判断显示或者隐藏多任务
 - ShelfPeekAnim 用来处理多任务 HIDE, PEEK, and OVERVIEW 状态之间切换时的动画
 - InputConsumer 事件消费者，有多种实现类，会根据系统处于不同的状态而生成不同的事件处理器，后面代码会详细介绍。
 - RecentsView RecentsView 有两个子类，一个是 LauncherRecentsView，一个是 FallbackRecentsView，FallbackRecentsView 显示在 RecentsActivity 中，用在多任务作为一个单独的 Activity 运行的场景，比如简易模式情况下。
 - `QuickstepAppTransitionManagerImpl <- LauncherAppTransitionManager` 管理打开和关闭App时切换操作，比如设置切换动画等。
 - TaskViewSimulator 一个模拟TaskView 和 RecentsView 的工具类，当我们在应用界面上划启动多任务时，有一个跟随手指移动的卡片，这个卡片就是这个类模拟出来的TaskView。
 - TaskView 多任务里面的一个卡片
 - TaskThumbnailView 任务卡片里面的界面截图
 - MotionPauseDetector 检测手势暂停


## 手势启动流程

通过手势启动多任务时，SystemUI会把触摸事件传递给`TouchInteractionService.onInputEvent`来处理，`TouchInteractionService`会交由合适的`InputConsumer`来处理，我们这里就介绍由其他应用界面启动时的`OtherActivityInputConsumer`。然后，事件将有两个比较重要的处理节点，一个是通过`PagedView.onTouchEvent`来处理，然后再由`OtherActivityInputConsumer`自己来处理。
从应用界面启动多任务，首先是处理当前window做位移和缩放，然后在合适的时机显示多任务卡片。

下面是事件的一个简单处理流程：

```
TouchInteractionService.onInputEvent
　RecentsAnimationDeviceState.isInSwipeUpTouchRegion //判断是否在手势区域
  mConsumer = newConsumer //初始化事件消费者
    TouchInteractionService.newBaseConsumer //根据不同的场景来生成不同的事件，我们就以在其他应用界面启动多任务情况来分析，这时会初始化　OtherActivityInputConsumer
      createOtherActivityInputConsumer
        new OtherActivityInputConsumer
          
  new AssistantInputConsumer　// 如果点击语音助手区域，就实例化　AssistantInputConsumer
  OtherActivityInputConsumer.onMotionEvent
    CachedEventDispatcher.dispatchEvent
      RecentsView.getEventDispatcher.accept // 多任务界面进行事件处理
        PagedView.onTouchEvent //后面详细介绍
    ACTION_DOWN
      OtherActivityInputConsumer.startTouchTrackingForWindowAnimation
        createLauncherSwipeHandler
          new LauncherSwipeHandlerV2
            BaseSwipeUpHandlerV2()
              BaseSwipeUpHandlerV2.initStateCallbacks() // 注册一些Launcher状态切换时的回调，后面代码介绍
            SwipeUpAnimationLogic()
              new TaskViewSimulator
                new RecentsOrientedState
                  RecentsOrientedState.initFlags();
                    RecentsOrientedState.updateHomeRotationSetting()
              TaskViewSimulator.setLayoutRotation()
                RecentsOrientedState.update() // 更新 PagedOrientationHandler（横竖屏），处理屏幕旋转
                  new PagedOrientationHandler.LANDSCAPE
                  new PagedOrientationHandler.PORTRAIT
        MotionPauseDetector.setOnMotionPauseListener //注册手势停顿时的回调，这个回调在介绍BaseSwipeUpHandlerV2时会详细介绍，主要处理一些多任务的状态切换
        BaseSwipeUpHandler.initWhenReady
          RecentsModel.getTasks
          ActivityInitListener.register
              BaseSwipeUpHandlerV2.onActivityInit
                BaseDraggingActivity.runOnceOnStart(callback=BaseSwipeUpHandlerV2.onLauncherStart)
        TaskAnimationManager.startRecentsAnimation // 开始多任务动画
          new RecentsAnimationCallbacks
          ActivityManagerWrapper.getInstance().startRecentsActivity（intent, null, mCallbacks）//启动多任务activity
    ACTION_MOVE
      BaseSwipeUpHandler.updateDisplacement
        AnimatedFloat.updateValue
          BaseSwipeUpHandlerV2.updateFinalShift()
            BaseSwipeUpHandler.applyWindowTransform() // 处理当前拖动的多任务卡片的缩放和位移
              TaskViewSimulator.apply()
                TransformParams.applySurfaceParams() // 应用位移和缩放的参数
            BaseSwipeUpHandlerV2.updateLauncherTransitionProgress // 设置多任务的缩放
              LauncherAnimUtils.SCALE_PROPERTY.setValue(mRecentsView, scale) // 设置多任务的缩放值
              RecentsView.FULLSCREEN_PROGRESS.setValue
                RecentsView.setFullscreenProgress() // 设置缩放和全屏之间的一些参数：TaskView透明度,可见性，缩放
                  TaskView.setFullscreenProgress()
                    TaskView.updateCurrentFullscreenParams
                      TaskView.FullscreenDrawParams.setProgress // 设置TaskView参数
                    TaskThumbnailView.setFullscreenParams()
                      TaskThumbnailView.setFullscreenParams()
            BaseSwipeUpHandler.forceHideOverView() //滑动过程中判断上划多少后需要强制隐藏多任务
              BaseSwipeUpHandlerV2.setShelfState(HIDE,,)
                BaseSwipeUpHandlerV2.maybeUpdateRecentsAttachedState() // 滑动过程中可能会切换小窗模式，这时就要判断是否隐藏或这显示多任务
                  BaseActivityInterface.DefaultAnimationFactory.setRecentsAttachedToAppWindow
                    BaseActivityInterface.DefaultAnimationFactory.updatePageAnimation
      RecentsAnimationDeviceState.isFullyGesturalNavMode
      MotionPauseDetector.setDisallowPause() // 设置是否允许暂停标志位
      MotionPauseDetector.addPosition()
        MotionPauseDetector.checkMotionPaused() // 判断手势是否暂停
          MotionPauseDetector.updatePaused()
    ACTION_UP
      OtherActivityInputConsumer.finishTouchTracking
        BaseSwipeUpHandlerV2.onGestureEnded
          BaseSwipeUpHandlerV2.handleNormalGestureEnd
            BaseSwipeUpHandlerV2.setShelfState(ShelfAnimState.OVERVIEW,,) // 多任务切换到浏览状态
              AnimationFactory.setShelfState -> LauncherActivityInterface.prepareRecentsUI.DefaultAnimationFactory.setShelfState
                ShelfPeekAnim.setShelfState
              BaseSwipeUpHandlerV2.maybeUpdateRecentsAttachedState
                BaseActivityInterface.DefaultAnimationFactory.setRecentsAttachedToAppWindow
            BaseSwipeUpHandlerV2.animateToProgress // 多任务从当前手势位置恢复到默认位置
              BaseSwipeUpHandlerV2.animateToProgressInternal
                BaseSwipeUpHandlerV2.maybeUpdateRecentsAttachedState
```

```
// TouchInteractionService.java
    private void onInputEvent(InputEvent ev) {
        if (!(ev instanceof MotionEvent)) {
            Log.e(TAG, "Unknown event " + ev);
            return;
        }
        MotionEvent event = (MotionEvent) ev;

        TestLogging.recordMotionEvent(
                TestProtocol.SEQUENCE_TIS, "TouchInteractionService.onInputEvent", event);

        if (!mDeviceState.isUserUnlocked()) {
            return;
        }

        Object traceToken = TraceHelper.INSTANCE.beginFlagsOverride(
                TraceHelper.FLAG_ALLOW_BINDER_TRACKING);

        final int action = event.getAction();
        if (action == ACTION_DOWN) {
            // 对　ACTION_DOWN　事件的处理
            mDeviceState.setOrientationTransformIfNeeded(event);
            if (mDeviceState.isInSwipeUpTouchRegion(event)) {
            　　//判断点击是否在mSwipeSharedState的矩形区域中
                ......
                //如果在这个区域中那么就会执行newConsumer的来创建一个事件消费者
                mConsumer = newConsumer(prevGestureState, mGestureState, event);
                mUncheckedConsumer = mConsumer;
            } else if (mDeviceState.isUserUnlocked() && mDeviceState.isFullyGesturalNavMode()) {
                mGestureState = createGestureState(mGestureState);
                ActivityManager.RunningTaskInfo runningTask = mGestureState.getRunningTask();
                if (mDeviceState.canTriggerAssistantAction(event, runningTask)) {
                    // 判断此处是不是点击Google的语音助手的窗口，如果存在就给语音助手。
                    mUncheckedConsumer = new AssistantInputConsumer(
                            this,
                            mGestureState,
                            InputConsumer.NO_OP, mInputMonitorCompat,
                            mOverviewComponentObserver.assistantGestureIsConstrained());
                } else {
                    mUncheckedConsumer = InputConsumer.NO_OP;
                }
            } else {
                mUncheckedConsumer = InputConsumer.NO_OP;
            }
        } else {
            //其他事件
            if (mUncheckedConsumer != InputConsumer.NO_OP) {
                mDeviceState.setOrientationTransformIfNeeded(event);
            }
        }

        if (mUncheckedConsumer != InputConsumer.NO_OP) {
            switch (event.getActionMasked()) {
                case ACTION_DOWN:
                case ACTION_UP:
                    break;
                default:
                    ActiveGestureLog.INSTANCE.addLog("onMotionEvent", event.getActionMasked());
                    break;
            }
        }

        boolean cleanUpConsumer = (action == ACTION_UP || action == ACTION_CANCEL)
                && mConsumer != null
                && !mConsumer.getActiveConsumerInHierarchy().isConsumerDetachedFromGesture();
        mUncheckedConsumer.onMotionEvent(event);

        if (cleanUpConsumer) {
            reset();
        }
        TraceHelper.INSTANCE.endFlagsOverride(traceToken);
    }
```

在 TouchInteractionService.onInputEvent 中会初始化 InputConsumer 事件消费者，有多种实现类，会根据系统处于不同的状态而生成不同的事件处理器。

 - DelegateInputConsumer：抽象类
 - AssistantInputConsumer：语音助手消费事件
 - OverscrollInputConsumer：
 - DeviceLockedInputConsumer：
 - OtherActivityInputConsumer：非桌面情况下启动多任务的事件消费者
 - OverviewInputConsumer：在桌面或者最近任务界面处理事件的消费者
 - OverviewWithoutFocusInputConsumer：
 - ResetGestureInputConsumer：
 - ScreenPinnedInputConsumer：
 - SysUiOverlayInputConsumer：

### 启动桌面

调用 ActivityManagerWrapper.getInstance().startRecentsActivity 去启动多任务activity，这里也就是桌面activity

```
        mCallbacks = new RecentsAnimationCallbacks(activityInterface.allowMinimizeSplitScreen());
        mCallbacks.addListener(new RecentsAnimationCallbacks.RecentsAnimationListener() {
            @Override
            public void onRecentsAnimationStart(RecentsAnimationController controller,
                    RecentsAnimationTargets targets) {
                if (mCallbacks == null) {
                    // It's possible for the recents animation to have finished and be cleaned up
                    // by the time we process the start callback, and in that case, just we can skip
                    // handling this call entirely
                    return;
                }
                mController = controller;
                mTargets = targets;
                mLastAppearedTaskTarget = mTargets.findTask(mLastGestureState.getRunningTaskId());
                mLastGestureState.updateLastAppearedTaskTarget(mLastAppearedTaskTarget);
            }

            @Override
            public void onRecentsAnimationCanceled(ThumbnailData thumbnailData) {
                if (thumbnailData != null) {
                    // If a screenshot is provided, switch to the screenshot before cleaning up
                    activityInterface.switchRunningTaskViewToScreenshot(thumbnailData,
                            () -> cleanUpRecentsAnimation(thumbnailData));
                } else {
                    cleanUpRecentsAnimation(null /* canceledThumbnail */);
                }
            }

            @Override
            public void onRecentsAnimationFinished(RecentsAnimationController controller) {
                cleanUpRecentsAnimation(null /* canceledThumbnail */);
            }

            @Override
            public void onTaskAppeared(RemoteAnimationTargetCompat appearedTaskTarget) {
                if (mController != null) {
                    if (mLastAppearedTaskTarget == null
                            || appearedTaskTarget.taskId != mLastAppearedTaskTarget.taskId) {
                        if (mLastAppearedTaskTarget != null) {
                            mController.removeTaskTarget(mLastAppearedTaskTarget);
                        }
                        mLastAppearedTaskTarget = appearedTaskTarget;
                        mLastGestureState.updateLastAppearedTaskTarget(mLastAppearedTaskTarget);
                    }
                }
            }
        });
```

```
Launcher.onStart
  BaseDraggingActivity.onStart
    BaseSwipeUpHandlerV2.onLauncherStart //前面初始化时 BaseDraggingActivity.runOnceOnStart 注册
    　initAnimFactory.run()　// 当不是从桌面启动时走这里
      　　mAnimationFactory = LauncherActivityInterface.prepareRecentsUI //返回AnimationFactory，提供显示多任务的动画，并显示多任务，后面有专门对这个方法的详细介绍
        　　LauncherActivityInterface.notifyRecentsOfOrientation
        　　    BaseActivityInterface.DefaultAnimationFactory.initUI
                StateManager().goToState
                    RecentsViewStateController.setState
                        getContentAlphaProperty().set()
                            RecentsView.CONTENT_ALPHA.setValue
                                LauncherRecentsView.setContentAlpha
                                    RecentsView.setContentAlpha
                                        RecentsView.setVisibility
      　　BaseSwipeUpHandlerV2.maybeUpdateRecentsAttachedState // 显示或者隐藏多任务
        　　BaseActivityInterface.DefaultAnimationFactory.setRecentsAttachedToAppWindow
          　　BaseActivityInterface.DefaultAnimationFactory.updatePageAnimation //定义多任务其他卡片进入和退出的动画效果
```

多任务的显示：
Launcher.onStart->WindowTransformSwipeHandler.onLauncherStart->LauncherActivityControllerHelper.prepareRecentsUI->LauncherStateManager.goToState->RecentsViewStateController.setState->BaseRecentsViewStateController.setState->RecentsView.CONTENT_ALPHA.setValue -> RecentsView.setContentAlpha -> setVisibility

##　注册状态回调

当调用　mStateCallback.setState　切换状态时会执行相关的回调函数，这部分在BaseSwipeUpHandlerV2中详细介绍。



## 准备多任务UI

prepareRecentsUI() 返回AnimationFactory，提供显示多任务的动画，并显示多任务

```
// LauncherActivityInterface.java
    public AnimationFactory prepareRecentsUI(RecentsAnimationDeviceState deviceState,
            boolean activityVisible, Consumer<AnimatorPlaybackController> callback) {
        notifyRecentsOfOrientation(deviceState);
        DefaultAnimationFactory factory = new DefaultAnimationFactory(callback) {
            @Override
            public void setShelfState(ShelfAnimState shelfState, Interpolator interpolator,
                    long duration) {
                mActivity.getShelfPeekAnim().setShelfState(shelfState, interpolator, duration);
            }

            @Override
            protected void createBackgroundToOverviewAnim(BaseQuickstepLauncher activity,
                    PendingAnimation pa) {
                //Flyme.zhangxin modify {@
                if (activity.getDeviceProfile().isMultiWindowMode || SysUINavigationMode.getMode(activity) != SysUINavigationMode.Mode.NO_BUTTON) {
                    super.createBackgroundToOverviewAnim(activity, pa);
                }
                //@}

                if (!activity.getDeviceProfile().isVerticalBarLayout()
                        && SysUINavigationMode.getMode(activity) != Mode.NO_BUTTON) {
                    // Don't animate the shelf when the mode is NO_BUTTON, because we
                    // update it atomically.
                    pa.add(activity.getStateManager().createStateElementAnimation(
                            INDEX_SHELF_ANIM,
                            BACKGROUND_APP.getVerticalProgress(activity),
                            OVERVIEW.getVerticalProgress(activity)));
                }

                // Animate the blur and wallpaper zoom
                float fromDepthRatio = BACKGROUND_APP.getDepth(activity);
                float toDepthRatio = OVERVIEW.getDepth(activity);
                pa.addFloat(getDepthController(),
                        new ClampedDepthProperty(fromDepthRatio, toDepthRatio),
                        fromDepthRatio, toDepthRatio, LINEAR);

            }
        };

        BaseQuickstepLauncher launcher = factory.initUI();
        // Since all apps is not visible, we can safely reset the scroll position.
        // This ensures then the next swipe up to all-apps starts from scroll 0.
        launcher.getAppsView().reset(false /* animate */);
        return factory;
    }
```

```
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

## 多任务显示和隐藏的动画

多任务显示和隐藏的时机一个是调用maybeUpdateRecentsAttachedState来实现，它有两个重载的方法，它决定何时隐藏或者显示RecentsView，滑动过程中，会根据滑动的距离和速度来决定是只显示当前滑动图片或者是显示多任务，而且显示和隐藏多任务过程中可能要有切换动画，在滑动过程中，可能会切换小窗模式，这些时候都就要判断是否隐藏或者显示多任务。 
另外一个是在滑动卡片结束时或者在桌面界面上划手势结束时会启动显示多任务的逻辑。

```
      BaseSwipeUpHandlerV2.maybeUpdateRecentsAttachedState // 显示或者隐藏多任务
        BaseActivityInterface.DefaultAnimationFactory.setRecentsAttachedToAppWindow
          BaseActivityInterface.DefaultAnimationFactory.updatePageAnimation //定义多任务其他卡片进入和退出的动画效果
```

```
// BaseSwipeUpHandlerV2.java

    private void maybeUpdateRecentsAttachedState(boolean animate) {
        if (!mDeviceState.isFullyGesturalNavMode() || mRecentsView == null) {
            return;
        }
        RemoteAnimationTargetCompat runningTaskTarget = mRecentsAnimationTargets != null
                ? mRecentsAnimationTargets.findTask(mGestureState.getRunningTaskId())
                : null;
        boolean recentsAttachedToAppWindow;
        if (mGestureState.getEndTarget() != null) {
            recentsAttachedToAppWindow = mGestureState.getEndTarget().recentsAttachedToAppWindow;
            //Flyme.zhangxin add{@ 从当前卡片回退到应用并且其他卡片没有显示情况下没必要再让多任务显示
            if (enableCustomScale() && mGestureState.getEndTarget() == LAST_TASK && !mIsShelfPeeking && !mIsLikelyToStartNewTask) {
                recentsAttachedToAppWindow = false;
            }
            //@}
        } else if (mContinuingLastGesture
                && mRecentsView.getRunningTaskIndex() != mRecentsView.getNextPage()) {
            recentsAttachedToAppWindow = true;
        } else if (runningTaskTarget != null && isNotInRecents(runningTaskTarget)) {
            // The window is going away so make sure recents is always visible in this case.
            recentsAttachedToAppWindow = true;
        } else {
            recentsAttachedToAppWindow = mIsShelfPeeking || mIsLikelyToStartNewTask;
            //Flyme.zhangxin add{@
            if (enableCustomScale() && mShelfState == OVERVIEW && !recentsAttachedToAppWindow) {
                recentsAttachedToAppWindow = true;
            }
            //@}
        }
        //Flyme.zhangxin add{@
        mAnimationFactory.setGestureTarget(mShelfState, !mReadyGoToOverView || isQuickStartNewTask());
        //@}
        mAnimationFactory.setRecentsAttachedToAppWindow(recentsAttachedToAppWindow, animate);
    }
```

```
// BaseActivityInterface.java
        @Override
        public void setRecentsAttachedToAppWindow(boolean attached, boolean animate) {
            //Flyme.zhangxin add{@
            if (updatePageAnimation(attached, animate)) {
                // 如果是滑动过程中切换小窗时需要做的多任务的动画，这里直接返回
                return;
            }
            //@}
            // 后面就做进入多任务时的动画
            if (mIsAttachedToWindow == attached && animate) {
                return;
            }
            mIsAttachedToWindow = attached;
            RecentsView recentsView = mActivity.getOverviewPanel();
            Animator fadeAnim = mActivity.getStateManager()
                    .createStateElementAnimation(INDEX_RECENTS_FADE_ANIM, attached ? 1 : 0);

            float fromTranslation = attached ? 1 : 0;
            float toTranslation = attached ? 0 : 1;
            mActivity.getStateManager()
                    .cancelStateElementAnimation(INDEX_RECENTS_TRANSLATE_X_ANIM);
            if (!recentsView.isShown() && animate) {
                ADJACENT_PAGE_OFFSET.set(recentsView, fromTranslation);
            } else {
                fromTranslation = ADJACENT_PAGE_OFFSET.get(recentsView);
            }
            if (!animate) {
                ADJACENT_PAGE_OFFSET.set(recentsView, toTranslation);
            } else {
                mActivity.getStateManager().createStateElementAnimation(
                        INDEX_RECENTS_TRANSLATE_X_ANIM,
                        fromTranslation, toTranslation).start();
            }

            fadeAnim.setInterpolator(attached ? INSTANT : ACCEL_2);
            fadeAnim.setDuration(animate ? RECENTS_ATTACH_DURATION : 0).start();

        }
```

在其他应用界面，滑动手势结束时：

```
AnimationSuccessListener.onAnimationEnd()
    AnimatorPlaybackController.OnAnimationEndDispatcher.onAnimationSuccess()
        BaseActivityInterface.createActivityInterface.setEndAction().StateManager().goToState(OVERVIEW)
            StateManager.onStateTransitionEnd()
                LauncherRecentsView.onStateTransitionComplete()
                    LauncherRecentsView.setFreezeViewVisibility()
                        RecentsView.setVisibility()
```

在桌面时滑动手势结束时：

```
FlingAndHoldTouchController.goToOverviewOnDragEnd.onAnimationEnd()
    AnimationSuccessListener.onAnimationEnd
        FlingAndHoldTouchController.goToOverviewOnDragEnd.onAnimationSuccess(OVERVIEW)
            PortraitStatesTouchController.onSwipeInteractionCompleted()
                AbstractStateChangeTouchController.onSwipeInteractionCompleted()
                    FlingAndHoldTouchController.goToTargetState()
                        AbstractStateChangeTouchController.goToTargetState()
                            StateManager.goToState()
                                StateManager.onStateTransitionEnd()
                                    LauncherRecentsView.onStateTransitionComplete()
                                        LauncherRecentsView.setFreezeViewVisibility()
                                            RecentsView.setVisibility()
```

## 多任务缩放和位置

手势在滑动过程中多任务的缩放和位置由 `BaseSwipeUpHandlerV2.updateFinalShift()` 进行，具体介绍在　BaseSwipeUpHandlerV2。

## 快切

在滑动屏幕下方的一块区域时，会快速切换应用，其实这个还是多任务 RecentsView 的全屏切换，切换完成后再启动对应的应用。
快切操作主要时通过RecentsView的切换来完成的，然后在启动目标TaskView，事件的处理通过　CachedEventDispatcher.dispatchEvent　分发到　PagedView.onTouchEvent 处理。
  OtherActivityInputConsumer.onMotionEvent -> CachedEventDispatcher.dispatchEvent ->  RecentsView.getEventDispatcher.accept -> PagedView.onTouchEvent ->MotionEvent.ACTION_MOVE -> PortraitPagedViewHandler.set -> View::scrollBy

## 推荐文章

Android R中虚拟按键的详细设计与实现　https://blog.csdn.net/chen364567628/article/details/107525744
走近Android -R 11 手势的详细设计与实现源码分析　https://blog.csdn.net/chen364567628/article/details/109126604