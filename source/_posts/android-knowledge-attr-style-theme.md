---
title: Android -- Attr、Style和Theme
categories: Android
comments: true
tags: [Attr、Style和Theme]
description: Android Attr、Style和Theme 介绍
date: 2015-5-10 10:00:00
---

## 概述

Android Attr、Style和Theme 在我们的开发过程中经常打交道，UI 风格的多样化全靠它们来完成，本文对它们做一个简单的介绍。

## 概念说明

 - Attr:属性，风格样式的最小单元，比如TextView 的字体颜色 `android:textColor`。
 - Style:风格，它是一系列Attr的集合用以定义一个 View 的样式，比如height、width、padding等；
 - Theme:主题，它与Style作用一样，不同于Style作用于个一个单独View，而它是作用于Activity上或是整个应用。

## 属性

我们为 View 设置的每一个属性，在系统中都有相对应的定义，包括我们自定义的属性，也需要为其声明一下。
比如，我们设置 View 的宽度，使用 `layout_width`，那么在系统中声明的属性值如下：

```
<declare-styleable name="ViewGroup_Layout">
    <attr name="layout_width" format="dimension">
        <enum name="fill_parent" value="-1" />
        <enum name="match_parent" value="-1" />
        <enum name="wrap_content" value="-2" />
    </attr>
    ...
</declare-styleable>
```

对应了三个枚举值。
关于自定义属性以及属性更加详细的说明，参考我的博客[Android -- 自定义Attribute和Styl](http://www.heqiangfly.com/2016/07/02/android-attr-style-custom/)


## 风格

style 其实就是一个属性的集合，把View的不同的属性结合在一起，就形成了一个统一的风格。
首先在 style.xml 中定义一个 style：

```
    <style name="customText">
        <item name="android:textSize">20sp</item>
        <item name="android:textColor">#FF00FF00</item>
        <item name="android:background">#ffff0000</item>
    </style>
```

然后在布局文件中使用。

## 主题

Theme与Style使用同一个元素标签<style>，区别在于所包含的属性不同，并且使用的地方也不一样。Theme你需要设置到AndroidManifest.xml的<application>或者<activity>标签下，设置后，被设置的Activity或整个应用下所有的View都可以使用该<style>里面的属性了。

## 属性优先级

我们先来做一个测试，

先为 Activity 设置一个布局文件：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".attr.AttrActivity">
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="测试Attr"/>
</LinearLayout>
```

先在 style.xml 文件中为 application 定义一个主题：

```
    <style name="TestAppTheme" parent="Theme.AppCompat">
        <item name="android:textColor">#FF0000FF</item>
        <item name="android:textSize">20sp</item>
    </style>
```

在 AndroidManifest.xml 中设置：

```
    <application
        android:name=".TestApplication"
        android:theme="@style/TestAppTheme"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:persistent="true">
```

打开 Activity，字体颜色和大小和我们配置的相同。

接下来为 Activity 设置主题：

```
    <style name="TestActivityTheme" parent="Theme.AppCompat">
        <item name="android:textColor">#FFFF0000</item>
        <item name="android:textSize">10sp</item>
    </style>
```

在 AndroidManifest.xml 中设置：

```
        <activity android:name=".attr.AttrActivity"
            android:theme="@style/TestActivityTheme"></activity>
```

这时 TextView 的表现和 TestActivityTheme 相同，那么就表示，Activity 主题中的属性会覆盖掉 Application 中的主题中的属性值。
接下来为 TextView 设置 style：

```
    <style name="customText">
        <item name="android:textSize">20sp</item>
        <item name="android:textColor">#FF00FF00</item>
        <item name="android:background">#ffff0000</item>
    </style>
```

```
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        style="@style/customText"
        android:text="测试Attr"/>
```

这时 TextView 的表现就是 customText 设置的值了。
接下来在布局文件中为 TextView 设置属性。

```
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textColor="#ffff00ff"
        style="@style/customText"
        android:text="测试Attr"/>
```

这时 TextView 的颜色值是在布局文件中设置的颜色值。

综上，我们可以得到结论：Androd 使用属性的顺序为：布局文件中设置的值 > View style > Activity theme > Application theme。

## 获取 Attr

有些情况下，我们可能需要使用 theme 中的属性值，比如下面我们想让一个 TextView 直接显示我们在 theme 中定义的颜色，则可以如下做：

先在 attr.xml 中自定义一个属性组：

```
    <declare-styleable name="myCustomStyle">
        <attr name="myCutsomColor" format="color" />
    </declare-styleable>
```

然后把在主题中为该属性设置一个颜色值：

```
    <style name="TestActivityTheme" parent="Theme.AppCompat">
        <item name="android:textColor">#FFFF0000</item>
        <item name="android:textSize">10sp</item>
        <item name="myCutsomColor">#FF00FFFF</item>
    </style>
```

那么我们可以再 TextView 中使用：

```
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textColor="?attr/myCutsomColor"
        android:text="测试Attr"/>
```

Attr 的使用格式是：

```
?[*<package_name>*:][*<resource_type>*/]*<resource_name>*
```

如果是本应用中的attr使用，则可以省去<package_name>部分。
如果想要使用系统的属性值：

```
android:textColor="?android:textColorSecondary"
```

此处的textColor使用当前主题的android:textColorSecondary属性内容。因为资源工具知道此处是一个属性，所以省去了attr （完整写法：?android:attr/textColorSecondary）。
