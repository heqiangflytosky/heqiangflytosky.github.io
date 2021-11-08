---
title: Android -- 字体设置
categories: Android
comments: true
tags: [字体]
description: 介绍 Android 字体设置
date: 2019-5-6 10:00:00
---

## 概述

Android 开发中字体的设置是很常见的问题，本文就对开发中的常见的关于字体的设置做一些说明。

## 修改字体

### 修改组件的属性

 - android:typeface：设置字体类型，该属性只能设置Android默认的几种字体类型，比如：`normal|sans|serif|monospace`，设置其他类型会出错。
 - android:fontFamily：设置字体类型，可以看做是 `android:typeface` 属性的加强，可选项更多，另外，该属性的设置在编译器并不会验证属性值的正确与否，如果设置不正确，将会使用默认字体。使用该属性可以添加系统中添加的一些自定义字体等。
 - android:textStyle：设置字体风格，可选项只有 `normal|bold|italic`，对应 `普通|粗体|斜体`。

`android:typeface` 和 `android:fontFamily` 都是设置字体类型，如果同时设置，将会优先使用 `android:fontFamily` 值。
`android:typeface` 和 `android:fontFamily` 都可以和 `android:textStyle` 搭配使用。

### 利用主题修改全局字体

我们可以通过修改 Application 或者 Activity 的 `android:theme` 配置类设置字体：

```
    <style name="TextTheme" parent="AppTheme.NoActionBar">
        <item name="android:typeface">serif</item>
        <item name="android:fontFamily">SFDIN-medium</item>
        <item name="android:textStyle">bold</item>
    </style>
```

使用方法和在组件中配置是一样的，`android:typeface` 和 `android:fontFamily` 同时使用时 `android:fontFamily` 优先级更高，可以和 `android:textStyle` 搭配使用。
全局级的配置和组件级的配置同时有的话，组件级的优先级更高。

### 使用自定义字体

我们可以在应用级别使用自定义字体，这时就要使用 Typeface 类了。
Typeface 提供了下面的构造方法：

 - Typeface.create(String familyName, int style)
 - Typeface.create(Typeface family, int style)
 - Typeface.create(Typeface family, int weight, boolean italic)
 - Typeface.defaultFromStyle(int style)
 - Typeface.createFromAsset(AssetManager mgr, String path)
 - Typeface.createFromFile(File file)
 - Typeface.createFromFile(String path)

style 参数指代：Typeface.NORMAL、Typeface.BOLD、Typeface.ITALIC、Typeface.BOLD_ITALIC，分别表示常规样式、粗体、斜体、粗斜体。

```
        try {
            File file = new File("/system/fonts/Test-Bold.otf");
            Typeface typeface = Typeface.createFromFile(file);
            textView1.setTypeface(typeface,Typeface.NORMAL);
        } catch (Exception e) {
            e.printStackTrace();
        }
```

这种创建 Typeface 的方法要记得加 try catch 否则如果字体库不存在会崩溃。

```
        Typeface fontMedium = Typeface.create("SFPRO-bold", Typeface.NORMAL);
        textView1.setTypeface(fontMedium);
```

这种情况如果字体库不存在，会使用系统默认的字体。

## 字体库的位置

Android 字体库文件在 Andoird 源码中放在 `framework/base/data/fonts/` 下面，设备中在 `/system/fonts` 下。

## 关于字体sp dp px 

px 设置后固定，不会根据显示大小和字体大小变动
dp 跟随显示大小变动，根据density， 手机设置里面的设置显示大小修改了 mMetrics.density
sp 跟随显示大小和字体大小变动，根据 scaledDensity， 手机设置里面的设置字体大小修改了 fontScale

```
                mMetrics.scaledDensity = mMetrics.density *
                        (mConfiguration.fontScale != 0 ? mConfiguration.fontScale : 1.0f);
```

## 相关推荐

https://www.jianshu.com/p/36059c6bf958
https://blog.csdn.net/a282255307/article/details/76870441