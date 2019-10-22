---
title: Flutter -- 开发 Gank 客户端
categories: Flutter
comments: true
tags: [Flutter]
description: Flutter 开发的 Gank 客户端
date: 2019-8-6 10:00:00
---

各个版本的Gank客户端：

 - [Android 版本](http://www.heqiangfly.com/2016/11/08/android-app-ganktoutiao/)
 - [快应用版本](http://www.heqiangfly.com/2018/10/01/quick-app-demo-ganktoutiao/)
 - [Flutter 版本](http://www.heqiangfly.com/2019/08/06/flutter-gank-app/)
 - [Vue 版本](http://www.heqiangfly.com/2019/06/02/javascript-vue-gank-web-app/)

## 概述

接触 Flutter，照例，撸一个 Gank 客户端来练习。
每日分享技术干货和妹子图，还有供大家中午休息的休闲视频、美女图片，另外还实现了推荐干货功能。

[数据来源 Gank](http://gank.io/api)
[项目源码 Github](https://github.com/heqiangflytosky/gank_flutter)

## 应用展示

<img src="https://raw.githubusercontent.com/heqiangflytosky/gank_flutter/master/des_img/main.png" width="360" height="744"/> 

<img src="https://raw.githubusercontent.com/heqiangflytosky/gank_flutter/master/des_img/detail.png" width="360" height="744"/>

<img src="https://raw.githubusercontent.com/heqiangflytosky/gank_flutter/master/des_img/mine.png" width="360" height="744"/>

<img src="https://raw.githubusercontent.com/heqiangflytosky/gank_flutter/master/des_img/fav.png" width="360" height="744"/>

<img src="https://raw.githubusercontent.com/heqiangflytosky/gank_flutter/master/des_img/history.png" width="360" height="744"/>

<img src="https://raw.githubusercontent.com/heqiangflytosky/gank_flutter/master/des_img/about.png" width="360" height="744"/>

## 用到的第三方库

| 名称 | 功能 |
| :-------------: |:-------------:|
| dio | 网络请求框架 |
| json_annotation | Json 模板 |
| json_serializable | Json 模板 |
| flutter_webview_plugin | Web 组件 |
| flutter_spinkit | Loading 组件 |
| sqflite | 数据库 |
| flutter_linkify | 展示文本链接组件 |
| url_launcher | 打开给定的url |