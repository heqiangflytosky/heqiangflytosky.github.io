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
# 直接导入源码
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
这里如果你不想编译源码，只想导入进来查看源代码的话，也可以使用其他工程生成的 `android.ipr` 和 `android.iml` ，复制到根目录即可。
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
可以按照下图中来添加Android Framework支持，点击 + 号，然后选择 Android。

![配置图](/images/development-tool-import-source-to-android-studio/debug-add.png)

然后我们就可以发现，调试按钮已经可以点击了。

![配置图](/images/development-tool-import-source-to-android-studio/debug-added.png)

## 一些问题

### Choose Process -> Nothing to show

有时候遇到过点击调试按钮没有调试进程可以选择，打开`Project Structure`->`Project`，重新选择一下`Projct SDK`就可以了。

# 通过软连接轻量导入

由于上面的方法是把代码直接加载到 AS 中，由于 AOSP 代码数量庞大，这样无疑会吃掉大量内存，导致系统卡顿。
再介绍一个通过给代码建立软连接的方法来打开AOSP源码。
首先也需要 android.ipr和android.iml，生成的方式和上面是一样的。
将下面的命令保存为脚本文件（如：create-linked-project.sh），保存到源码根目录，并在源码根目录运行。

```
#!/bin/bash
######################################     Step 1    ##############################################
PARENT_PATH=$(dirname $(pwd))

# 获取源码目录的路径和文件夹名称. etc. "android-12.0.0_r13"
REAL_PROJECT_PATH=$(cd `dirname $0`; pwd)
REAL_PROJECT_NAME="${REAL_PROJECT_PATH##*/}"

# 创建软链接目录. etc. "android-12.0.0_r13-ln"
LINKED_PROJECT_NAME="${REAL_PROJECT_NAME}-ln"
LINKED_PROJECT_PATH="${PARENT_PATH}/${LINKED_PROJECT_NAME}"
mkdir -p ${LINKED_PROJECT_PATH}

######################################     Step 2    ##############################################
# 复制 android.ipr、android.iml 到软链接目录
cp android.ipr ${LINKED_PROJECT_PATH}
cp android.iml ${LINKED_PROJECT_PATH}

######################################     Step 3    ##############################################
# 链接 build、device 目录，常用于机型相关配置文件修改。
ln -sf ${REAL_PROJECT_PATH}/build ${LINKED_PROJECT_PATH}
ln -sf ${REAL_PROJECT_PATH}/device ${LINKED_PROJECT_PATH}

# 链接 frameworks/base、frameworks/libs ，用于SystemUI、Framework开发。
mkdir -p ${LINKED_PROJECT_PATH}/frameworks
ln -sf ${REAL_PROJECT_PATH}/frameworks/base ${LINKED_PROJECT_PATH}/frameworks
ln -sf ${REAL_PROJECT_PATH}/frameworks/libs ${LINKED_PROJECT_PATH}/frameworks

# 链接 packages/apps/Launcher3 ，用于桌面应用的开发。
mkdir -p ${LINKED_PROJECT_PATH}/packages/apps
ln -sf ${REAL_PROJECT_PATH}/packages/apps/Launcher3 ${LINKED_PROJECT_PATH}/packages/apps

# 链接 packages/apps/Settings ，用于设置应用的开发。
mkdir -p ${LINKED_PROJECT_PATH}/packages/apps
ln -sf ${REAL_PROJECT_PATH}/packages/apps/Settings ${LINKED_PROJECT_PATH}/packages/apps

# 链接 out ，用于给Android Studio添加依赖，解决代码报红的问题。
mkdir -p ${LINKED_PROJECT_PATH}/out/target/common
ln -sf ${REAL_PROJECT_PATH}/out/target/common/obj ${LINKED_PROJECT_PATH}/out/target/common/
```
如果要链接其他目录，则按需修改 Step 3下方的命令即可。
然后会在源码目录的同级目录生成一个包含软连接的目录：`<dir name>-ln`。
然后直接打开该目录的 android.ipr即可。

如果遇到找不到符号的问题，可以把out目录中生成的中间文件添加进来，比如：
`out/target/common/obj/APPS/SystemUI_intermediates/classes.jar`,
`out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classed-header.jar`,
`out/target/common/obj/JAVA_LIBRARIES/services_intermediates/classes.jar`,
