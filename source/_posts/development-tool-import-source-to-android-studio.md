---
title: Android Studio 导入 Android 源码
categories: 开发工具
comments: true
keywords: SourceCode, Android Studio
tags: [Android Studio, 开发工具]
description: 介绍一些如何使用 Android Studio 导入 Android 源码以及一些配置方法
date: 2016-12-3 10:00:00
---
Android 的源码代码量是非常大的，也有多种代码编辑器来供我们选择来去阅读Android源码，虽然在 Eclipse 和 SourceInsight 上阅读 Android 源码也能带来很好的体验，但习惯于用 Android Studio 来作为应用开发工具的大家肯定也希望用它来阅读源代码。本文将介绍如何将 Android 源码导入到 Android Studio 中来的技巧。
<!-- more -->

## 导入源码
### 修改Android Studio的配置文件
由于导入源码时需要消耗大量内存，所以建议修改`studio64.vmoptions`文件：
```
-Xms1024m
-Xmx1024m
```

### 生成导入到Android Studio所需的配置文件
首先要编译一次源码，然后看有没有`out/host/linux-x86/framework/idegen.jar`
如果没有的话就执行一下下面的命令，生成`out/host/linux-x86/framework/idegen.jar`：
```
source build/envsetup.sh
mmm development/tools/idegen/
```
然后执行一下下面的命令：
```
development/tools/idegen/idegen.sh
```
会在根目录下面生成`android.ipr`和`android.iml`。
`android.ipr` 一般保存了工程相关的设置，比如modules和modules libraries的路径，编译器配置，入口点等。
`android.iml` 用来描述modules。它包括modules路径、 依赖关系，顺序设置等。一个项目可以包含多个 *.iml 文件。
到这一步我们其实就可以导入到Android Studio里面去了。

### 过滤一些模块
如果把Android所有的源码全部导入到Android Studio里面去，工程将会非常大，而且会很耗时间，那么我们就可以把不需要的模块给过滤掉。
打开`android.iml`文件，加入以下代码，修改`excludeFolder`的配置：
```
<excludeFolder url="file://$MODULE_DIR$/.repo"/>
<excludeFolder url="file://$MODULE_DIR$/abi"/>
<excludeFolder url="file://$MODULE_DIR$/frameworks/base/docs"/>
<excludeFolder url="file://$MODULE_DIR$/art"/>
<excludeFolder url="file://$MODULE_DIR$/bionic"/>
<excludeFolder url="file://$MODULE_DIR$/bootable"/>
<excludeFolder url="file://$MODULE_DIR$/build"/>
<excludeFolder url="file://$MODULE_DIR$/cts"/>
<excludeFolder url="file://$MODULE_DIR$/dalvik"/>
<excludeFolder url="file://$MODULE_DIR$/developers"/>
<excludeFolder url="file://$MODULE_DIR$/development"/>
<excludeFolder url="file://$MODULE_DIR$/device"/>
<excludeFolder url="file://$MODULE_DIR$/docs"/>
<excludeFolder url="file://$MODULE_DIR$/external"/>
<excludeFolder url="file://$MODULE_DIR$/hardware"/>
<excludeFolder url="file://$MODULE_DIR$/kernel-3.18"/>
<excludeFolder url="file://$MODULE_DIR$/libcore"/>
<excludeFolder url="file://$MODULE_DIR$/libnativehelper"/>
<excludeFolder url="file://$MODULE_DIR$/ndk"/>
<excludeFolder url="file://$MODULE_DIR$/out"/>
<excludeFolder url="file://$MODULE_DIR$/pdk"/>
<excludeFolder url="file://$MODULE_DIR$/platform_testing"/>
<excludeFolder url="file://$MODULE_DIR$/prebuilts"/>
<excludeFolder url="file://$MODULE_DIR$/rc_projects"/>
<excludeFolder url="file://$MODULE_DIR$/sdk"/>
<excludeFolder url="file://$MODULE_DIR$/system"/>
<excludeFolder url="file://$MODULE_DIR$/tools"/>
<excludeFolder url="file://$MODULE_DIR$/trusty"/>
<excludeFolder url="file://$MODULE_DIR$/vendor"/>
```
这样我们就只导入了`frameworks`和`packages`的代码。

### 导入
`File` -> `Open` 选择源码目录就可以导入了，整个导入过程需要3分钟左右，然后就可以去阅读源码了。

## 一些配置
### 设置SDK和JDK

![配置图](/images/development-tool-import-source-to-android-studio/project.png)

### 解决源码中跳转错误问题
设置`Modules`的依赖：
先将所有依赖删掉，只留下图中的两个(注意:这里删除全部只是为了方便。如果确实用到了.jar，再将它们的路径添加进来就可以了）。
![配置图](/images/development-tool-import-source-to-android-studio/modules-add.png)

点击上图中的'+'并选择上图中的`Jars or directories`选项，依次将`frameworks`和`external`文件夹添加进来。如图：

![配置图](/images/development-tool-import-source-to-android-studio/modules-added.png)

推荐把frameworks和external这两个移到最上面，这样在代码跳转时会优先从这两个文件夹下查找，而不是在Android.jar中查找。

### Debug源码
我们可以通过给刚导入的工程在`Modules`中添加`Android Framework`来让AS将它作为一个Android工程，从而方便我们调试代码。

![配置图](/images/development-tool-import-source-to-android-studio/debug-add.png)

可以按照上图中来添加Android Framework支持。
然后我们就可以发现，调试按钮已经可以点击了。

![配置图](/images/development-tool-import-source-to-android-studio/debug-added.png)

有时候遇到过点击调试按钮没有调试进程可以选择，打开`Project Structure`->`Project`，重新选择一下`Projct SDK`就可以了。
