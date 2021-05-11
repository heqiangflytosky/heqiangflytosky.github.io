---
title: Android -- Bitmap 内存管理
categories: Android
comments: true
tags: [Bitmap]
description: 分析 Bitmap 内存管理
date: 2017-2-12 10:00:00
---

## 概述

在 Android 中，我们经常和 Bitmap 打交道，处理 Bitmap 也使我们经常遇到一些难缠的问题。因此我们需要储备一些高效使用 Bitmap 的只是来提升我们的开发技能。

在 Bitmap 的使用过程中，我们会遇到下面的问题：

 - 使用 Bitmap 会很容易使我们的 App 内存变得很大，比如，一个 4048x3036 像素的图片，如果我们使用 ARGB_8888 的配置来存储，那么就占用了48M的内存（4048x3036x4 bytes），这么大的内存使用要求将会显著提高我们 APP 的内存占用。
 - 在 UI 线程中加载图片会显著拖累APP的性能，并有可能出现ANR。因此，在使用 Bitmap 时需要注意进行管理我们使用的线程。
 - 当需要再 App 中加载大量图片时，我们需要合理地使用内存缓存和磁盘缓存，否则会影响App的响应速度和流畅性。

现在，已经有很多的开源的图片处理框架，比如Fresco、Glide、Picasso等等，它们已经实现了图片获取、解码，显示和缓存等，推荐大家使用。
关于 Android App 的内存优化，我们就很有必要来关注 Bitmap 内存，因为，它一般是我们内存占用过的大户，管理不当很容易导致内存不足，也是很多 OOM 的元凶之一。
本文就来介绍一下 Bitmap 内存方法的知识。

[Android 官方文档：Handling bitmaps](https://developer.android.com/topic/performance/graphics)
[Android 官方文档：Managing Bitmap Memory](https://developer.android.com/topic/performance/graphics/manage-memory)

## Bitmap 的内存计算

比如我们构造 Bitmap 时设置宽200，高400，色彩模式为 `Bitmap.Config.ARGB_8888`，那么它的像素数据在内存中的大小就为 200x400x4 byte，色彩模式为 `Bitmap.Config.RGB_565`，那么大小就是200x400x2 byte，这些在内存中是固定大小的。

## 关于图片压缩

比如我们用 `bitmap.compress(Bitmap.CompressFormat.PNG,100,fos);` 转化为字节流以后发现获取的 `fos.toByteArray()` 变小了，这是因为进行了压缩的缘故，相同宽高，不同的 bitmap 对象，压缩以后大小是不一样的，因为不同的 bitmap 色彩丰富程度不一样，表达的信息不一样，最终能压缩的大小也不一样。
`bitmap.compress` 压缩是质量压缩，是因为它不会减少图片的像素。图片压缩可以分为尺寸压缩和质量压缩，它是在保持像素的前提下改变图片的宽高、位深及透明度等，来达到压缩图片的目的。进过它压缩的图片文件大小会有改变，但是导入成 bitmap 后占得内存是不变的。因为要保持像素不变，所以它就无法无限压缩，到达一个值之后就不会继续变小了。所以我们发现有时候设置 `compress(CompressFormat format, int quality, OutputStream stream)` quality参数不起作用。

## Bitmap 内存管理

关于 Bitmap 内存的管理，Android 各个版本也是不同的，现在我们来介绍一下这个演变过程：

 - 在Android 2.2 (API level 8)以及之前，当垃圾回收发生时，应用的线程是会被暂停的，这会导致一个延迟滞后，并降低系统效率。 从Android 2.3开始，添加了并发垃圾回收的机制， 这意味着在一个Bitmap不再被引用之后，它所占用的内存会被立即回收。
 - 在Android 2.3.3 (API level 10)以及之前, 一个 Bitmap 的像素级数据（pixel data）是存放在 Native 内存空间中的。 这些数据与 Bitmap 对象本身是隔离的，Bitmap 对象本身被存放在 Dalvik 堆中。我们无法预测在 Native 内存中的像素级数据何时会被释放，这意味着程序容易超过它的内存限制并且崩溃。 自Android 3.0 (API Level 11)开始， 像素级数据则是与 Bitmap 本身一起存放在 Dalvik 堆中。自 Android 8.0 (API level 26) 开始，像素的内存又放到了 Native 内存中。

这里声明一点，Bitmap 对象本身和其他Java对象一样是放在 Java Heap 中存储的。
关于 Bitmap 的存储区域的位置我们现在来在真机上进行一下测试，来验证一下前面的说法。
先来看一下测试代码：

```
    Map<String, Bitmap> mBitmaps = new HashMap<>();
    private void testLoadBitmap() {
        Bitmap bitmap = Bitmap.createBitmap(2*1024,1024,Bitmap.Config.ARGB_8888);
        bitmap.eraseColor(0xffffffff);
        Log.e("Test","图片大小 = "+bitmap.getByteCount());
        mBitmaps.put(String.valueOf(System.currentTimeMillis()),bitmap);
    }
```

代码中创建了一个2048x1024大小的Bitmap，采用 ARGB_8888 存储，那么它的大小为 2048x1024x4 byte = 8M。
我们通过一个按钮，点击一次创建一个 Bitmap 并放到 HashMap 中存储。
这里把创建的 Bitmap 放到 HashMap 里面是保持对 Bitmap 的引用，防止内存回收掉。
我们来通过 Android Studio 自带的 Android Profiler 工具来查看内存。
首先我们在一台 Android 7 的手机上做测试，内存使用情况如下，我们发现，Bitmap 的内存是在 Java 堆上创建的。
当创建的 Bitmap 内存超过 Android 内存限制时，会发生 OOM。
这个内存限制大小可以通过 `adb shell getprop |grep vm` 命令来查看，具体参考 [Android 性能优化工具篇 -- 内存查看和分析工具 ](http://www.heqiangfly.com/2018/10/10/android-performance-optimization-tools-ram-analysis/) 一文。

![效果图](/images/android-knowledge-about-bitmap-memory-management/image1.png)

下面是一台 Android 9 的手机上做的测试，Bitmap 的内存是在 Native 内存空间中创建的。Native heap的增长并不受dalvik vm heapsize的限制，只要RAM有剩余空间，就可以一直在native heap上申请空间，当然如果 RAM快耗尽，memory killer会杀进程释放RAM。

![效果图](/images/android-knowledge-about-bitmap-memory-management/image2.png)

如果我们在创建完 Bitmap 后调用 `mBitmaps.clear();`，会发现 Native 内存马上会回收回到正常水平，这也说明对于 Bitmap 的 Native 内存也是可以自动回收的，不需要我们额外干预。

## Bitmap 内存优化

关于一些 Bitmap 内存优化方面的知识，也可以看一下我的另外一篇博客[Android 性能优化实战篇 -- 内存优化](http://www.heqiangfly.com/2018/10/08/android-performance-optimization-reduce-ram/)，里面有部分内容也是介绍这方面知识的。

### inBitmap

这里我们针对 inBitmap 进行一些测试：
首先我们先创建一个Bitmap数据，然后通过 `BitmapFactory.decodeByteArray` 来解码生成图片。
这里为什么不用 

```
    private byte[] mBitmapData;
    private void testLoadBitmap() {
        Bitmap bitmap = Bitmap.createBitmap(20 * 1024, 1024, Bitmap.Config.ARGB_8888);
        bitmap.eraseColor(0xffffffff);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        bitmap.compress(Bitmap.CompressFormat.PNG, 100, baos);
        mBitmapData = baos.toByteArray();
    }
```

```
    private void testDecodeBitmpa () {
        Log.e("Test","mBitmapData.length = "+mBitmapData.length);

        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inPreferredConfig = Bitmap.Config.ARGB_8888;
        Bitmap bitmap = BitmapFactory.decodeByteArray(mBitmapData,0,mBitmapData.length,options);
        mBitmaps.put(System.currentTimeMillis()+"",bitmap);
    }
```

![效果图](/images/android-knowledge-about-bitmap-memory-management/image3.png)

可以看到，每次解码图片，native 内存都增大80M左右。
那么如果我们设置了 inBitmap 呢？

```
    private Bitmap mInBitmap;
    private void testDecodeBitmpa () {
        Log.e("Test","mBitmapData.length = "+mBitmapData.length);
        if (mInBitmap == null) {
            mInBitmap = Bitmap.createBitmap(20 * 1024, 1024, Bitmap.Config.ARGB_8888);
            mInBitmap.eraseColor(0xffffffff);
        }
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inPreferredConfig = Bitmap.Config.ARGB_8888;
        options.inBitmap = mInBitmap;
        Bitmap bitmap = BitmapFactory.decodeByteArray(mBitmapData,0,mBitmapData.length,options);
        mBitmaps.put(System.currentTimeMillis()+"",bitmap);
    }
```

![效果图](/images/android-knowledge-about-bitmap-memory-management/image4.png)

可以看到，每次解码图片，native 内存没有再增加，说明重用了 mInBitmap 的内存。
这里用 `BitmapFactory.decodeResource` 测试也是 OK 的，效果一样。

```
    private Bitmap mInBitmap;
    private void testDecodeBitmpa () {
        if (mInBitmap == null) {
            mInBitmap = Bitmap.createBitmap(20 * 1024, 1024, Bitmap.Config.ARGB_8888);
            mInBitmap.eraseColor(0xffffffff);
        }
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inPreferredConfig = Bitmap.Config.RGB_565;
        options.inBitmap = mInBitmap;
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(),R.drawable.big,options);
        mBitmaps.put(System.currentTimeMillis()+"",bitmap);
    }
```