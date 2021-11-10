---
title: SystemUI -- 状态栏字体和图标颜色随背景而改变
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍状态栏字体和图标颜色随背景而改变的原理
date: 2021-6-1 10:00:00
---

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
        mNonAdaptedColor = 0xffff0000;
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

## system_server

主要是根据当前的top window的SystemUiVisibility得到SystemUiVisibility数组，然后传给SystemUI。

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