---
title: Android -- Launcher 独立编译
categories:  Android Launcher
comments: true
tags: [Launcher3]
description: 介绍在 Android Studio 中独立编译 Android R Launcher3。
date: 2021-1-6 10:00:00
---

## 概述

Android 中 Launcher3 是在源码中编译的，那么它能不能在 Android Studio 中以独立应用的形式编译呢？答案是肯定的，本文就来介绍一下编译的方法，代码基于 Android R。

## 下载源码

打开 https://android.googlesource.com/，页面搜索 Launcher3，点击进去，复制clone地址：git clone https://android.googlesource.com/platform/packages/apps/Launcher3，切换到 android11-release 分支。
导入到 Android Studio 中，我这里用的是 Android Studio 3.5。

## 问题

1.

```
ERROR: Unable to find a matching configuration of project :IconLoader: None of the consumable configurations have attributes.
```

Android R中这部分代码放到了framework中，源码下载 `git clone https://android.googlesource.com/platform/frameworks/libs/systemui`，切换到 android11-release 分支。然后将代码copy到 Launcher 的 iconloaderlib 目录。
或者找到 sysui_shared.jar 放到 libs 目录下面。
我这里采用下载源码的方式。

2.

```
org.gradle.api.internal.tasks.TaskDependencyResolveException: Could not determine the dependencies of task ':IconLoader:compileDebugJavaWithJavac'.
	at org.gradle.api.internal.tasks.CachingTaskDependencyResolveContext.getDependencies(CachingTaskDependencyResolveContext.java:68)
	at org.gradle.execution.plan.TaskDependencyResolver.resolveDependenciesFor(TaskDependencyResolver.java:46)
	at org.gradle.execution.plan.LocalTaskNode.getDependencies(LocalTaskNode.java:111)
	at org.gradle.execution.plan.LocalTaskNode.resolveDependencies(LocalTaskNode.java:79)
	at org.gradle.execution.plan.DefaultExecutionPlan.addEntryTasks(DefaultExecutionPlan.java:177)
	......
```

用命令行编译就好了，`./gradlew assembleAospWithoutQuickstep`。或者在Android Studio中只编译assembleAospWithoutQuickstep任务。
assembleAospWithoutQuickstep任务编译出来的是不带多任务的版本，调多任务时会调用系统桌面的多任务。

3.

```
ERROR: Failed to find Platform SDK with path: platforms;android-R
```

修改 gradle.properties：

```
//COMPILE_SDK=android-R
COMPILE_SDK=android-30
```

4.

```
QLauncher3/iconloaderlib/src_full_lib/com/android/launcher3/icons/SimpleIconCache.java:69: 错误: 找不到符号
            int index = mUserSerialMap.indexOfKey(user.getIdentifier());
                                                      ^
  符号:   方法 getIdentifier()
  位置: 类型为UserHandle的变量 user
QLauncher3/iconloaderlib/src_full_lib/com/android/launcher3/icons/SimpleIconCache.java:74: 错误: 找不到符号
            mUserSerialMap.put(user.getIdentifier(), serial);
                                   ^
  符号:   方法 getIdentifier()
  位置: 类型为UserHandle的变量 user
QLauncher3/iconloaderlib/src_full_lib/com/android/launcher3/icons/SimpleIconCache.java:87: 错误: 找不到符号
        return info.isInstantApp();
                   ^
  符号:   方法 isInstantApp()
  位置: 类型为ApplicationInfo的变量 info
3 个错误

```

将user.getIdentifier()换成user.hashCode(),
将info.isInstantApp()换成mContext.getPackageManager().isInstantApp(info.packageName);


5.

错误: 程序包com.android.launcher3.logger.LauncherAtom不存在
在 build.gradle 中添加

```
option "java_package=launcher_atom.proto|com.android.launcher3.logger"
```

错误: 程序包com.android.launcher3.userevent.LauncherLogProto.Target不存在
错误: 程序包com.android.launcher3.userevent.LauncherLogProto不存在
修改包名com.android.launcher3.userevent.LauncherLogProto 为 com.android.launcher3.userevent.nano.LauncherLogProto
先这样改，后面这部分代码会注释掉。

错误: 找不到符号com.google.protobuf.InvalidProtocolBufferException
修改成Exception

/FolderInfo.java:86: 错误: 不兼容的类型: int无法转换为Attribute
mLogAttribute修改成int类型

6.

剩下的再找不到，那就注释掉吧，logger 和 userevent 这部分是打印日志和统计使用，没有太大作用。编译不通过的部分可删掉。
比如：
ItemInfo.java:34 错误: 找不到符号import static com.android.launcher3.logger.LauncherAtom.ContainerInfo.ContainerCase.CONTAINER_NOT_SET;

把这部分代码注释掉。

7.
 错误: 程序包com.google.protobuf.nano不存在
 错误: 程序包com.android.systemui.shared.system不存在
 错误: 程序包com.android.systemui.plugins.annotations不存在
 
在maven上下载 protobuf-javanano jar包，修改名字launcher_protos.jar。
网上下载 plugin_core.jar。

项目根目录下新建一个libs文件夹，将这2个jar放进去,并修改build.gradle，去引用这2个jar。

```
    implementation fileTree(dir: "libs", include: 'launcher_protos.jar')

    ......

    // Required for AOSP to compile. This is already included in the sysui_shared.jar
    withoutQuickstepImplementation fileTree(dir: "libs", include: 'plugin_core.jar')
```


编译成功，安装到手机，去应用管理里面把系统桌面的默认应用设置一下，桌面正常运行起来。

源码请参考[GitHub](https://github.com/heqiangflytosky/QLauncher3/tree/android11)


## 参考文章

https://zhuanlan.zhihu.com/p/141243776
https://github.com/HqyBryan/Launcher3

https://www.jianshu.com/p/cc333ee61554
https://github.com/huolizhuminh/Launcher3