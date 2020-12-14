---
title: Android JNI -- 传递对象
categories:  Android
comments: true
tags: [JNI]
description: 介绍 JNI 传递对象
date: 2015-12-12 10:00:00
---

## 概述

Jni 开发时有时需要把 Java 对象传递给 C++，或者把对象从 C++ 返回给 Java，本文就这方面的知识做一些介绍。

## C++ 向 Java 传递对象

```
//Paras.java
public class Paras {
    private int vInt;
    private String vString;
    public int pInt;
    
    public void setvInt(int value) {
        vInt = value;
    }

    public int getvInt() {
        return vInt;
    }

    public String getvString() {
        return vString;
    }

    public void setvString(String vString) {
        this.vString = vString;
    }

}
```

```
//JniUtils.java
public class JniUtils {
    static {
        System.loadLibrary("JniDemo");
    }
    ......

    public native Paras testObject();
}
```

```
// testJni.cpp
extern "C"
JNIEXPORT jobject JNICALL Java_com_example_heqiang_testsomething_jni_JniUtils_testObject(JNIEnv *env, jobject obj) {
    LOGD("call testObject sucessfully");

    jclass  jcls = env->FindClass("com/example/heqiang/testsomething/jni/Paras");
    // 创建对象
    jobject jniObj = env->AllocObject(jcls);
    // 获取方法ID
    jmethodID set1 = env->GetMethodID(jcls,"setvInt","(I)V");
    jmethodID set2 = env->GetMethodID(jcls,"setvString","(Ljava/lang/String;)V");
    // 获取字段ID
    jfieldID filed1 = env->GetFieldID(jcls, "pInt", "I");

    // 调用方法
    env->CallVoidMethod(jniObj,set1,100);

    jstring p1 = env->NewStringUTF("Test2");
    env->CallVoidMethod(jniObj,set2,p1);

    // 设置字段
    env->SetIntField(jniObj, filed1, 200);
    
    env->DeleteLocalRef(jcls);

    return jniObj;
}
```

## Java 向 C++ 传递对象

JniUtils 添加 `testObject2(Paras p)` 方法：

```
public class JniUtils {
    static {
        System.loadLibrary("JniDemo");
    }
    ......

    public native void testObject2(Paras p);
}
```

```
// testJni.cpp
extern "C"
JNIEXPORT void  JNICALL Java_com_example_heqiang_testsomething_jni_JniUtils_testObject2(JNIEnv *env, jobject obj, jobject pObj) {
    LOGD("call testObject2 sucessfully");

    jclass jcls = env->GetObjectClass(pObj);
    // 获取方法ID
    jmethodID get1 = env->GetMethodID(jcls,"getvInt","()I");
    jmethodID get2 = env->GetMethodID(jcls,"getvString","()Ljava/lang/String;");
    // 获取字段ID
    jfieldID filed1 = env->GetFieldID(jcls, "pInt", "I");

    // 调用方法
    jint r1 = env->CallIntMethod(pObj,get1);
    jstring r2 = static_cast<jstring>(env->CallObjectMethod(pObj, get2));
    // 获取字段值
    jint r3 = env->GetIntField(pObj,filed1);

    const char *r2c = env->GetStringUTFChars(r2, 0);
    LOGD("call testObject2 sucessfully %d, %s, %d",r1,r2c,r3);
}
```
