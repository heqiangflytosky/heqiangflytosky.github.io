---
title: Android 性能优化工具篇 -- LeakCanary 检测内存泄漏
categories: Android性能优化
comments: true
tags: [内存优化]
description: 介绍使用 LeakCanary 检测内存泄漏的方法
date: 2015-10-9 10:00:00
---

## 概述

在Android中检测内存泄露，除了我们熟知的MAT分析外，还有另外一种方法公选择：LeakCanary。
leakcanary是一个由著名的GitHub开源组织Square贡献的开源项目。
具体可以参考下面文章：
[Github 地址](https://github.com/square/leakcanary)
[LeakCanary 中文使用说明](http://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)
[demo](https://github.com/liaohuqiu/leakcanary-demo)

## 接入方法

使用方法比较简单：
在 build.gradle 中加入

```
 dependencies {
   debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
   releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
 }
```

在 Application 中：

```
import com.squareup.leakcanary.LeakCanary;
public class ExampleApplication extends Application {

  @Override public void onCreate() {
    super.onCreate();
    LeakCanary.install(this);
  }
}
```

这样在debug build 中，如果检测到某个 activity 有内存泄露，LeakCanary 就是自动地显示一个通知。
使用：
使用 RefWatcher 监控那些本该被回收的对象。

```
RefWatcher refWatcher = {...};

// 监控
refWatcher.watch(schrodingerCat);
```

LeakCanary.install() 会返回一个预定义的 RefWatcher，同时也会启用一个 ActivityRefWatcher，用于自动监控调用 Activity.onDestroy() 之后泄露的 activity。

```
public class ExampleApplication extends Application {

  public static RefWatcher getRefWatcher(Context context) {
    ExampleApplication application = (ExampleApplication) context.getApplicationContext();
    return application.refWatcher;
  }

  private RefWatcher refWatcher;

  @Override public void onCreate() {
    super.onCreate();
    refWatcher = LeakCanary.install(this);
  }
}

```

使用 RefWatcher 监控 Fragment：

```
public abstract class BaseFragment extends Fragment {

  @Override public void onDestroy() {
    super.onDestroy();
    RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
    refWatcher.watch(this);
  }
}
```

在这个过程中你有可能需要设置 File -> Settings ->Build,Execution,Deployment -> Build Tools -> Gradle, Then uncheck "Offline work" on the right.

## 注意问题

AndroidExcludedRefs.java忽略的泄漏列表
https://github.com/square/leakcanary/blob/04b0b596dc0119d5056f63186d3f972703e8b507/leakcanary-android/src/main/java/com/squareup/leakcanary/AndroidExcludedRefs.java

Leakcanary部分泄露警报无需修复
http://lynn8570.github.io/posts/some%20memory%20leaks%20that%20no%20need%20to%20be%20fixed/

 1. InputMethodManager.sInstance泄露
 2. AsyncQueryHandler 没有quit
 3. TextLine.sCached 泄露  https://github.com/square/leakcanary/pull/283/files