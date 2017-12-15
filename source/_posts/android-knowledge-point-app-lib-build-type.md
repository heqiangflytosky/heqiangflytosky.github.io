---
title: Android 解决Lib Module 和 App Module 的 buildType 不一样的问题
categories: Android
comments: true
tags: [Android, buildType]
description: 介绍如何解决 Android 多模块开发过程中遇到的 Lib Module 和 App Module 的 buildType 不一样的问题
date: 2016-3-19 10:00:00
---

## 概述

在前一篇文章 [Android BuildConfig的使用](http://www.heqiangfly.com/2016/03/18/android-knowledge-point-buildconfig/) 中介绍了 `BuildConfig` 的使用，那么在使用的过程中你也会会遇到这样的问题，Lib Module 和 App Module 的 `BuildConfig.DEBUG` 的值是不一样的。
这是因为默认情况下依赖库总是以 release 模式打包, 导致 `BuildConfig.DEBUG` 为 `false`。那么本文将带来这种问题的集中解决办法。

## 问题描述

假设在应用中有下面模块，App、ModuleA、ModuleB 和 Common 模块，Common 模块提供了一些公用代码来供其他模块来使用，现在 App 模块 引入了 Common 模块，来看下面在 APP Module 中的一段代码：

```java
        Log.e("Test","App debugMode = "+BuildConfig.DEBUG);
        Log.e("Test","Common debugMode = "+com.android.hq.common.BuildConfig.DEBUG);
```

如果以 Debug 模式编译，打印结果为：true 和  false；
这也就印证了前面我们的说法。
下面带来集中解决方案。

## 解决方案

### 使用高版本的 Gradle

使用Gradle 3.0 以上版本不会有这个问题

### Common Module 提供其他版本

1 在 Common 模块的 build.gradle 文件中添加下面代码：

```java
android {

    ......
    publishNonDefault true
}
```

2 在 App 模块的 build.gradle 文件中添加下面代码：

```java
dependencies {
    //依赖library
    //compile project(':common')
    debugCompile project(path: ':common', configuration: 'debug')
    releaseCompile project(path: ':common', configuration: 'release')
}
```

来修改 Common 模块的应用方式，再来看上面的代码的打印：结果为 true 和 true。
问题解决

### 在 Common 中使用反射来获取 App 中的 BuildConfig.DEBUG

代码：

```java
public class AppBuildConfig {
    private static Boolean sIsDebug = null;
    public static boolean getBuildConfigDebugValue() {
        if (sIsDebug == null) {
            try {
                Class<?> clazz = Class.forName("包名.BuildConfig");
                Field field = clazz.getField("DEBUG");
                sIsDebug = (Boolean) field.get(null);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        return sIsDebug != null ? sIsDebug : false;
    }
}
```

### 使用 ApplicationInfo.FLAG_DEBUGGABLE 来判断 DEBUG Type

我们反编译 Debug 包和 Release 包对比看看有没有其他的区别，会发现他们 AndroidManifest.xml 中 application 节点的 `android:debuggable` 值是不同的。Debug 包值为 `true`，Release 包值为 `false`，这是编译自动修改的。所以我们考虑通过 `ApplicationInfo` 的这个属性去判断是否是 Debug 版本，如下：

```java
public class AppUtils {
 
    private static Boolean isDebug = null;
 
    public static boolean isDebug() {
        return isDebug == null ? false : isDebug.booleanValue();
    }

    public static void syncIsDebug(Context context) {
        if (isDebug == null) {
            isDebug = context.getApplicationInfo() != null &&
                    (context.getApplicationInfo().flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0;
        }
    }
}
```

## 引申问题

上面的方法 2 中，如果 App 依赖 ModuleA 和 Common，ModuleA 又依赖 Common，在Common 中使用新特性发布了全版本，在 App 中使用方法 2 中的特性进行控制，App 对 ModuleA 不做控制，ModuleA 对 Common 也不做控制，这种方法就会有下面的报错：

```
Error:Execution failed for task ':app:processDebugResources'.
> Error: more than one library with package name 'com.android.hq.common'
  You can temporarily disable this error with android.enforceUniquePackageName=false
  However, this is temporary and will be enforced in 1.0
```

因为当应用 App 是 debug 的时候，库 Common 是被新特性控制成 debug 的了，另一边库 ModuleA 只默认构建 release 版本，就自然使用了 release ，而库 ModuleA 依赖的库 Common 因为是普通依赖，自然也是默认的 release。
这样整个项目中就会存在一个 debug 的库 Common 和一个 release 的库 Common，Gradle 就报了构建错误：`more than one library with package name：XXX`。

### 解决办法

让 ModuleA 也发布全版本，ModuleA 也用新特性控制 Common。
App 模块的 build.gradle ：

```
dependencies {
    //依赖library
    //compile project(':common')
    debugCompile project(path: ':common', configuration: 'debug')
    releaseCompile project(path: ':common', configuration: 'release')

    //compile project(':modulea')
    debugCompile project(path: ':modulea', configuration: 'debug')
    releaseCompile project(path: ':modulea', configuration: 'release')
}
```

ModuleA 模块的 build.gradle ：

```
android {
    ......
    publishNonDefault true
}
dependencies {
    ......
    //compile project(':common')
    debugCompile project(path: ':common', configuration: 'debug')
    releaseCompile project(path: ':common', configuration: 'release')
}
```

## 参考文章
https://www.cnblogs.com/bellkosmos/p/6437171.html
http://www.jianshu.com/p/1907bffef0a3
http://www.trinea.cn/android/android-whether-debug-mode-why-buildconfig-debug-always-false/

