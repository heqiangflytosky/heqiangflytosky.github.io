---
title: Android JetPack
categories: Android 架构
comments: true
tags: [JetPack]
description: 本文介绍 Android JetPack 开发工具套件
date: 2018-9-2 10:00:00
---

相关博客：
[Android Architecture Components -- Room](http://www.heqiangfly.com/2017/12/02/android-architecture-components-room/)
[Android Architecture Components -- LifeCycle](http://www.heqiangfly.com/2018/09/10/android-architecture-components-lifecycle/)
[Android Architecture Components -- LiveData](http://www.heqiangfly.com/2018/09/16/android-architecture-components-livedata/)
[Android Architecture Components -- ViewModel ](http://www.heqiangfly.com/2018/09/20/android-architecture-components-viewmodel/)

## 概述

Google 在 2018 Google I/O 大会上推出了 Android Jetpack，它包含了开发库、工具以及最佳实践指南。
Jetpack 是背包式飞行器的意思，是一种小巧的个人飞行器。顾名思义，通过使用Android  Jetpack 这套组件，可以帮助开发者更高效、更容易地构建优秀的应用。为此，Android 官方团队也把 Jetpack 的Logo 设计成了一个背着火箭的小机器人。
除了在 2018 Google I/O 大会上新添加的功能外，它还把以前退出的一些功能囊括其中，比如 Architecture Components 等。并且，Android 团队将继续在Android平台上为 Jetpack 添加新的功能。
下面来看一下 Jetpack 包含了哪些功能，下图会随着 Jetpack 功能的更新保持同步更新：

![效果图](https://raw.githubusercontent.com/googlesamples/android-sunflower/master/screenshots/jetpack_donut.png)

接下来也会通过一系列的博客来介绍 Jetpack 的使用。
先来看一下 Jetpack 中的架构组件，这些组件关联使用将更能发挥它的作用，我们将采取循序渐进，由浅入深的方式来逐步介绍这些组件的使用。
Google 官方Demo源码：
https://github.com/googlecodelabs/android-lifecycles
https://github.com/googlesamples/android-architecture-components
https://github.com/googlesamples/android-sunflower

## 关于AndroidX

先来个插曲介绍一下 AndroidX，后面 Android 会把 support 库放到 AndroidX 中，仅仅会改变包名和 Maven 库的名字，类的名字不会改变。

## 添加依赖汇总

[官方文档](https://developer.android.google.cn/topic/libraries/architecture/adding-components)

### 添加 Google Maven 仓库

在project下的build.gradle中添加Maven仓库

```
allprojects {
    repositories {
        jcenter()
        google()//添加Google Maven仓库
    }
}
```

### 添加依赖

#### LifeCycle

包含 LiveData 和 ViewModel 的依赖。

##### Android X

```
dependencies {
    def lifecycle_version = "2.0.0"

    // ViewModel and LiveData
    implementation "androidx.lifecycle:lifecycle-extensions:$lifecycle_version"
    // alternatively - just ViewModel
    implementation "androidx.lifecycle:lifecycle-viewmodel:$lifecycle_version" // use -ktx for Kotlin
    // alternatively - just LiveData
    implementation "androidx.lifecycle:lifecycle-livedata:$lifecycle_version"
    // alternatively - Lifecycles only (no ViewModel or LiveData). Some UI
    //     AndroidX libraries use this lightweight import for Lifecycle
    implementation "androidx.lifecycle:lifecycle-runtime:$lifecycle_version"

    annotationProcessor "androidx.lifecycle:lifecycle-compiler:$lifecycle_version" // use kapt for Kotlin
    // alternately - if using Java8, use the following instead of lifecycle-compiler
    implementation "androidx.lifecycle:lifecycle-common-java8:$lifecycle_version"

    // optional - ReactiveStreams support for LiveData
    implementation "androidx.lifecycle:lifecycle-reactivestreams:$lifecycle_version" // use -ktx for Kotlin

    // optional - Test helpers for LiveData
    testImplementation "androidx.arch.core:core-testing:$lifecycle_version"
}

```

##### AndroidX 之前版本

```
dependencies {
    def lifecycle_version = "1.1.1"

    // ViewModel and LiveData
    implementation "android.arch.lifecycle:extensions:$lifecycle_version"
    // alternatively - just ViewModel
    implementation "android.arch.lifecycle:viewmodel:$lifecycle_version" // use -ktx for Kotlin
    // alternatively - just LiveData
    implementation "android.arch.lifecycle:livedata:$lifecycle_version"
    // alternatively - Lifecycles only (no ViewModel or LiveData).
    //     Support library depends on this lightweight import
    implementation "android.arch.lifecycle:runtime:$lifecycle_version"

    annotationProcessor "android.arch.lifecycle:compiler:$lifecycle_version" // use kapt for Kotlin
    // alternately - if using Java8, use the following instead of compiler
    // 比如你想使用 DefaultLifecycleObserver 就要依赖 java8
    implementation "android.arch.lifecycle:common-java8:$lifecycle_version"

    // optional - ReactiveStreams support for LiveData
    implementation "android.arch.lifecycle:reactivestreams:$lifecycle_version"

    // optional - Test helpers for LiveData
    testImplementation "android.arch.core:core-testing:$lifecycle_version"
}

```

#### Room

##### AndroidX

```
dependencies {
    def room_version = "2.1.0-alpha03"

    implementation "androidx.room:room-runtime:$room_version"
    annotationProcessor "androidx.room:room-compiler:$room_version" // use kapt for Kotlin

    // optional - RxJava support for Room
    implementation "androidx.room:room-rxjava2:$room_version"

    // optional - Guava support for Room, including Optional and ListenableFuture
    implementation "androidx.room:room-guava:$room_version"

    // optional - Coroutines support for Room
    implementation "androidx.room:room-coroutines:$room_version"

    // Test helpers
    testImplementation "androidx.room:room-testing:$room_version"
}

```

##### AndroidX 之前版本

```
dependencies {
    def room_version = "1.1.1"

    implementation "android.arch.persistence.room:runtime:$room_version"
    annotationProcessor "android.arch.persistence.room:compiler:$room_version" // use kapt for Kotlin

    // optional - RxJava support for Room
    implementation "android.arch.persistence.room:rxjava2:$room_version"

    // optional - Guava support for Room, including Optional and ListenableFuture
    implementation "android.arch.persistence.room:guava:$room_version"

    // Test helpers
    testImplementation "android.arch.persistence.room:testing:$room_version"
}

```





```
dependencies {
    // ViewModel and LiveData
    implementation "android.arch.lifecycle:extensions:1.1.1"
    // alternatively, just ViewModel
    implementation "android.arch.lifecycle:viewmodel:1.1.1"
    // alternatively, just LiveData
    implementation "android.arch.lifecycle:livedata:1.1.1"
 
    annotationProcessor "android.arch.lifecycle:compiler:1.1.1"
 
    // Room (use 1.1.0-beta3 for latest beta)
    implementation "android.arch.persistence.room:runtime:1.0.0"
    annotationProcessor "android.arch.persistence.room:compiler:1.0.0"
 
    // Paging
    implementation "android.arch.paging:runtime:1.0.0-rc1"
 
    // Test helpers for LiveData
    testImplementation "android.arch.core:core-testing:1.1.1"
 
    // Test helpers for Room
    testImplementation "android.arch.persistence.room:testing:1.0.0"
}

```