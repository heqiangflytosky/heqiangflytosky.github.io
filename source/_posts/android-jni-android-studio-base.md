---
title: Android JNI -- Android Studio JNI 开发基础
categories:  Android
comments: true
tags: [JNI]
description: Android Studio JNI 开发基础
date: 2015-12-10 10:00:00
---

## 概述

本文介绍如何在 Android Studio 环境下进行 JNI 开发。

## 准备工作

首先要在 AS 的 Project Structure 中配置 Android NDK Location。

## 开发

### Java 类

首先创建一个声明 native 方法的 Java 类：

```
public class JniUtils {
    public static native String getString();
}
```

rebuild 一下工程，然后就可以在 `JniUtils` 类所在的 Module 的  `build/intermediates/classes/debug/<包名>` 下面找到 JniUtils.class 文件。

### 生成头文件

进入 `build/intermediates/classes/debug` 目录，执行：

```
javah -jni com.example.heqiang.testsomething.util.JniUtils
```

就会在当前目录生成 com_example_heqiang_testsomething_util_JniUtils.h 头文件。当然这个头文件的文件名你也可以自定义成其他。

### JNI 开发

在 src/main 路径下新建一个名为 jni 的文件夹，再将前面生成的头文件放到该目录下面。
然后可以在目前下面创建 native 文件，名字可以随意定。

```
#include "com_example_heqiang_testsomething_util_JniUtils.h"

JNIEXPORT jstring JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_getKey(JNIEnv *env, jobject obj) {
    return (*env)->NewStringUTF(env,"abcdefghijk");
}
```

## 编译

### AS 自动编译

首先配置 build.gradle :

```
    defaultConfig {
        ndk {
            ndk {
                moduleName "JniDemo"                       //生成的so名字
                abiFilters "armeabi", "armeabi-v7a", "x86" //输出指定三种abi体系结构下的so库
            }
        }

    }
```

触发编译，这个时候可以会报错：

```
Error:Execution failed for task ':app:compileDebugNdk'.
> Error: NDK integration is deprecated in the current plugin.  Consider trying the new experimental plugin.  For details, see http://tools.android.com/tech-docs/new-build-system/gradle-experimental.  Set "$USE_DEPRECATED_NDK=true" in gradle.properties to continue using the current NDK integration.
```

在 gradle.properties 文件中配置：

```
android.useDeprecatedNdk=true
```

然后再次编译，在 `build/intermediates/ndk/debug/` 目录下会找到生成的各个平台的 so 文件。

### 手动编译

上面的方法容易受到 AS 版本的影响，下面来介绍创建 Android.mk 文件的方法来生成so。
其实在上面的方法中在 build/intermediates/ndk/debug 路径下也有生成 Android.mk 文件。
首先我们在 jni 目录下面创建 Application.mk 文件：

```
APP_ABI := armeabi,armeabi-v7a,arm64-v8a
```

创建 Android.mk：

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := JniDemo
LOCAL_SRC_FILES := Constants.c

include $(BUILD_SHARED_LIBRARY)
```

在 jni 目录下面运行：

```
ndk-build 
```

这样会在 `src/main/libs` 下面生成 so 文件。
然后在 build.gradle 中配置:

```
        sourceSets.main {
            // JNI build
            jniLibs.srcDirs = ['src/main/libs'] //set libs as .so's location instead of jni
            jni.srcDirs = []                    //disable automatic ndk-build call with auto-generated Android.mk file
        }
```

## 加载so

在 JniUtils 类中添加下面代码记载so：

```
public class JniUtils {
    static {
        System.loadLibrary("JniDemo");
    }
    public static native String getString();
}
```

然后就可以通过 `JniUtils.getString()` 

