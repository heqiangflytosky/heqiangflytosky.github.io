---
title: Android 图形系统 -- SurfaceView 使用
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 SurfaceView 的使用
date: 2016-9-8 10:00:00
---

## SurfaceView 简介

### 什么是 SurfaceView

`SurfaceView` 是 Android 中一种比较特殊的 `View`，它跟平时时候的 `TextView`、`Button` 等 最大的区别是它跟它的视图容器并不是在同一个视图层上。
`SurfaceView` 的工作方式是创建一个置于应用窗口之后的新窗口。在屏幕显示的视图层中嵌入了一块用做图像绘制的独立的 `Surface` 视图，它不与宿主窗口共享同一个绘图表面。相当于在屏幕上挖了个洞来显示它所绘制的图像。
`SurfaceView` 窗口刷新的时候不需要重绘应用程序的窗口。另外，`SurfaceView` 的绘制也可以在一个独立的线程中完成，所以对 `SurfaceView` 的绘制并不会影响到主线程的运行。因此可以实现复杂而高效的UI。

### 为什么要使用 SurfaceView

SurfaceView 一般用来实现动态的或者比较复杂的图像还有动画的显示。

### 相关的几个类

 - SurfaceHolder：顾名思义，就是 `Surface` 的持有者，通过它的对象我们可以操作 `Surface`。`SurfaceView.getHolder()`方法可以获得 `SurfaceView` 的 `SurfaceHolder` 对象。
 - SurfaceHolder.Callback：实现 `Surface` 生命周期的回调方法。

## SurfaceView 的使用

```
public class MySurfaceView extends SurfaceView implements SurfaceHolder.Callback{

    public MySurfaceView(Context context) {
        super(context);
    }

    public MySurfaceView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public MySurfaceView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    private void init() {
        SurfaceHolder holder = getHolder();
        //设置Surface生命周期回调
        holder.addCallback(this);
    }

    @Override
    public void surfaceCreated(final SurfaceHolder surfaceHolder) {
        Canvas canvas = surfaceHolder.lockCanvas();
        // dosomething
        surfaceHolder.unlockCanvasAndPost(canvas);
    }

    @Override
    public void surfaceChanged(SurfaceHolder surfaceHolder, int i, int i1, int i2) {

    }

    @Override
    public void surfaceDestroyed(SurfaceHolder surfaceHolder) {

    }
}
```

上面的自定义 `MySurfaceView` 继承自 `SurfaceView`，并实现 `SurfaceHolder.Callback` 接口。
对于实现 `SurfaceHolder.Callback` 接口，其实就是实现了它的几个关于 `Surface` 的生命周期的方法：

 - surfaceCreated()
 - surfaceChanged()
 - surfaceDestroyed()

SurfaceView的SurfaceHolder提供了两个lockCanvas方法：

 - lockCanvas()：锁住整张画布，绘画完成后也更新整张画布的内容到屏幕上
 - lockCanvas(Rect inOutDirty)：锁住画布中的某个区域，绘画完成后也只更新这个区域的内容到屏幕。只更新必要的画面内容以节省时间，提高程序运行的效率，适用于大动态画面的场景。

通过这两个方法来获取当前的 `Canvas` 绘图对象。接下来就可以在画布上进行绘制操作了，就和普通的 `View` 上面绘制没有什么两样了。

## SurfaceView 特点

### 多线程绘图

SurfaceView 的绘制并不会影响到主线程的运行，因此可以实现复杂而高效的UI。

```
    @Override
    public void surfaceCreated(final SurfaceHolder surfaceHolder) {
        // 先绘制蓝色背景色
        final Canvas canvas = surfaceHolder.lockCanvas();
        canvas.drawColor(Color.BLUE);
        surfaceHolder.unlockCanvasAndPost(canvas);
        // 启动一个后台线程绘制两个颜色块
        new Thread(new Runnable() {
            @Override
            public void run() {
                Canvas canvas = surfaceHolder.lockCanvas(new Rect(200,200,400,400));
                Paint paint1 = new Paint();
                paint1.setColor(Color.GREEN);
                canvas.drawRect(new Rect(200,200,400,400), paint1);
                surfaceHolder.unlockCanvasAndPost(canvas);

                canvas = surfaceHolder.lockCanvas(new Rect(20,20,200,200));
                Paint paint = new Paint();
                paint.setColor(Color.RED);
                canvas.drawRect(new Rect(20,20,200,200), paint);
                surfaceHolder.unlockCanvasAndPost(canvas);
            }
        }).start();
    }
```

### 双缓冲

其实这里叫双缓冲也不太合适，应该叫多缓冲，因为各个手机厂商的定制Rom设置的缓冲数量并不一样。
什么是双缓冲呢？我们先通过一个例子来了解一下：
我们先通过下面代码画一个方块A：

```
    @Override
    public void surfaceCreated(final SurfaceHolder surfaceHolder) {
        // 方块A
        Canvas canvas = surfaceHolder.lockCanvas();
        Paint paint = new Paint();
        paint.setColor(Color.RED);
        canvas.drawRect(new Rect(0,0,200,200), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);
    }
```

通过 `lockCanvas()` 获取整个画布，得到下面的结果：

<img src="/images/android-graphic-system-surfaceview/buffer_1.png" width="270" height="540"/>

再通过下面的代码画第二个方块B：

```
    @Override
    public void surfaceCreated(final SurfaceHolder surfaceHolder) {
        // 方块A
        Canvas canvas = surfaceHolder.lockCanvas();
        Paint paint = new Paint();
        paint.setColor(Color.RED);
        canvas.drawRect(new Rect(0,0,200,200), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);

        // 方块B
        canvas = surfaceHolder.lockCanvas();
        paint.setColor(Color.GREEN);
        canvas.drawRect(new Rect(200,0,400,200), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);
    }
```

得到下面的结果：

<img src="/images/android-graphic-system-surfaceview/buffer_2.png" width="270" height="540"/>

这里只出现了方块B，那么以前画的方块A，去哪里了，前面不是说 `lockCanvas()` 得到的画布是会保留以前的绘制内容的吗？
我们接着画方块，知道画第四个方块D的时候，方块A终于出现在画布上了：

```
    @Override
    public void surfaceCreated(final SurfaceHolder surfaceHolder) {
        // 方块A
        Canvas canvas = surfaceHolder.lockCanvas();
        Paint paint = new Paint();
        paint.setColor(Color.RED);
        canvas.drawRect(new Rect(0,0,200,200), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);

        // 方块B
        canvas = surfaceHolder.lockCanvas();
        paint.setColor(Color.GREEN);
        canvas.drawRect(new Rect(200,0,400,200), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);

        // 方块C
        canvas = surfaceHolder.lockCanvas();
        paint.setColor(Color.CYAN);
        canvas.drawRect(new Rect(0,200,200,400), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);

        // 方块D
        canvas = surfaceHolder.lockCanvas();
        paint.setColor(Color.GRAY);
        canvas.drawRect(new Rect(200,200,400,400), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);
    }
```

<img src="/images/android-graphic-system-surfaceview/buffer_4.png" width="270" height="540"/>

这是为什么呢？`Surface` 在更新画面的时候使用了缓冲机制，也就是在画布和屏幕之间建立了多个跟屏幕一样大小的内存区域，即缓冲区。
在 Surface 刚建立起来的时候，它的两个缓冲区内存中是没有任何图像的，即都是全黑。

 1.在画方块A前，我使用 `lockCanvas()` 接口，于是 Surface 锁住“缓冲区0”的全部区域，往画布上画方块A，被送到“缓冲区0”中。画完这1帧，使用  `unlockCanvasAndPost()` 接口，则会将“缓冲区0”中所有的图像都送到屏幕中进行显示，画面显示正常。
 2.在画方块B前，我使用 `lockCanvas()` 接口，于是 Surface 锁住“缓冲区1”的全部区域，往画布上画方块B，被送到“缓冲区1”中。画完这1帧，使用  `unlockCanvasAndPost()` 接口，则会将“缓冲区1”中所有的图像都送到屏幕中进行显示，画面只显示方块B。
 3....
 4.在画方块D前，我使用 `lockCanvas()` 接口，于是 Surface 锁住“缓冲区0”的全部区域，往画布上画方块D，和原先的方块A一起都被送到“缓冲区0”中。画完这1帧，使用  `unlockCanvasAndPost()` 接口，则会将“缓冲区0”中所有的图像都送到屏幕中进行显示，画面显示方块A和D。
 因此，我们在平时使用的时候就要注意，要避免上面的情况，就要保证每一帧绘制前，要把上一帧的内容都再绘制一遍。
下面来看一下下面代码的绘制效果：

```
    @Override
    public void surfaceCreated(final SurfaceHolder surfaceHolder) {

        // 方块A
        Canvas canvas = surfaceHolder.lockCanvas();
        if (mBitmapCache == null) {
            mBitmapCache = Bitmap.createBitmap(canvas.getWidth(), canvas.getHeight(), Bitmap.Config.ARGB_8888);
        }
        Canvas canvasA = new Canvas(mBitmapCache);
        Paint paint = new Paint();
        paint.setColor(Color.RED);
        canvasA.drawRect(new Rect(0,0,200,200), paint);
        // 清除画布
        canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);
        canvas.drawBitmap(mBitmapCache, 0, 0, paint);
        surfaceHolder.unlockCanvasAndPost(canvas);

        // 方块B
        canvas = surfaceHolder.lockCanvas();
        Canvas canvasB = new Canvas(mBitmapCache);
        paint.setColor(Color.GREEN);
        canvasB.drawRect(new Rect(200,0,400,200), paint);
        // 清除画布
        canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);
        canvas.drawBitmap(mBitmapCache, 0, 0, paint);
        surfaceHolder.unlockCanvasAndPost(canvas);

        // 方块C
        canvas = surfaceHolder.lockCanvas();
        Canvas canvasC = new Canvas(mBitmapCache);
        paint.setColor(Color.CYAN);
        canvasC.drawRect(new Rect(0,200,200,400), paint);
        // 清除画布
        canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);
        canvas.drawBitmap(mBitmapCache, 0, 0, paint);
        surfaceHolder.unlockCanvasAndPost(canvas);

        // 方块D
        canvas = surfaceHolder.lockCanvas();
        Canvas canvasD = new Canvas(mBitmapCache);
        paint.setColor(Color.GRAY);
        canvasD.drawRect(new Rect(200,200,400,400), paint);
        // 清除画布
        canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);
        canvas.drawBitmap(mBitmapCache, 0, 0, paint);
        surfaceHolder.unlockCanvasAndPost(canvas);
    }
```

这里使用了 `Bitmap` 作为缓存来保存上一帧的数据，每一帧的数据都是在上一帧的基础上来画的。

接下来我们换一种绘制方式：

```
    @Override
    public void surfaceCreated(final SurfaceHolder surfaceHolder) {
        // 方块A
        Canvas canvas = surfaceHolder.lockCanvas(new Rect(0,0,200,200));
        Paint paint = new Paint();
        paint.setColor(Color.RED);
        canvas.drawRect(new Rect(0,0,200,200), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);

        // 方块B
        canvas = surfaceHolder.lockCanvas(new Rect(200,0,400,200));
        paint.setColor(Color.GREEN);
        canvas.drawRect(new Rect(200,0,400,200), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);

        // 方块C
        canvas = surfaceHolder.lockCanvas(new Rect(0,200,200,400));
        paint.setColor(Color.CYAN);
        canvas.drawRect(new Rect(0,200,200,400), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);

        // 方块D
        canvas = surfaceHolder.lockCanvas(new Rect(200,200,400,400));
        paint.setColor(Color.GRAY);
        canvas.drawRect(new Rect(200,200,400,400), paint);
        surfaceHolder.unlockCanvasAndPost(canvas);
    }
```

<img src="/images/android-graphic-system-surfaceview/buffer_5.png" width="270" height="540"/>

这下四个方块都显示了，这是因为 `lockCanvas(Rect dirty)` 锁住画布中的某个区域，绘画完成后也只更新当前缓冲区的这个区域的内容到屏幕。那么这个区域以外的部分都会保留在屏幕上面。

### 硬件加速

我们在 `surfaceCreated` 方法中加断点调试 `canvas.isHardwareAccelerated();` 发现返回 false，而且 canvas 对象为 `Surface$CompatibleCanvas` 对象。
`CompatibleCanvas` 类是 `Surface` 的一个内部类，继承自 `Canvas` ，包含一个矩阵对象 `Matrix`，覆盖了 `setMatrix` 和 `getMatrix` 方法。我们知道 `Canvas` 是不起用硬件加速的，底层使用 Skai 来进行绘制，因此 `CompatibleCanvas` 也是没有使用硬件加速的。这就是为什么通过 `SurfaceView.getHolder().lockCanvas()` 得到的 `Canvas` 是软绘制的原因。
那这样来说是不是我们就无法使用硬件加速进行GPU渲染了呢？答案是否定的，我们可以通过下面两种方式使用GPU渲染：

 - 自已创建 OpenGL 上下文，接入3D引擎。具体可以参考：`systemui/ImageWallpaper.java`，里面提供了用 OpenGL 和 Canvas 来绘制壁纸的两种方法。
 - 使用 `GLSurfaceView`，这个后面博客会介绍。
 - Android 6.0 版本的 `Surface` 类提供了一个 `lockHardwareCanvas` 方法，用此方法可以得到硬件加速的 `Canvas`。

下面来看一段使用 `lockHardwareCanvas` 进行GPU渲染的方法：

```
    @Override
    public void surfaceCreated(final SurfaceHolder surfaceHolder) {
        Canvas canvas = surfaceHolder.getSurface().lockHardwareCanvas();
        Paint paint = new Paint();
        paint.setColor(Color.RED);
        canvas.drawRect(new Rect(0,0,200,200), paint);
        surfaceHolder.getSurface().unlockCanvasAndPost(canvas);
    }
```

通过调试发现 `canvas` 对象为 `DisplayListCanvas` 对象，它是使用GPU渲染的，后面会进行详细介绍。


## 和普通View的差异

### 无法做旋转等动画

SurfaceView 提供一个直接的绘图表面（Surface）嵌入到视图结构层次中。你可以控制这个Surface的格式，大小，SurfaceView负责在屏幕上正确的摆放Surface。简单说就是SurfaceView拥有自己的Surface，它与宿主窗口是分离的。
SurfaceView 在 7.0 以前版本是不支持平移，缩放，旋转等动画，在7.0 以后版本可以支持平移，缩放的动画操作。但无法进行旋转操作。
如图是将 Surface 旋转30°后的效果，发现绘制内容并没有跟随旋转。

<img src="/images/android-graphic-system-surfaceview/rotate_1.jpg" width="270" height="540"/>
<img src="/images/android-graphic-system-surfaceview/rotate_2.jpg" width="270" height="540"/>

## 视频播放器

Android 提供的组件 `VideoView` 是使用 `SurfaceView` 的。感兴趣的可以参考源码。
另外，请参考我的github的Demo：[FloatWindowPlayer](https://github.com/heqiangflytosky/FloatWindowPlayer) 中的 SurfaceView 部分，实现了悬浮窗播放器。

## 和 TextureView 对比

TextureView 的特点是支持旋转等动画，但是它必须在硬件加速的窗口中使用，占用内存比SurfaceView高，在5.0以前在主线程渲染，5.0以后有单独的渲染线程。

|  | SurfaceView | TextureView |
|:-------------:|:-------------:|:-------------:|
| 内存 | 低 | 高 |
| 绘制 | 及时 | 1-3帧的延迟 |
| 耗电 | 低 | 高 |
| 动画和截图 | 不支持 | 支持 |

综合以上对比，那么我们的做视频播放器时应该如何选择呢？
从性能和安全性角度出发，使用播放器优先选SurfaceView。
由于 SurfaceView 有自己独立的 Window，因此 SurfaceView 也不能放到 ListView 或者 ScrollView中，因此，在列表中播放视频就无法实现了，只能选择 TextureView。

## 推荐文章

小窗播放视频的原理和实现（上）：https://cloud.tencent.com/developer/article/1034235
小窗播放视频的原理和实现（下）：https://cloud.tencent.com/developer/article/1047885