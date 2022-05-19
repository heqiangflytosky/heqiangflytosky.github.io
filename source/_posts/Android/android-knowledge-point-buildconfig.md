---
title: Android BuildConfig的使用
categories: Android
comments: true
tags: [Android, BuildConfig]
description: 介绍如何在 Android 开发中 BuildConfig 类的用法
date: 2016-2-2 10:00:00
---

## 概述
`BuildConfig` 是 Android Studio 在 Build 之后自动生成的一个类，顾名思义，我们可以在里面定义一些编译配置相关的变量。
在不同的编译模式下会生成不同的变量，比如在 Debug 和 Release 模式下，我们可以利用这些变量来方便不同编译环境下的开发，比如日志的打印等等。
我们可以在哪里找到 `BuildConfig` 这个类呢？如果我们编译的是 Debug 版本的应用，可以在下面的目录中找到：
```
./app/build/generated/source/buildConfig/debug/<你的包名>/BuildConfig.java
```

下面来看一下默认情况下 `BuildConfig` 类：
```java
/**
 * Automatically generated file. DO NOT MODIFY
 * 这个类是自动生成的，不要试图去修改它
 */
public final class BuildConfig {
  public static final boolean DEBUG = Boolean.parseBoolean("true");
  public static final String APPLICATION_ID = "com.example.heqiang.testsomething";
  public static final String BUILD_TYPE = "debug";
  public static final String FLAVOR = "";
  public static final int VERSION_CODE = 1;
  public static final String VERSION_NAME = "1.0";
}
```

## 自定义配置

我们可以在 `build.gradle` 中使用 `buildConfigField` 方法来添加自己的自定义常量。

```java
    buildTypes {
        debug{
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            buildConfigField("String", "TEST_STRING", "\"debug\"")
            buildConfigField("boolean", "IS_DEBUG", "true")
        }
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            buildConfigField("String", "TEST_STRING", "\"release\"")
            buildConfigField("boolean", "IS_DEBUG", "false")
        }
```

在 `build.gradle` 的 `buildTypes` 节点 `debug` 和 `release` 节点中分别添加同名的变量配置信息，如上面的代码所示。
sync 之后 `BuildConfig` 类就会自动重新生成：

```java
public final class BuildConfig {
  public static final boolean DEBUG = Boolean.parseBoolean("true");
  public static final String APPLICATION_ID = "com.example.heqiang.testsomething";
  public static final String BUILD_TYPE = "debug";
  public static final String FLAVOR = "";
  public static final int VERSION_CODE = 1;
  public static final String VERSION_NAME = "1.0";
  // Fields from build type: debug
  public static final boolean IS_DEBUG = true;
  public static final String TEST_STRING = "debug";
}
```

`buildConfigField` 定义自定义常量时后面的括号也可以去掉，下面的用法也是可以的： `buildConfigField "boolean", "IS_DEBUG", "true" `。




