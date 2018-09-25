---
title: Android 图形系统 -- 硬件加速渲染
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 LockCanvas() 方法获取的 Canvas 无法使用硬件加速的原因。
date: 2016-9-5 10:00:00
---

我们在开发过程中经常会遇到硬件加速这个概念，

## 硬件加速的设置

Android 3.0 (API level 11), 开始支持
所有的 `View` 的 canvas 都会使用GPU，但是硬件的加速会占有一定的RAM。
在API >= 14上，默认是开启的，如果你的应用只是标准的 `View` 和 `Drawable`，全局都打开硬件加速，是不会有任何问题的。
然而，硬件加速并不支持所有的2D画图的操作，这时开着它，可能会影响到你的自定义控件或者绘画，出现异常等行为，
所以android对于硬件加速提供了可选性
硬件加速的级别
Application

```
<application 
    android:hardwareAccelerated="false" 
...>
</application>
```
Activity

```
<application 
    android:hardwareAccelerated="true">
    <activity ... />
    <activity android:hardwareAccelerated="false" />
</application>
```
Window

```
getWindow().setFlags(
   WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED,
   WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED);
```

View

```
myView.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
```

application 和 activity 都可以单独控制硬件加速的打开和关闭。
window级别无法关闭硬件加速，可以开启硬件加速。
view级别只能关闭硬件加速，无法开启硬件加速。
通过window开启硬件加速后，`onDraw` 的  `canvas` 是 `DisplayListCanvas` 实例。
另外只要通过 `View` 的 `setLayerType` 设置了关闭或者开启（虽然不能成功），`onDraw` 的  `canvas` 都变成了 `Canvas` 实例。也就是说 通过 `View` 的 `setLayerType` 可以把 `View` 中的画布换成  `Canvas`，具体是不是硬件加速是由 `Canvas` 类来决定。这部分的源码分析我们后面再介绍。
可以在 application 中关闭，activity 中打开；
可以在 application 中打开，activity 中关闭；

## 硬件加速的限制

目前，Android对硬件加速的支持并非完美，有些绘制操作在开启硬件加速的情况下不能正常工作。
[硬件加速不支持的操作列表](https://developer.android.com/guide/topics/graphics/hardware-accel#unsupported)

## 硬件加速之Canvas

在实际开发中或许你有下面的经历：
尽管已经设置了硬件加速，通过 `TextureView.lockCanvas()` 或者通过 `SurfaceView.getHolder().lockCanvas()` 得到的 `Canvas` 通过打印 `Canvas.isHardwareAccelerated()` 会返回 false。而 `TextureView.isHardwareAccelerated()` 或者 `SurfaceView.isHardwareAccelerated()` 是返回true的。
这是正确的，通过 `lockCanvas()` 得到的 `Canvas` 只能用软件绘制的。
如果想通过硬件渲染，只能通过 `lockHardwareCanvas` 或者调用OpenGL接口实现。
具体可以参考：systemui/ImageWallpaper.java，里面提供了用 `OpenGL` 和 `Canvas` 来绘制壁纸的两种方法。
先来看一下 `View` 的情况：

```
public class CustomView extends TextView {
    ......

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Bitmap bitmap1 = BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher);
        canvas.drawBitmap(bitmap1,0,0,new Paint());
    }
}
```

如果开启了硬件加速，在 `onDraw` 这里打断点，你会发现 `canvas` 其实是 `DisplayListCanvas` 实例。
通过调试 `canvas.isHardwareAccelerated();` 的执行结果是返回 true。

如果通过 application 或者 activity 关闭了硬件加速，`onDraw` 的  `canvas` 其实是 `Surface.CompatibleCanvas` 实例。
如果通过 view 关闭了硬件加速，`onDraw` 的  `canvas` 其实是 `Canvas` 实例。

使用多种方式(`View/SurfaceView/TextureView`)实现高效绘制
能实现使用GPU绘制的只有 `View`，因为它使用 `SurfaceView/TextureView` 实现时并没有使用到 OpenGL 的 API，`canvas.drawText(...)` 的 `Canvas` 是使用 CPU 的，所以如果在CPU资源缺少的情况下效率并不高。
总结可得，使用 `Canvas` 要注意是否有硬件加速，一般的 View 是有的，而 SurfaceView 和 TextureView 通过 `lockCanvas()` 得到的 `Canvas` 是没有的。
