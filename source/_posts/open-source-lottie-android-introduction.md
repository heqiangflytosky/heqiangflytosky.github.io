---
title: 开源项目- Lottie 简介
categories: Android开源项目
comments: true
tags: [Android, Lottie]
description: 介绍lottie-android的基本用法
date: 2017-02-07 10:00:00
---
Airbnb在GitHub上面开源了一个项目lottie-android，最近火的不要不要的，牢牢占据Trending排行榜（日、周、月）首位，下面我们就见识一下这个项目。
首先放上Lottie在GitHub上面的项目地址：[Android](https://github.com/airbnb/lottie-android)，[iOS](https://github.com/airbnb/lottie-ios), 和[React Native](https://github.com/airbnb/lottie-react-native)。
<!-- more -->
## Lottie简介
Lottie是一个为Android和IOS设备提供的一个开源框架，它能够解析通过[Adobe After Effects](http://www.adobe.com/products/aftereffects.html) 软件做出来的动画，动画文件通过[Bodymovin](https://github.com/bodymovin/bodymovin)导出json文件，就可以通过Lottie中的LottieAnimationView来使用了。
[Bodymovin](https://github.com/bodymovin/bodymovin)是一个After Effects的插件，它由Hernan Torrisi开发。

我们先看看官方给出的实现的动画效果：

<img src="https://raw.githubusercontent.com/airbnb/lottie-android/master/gifs/Example1.gif"/>

<img src="https://raw.githubusercontent.com/airbnb/lottie-android/master/gifs/Example2.gif"/>

<img src="https://raw.githubusercontent.com/airbnb/lottie-android/master/gifs/Example3.gif"/>

<img src="https://raw.githubusercontent.com/airbnb/lottie-android/master/gifs/Community 2_3.gif"/>

<img src="https://raw.githubusercontent.com/airbnb/lottie-android/master/gifs/Example4.gif"/>

这些动画如果让你实现起来，你可能会觉得很麻烦，但是通过Lottie这一切就变得很容易。
想了解更多请参考[官方介绍](http://airbnb.design/introducing-lottie/)

## 使用方法
首先由视觉设计师通过Adobe After Effects做好这些动画，这个比我们用代码来实现会容易的很多，然后Bodymovin导出json文件，这些json文件描述了该动画的一些关键点的坐标以及运动轨迹，然后把json文件放到项目的app/src/main/assets目录下，代码中在build.gradle中添加依赖：
```
dependencies {
  compile 'com.airbnb.android:lottie:1.0.1'
}
```
在布局文件上加上：
```xml
<com.airbnb.lottie.LottieAnimationView
        android:id="@+id/animation_view"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:lottie_fileName="hello-world.json"
        app:lottie_loop="true"
```
或者代码中实现：
```java
LottieAnimationView animationView = (LottieAnimationView) findViewById(R.id.animation_view);
animationView.setAnimation("hello-world.json");
animationView.loop(true);
```
此方法将加载文件并在后台解析动画，并在完成后异步开始呈现动画。
Lottie只支持Jellybean (API 16)或以上版本。
通过源码我们可以发现`LottieAnimationView`是继承自`AppCompatImageView`，我们可以像使用其他`View`一样来使用它。
```
LottieAnimationView animationView = (LottieAnimationView) findViewById(R.id.animation_view);
```
甚至可以从网络上下载json数据：
```
 LottieComposition composition = LottieComposition.fromJson(getResources(), jsonObject, (composition) -> {
     animationView.setComposition(composition);
     animationView.playAnimation();
 });
```
或者使用
```
setAnimation(JSONObject);
```

我们还可以控制动画或者添加监听器：
```
animationView.addAnimatorUpdateListener((animation) -> {
    // Do something.
});
animationView.playAnimation();
...
if (animationView.isAnimating()) {
    // Do something.
}
...
animationView.setProgress(0.5f);
...
// Custom animation speed or duration.
ValueAnimator animator = ValueAnimator.ofFloat(0f, 1f)
    .setDuration(500);
animator.addUpdateListener(animation -> {
    animationView.setProgress(animation.getAnimatedValue());
});
animator.start();
...
animationView.cancelAnimation();
```
`LottieAnimationView`是使用`LottieDrawable`来渲染动画的，如果有必要，还可以直接使用`LottieDrawable`：
```
LottieDrawable drawable = new LottieDrawable();
LottieComposition.fromAssetFileName(getContext(), "hello-world.json", (composition) -> {
    drawable.setComposition(composition);
});
```
如果动画会被频繁的复用，`LottieAnimationView`有一套缓存策略，可以使用
```
LottieAnimationView#setAnimation(String, CacheStrategy)
```
来实现它，`CacheStrategy`可以是`Strong`，`Weak`或者是`None`，这样`LottieAnimationView`就可以持有一个已经加载和解析动画的强引用或者弱引用。

先简单介绍到这里，如需要了解更多请看[Lottie的源码分析](http://www.heqiangfly.com/2017/02/07/open-source-lottie-android-source-code-analysis/)。

