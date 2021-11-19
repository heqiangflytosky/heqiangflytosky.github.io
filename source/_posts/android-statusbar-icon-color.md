---
title: SystemUI -- 状态栏字体和图标变色原理
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍状态栏字体和图标变色原理
date: 2021-6-1 10:00:00
---

## 概述

App 端设置SystemUIVisibility的方式几种。一是在任意一个已经显示在窗口上的控件调用View.setSystemUiVisibility(),二是直接在窗口的 LayoutParams.systemUiVisibility 上进行设置并通过WindowManager.updateViewLayout()方法使其生效，还有就是通过 `getWindow().getDecorView().getWindowInsetsController().setSystemBarsAppearance(WindowInsetsController.APPEARANCE_LIGHT_STATUS_BARS, WindowInsetsController.APPEARANCE_LIGHT_STATUS_BARS)` 的方式。

## SystemUI 实现图标变色的方法

代码基于 Android R。
切换应用的时候我们可以看到，不同风格颜色的应用在切换时状态栏的数字和icon图标可能会改变颜色，那么这是怎么实现的呢？
实现这个功能，需要实现DarkReceiver这个接口。

```
// DarkIconDispatcher.java
    @ProvidesInterface(version = DarkReceiver.VERSION)
    interface DarkReceiver {
        int VERSION = 1;
        void onDarkChanged(Rect area, float darkIntensity, int tint);
    }
```

我们就以状态栏的数字时钟为例来介绍。

Clock.java 实现了 DarkReceiver，当收到 onDarkChanged() 回调的时候，会根据 area 和 tint 的值来获取字体需要设置的颜色。

```
    @Override
    public void onDarkChanged(Rect area, float darkIntensity, int tint) {
        mNonAdaptedColor = DarkIconDispatcher.getTint(area, this, tint);
        if (!mUseWallpaperTextColor) {
            setTextColor(mNonAdaptedColor);
        }
    }
```

然后在 onAttachedToWindow() 中注册一下监听，那么当应用的变化时就会通过 DarkIconDispatcherImpl 来发送通知。

```
    @Override
    protected void onAttachedToWindow() {
            ......
            if (mShowDark) {
                Dependency.get(DarkIconDispatcher.class).addDarkReceiver(this);
            }
            ......
    }
```

那么什么时候会回调呢？下面我们就来介绍一下这个流程。

## App

首先要有 App 端设置当前应用的 StatusBar 类型，比如：View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR 或者 WindowInsetsController.APPEARANCE_LIGHT_STATUS_BARS。

先看通过 View.setSystemUiVisibility() 来设置：

```
// View.java
    @Deprecated
    public void setSystemUiVisibility(int visibility) {
        if (visibility != mSystemUiVisibility) {
            // 更新View自己的属性值
            mSystemUiVisibility = visibility;
            if (mParent != null && mAttachInfo != null && !mAttachInfo.mRecomputeGlobalAttributes) {
                //通知父控件这一变化
                mParent.recomputeViewAttributes(this);
            }
        }
    }
```

然后在addWindow时会调用 ViewRootImpl.collectViewAttributes() 方法收集控件树中每个 View 所保存的 SystemUIVisibility。 如下:

```
ActivityThread.handleResumeActivity()
    WindowManagerImpl.addView()
        WindowManagerGlobal.addView()
            ViewRootImpl.setView()
                ViewRootImpl.collectViewAttributes()
                    ViewGroup.dispatchCollectViewAttributes
                        View.dispatchCollectViewAttributes
                            View.performCollectViewAttributes
                ViewRootImpl.adjustLayoutParamsForCompatibility()
                mWindowSession.addToDisplayAsUser
```

```
    private boolean collectViewAttributes() {
        if (mAttachInfo.mRecomputeGlobalAttributes) {
            //Log.i(mTag, "Computing view hierarchy attributes!");
            mAttachInfo.mRecomputeGlobalAttributes = false;
            boolean oldScreenOn = mAttachInfo.mKeepScreenOn;
            mAttachInfo.mKeepScreenOn = false;
            //清空所保存的SystemUiVisibility
            mAttachInfo.mSystemUiVisibility = 0;
            mAttachInfo.mHasSystemUiListeners = false;
            //遍历整个控件树
            mView.dispatchCollectViewAttributes(mAttachInfo, 0);
            //移除被禁用的SystemUiVisibility标记
            mAttachInfo.mSystemUiVisibility &= ~mAttachInfo.mDisabledSystemUiVisibility;
            WindowManager.LayoutParams params = mWindowAttributes;
            mAttachInfo.mSystemUiVisibility |= getImpliedSystemUiVisibility(params);
            mCompatibleVisibilityInfo.globalVisibility =
                    (mCompatibleVisibilityInfo.globalVisibility & ~View.SYSTEM_UI_FLAG_LOW_PROFILE)
                            | (mAttachInfo.mSystemUiVisibility & View.SYSTEM_UI_FLAG_LOW_PROFILE);
            dispatchDispatchSystemUiVisibilityChanged(mCompatibleVisibilityInfo);
            if (mAttachInfo.mKeepScreenOn != oldScreenOn
                    || mAttachInfo.mSystemUiVisibility != params.subtreeSystemUiVisibility
                    || mAttachInfo.mHasSystemUiListeners != params.hasSystemUiListeners) {
                applyKeepScreenOnFlag(params);
                //将SystemUiVisibility保存到窗口的LayoutParams
                params.subtreeSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
                params.hasSystemUiListeners = mAttachInfo.mHasSystemUiListeners;
                mView.dispatchWindowSystemUiVisiblityChanged(mAttachInfo.mSystemUiVisibility);
                return true;
            }
        }
```

```
View.java
    void performCollectViewAttributes(AttachInfo attachInfo, int visibility) {
        if ((visibility & VISIBILITY_MASK) == VISIBLE) {
            if ((mViewFlags & KEEP_SCREEN_ON) == KEEP_SCREEN_ON) {
                attachInfo.mKeepScreenOn = true;
            }
            // 这里进行或操作，把子View树的 mSystemUiVisibility 都保存在 AttachInfo 中
            attachInfo.mSystemUiVisibility |= mSystemUiVisibility;
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnSystemUiVisibilityChangeListener != null) {
                attachInfo.mHasSystemUiListeners = true;
            }
        }
    }
```

计算 insetsFlags.appearance

```
ViewRootImpl.java
    public static void adjustLayoutParamsForCompatibility(WindowManager.LayoutParams inOutParams) {
        if (sNewInsetsMode != NEW_INSETS_MODE_FULL) {
            return;
        }
        final int sysUiVis = inOutParams.systemUiVisibility | inOutParams.subtreeSystemUiVisibility;
        final int flags = inOutParams.flags;
        final int type = inOutParams.type;
        final int adjust = inOutParams.softInputMode & SOFT_INPUT_MASK_ADJUST;

        // 更新 inOutParams.insetsFlags.appearance，这个值传递给WMS后计算状态栏颜色时需要用到
        if ((inOutParams.privateFlags & PRIVATE_FLAG_APPEARANCE_CONTROLLED) == 0) {
            inOutParams.insetsFlags.appearance = 0;
            if ((sysUiVis & SYSTEM_UI_FLAG_LOW_PROFILE) != 0) {
                inOutParams.insetsFlags.appearance |= APPEARANCE_LOW_PROFILE_BARS;
            }
            if ((sysUiVis & SYSTEM_UI_FLAG_LIGHT_STATUS_BAR) != 0) {
                inOutParams.insetsFlags.appearance |= APPEARANCE_LIGHT_STATUS_BARS;
            }
            if ((sysUiVis & SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR) != 0) {
                inOutParams.insetsFlags.appearance |= APPEARANCE_LIGHT_NAVIGATION_BARS;
            }
        }

        if ((inOutParams.privateFlags & PRIVATE_FLAG_BEHAVIOR_CONTROLLED) == 0) {
            if ((sysUiVis & SYSTEM_UI_FLAG_IMMERSIVE_STICKY) != 0
                    || (flags & FLAG_FULLSCREEN) != 0) {
                inOutParams.insetsFlags.behavior = BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE;
            } else if ((sysUiVis & SYSTEM_UI_FLAG_IMMERSIVE) != 0) {
                inOutParams.insetsFlags.behavior = BEHAVIOR_SHOW_BARS_BY_SWIPE;
            } else {
                inOutParams.insetsFlags.behavior = BEHAVIOR_SHOW_BARS_BY_TOUCH;
            }
        }

        inOutParams.privateFlags &= ~PRIVATE_FLAG_INSET_PARENT_FRAME_BY_IME;

        if ((inOutParams.privateFlags & PRIVATE_FLAG_FIT_INSETS_CONTROLLED) != 0) {
            return;
        }

        int types = inOutParams.getFitInsetsTypes();
        boolean ignoreVis = inOutParams.isFitInsetsIgnoringVisibility();

        if (((sysUiVis & SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN) != 0
                || (flags & FLAG_LAYOUT_IN_SCREEN) != 0)
                || (flags & FLAG_TRANSLUCENT_STATUS) != 0) {
            types &= ~Type.statusBars();
        }
        if ((sysUiVis & SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION) != 0
                || (flags & FLAG_TRANSLUCENT_NAVIGATION) != 0) {
            types &= ~Type.systemBars();
        }
        if (type == TYPE_TOAST || type == TYPE_SYSTEM_ALERT) {
            ignoreVis = true;
        } else if ((types & Type.systemBars()) == Type.systemBars()) {
            if (adjust == SOFT_INPUT_ADJUST_RESIZE) {
                types |= Type.ime();
            } else {
                inOutParams.privateFlags |= PRIVATE_FLAG_INSET_PARENT_FRAME_BY_IME;
            }
        }
        inOutParams.setFitInsetsTypes(types);
        inOutParams.setFitInsetsIgnoringVisibility(ignoreVis);

        // The fitting of insets are not really controlled by the clients, so we remove the flag.
        inOutParams.privateFlags &= ~PRIVATE_FLAG_FIT_INSETS_CONTROLLED;
    }
```

```
ViewRootImpl.doTraversal()
    ViewRootImpl.performTraversals()
        ViewRootImpl.adjustLayoutParamsForCompatibility()
        ViewRootImpl.relayoutWindow() 
```

将 WindowManager.LayoutParams 传递到 server 端。会把LayoutParams.subtreeSystemUiVisibility、insetsFlags.appearance 以及LayoutParams.sytemUiVisibility字段送入WMS.


## system_server

```
WindowManagerService.relayoutWindow()
    WindowSurfacePlacer.performSurfacePlacement()
        WindowSurfacePlacer.performSurfacePlacementLoop()
            RootWindowContainer.performSurfacePlacement()
                RootWindowContainer.performSurfacePlacementNoTrace()
                   RootWindowContainer.applySurfaceChangesTransaction()
                       DisplayContent.applySurfaceChangesTransaction()
                           DisplayPolicy.finishPostLayoutPolicyLw()
                               DisplayPolicy.updateSystemUiVisibilityLw()
                                    DisplayPolicy.updateLightStatusBarAppearanceLw()
```

```
WindowManagerService.java

    public int relayoutWindow(Session session, IWindow client, int seq, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags,
            long frameNumber, Rect outFrame, Rect outContentInsets,
            Rect outVisibleInsets, Rect outStableInsets, Rect outBackdropFrame,
            DisplayCutout.ParcelableWrapper outCutout, MergedConfiguration mergedConfiguration,
            SurfaceControl outSurfaceControl, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls, Point outSurfaceSize,
            SurfaceControl outBLASTSurfaceControl) {
            
            ......
            if (attrs != null) {
                displayPolicy.adjustWindowParamsLw(win, attrs, pid, uid);
                win.mToken.adjustWindowParams(win, attrs);
                // if they don't have the permission, mask out the status bar bits
                if (seq == win.mSeq) {
                    int systemUiVisibility = attrs.systemUiVisibility
                            | attrs.subtreeSystemUiVisibility;
                    if ((systemUiVisibility & DISABLE_MASK) != 0) {
                        if (!hasStatusBarPermission(pid, uid)) {
                            systemUiVisibility &= ~DISABLE_MASK;
                        }
                    }
                    //保存到WindowState.mSystemUiVisibility
                    win.mSystemUiVisibility = systemUiVisibility;
                }
```

然后获取当前的top window的SystemUiVisibility得到SystemUiVisibility数组，然后传给SystemUI。

```
DisplayPolicy.updateSystemUiVisibilityLw()
    DisplayPolicy.updateLightStatusBarAppearanceLw()
        PolicyControl.getSystemUiVisibility()
        InsetsFlags.getAppearance()
    new AppearanceRegion()
    StatusBarManagerService.onSystemBarAppearanceChanged()
        IStatusBar.onSystemBarAppearanceChanged() // 回调到SystemUI
```

```
    int updateSystemUiVisibilityLw() {
        // 获取 fullscreenAppearance
        final int fullscreenAppearance = updateLightStatusBarAppearanceLw(0 /* vis */,
                mTopFullscreenOpaqueWindowState, mTopFullscreenOpaqueOrDimmingWindowState);
        final int dockedAppearance = updateLightStatusBarAppearanceLw(0 /* vis */,
                mTopDockedOpaqueWindowState, mTopDockedOpaqueOrDimmingWindowState);
        final boolean inSplitScreen =
                mService.mRoot.getDefaultTaskDisplayArea().isSplitScreenModeActivated();
        ......
        //根据 fullscreenAppearance 创建 AppearanceRegion
        final AppearanceRegion[] appearanceRegions = inSplitScreen
                ? new AppearanceRegion[]{
                        new AppearanceRegion(fullscreenAppearance, fullscreenStackBounds),
                        new AppearanceRegion(dockedAppearance, dockedStackBounds)}
                : new AppearanceRegion[]{
                        new AppearanceRegion(fullscreenAppearance, fullscreenStackBounds)};
        String cause = win.toString();
        mHandler.post(() -> {
            ......
                statusBar.onSystemBarAppearanceChanged(displayId, appearance,
                        appearanceRegions, isNavbarColorManagedByIme);
                statusBar.topAppWindowChanged(displayId, isFullscreen, isImmersive);

                ......
            }
        });
        return diff;
    }
    
        @Override
        public void onSystemBarAppearanceChanged(int displayId, @Appearance int appearance,
                AppearanceRegion[] appearanceRegions, boolean navbarColorManagedByIme) {
            final UiState state = getUiState(displayId);
            if (!state.appearanceEquals(appearance, appearanceRegions, navbarColorManagedByIme)) {
                state.setAppearance(appearance, appearanceRegions, navbarColorManagedByIme);
            }
            if (mBar != null) {
                try {
                    // 回调到SystemUI
                    mBar.onSystemBarAppearanceChanged(displayId, appearance, appearanceRegions,
                            navbarColorManagedByIme);
                } catch (RemoteException ex) { }
            }
        }
```

## SystemUI

SystemUI根据system_server传递过来的 AppearanceRegion[] 的appearance得到当前需要的状态栏图标的颜色。

```
CommandQueue.onSystemBarAppearanceChanged()
    CommandQueue.H.handleMessage()
        MSG_SYSTEM_BAR_APPEARANCE_CHANGED
            StatusBar.onSystemBarAppearanceChanged()
                LightBarController.onStatusBarAppearanceChanged()
                    mAppearanceRegions = appearanceRegions // 赋值 mAppearanceRegions
                    LightBarController.onStatusBarModeChanged()
                        LightBarController.updateStatus()
                            LightBarTransitionsController.isLight() //
                            LightBarTransitionsController.setIconsDark(true)
                                LightBarTransitionsController.animateIconTint()
                                    LightBarTransitionsController.setIconTintInternal()
                            LightBarTransitionsController.setIconsDark(false)
```

如果状态有变化，就更新状态：

```
// LightBarController.java
    private void updateStatus() {
        final int numStacks = mAppearanceRegions.length;
        int numLightStacks = 0;

        // We can only have maximum one light stack.
        int indexLightStack = -1;
        // 根据 AppearanceRegions 的 mAppearance 来获取 numLightStacks 的个数。
        for (int i = 0; i < numStacks; i++) {
            if (isLight(mAppearanceRegions[i].getAppearance(), mStatusBarMode,
                    APPEARANCE_LIGHT_STATUS_BARS)) {
                numLightStacks++;
                indexLightStack = i;
            }
        }

        // 如果所有的 stacks 都是 light, 那么就用黑色图标
        if (numLightStacks == numStacks) {
            mStatusBarIconController.setIconsDarkArea(null);
            mStatusBarIconController.getTransitionsController().setIconsDark(true, animateChange());

        }

        // 如果没有一个stack 是 light, 那么就用白色图标
        else if (numLightStacks == 0) {
            mStatusBarIconController.getTransitionsController().setIconsDark(
                    false, animateChange());
        }

        // 如果不是所有的stack都一样
        else {
            mStatusBarIconController.setIconsDarkArea(
                    mAppearanceRegions[indexLightStack].getBounds());
            mStatusBarIconController.getTransitionsController().setIconsDark(true, animateChange());
        }
    }
    
    private static boolean isLight(int appearance, int barMode, int flag) {
        final boolean isTransparentBar = (barMode == MODE_TRANSPARENT
                || barMode == MODE_LIGHTS_OUT_TRANSPARENT);
        final boolean light = (appearance & flag) != 0;
        return isTransparentBar && light;
    }
```

AppearanceRegion 是指定区域指定 mAppearance，后面SystemUI会根据这个 mAppearance来计算出需要显示的颜色值。

```
    private void animateIconTint(float targetDarkIntensity, long delay,
            long duration) {
        if (mNextDarkIntensity == targetDarkIntensity) {
            return;
        }
        if (mTintAnimator != null) {
            mTintAnimator.cancel();
        }
        mNextDarkIntensity = targetDarkIntensity;
        mTintAnimator = ValueAnimator.ofFloat(mDarkIntensity, targetDarkIntensity);
        // 更新Icon的颜色值
        mTintAnimator.addUpdateListener(
                animation -> setIconTintInternal((Float) animation.getAnimatedValue()));
        mTintAnimator.setDuration(duration);
        mTintAnimator.setStartDelay(delay);
        mTintAnimator.setInterpolator(Interpolators.LINEAR_OUT_SLOW_IN);
        // 开始变色动画
        mTintAnimator.start();
    }
```

```
    private void setIconTintInternal(float darkIntensity) {
        mDarkIntensity = darkIntensity;
        dispatchDark();
    }
```

```
LightBarTransitionsController.setIconTintInternal()
    mDarkIntensity = darkIntensity;
    LightBarTransitionsController.dispatchDark()
        DarkIconDispatcherImpl.applyDarkIntensity()
            mIconTint = XX
            DarkIconDispatcherImpl.applyIconTint()
                DarkReceiver.onDarkChanged() // 发送通知给所有的Receivers
                    DarkIconDispatcher.getTint() // 获取颜色
```

根据 mDarkIntensity 来得到 mIconTint。

```
    @Override
    public void applyDarkIntensity(float darkIntensity) {
        // 设置 mDarkIntensity
        mDarkIntensity = darkIntensity;
        boolean isCts = SystemUICommonUtils.isCtsRunningStatic();
        // 根据 darkIntensity 来得到 mIconTint
        mIconTint = (int) ArgbEvaluator.getInstance().evaluate(darkIntensity,
                mLightModeIconColorSingleTone, isCts ? StatusBarFilterControl.DARK_FILTER_COLOR_FOR_CTS : mDarkModeIconColorSingleTone);
        applyIconTint();
    }
```

根据 mIconTint 来得到图标颜色值。

```
    static int getTint(Rect tintArea, View view, int color) {
        // 看是否是在指定的区域，如果在，就设置为 color
        if (isInArea(tintArea, view)) {
            return color;
        } else {
            return DEFAULT_ICON_TINT;
        }
    }
    
    static boolean isInArea(Rect area, View view) {
        if (area.isEmpty()) {
            return true;
        }
        sTmpRect.set(area);
        view.getLocationOnScreen(sTmpInt2);
        int left = sTmpInt2[0];

        int intersectStart = Math.max(left, area.left);
        int intersectEnd = Math.min(left + view.getWidth(), area.right);
        int intersectAmount = Math.max(0, intersectEnd - intersectStart);

        boolean coversFullStatusBar = area.top <= 0;
        boolean majorityOfWidth = 2 * intersectAmount > view.getWidth();
        return majorityOfWidth && coversFullStatusBar;
    }
```

决定状态栏颜色的属性依次是：

LightBarController.mAppearanceRegions[] -> LightBarTransitionsController.mDarkIntensity -> DarkIconDispatcher.mIconTint


## 参考文章

https://blog.csdn.net/csdnxialei/article/details/95059041
https://www.codeleading.com/article/3648999178/
