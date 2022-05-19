---
title: Android 常用调试知识
categories: Android
comments: true
tags: [Android]
description: 一些 Android 常用调试知识汇总
date: 2014-4-10 10:00:00
---

## 运行过程中等待调试器

我们常常有这样的需求，在应用启动过程中进行调试，那么就要运用等待调试器功能。
有两种方法可以实现：

 1. 在设置里面进行设置
  1. 进入设置-->辅助功能-->开发者选项；如果没有打开开发者模式，在拨号里面输入*#*#6961#*#*；
  2. 找到选择调试应用，打开选择你要调试的应用；
  3. 再把等待调试器选项打开；
  4. 这样你要选择调试的应用在启动过程中就自动进入了调试模式；
 2. 代码中设置 `android.os.Debug.waitForDebugger()`，这里当程序执行到这里时会等待调试器，如果我们连接调试器后才会继续执行。

## 修改eclipse debug的代码源文件的查找路径

进入Debug窗口，选中debug的项目，右键单击，在弹出的菜单中选择“Edit Source Lookup ......”，选择Add，在弹出框中选择project，就可以在里面改动和添加了源文件的查找路径了。

## 强制执行onTrimMemory()回收应用内存

 - `adb shell dumpsys gfxinfo com.android.test -cmd trim 80`
 - ` am send-trim-memory [--user <USER_ID>] <PROCESS>
[HIDDEN|RUNNING_MODERATE|BACKGROUND|RUNNING_LOW|MODERATE|RUNNING_CRITICAL|COMPLETE]`

## App Crash没有打印

如果App在某种情况下一言不合的挂掉了，而且没有任何Crash堆栈的打印，不要怀疑了，赶快看看是不是触发了finish()导致的，本人已踩坑。

## 调试SystemUI进程等后台进程时自动断开的问题

断点调试SystemUI时你会发现进程会自己挂掉，打开 开发者选项->显示所有“应用无响应”(ANR) 选项就好了。

## 使用Android Studio来查看依赖列表

有时候我们应用在编译的时候会遇到类似的问题：

```
Error:Error converting bytecode to dex:
Cause: com.android.dex.DexException: Multiple dex files define Landroid/support/v4/accessibilityservice/AccessibilityServiceInfoCompat$AccessibilityServiceInfoVersionImpl;
```

或者：

```
Error:java.io.IOException: Duplicate zip entry [support-annotations-25.2.0.jar:android/support/annotation/ColorRes.class]
```

这种问题一般就是引用的第三方库里面重复应用了一些包导致的问题，这个时候我们就需要知道是哪些包重复引用了。
我们除了可以用命令行 

```
./gradlew <projectname>:dependencies --configuration compile 
```

projectname 为 settings.gradle 里面配置的各个 project，如果没有配置，直接运行 

```
./gradlew dependencies --configuration compile
```

比如 

```
./gradlew app:dependencies --configuration compile
```

来查看项目的依赖列表之外，还可以在 Android Studio 里面查看。
方法如图：

![效果图](/images/android-knowledge-debug-skill-one/android-debug-skill-show-dependencies.png)

就可以在Gradle Console窗口里面看到项目的依赖库列表。 
[参考](http://stackoverflow.com/questions/20989317/multiple-dex-files-define-landroid-support-v4-accessibilityservice-accessibility)