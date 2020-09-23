---
title: Android 文件存储以及获取目录路径方法汇总
categories: Android
comments: true
tags: [Android]
description: 介绍 Android 中常用的获取目录路径的方法
date: 2015-3-20 10:00:00
---

## 关于 Android 存储的基础知识

Android 开发中经常会使用 Android 存储的相关知识，首先先介绍下面几个知识点。

### 内部存储和外部存储

Android 中的文件存储可以分为内部存储和外部存储。
内部存储是指这些文件默认是只能被你的应用访问到，且一个应用所创建的所有文件都在和应用包名相同的目录下。也就是说应用创建于内部存储的文件，与这个应用是关联起来的。当一个应用卸载之后，内部存储中的这些文件也被删除。从技术上来讲如果你在创建内部存储文件的时候将文件属性设置成可读，其他app能够访问自己应用的数据，前提是他知道你这个应用的包名，如果一个文件的属性是私有（private），那么即使知道包名其他应用也无法访问。
内部存储一般用 `Context` 来获取和操作。
访问内部存储的API方法：

 - Environment.getDataDirectory() 
 - getFilesDir().getAbsolutePath() 
 - getCacheDir().getAbsolutePath() 
 - getDir(“myFile”, MODE_PRIVATE).getAbsolutePath()

Android 外部存储可以分为两类：一个是机身存储的外部存储，另外一个是扩展的SD卡。他们统称为外部存储。
机身外部存储的路径在 Android4.4 以后是 /storage/emulated/0/，SD卡是 /storage/B3E4-1711/。
访问外部存储的API方法：

 - Environment.getExternalStorageDirectory().getAbsolutePath() 
 - Environment.getExternalStoragePublicDirectory(“”).getAbsolutePath()
 - getExternalFilesDir(“”).getAbsolutePath() 
 - getExternalCacheDir().getAbsolutePath() 


## 获取文件存储路径的方法

### Environment中的方法

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

在Android Q之后，由于对存储权限的收紧，`getExternalStorageDirectory` 已经标记位弃用的方法，此方法返回的路径不再可供应用程序直接访问。通过迁移到`Context＃getExternalFilesDir（String）`，`MediaStore`或`Intent＃ACTION_OPEN_DOCUMENT`之类的替代方案，应用程序可以继续访问共享/外部存储中存储的内容。

### Context中的方法

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
| getObbDir() | /storage/emulated/0/Android/obb/com.example.hq.testsomething |

`getExternalFilesDir(String type)` 的参数可以自定义文件夹，比如 'getExternalFilesDir("test")' 的目录就是 '/storage/emulated/0/Android/data/com.example.hq.testsomething/files/test'

## Android 不同版本存储路径的区别

| 方法名称 | Android 版本 | 目录路径 |
|:-------------:|:-------------:|:-------------:|
| getExternalStorageDirectory() | 4.0 | /mnt/sdcard |
| getExternalStorageDirectory() | 4.1 | /storage/sdcard0 |
| getExternalStorageDirectory() | 4.4 | /storage/emulated/0 |
| getExternalStorageDirectory() | 6.0 | /storage/emulated/0 |
| getFilesDir() | 4.0 | /data/data/com.example.hq.testsomething/files |
| getFilesDir() | 4.1 | /data/data/com.example.hq.testsomething/files |
| getFilesDir() | 4.4 | /data/data/com.example.hq.testsomething/files |
| getFilesDir() | 6.0 | /data/user/0/com.example.hq.testsomething/files |

因此，我们需要获取文件存储路径时，不要用字符串来写死文件路径，这样容易带来Android版本兼容性文件，而是要用 Android 提供的方法来获取。虽然 Android 为了兼容以前的版本，建立了软连接（sdcard/、mnt/sdcard、storage/emulated/0）来指向外部存储。
