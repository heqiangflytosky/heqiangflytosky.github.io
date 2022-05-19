---
title: Android -- IconCache
categories:  Android Launcher
comments: true
tags: [Launcher3]
description: 介绍 IconCache
date: 2021-2-6 10:00:00
---


## 概述

IconCache　是一个缓存类，缓存了跟Icon相关的信息，实际它主要缓存了应用的桌面图标　bitmap　和　label，除了memory缓存，还持久化到了数据库　app_icons.db 中。
IconCache 继承自 BaseIconCache，这个类在　`frameworks/libs/systemui/iconloaderlib/src/com/android/launcher3/icons/cache/`　目录下。
IconCache　在　LauncherAppState　中实例化，LauncherAppState　和　AllAppsList　持有对它的引用。

## 相关类

LauncherIcons 一个用于处理图标样式的帮助类，在缓存图标之前，会通过该类进行标准化处理。它继承自　BaseIconFactory，这个类在　`frameworks/libs/systemui/iconloaderlib/src/com/android/launcher3/icons/` 目录下。提供下面的一些方法：

 - createIconBitmap
 - createBadgedIconBitmap
 - createScaledBitmapWithoutShadow

缓存图标的更新时通过　SerializedIconUpdateTask　任务来完成。

IconCache 中有　CachingLogic　的三个实现类的对象，分别是：

 - ComponentCachingLogic
 - ShortcutCachingLogic
 - LauncherActivityCachingLogic

他们代表了不同组件图标的缓存逻辑。

## 图标处理

### application 图标

1.get icon

```
LoaderTask.run
    LoaderTask.loadWorkspace
        LoaderCursor.getAppShortcutInfo
            IconCache.getTitleAndIcon
                BaseIconCache.cacheLocked
                    mCache.put // 放入memory缓存
                    BaseIconCache.getEntryFromDB
                    LauncherActivityCachingLogic.loadIcon // 如果数据库中没有图标
                        IconProvider.getIcon
                        　　IconUpdateUtils.getThemeIconDrawable  // **首先尝试获取主题定制图标(ROM定制)
                            LauncherActivityInfo.getIcon // 默认图标获取，原生方法获取 Activity 图标
                        BaseIconFactory.createBadgedIconBitmap
                        　　BaseIconFactory.normalizeAndWrapToAdaptiveIcon
                        　　BaseIconFactory.createIconBitmap
                BaseIconCache.applyCacheEntry
```

2.set icon

```
BaseLoaderResults.bindWorkspaceItems
    Launcher.bindItems
        Launcher.createShortcut
            BubbleTextView.applyFromWorkspaceItem
                BubbleTextView.applyIconAndLabel
                    BubbleTextView.setIcon
                    BubbleTextView.setText
```

3. save icon

在LauncherModel的加载桌面数据流程里，也会初始化所有图标的信息。在执行完所有的load和bind之后，会进行更新数据库图标操作。在　IconCacheUpdateHandler　扫描到所有应用后，会开启一个线程　SerializedIconUpdateTask　进行更新图标操作，把图标缓存到内存和数据库里。

```
LoaderTask.run
    IconCacheUpdateHandler.updateIcons
        IconCacheUpdateHandler.updateIconsPerUser
            SerializedIconUpdateTask.run
                BaseIconCache.addIconToDBAndMemCache
                    mCache.get // 如果不做替换，就直接从memory缓存中获取
                    LauncherActivityCachingLogic.loadIcon // 如果要做替换，就再次向系统获取图标
                        //内部调用参考get icon部分
                    mCache.put // 再写入memory缓存
                    BaseIconCache.newContentValues
                    BaseIconCache.addIconToDB // 写入数据库
```

### appwidget 图标

get icon

```
LoaderTask.run
    LoaderTask.loadWorkspace
        ITEM_TYPE_CUSTOM_APPWIDGET
        IconCache.getTitleAndIconForApp
            BaseIconCache.getEntryForPackageLocked
                BaseIconCache.getEntryFromDB
                PackageItemInfo.loadIcon // 如果数据库中没有缓存
                BaseIconFactory.createBadgedIconBitmap
                　　BaseIconFactory.normalizeAndWrapToAdaptiveIcon
                　　BaseIconFactory.createIconBitmap
                BaseIconCache.newContentValues 
                BaseIconCache.addIconToDB // 写入数据库
```

## 推荐文章

https://www.pianshen.com/article/2581187899/
https://blog.csdn.net/qinhai1989/article/details/86706877