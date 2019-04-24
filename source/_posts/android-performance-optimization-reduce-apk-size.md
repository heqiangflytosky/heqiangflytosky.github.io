---
title: Android 性能优化实战篇 -- APK 瘦身
categories: Android性能优化
comments: true
tags: [APK 瘦身]
description: 介绍 APK 瘦身的一些方法
date: 2017-7-28 10:00:00
---

## 概述

随着业务的发展和需求的增加，APK包的体积会越来越大，这样会导致固件体积的增大，也会影响应用运行效率，过大的 APK 包也影响了下载安装体验。因此，APK 瘦身也成为我们对应用性能优化的一项重要工作。
[Android 官方瘦身文档](https://developer.android.com/topic/performance/reduce-apk-size)
Android Studio 2.2 版本以后提供了一个帮助我们分析 APK 包的工具 -- APK Analyzer，具体用法可以参考这篇文章 [Android 性能优化工具篇 -- APK Analyzer](http://www.heqiangfly.com/2017/07/26/android-performance-optimization-apk-analyzer/)。

## APK 包结构

在探讨如何精简 APK 之前，先来了解一下 APK 的结构，这样会使我们的工作有方向性。
APK 文件由 Zip 压缩文件组成。这些文件包括 Java 类文件、资源文件和包含已编译资源的文件。 

 - META-INF/：包含 CERT.SF 和 CERT.RSA 签名文件，以及 MANIFEST.MF 清单文件。
 - assets/：包含应用的资源；应用可以使用 AssetManager 对象检索这些资源。
 - res/：包含未编译到 resources.arsc 中的资源。
 - lib/：包含特定于处理器软件层的编译代码。此目录包含每种平台类型的子目录，如 armeabi、armeabi-v7a、arm64-v8a、x86、x86_64 和 mips。
 - resources.arsc：包含已编译的资源。此文件包含 res/values/ 文件夹的所有配置中的 XML 内容。打包工具会提取此 XML 内容，将其编译为二进制文件形式，并将相应内容进行归档。此内容包括语言字符串和样式，以及未直接包含在 resources.arsc 文件中的内容（例如布局文件和图片）的路径。
 - classes.dex：包含以 Dalvik/ART 虚拟机可理解的 DEX 文件格式编译的类。
 - AndroidManifest.xml：包含核心 Android 清单文件。此文件列出了应用的名称、版本、访问权限和引用的库文件。该文件使用 Android 的二进制 XML 格式。


## 导致 APK 体积增大的问题

- 应用自身资源问题
 - 资源冗余问题：包括一些无用图片资源、string资源和so文件等。
 - 不合理的使用资源：可以使用代码使用的效果尽量不要用图片，比如一些纯色、渐变图片和背景图片等。
 - 引入第三方 aar 时带入的一些冗余资源：可能会引入业务完全没有使用的一些资源，比如asset, so, res等。
- 应用自身代码问题
 - 代码冗余问题：包括一些无用的 java 类、方法和变量。
 - 代码质量不高，重用性不高，有很多相同逻辑代码。
 - 代码问题，一些编程习惯导致编译后的dex文件增大。
 - proguard文件keep了过多的类，影响压缩效果
- 打包集成问题
 - 可以去掉一些没用的语言资源、不同分辨率图片资源
 - 可以去掉一些cpu架构的so
 - 引入的多个sdk包中有相同的资源


## 瘦身方案

### 应用自身冗余资源问题优化

#### 去除无用的资源

可以使用  Android Studio 中提供的静态代码分析器 lint 工具来检测 res/ 文件夹中未被代码引用的资源。当 lint 工具发现项目中有可能未使用的资源时，会显示一条消息，如下例所示。 

```
    res/layout/preferences.xml: Warning: The resource R.layout.preferences appears
        to be unused [UnusedResources]
```

**注意**：lint 工具不会扫描 assets/ 文件夹、通过反射引用的资源或已链接到应用的库文件。此外，它也不会移除资源，只会提醒您它们的存在。
可以两种方式来运行，可以通过 Android Studio菜单栏的 Analyze -> Inspect Code 来运行；还可以通过 lint 命令行来运行。
lint 具体的使用会有一篇文章来专门介绍。

我们还可以在 build.gradle 中启用 shrinkResources 来自动移除添加到代码的库中包含未使用的资源。

```
    android {
        // Other settings

        buildTypes {
            release {
                minifyEnabled true
                shrinkResources true
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            }
        }
    }
```

其实这里并不是彻底删除，而是保留文件名，但是没有内容。比如图片可能就一个占位图片（67B）。layout是一个空的xml文件（47B）：

```
<?xml version="1.0" encoding="utf-8"?>
<x />
```

要启用 shrinkResources，您必须启用代码压缩功能，即设置 minifyEnabled true。
**注意**：string.xml中没有被引用的字符串资源不会被删除。

#### 使用可绘制对象

某些图片不需要静态图片资源，可以在运行时通过代码动态绘制图片。Drawable 对象（XML 中为 `<shape>`）会占用 APK 中的少量空间。此外，XML Drawable 对象会生成符合 Material Design 准则的单色图片。
一些简单的动画效果也可以通过代码绘制对象来实现。

#### 重复使用资源

通过代码重用一些资源，比如通过旋转等特效来使用已有图片生成所需要的图片。

#### 压缩图片

使用一些压缩技术来压缩图片

#### 使用 WebP 文件格式

#### 使用矢量图形

#### 将矢量图形用于动画图片

#### 资源混淆压缩

可以使用微信的开源工具 AndResGuard 进行资源文件混淆以及7z压缩。
[AndResGuard Giyhub 地址](https://github.com/shwenzhang/AndResGuard)

### 应用自身编码问题优化

#### 压缩代码

要通过 ProGuard 启用代码压缩，可以在 build.gradle 文件内相应的构建类型中配置 minifyEnabled true。

#### 移除无用的代码

#### 资源引用优化

复用系统自带的资源。Android系统本身内置了很多的资源，例如字符串/颜色/图片/动画/样式以及简单布局等等，这些资源都可以在应用程序中直接引用。这样做不仅仅可以减少应用程序的自身负重，减小APK的大小，另外还可以一定程度上减少内存的开销，复用性更好。但是也有必要留意Android系统的版本差异性，对那些不同系统版本上表现存在很大差异，不符合需求的情况，还是需要应用程序自身内置进去。

#### 避免使用枚举

以前博客也有介绍，单个枚举会使应用的 classes.dex 文件增加大约 1.0 到 1.4 KB 的大小。这些增加的大小会快速累积，产生复杂的系统或共享库。如果可能，请考虑使用 @IntDef 注释和 ProGuard 移除枚举并将它们转换为整数。此类型转换可保留枚举的各种安全优势。

### 打包集成问题优化

#### 语言资源优化

我们可以再 build.gradle 文件中选择性配置我们要打包的语言资源，对于一些不常用的语言可以选择性的不打包，这样可以节省一些体积。

```
android {
    defaultConfig {
        resConfigs "zh-rCN","zh-rHK","zh-rTW"
    }
```

加入这样的配置就只把默认 default、zh-rCN、zh-rHK 和 zh-rTW 四种语言资源打包进 APK。

#### 图片资源优化

同样也可以在 build.gradle 文件中选择性配置我们要打包的图片资源。

在老版本的 gradle 中可以使用下面的方式选择性的打包图片：

```
resConfigs "xxhdpi", "xxxhdpi"
```

但是在新版本中该方式已经被废弃，替代方案是在 gradle 中使用 splits 根据不同的 ABI 以及不同的屏幕密度分别打包。
[Android 官方文档](https://developer.android.com/studio/build/configure-apk-splits.html)

```
android {
  ...
  splits {

    // Configures multiple APKs based on screen density.
    density {

      // Configures multiple APKs based on screen density.
      enable true

      // Specifies a list of screen densities Gradle should not create multiple APKs for.
      exclude "ldpi", "xxhdpi", "xxxhdpi"

      // Specifies a list of compatible screen size settings for the manifest.
      compatibleScreens 'small', 'normal', 'large', 'xlarge'
    }
  }
}
```

### 插件化

插件化除了可以解决方法数超标的问题，同时如果插件放到服务器，在需要时再下载下来，可以减少安装包大小。但是需要注意的是，因为使用时需要临时下载，所以建议在用户使用率不高的功能模块上使用，或者使用预加载。

## 瘦身方法成本收益对比