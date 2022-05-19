---
title: Android 性能优化工具篇 -- APK Analyzer
categories: Android性能优化
comments: true
tags: [APK Analyzer]
description: 介绍 APK Analyzer 工具的使用
date: 2017-7-26 10:00:00
---

## 概述

APK Analyzer 顾名思义就是 APK 分析器，它是 Android Studio 2.2 版本开始提供的工具。通过它我们可以很方便的了解 APK 包的结构，因此它也是我们开发过程中进行 APK 瘦身 的一大利器。
这里将它的用法做一下归纳和总结。

[官方文档](https://developer.android.com/studio/build/apk-analyzer.html)

先来概述一下 APK Analyzer 可以实现的一些功能：

 - 查看 APK 压缩包的信息：包含的文件及其大小；原文件以及压缩后的大小；
 - 查看 APK 的文件：比如 AndroidManifest.xml 文件；资源文件(resources.arsc、layout文件、res、assets等)等；
 - 查看 DEX 文件：类信息、包信息等；
 - 比较两个 APK 文件；

## 使用方法

使用 APK Analyzer 打开 APK 的方式：

 - 直接拖拽 APK 到 Android Studio 的编辑窗口
 - 切换到 Project 视图，双击 APK 文件
 - 在菜单栏中选择 Build -> Analyzer APK，然后选择 APK 文件
 
下图为打开 APK 后的视图：

![效果图](/images/android-performance-optimization-apk-analyzer/open.png)

从这个视图上面我们可以得到下面的信息：

 - APK 的包名和版本信息
 - APK 包大小，包括 APK 解压后的大小和压缩后的大小
 - APK 包的结构信息，包含各个目录的大小（包括解压后的大小和压缩后的大小）
 
## 查看文件

### 查看资源文件

包含 resources.arsc、layout文件、res、assets等。
点击 resources.arsc，在视图的下方区域就是显示 resources.arsc 的各种资源列表：attr，drawable、dimen、string等，点击列表的项便会展示该项的所有资源信息，包含资源的个数和配置维度的个数：

![效果图](/images/android-performance-optimization-apk-analyzer/resources.png)

其他资源的打开方式也是一样，点击既可在视图中展示。

### 查看 AndroidManifest.xml 文件

点击既可展示 AndroidManifest.xml 文件的详细信息。

### 查看 DEX 文件

APK Analyzer 的浏览 DEX 文件的功能可以使我们快速了解 DEX 文件的信息，我们能看到类、包、总的引用和声明个数，这些信息能够帮助我们决定是否使用multi-dex或者移除依赖使得满足64K方法数限制。

![效果图](/images/android-performance-optimization-apk-analyzer/dex1.png)

上图展示了一个 APK 中的一个 DEX 文件。每个包、类、方法都列有 Defined Method、Referenced Method 和 Size。Referenced Method列是DEX文件中引用的全部方法，它包含了你定义的方法、依赖的library、定义在标准Java和Android包中的方法。Defined Method列只包含了定义在DEX文件中方法，因此它是Referenced Method方法的子集。注意当你引入一个依赖，在依赖中定义的方法会包含在Defined Method和Referenced Method中。还要注意，混淆压缩也会改变DEX文件的内容。
Size 是指选中的包、类、方法的大小。
下面来介绍一个工具栏的用法。工具栏是 Android Studio 3.0 才加入的功能。

 - 点击 Show fields （f按钮）可以切换展示或者隐藏类的变量。
 - 点击 Show methods （m按钮）可以切换展示或者隐藏类的方法。
 - 点击 Show all referenced methods or fields（mf按钮）可以切换展示所有引用的类、方法和变量或者只展示定义在DEX文件中类、方法和变量。
 - Load Proguard mappings：如果我们用 APK Analyzer 浏览混淆过的 APK，包名和类名都是混淆过的，影响我们查看，这是我们可以用这个功能导入 mapping.txt 文件。这样就相当于还原了混淆文件。如下图。

![效果图](/images/android-performance-optimization-apk-analyzer/dex2.png)

## 对比 两个 APK 