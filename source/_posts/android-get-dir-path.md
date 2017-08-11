---
title: Android 获取目录路径方法汇总
categories: Android
comments: true
tags: [Android]
description: 介绍 Android 中常用的获取目录路径的方法
date: 2015-3-20 10:00:00
---

## Environment中的方法

先来看一下 `Environment` 中的一些方法，主要是获取一些系统的目录：

| 方法名称 | 目录路径 |
|:-------------:|:-------------:|
| getDataAppDirectory() | /data |
| getDataDirectory() | /cache |
| getExternalStorageDirectory() | /storage/emulated/0 |
| getExternalStoragePublicDirectory(Environment.DIRECTORY_MUSIC) | /storage/emulated/0/Music |
| getRootDirectory() | /system |

`getExternalStoragePublicDirectory(String type)` 支持的类型有：

 - DIRECTORY_MUSIC
 - DIRECTORY_PODCASTS
 - DIRECTORY_RINGTONES
 - DIRECTORY_ALARMS
 - DIRECTORY_NOTIFICATIONS
 - DIRECTORY_PICTURES
 - DIRECTORY_MOVIES
 - DIRECTORY_DOWNLOADS
 - DIRECTORY_DCIM
 - DIRECTORY_DOCUMENTS
 - 任何在/storage/emulated/0/的目录


## Context中的方法

主要是获取一些和应用相关的目录：

| 方法名称 | 目录路径 |
|:-------------:|:-------------:|
| getPackageCodePath() | /data/app/com.example.hq.testsomething-1/base.apk |
| getPackageResourcePath() | /data/app/com.example.hq.testsomething-1/base.apk |
| getCacheDir() | /data/user/0/com.example.hq.testsomething/cache |
| getDatabasePath("test") | /data/user/0/com.example.hq.testsomething/databases/test |
| getDir("test", Context.MODE_PRIVATE) | /data/user/0/com.example.hq.testsomething/app_test |
| getExternalCacheDir() | /storage/emulated/0/Android/data/com.example.hq.testsomething/cache |
| getExternalFilesDir(Environment.DIRECTORY_MUSIC) | /storage/emulated/0/Android/data/com.example.hq.testsomething/files/Music |
| getExternalFilesDir(null) | /storage/emulated/0/Android/data/com.example.hq.testsomething/files |
| getFilesDir() | /data/user/0/com.example.hq.testsomething/files |

`getExternalFilesDir(String type)` 的参数类型和前面的 `getExternalStoragePublicDirectory(String type)` 一样。
