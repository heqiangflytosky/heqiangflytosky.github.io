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
相关文档：
[Android Developer:向您的项目添加 C 和 C++ 代码](https://developer.android.com/studio/projects/add-native-code) 
[Android Developer:ndk-build](https://developer.android.com/ndk/guides/ndk-build)
[Android Developer:CMake](https://developer.android.com/ndk/guides/cmake)

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
另外，这个头文件也不是必须的，可以生成，也可以省去。

### JNI 开发

在 src/main 路径下新建一个名为 jni 的文件夹，再将前面生成的头文件放到该目录下面。
然后可以在目前下面创建 native 文件，名字可以随意定。

```
#include "com_example_heqiang_testsomething_util_JniUtils.h"
#include <android/log.h>

#define  LOG_TAG    "TestJNI"
#define  LOGD(...)  __android_log_print(ANDROID_LOG_DEBUG,LOG_TAG,__VA_ARGS__)
#define  LOGE(...)  __android_log_print(ANDROID_LOG_ERROR,LOG_TAG,__VA_ARGS__)

JNIEXPORT jstring JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_getString(JNIEnv *env, jobject obj) {
    LOGD("call jni sucessfully");
    return (*env)->NewStringUTF(env,"abcdefghijklmn");
}
```

这介绍一个自动生成 jni 方法的窍门，光标定位到 Java 的native 方法上面，然后按快捷键，会有 `create funtion XXXX` 选项，选择后会在 c 文件里面自动生成对应的 jni 方法。

## 编译

### AS 自动编译

支持两种方式：

 - ndk-build + Android.mk + Application.mk 
 - CMake + CMakeLists.txt 

#### 使用 ndk-build

首先配置 build.gradle :

```
android {
    ...

    defaultConfig {
	...
        externalNativeBuild {
            ndkBuild {
		// 这配置的 abiFilters 只是指定我们自动生成的so的abiFilters，
		// 和ndk配置的不同,ndk 是配置打包到apk的so类型
                abiFilters "armeabi-v7a"
            }
        }

    }

    externalNativeBuild {
        ndkBuild{
            path "src/main/jni/Android.mk"
        }
    }
```

这种方法会把 cpp 文件添加到当前 Android Studio 中，方便我们阅读jni代码。
触发编译，这个时候可以会报错：

```
Error:Execution failed for task ':app:compileDebugNdk'.
> Error: NDK integration is deprecated in the current plugin.  Consider trying the new experimental plugin.  For details, see http://tools.android.com/tech-docs/new-build-system/gradle-experimental.  Set "$USE_DEPRECATED_NDK=true" in gradle.properties to continue using the current NDK integration.
```

在 gradle.properties 文件中配置：

```
android.useDeprecatedNdk=true
```



#### 使用 cmake

```
// CMakeLists.txt
cmake_minimum_required(VERSION 3.4.1)

# 添加头文件路径，demo中暂时不需要
# include_directories(src/main/cpp/include)

# 添加动态库
add_library( # Specifies the name of the library.
        JniDemo

        # Sets the library as a shared library.
        SHARED

        # Provides a relative path to your source file(s).
        src/main/jni/Constants.c )

# 连接动态库，需要使用 log 库。
target_link_libraries(
        JniDemo
        log
)
```

配置gradle：

```
android {
    ...

    defaultConfig {
        ...

        externalNativeBuild {
            cmake {
                abiFilters  "armeabi-v7a"
            }

        }

    }

    externalNativeBuild {
        cmake{
            path "CMakeLists.txt"
        }
    }
```


在 app/build/intermediates/cmake/ 目录下生成so。

### 手动编译

#### ndk-build

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
LOCAL_LDLIBS    := -llog

LOCAL_MODULE    := JniDemo
LOCAL_SRC_FILES := Constants.c \
	Test.cpp

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
            jni.srcDirs = []                    //disable automatic ndk-build call with auto-generated Android.mk file 禁用通过Gradle来编译本地c/c++代码
        }
```

#### cmake



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

## c 和 c++ 开发的异同

使用 c++ 开发 jni 和使用 c 语言还是有些差别的。

1.文件后缀改成 cpp
2.在 jni.h 文件中有两套代码。一套是支持c的， 一套是支持c++的。

比如，c++ 中 getString 方法要写成：

```
JNIEXPORT jstring JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_getString(JNIEnv *env, jobject obj) {
    LOGD("call jni sucessfully");
    return env->NewStringUTF("abcdefghijklmn");
}
```
3.在方法上面加上 extern "C" 

```
extern "C" 
JNIEXPORT jstring JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_getString(JNIEnv *env, jobject obj) {
    LOGD("call jni sucessfully");
   
```

## native 调用 Java

前面我们讲了 Java 调用 c/c++，接下来讲一下如果用 c/c++ 调用 Java 方法。
值得注意的是，在 native 的主线程和子线程调用 Java 的方法是不一样的。

### 主线程调用

要想通过 native 调用 Java，就需要有个 Java 的对象，获取这个对象也有两个方法：

 - Java 层调用 native 方法时，会有个对象 jobject 参数，这个就是 Java 的当前对象。
 - 通过C/C++创建java对象

先来看第一种方法：

```
// JniUtils.java

public class JniUtils {
    static {
        System.loadLibrary("JniDemo");
    }
    public static native String getString();
    public native void setUp();
    public native void callBack();
}

```

```
//Constants.c

JavaVM* javaVM = NULL;
jobject j_obj = NULL;
jmethodID j_success = NULL;

JNIEXPORT jstring JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_getString(JNIEnv *env, jobject obj) {
    LOGD("call getString sucessfully");
    return (*env)->NewStringUTF(env,"abcdefghijklmn1122");
}

JNIEXPORT void JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_setUp(JNIEnv *env, jobject obj) {
    LOGD("call setUp sucessfully");
    j_obj = (*env)->NewGlobalRef(env,obj);
    jclass  jlz = (*env)->GetObjectClass(env,obj);

    j_success = (*env)->GetMethodID(env,jlz, "onSucess", "(Ljava/lang/String;)V");
}

JNIEXPORT void JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_callBack(JNIEnv *env, jobject obj) {
    LOGD("call callback sucessfully");
    callJava(env, obj);
}

void callJava(JNIEnv *env,jobject obj) {
    jstring para = (*env)->NewStringUTF(env, "Test");
    (*env)->CallVoidMethod(env,obj, j_success, para);
}

```

再来看第二种方法：
创建一个 Java 类：

```
package com.example.heqiang.testsomething.util;

import android.util.Log;

public class JniClass {
    public void onSuccess(String msg) {
        Log.e("Test","msg2 = "+msg);
    }
}
```

```
//Constants.c
JavaVM* javaVM = NULL;

jclass j_jniClass = NULL;
jmethodID j_success2 = NULL;

JNIEXPORT jstring JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_getString(JNIEnv *env, jobject obj) {
    LOGD("call getString sucessfully");
    return (*env)->NewStringUTF(env,"abcdefghijklmn1122");
}

JNIEXPORT void JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_setUp(JNIEnv *env, jobject obj) {
    LOGD("call setUp sucessfully");

    j_jniClass = (*env)->FindClass(env,"com/example/heqiang/testsomething/util/JniClass");
    // findclass 返回的是一个 local reference，函数结束运行时会被销毁，不能赋值给一个全局变量
    // 必须在这里转换一下，否则会有 JNI DETECTED ERROR IN APPLICATION: use of deleted local reference 崩溃
    j_jniClass = (*env)->NewGlobalRef(env,j_jniClass);
    j_success2 = (*env)->GetMethodID(env,j_jniClass,"onSuccess","(Ljava/lang/String;)V");
}

JNIEXPORT void JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_callBack(JNIEnv *env, jobject obj) {
    LOGD("call callback sucessfully");
    callJava(env, obj);
}

void callJava(JNIEnv *env,jobject obj) {
    jstring para2 = (*env)->NewStringUTF(env, "Test2");
    jmethodID init = (*env)->GetMethodID(env,j_jniClass,"<init>","()V");

    // 创建对象，下面两种方式都可以
    jobject jniObj = (*env)->NewObject(env,j_jniClass,init);
    //jobject jniObj = (*env)->AllocObject(env,j_jniClass);

    (*env)->CallVoidMethod(env,jniObj, j_success2, para2);
}
```

### 子线程调用

JavaVM是属于Java进程的，每个进程只有一个JavaVM，而这个JavaVM可以被多线程共享，但是JNIEnv和jobject是属于线程私有的，不能共享。
要在子线程函数里使用AttachCurrentThread()和DetachCurrentThread()这两个函数，在这两个函数之间加入回调java方法所需要的代码。AttachCurrentThread方法用来获取到当前线程中的JNIEnv指针。

```
JavaVM* javaVM = NULL;
//jobject j_obj = NULL;
//jmethodID j_success = NULL;

jclass j_jniClass = NULL;
jmethodID j_success2 = NULL;

pthread_t pt;

//当动态库被加载时这个函数被系统调用
jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    LOGD("JNI_OnLoad");
    jint result = -1;
    //保存全局JVM以便在子线程中使用
    //javaVM = vm;
    JNIEnv* env;

    if ((*vm)->GetEnv(vm,(void**)&env, JNI_VERSION_1_4) != JNI_OK)
    {
        return result;
    }
    return JNI_VERSION_1_4;
}

JNIEXPORT jstring JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_getString(JNIEnv *env, jobject obj) {
    LOGD("call getString sucessfully");
    return (*env)->NewStringUTF(env,"abcdefghijklmn1122");
}

JNIEXPORT void JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_setUp(JNIEnv *env, jobject obj) {
    LOGD("call setUp sucessfully");
    //保存全局JVM以便在子线程中使用
    // 在 JNI_OnLoad 中保存也行
    (*env)->GetJavaVM(env,&javaVM);

    j_jniClass = (*env)->FindClass(env,"com/example/heqiang/testsomething/util/JniClass");
    // findclass 返回的是一个 local reference，函数结束运行时会被销毁，不能赋值给一个全局变量
    // 必须在这里转换一下，否则会有 JNI DETECTED ERROR IN APPLICATION: use of deleted local reference 崩溃
    j_jniClass = (*env)->NewGlobalRef(env,j_jniClass);
    j_success2 = (*env)->GetMethodID(env,j_jniClass,"onSuccess","(Ljava/lang/String;)V");
}

void *thread_fun(void* arg){
    LOGD("call thread_fun sucessfully");
    JNIEnv *jniEnv;
    if((*javaVM)->AttachCurrentThread(javaVM,&jniEnv, 0) != JNI_OK)
    {
        LOGE("%s: AttachCurrentThread() failed", __FUNCTION__);
        return NULL;
    }

    callJava(jniEnv,NULL);

    if((*javaVM)->DetachCurrentThread(javaVM) != JNI_OK)
    {
        LOGE("%s: DetachCurrentThread() failed", __FUNCTION__);
    }
    
    pthread_exit(0);
}

JNIEXPORT void JNICALL Java_com_example_heqiang_testsomething_util_JniUtils_callBack(JNIEnv *env, jobject obj) {
    LOGD("call callback sucessfully");
    //callJava(env, obj);
    pthread_create(&pt, NULL,thread_fun,NULL);
}

void callJava(JNIEnv *env,jobject obj) {
    LOGD("call callJava sucessfully");

    jstring para2 = (*env)->NewStringUTF(env, "Test2");
    jmethodID init = (*env)->GetMethodID(env,j_jniClass,"<init>","()V");

    // 创建对象，下面两种方式都可以
    jobject jniObj = (*env)->NewObject(env,j_jniClass,init);
    //jobject jniObj = (*env)->AllocObject(env,j_jniClass);

    (*env)->CallVoidMethod(env,jniObj, j_success2, para2);
}


```

## 使用 RegisterNatives 注册本地方法

前面在jni中写本地方法时，我们使用了 Java_classpath_className_nativeMethodName 的形式，比如 Java_com_example_heqiang_testsomething_jni_JniUtils_getString，函数名很长，而且当类名变了的时候，函数名必须一个一个的改，挺麻烦的。
实际上 jvm 也同时提供了直接RegisterNative方法手动的注册native方法，下面我们就把上面的 getString 方法的实现方式修改一下。

```
JNIEXPORT jstring JNICALL getString(JNIEnv *env, jobject obj) {
    LOGD("call 11 getString sucessfully");
    return (*env)->NewStringUTF(env,"abcdefghijklmn1122");
}

// 建立jni映射表，将c和java的函数关联起来
const JNINativeMethod methods[]={
        {"getString","()Ljava/lang/String;",(jobject *)getString},
};

//当动态库被加载时这个函数被系统调用
jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    LOGD("JNI_OnLoad");
    jint result = JNI_ERR;
    //javaVM = vm;
    JNIEnv* env;

    if ((*vm)->GetEnv(vm,(void**)&env, JNI_VERSION_1_4) != JNI_OK)
    {
        return result;
    }

    // 获取对应的 Java class
    jclass cls = (*env)->FindClass(env, "com/example/heqiang/testsomething/jni/JniUtils");
    if (cls == NULL) {
        return JNI_ERR;
    }

    //注册映射表
    if((*env)->RegisterNatives(env,cls,methods,1)<0){
        return JNI_ERR;
    }

    return JNI_VERSION_1_4;
}
```

## 小技巧

### 获取签名参数

进入 class 文件所在的目录，然后运行： `javap -s -p JniUtils.class `

```
Compiled from "JniUtils.java"
public class com.example.heqiang.testsomething.util.JniUtils {
  public com.example.heqiang.testsomething.util.JniUtils();
    descriptor: ()V

  public static native java.lang.String getString();
    descriptor: ()Ljava/lang/String;

  public native void setUp();
    descriptor: ()V

  public native void callBack();
    descriptor: ()V

  public void onSucess(java.lang.String);
    descriptor: (Ljava/lang/String;)V

  static {};
    descriptor: ()V
}

```

可以得到每个方法的签名参数。

签名对照表：

| Java 类型 | 符号 |
|:-------------:|:-------------:|
| boolean | Z |
| byte | B |
| char | C |
| short | S |
| int | I |
| long | L |
| float | F |
| double | D |
| void | V |
| object 对象 | LClassName;L类名 |
| Arrays | [array-type[数组类型 |
| methods方法 | (argument-types)return-type(参数类型)返回类型 |

### 类型转换

jstring -> char* ：

```
    jstring r2 = static_cast<jstring>(env->CallObjectMethod(pObj, get2));

    const char *r2c = env->GetStringUTFChars(r2, 0);
    LOGD("call testObject2 sucessfully %s",r2c);
```

### 其他

jni 里面没有 CallStringMethod 方法，我们可以调用 CallObjectMethod 方法，然后对返回值进行强制转换为 jstring 即可。