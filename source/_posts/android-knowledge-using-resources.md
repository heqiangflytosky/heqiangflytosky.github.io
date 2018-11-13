---
title: Android  应该了解的知识系列 - 资源引用的那些事儿
categories: Android
comments: true
tags: [Android]
description: 介绍Android 中资源引用的几种方法
date: 2015-2-20 10:00:00
---
android-knowledge-using-resources.md
## 概述

Android 开发过程中避免不了的会引用 xml 资源，现在针对资源的引用方式总结如下。

## 资源引用方式

先介绍一下下面一些名称代表的意义：

 - package_name：资源文件所在的包名
 - resource_type：资源文件的类型，比如string、color等
 - resource_name：资源文件的名称

### 引用自定义资源

> `@[<package_name>:]<resource_type>/<resource_name>`

比如：

 - `android:textColor="@color/Blue_1"`

另外：

 - @+id/资源ID名： 表示创建一个新的资源ID
 - @id/资源ID名： 应用现有已定义的资源ID，包括系统ID
 - @android:id/资源ID名：引用系统ID，其等效于@id/资源ID名

### 引用系统资源

#### 引用 public 资源

 public 资源是指在 `<sdk_path>/platforms/android-21/data/res/values/public.xml` 中定义的资源

引用方式：

> `@android:<resource_type>/<resource_name>`

比如：

 - `android:textColor="@android:color/black"`

#### 引用非 public 资源

引用方式：

> `@*android:<resource_type>/<resource_name>`

这种引用方式是google不推荐使用的。

如果 Java 代码中引用可以`com.android.internal.R.<resource_type>.<resource_name>`，但是这种方式第三方应用是不适用的，可以通过反射 `com.android.internal.R` 的方式实现。
或者是通过在 xml 引用，在通过 `getResources` 方式使用。

### 引用主题属性

> `?[<package_name>:]<resource_type>/<resource_name>`

？表示从当前 Theme 中查找引用的资源名，这个 google 叫预定义样式，用在多主题时的场景，属性值会随着主题而改变。这样我们就不必提供具体的值，而可以通过改变主题来改变UI元素的外观。
用法：

 - `android:textColor="?color"`：引用自定义的
 - `android:textColor="?attr/color"`：引用自定义的，和上面等效
 - `android:textColor="?android:colorButtonNormal"`：引用系统的
 - `android:textColor="?android:attr/colorButtonNormal"`：引用系统的，和上面等效
 - `android:textColor="?attr/android:colorButtonNormal"`：引用系统的，和上面等效


