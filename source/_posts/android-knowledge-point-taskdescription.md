---
title: Android -- 使用 TaskDescription 定制任务管理器 Title
categories: Android
comments: true
tags: [Android, TaskDescription]
description: 介绍使用 TaskDescription 定制任务管理器 Title
date: 2017-10-8 10:00:00
---

## 概述

最近有个需求，就是要代码动态修改 APP 在任务管理器中名称显示，不要问我问什么有这样的需求，因为需求就这样。哈哈……
对于如何控制 APP 在任务管理器中的 Title，我们可能知道，可以通过在 AndroidManifest.xml 中设置 `application` 或者主 `activity` 的 `android:label` 来实现，而且 `activity` 的优先级高于 `application`，也就是说两者都设置这个标签的话，主 `activity` 的值覆盖 `application`,在桌面上的 APP 名称和 `activity` 的 `title` 的名称都是 `activity` 的 `label` 值。
但是 `label` 的值在代码中是无法进行动态设置的，而且 `ActivityInfo` 的生成是在 AMS 进程进行的，想要修改也不太容易，后面甚至想到了用 HOOK 技术 HOOK PMS 以及 AMS 相关 API 的方法。
由于一直想当然的认为任务管理器中也是读取的是 `ActivityInfo` 的 `labelRes` 或者 `nonLocalizedLabel` 来实现的，因此就一直在修改 `android:label` 上想办法。
这里再来个插曲介绍一下 `nonLocalizedLabel`。

```java
public class PackageItemInfo {
    ......
    public int labelRes;
    
    public CharSequence nonLocalizedLabel;
    ......
}
```

它是怎么赋值的呢？

```java
            TypedValue v = args.sa.peekValue(args.labelRes);
            if (v != null && (outInfo.labelRes=v.resourceId) == 0) {
                outInfo.nonLocalizedLabel = v.coerceToString();
            }
```

看明白了吗？
这里的 `labelRes` 就代表 `android:label` 的资源ID，如果资源ID为0,那么表示不是通过给资源ID的方式来赋值的，可能就直接给 `android:label` 了一个字符串。
类似这样的：

```java
            android:label="Test"
```

## 使用TaskDescription

后面通过查看任务管理器的源码发现，里面用到了一个 `TaskDescription`，通过获取 `TaskDescription` 来获得 Task 的 Title。
`TaskDescription` 是 Android 5.0 加入的一个类，通过它可以设置或者获取任务列表里面的 `Activity` 信息。
`Activity` 提供了 `setTaskDescription()` 方法，其需要 `TaskDescription` 实例，而 `TaskDescription` 提供了多个构造器，注意 `color` 传入必须是非透明。
在 `Activity` 里面使用下面代码解决了该需求问题：

```java
            String title = ...;
            Bitmap icon = ...;
            setTaskDescription(new ActivityManager.TaskDescription(
                    title, icon, getResources().getColor(R.color.colorPrimary)));
```
