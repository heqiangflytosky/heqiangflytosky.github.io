---
title: Android -- 状态栏导航栏相关设置
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍 Android 状态栏导航栏相关设置
date: 2021-5-1 10:00:00
---


## 设置状态栏

### 沉浸式状态栏：
https://www.jianshu.com/p/e1b716b1d508
https://www.jianshu.com/p/752f4551e134
https://www.jianshu.com/p/1f2ce8209f24

FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS
如果软件没有对状态栏做处理，状态栏自然状态是颜色不透明（全黑）、字体为白色。

### 设置状态栏透明

只有4.4以后才可以设置，4.4以下只会显示默认状态栏。
4.4以后之所以可以设置，是因为API新添属性windowTranslucentStatus。

第一种方式，通过 style xml 设置：
```
    <style name="StatusBarActivityTheme" parent="Theme.AppCompat.Light">
        <!-- Customize your theme here. -->

        <item name="android:windowTranslucentStatus">true</item>
    </style>
```

第二种方式，通过代码设置：

```
        if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT){
            getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        }
```

你会发现这是后状态栏还是有颜色，这是因为有状态栏背景存在。

### 设置状态栏颜色

第一种方式：

```
    <style name="StatusBarActivityTheme" parent="Theme.AppCompat.Light">
        <item name="colorPrimary">#0000ff</item> // titlebar等主题颜色
        <item name="colorPrimaryDark">#0000ff</item> // 状态栏颜色
        <item name="colorAccent">#0000ff</item>
    </style>
```

第二种方式：

```
    <style name="StatusBarActivityTheme" parent="Theme.AppCompat.Light">
        <item name="android:statusBarColor">#00ff00</item>
    </style>
```

第三种方式：

```
getWindow().setStatusBarColor(0xffff0000);
```

### 设置状态栏图标颜色

设置黑色图标：

```
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        getWindow().getDecorView().getWindowInsetsController()
                .setSystemBarsAppearance(WindowInsetsController.APPEARANCE_LIGHT_STATUS_BARS,
                        WindowInsetsController.APPEARANCE_LIGHT_STATUS_BARS);
        }
```

设置白色图标：

```
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            getWindow().getDecorView().getWindowInsetsController()
                    .setSystemBarsAppearance(0,
                            WindowInsetsController.APPEARANCE_LIGHT_STATUS_BARS);
        }
```

## 设置导航栏

### 设置导航栏颜色

```
getWindow().setNavigationBarColor(0xff000000);
```

### 设置导航栏图标颜色

设置为黑色图标：

```
mWindowInsetsController.setSystemBarsAppearance(APPEARANCE_LIGHT_NAVIGATION_BARS, APPEARANCE_LIGHT_NAVIGATION_BARS);
```

设置为白色图标：

```
mWindowInsetsController.setSystemBarsAppearance(0, APPEARANCE_LIGHT_NAVIGATION_BARS);
```

### 获取导航栏高度

可通过 WindowInsets 进行计算，方式如下：
1、监听 DecorView 的 WindowInsets

```
getWindow().getDecorView().setOnApplyWindowInsetsListener(new View.OnApplyWindowInsetsListener() {
    @Override
    public WindowInsets onApplyWindowInsets(View v, WindowInsets insets) {
        //可以在此监听状态栏、导航栏显示变化，作延迟隐藏/ui变化处理
        mWindowInsets = insets;
        return insets;
    }
});

```

2、拿到导航栏的 Inset 计算

```
Insets insets = mWindowInsets.getInsets(WindowInsets.Type.navigationBars());
int height = insets.bottom - insets.top;
return height;
```

## 去掉ActionBar

```
    <style name="StatusBarActivityTheme" parent="Theme.AppCompat.Light">
        <item name="windowActionBar">false</item>
        <item name="windowNoTitle">true</item>
    </style>
```

或者：

```
    <style name="StatusBarActivityTheme" parent="Theme.AppCompat.Light.NoActionBar">
    </style>
```

## 隐藏和显示状态栏和状态栏

下面来介绍如何隐藏和显示状态栏和状态栏来实现全屏。

隐藏：

```
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            // 隐藏状态栏
            //getWindow().getDecorView().getWindowInsetsController().hide(WindowInsets.Type.statusBars());
            // 隐藏导航栏
            //getWindow().getDecorView().getWindowInsetsController().hide(WindowInsets.Type.navigationBars());
            // 隐藏状态栏和导航栏
            getWindow().getDecorView().getWindowInsetsController().hide(WindowInsets.Type.systemBars());
        }else{
            // 全屏显示，隐藏状态栏和导航栏，拉出状态栏和导航栏显示一会儿后消失。
            getWindow().getDecorView().setSystemUiVisibility(
                    View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                            | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                            | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                            | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                            | View.SYSTEM_UI_FLAG_FULLSCREEN
                            | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY);
        }
```

Android系统API提供了三种全屏显示模式：Lean Back, Immersive, Immersive Sticky。 这三种全屏模式都可以隐藏系统栏，区别就是系统栏显示方式的不同。

 - Lean Back模式应用在不需要用户频繁的点击屏幕的情形下，比如视频播放APP。该模式下，用户只需要单击屏幕，System bar将会自动显示处理，无需做更多的操作。使能这种模式，只需要调用 setSystemUiVisibility() 传递两个参数 SYSTEM_UI_FLAG_FULLSCREEN 和 SYSTEM_UI_FLAG_HIDE_NAVIGATION。
 - Immersive模式应用在用户需要频繁与屏幕交互的情形下，比如游戏，图片浏览，阅读等等。该模式下，用户需要在屏幕边缘向内滑动才能呼出system bar。使能这种模式，需要调用setSystemUiVisibility() 传递 SYSTEM_UI_FLAG_IMMERSIVE ， SYSTEM_UI_FLAG_FULLSCREEN 和 SYSTEM_UI_FLAG_HIDE_NAVIGATION。
 - Sticky immersive模式与Immersive模式的区别是：System bar 呼出时，System bar是半透明显示的， 同时System bar会自动隐藏。这种模式的好处是如果用户误操作导致system bar显示出来后，无需做什么其它操作，system bar会自动隐藏，在有些引用场景下，这是一个比较好的用户体验。使能这种模式，需要调用 setSystemUiVisibility() 并将 SYSTEM_UI_FLAG_IMMERSIVE_STICKY 标志与 SYSTEM_UI_FLAG_FULLSCREEN 和 SYSTEM_UI_FLAG_HIDE_NAVIGATION 一起传递。

 - View.SYSTEM_UI_FLAG_LAYOUT_STABLE：全屏显示时保证尺寸不变。
 - View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN：Activity全屏显示，状态栏显示在Activity页面上面。
 - View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION：效果同View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
 - View.SYSTEM_UI_FLAG_HIDE_NAVIGATION：隐藏导航栏
 - View.SYSTEM_UI_FLAG_FULLSCREEN：Activity全屏显示，且状态栏被隐藏覆盖掉。
 - View.SYSTEM_UI_FLAG_VISIBLE：Activity非全屏显示，显示状态栏和导航栏。
 - View.INVISIBLE：Activity伸展全屏显示，隐藏状态栏。
 - View.SYSTEM_UI_LAYOUT_FLAGS：效果同View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
 - View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY：必须配合View.SYSTEM_UI_FLAG_FULLSCREEN和View.SYSTEM_UI_FLAG_HIDE_NAVIGATION组合使用，达到的效果是拉出状态栏和导航栏后显示一会儿消失。

显示：

```
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            // 显示状态栏
            //getWindow().getDecorView().getWindowInsetsController().show(WindowInsets.Type.statusBars());
            // 显示导航栏
            //getWindow().getDecorView().getWindowInsetsController().show(WindowInsets.Type.navigationBars());
            // 显示状态栏和导航栏
            getWindow().getDecorView().getWindowInsetsController().show(WindowInsets.Type.systemBars());
        }else{
            getWindow().getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_VISIBLE);
        }
```

隐藏状态栏还可以用下面方法实现：

```
getWindow().addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
//或者
getWindow().setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN, WindowManager.LayoutParams.FLAG_FULLSCREEN);
```

或者在主题中配置全屏属性：

```
<item name="android:windowFullscreen">true</item>
```

屏幕边沿下滑后可以显示状态栏，随后自动消失。

显示状态栏：

```
// 显示状态栏
getWindow().clearFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
```

如果发现导航栏隐藏后，原来的导航栏区域无法显示内容，可看看是不是设置了布局或者代码设置了 `android:fitsSystemWindows` 为 true 导致的。

## 定制状态栏及导航栏的显示/隐藏动画

WindowInsetsController 提供了定制显示隐藏系统栏的动画的方法：

```
void controlWindowInsetsAnimation(@InsetsType int types, long durationMillis,
        @Nullable Interpolator interpolator,
        @Nullable CancellationSignal cancellationSignal,
        @NonNull WindowInsetsAnimationControlListener listener);
```

## fitsSystemWindows

该属性让view可以根据系统窗口(如status bar)来调整自己的布局，如果值为true,就会调整view的paingding属性来给system windows留出空间。
fitsSystemWindow 默认是true，表示页面布局(内容区)不会扩展到状态栏，会针对透明的状态栏会自动添加一个值等于状态栏高度的paddingTop；针对透明的系统导航栏会自动添加一个值等于导航栏高度的paddingBottom，当fitsSystemWindows = false时，表示页面布局(内容区)扩展到状态栏。
如果出现statusbar或者navigatinobar把界面盖住的情况，一般都是fits设置的有问题。