---
title: Retrofit 2 源码分析 -- 启动篇
categories: Android开源项目
comments: true
tags: [Android开源项目, Retrofit]
description: Retrofit 2 源码的简单介绍
date: 2017-11-6 10:00:00
---

使用过 Retrofit 的同学都知道用 Retrofit 进行网络请求是非常便利的，学会了使用之后，我们一般都按捺不住要去研究源码去一查究竟了。
Retrofit 其实就是对 OKHttp 做了一层封装，使用了一些精妙的设计模式，使用面向接口的方式进行网络请求。

## 下载源码

[GitHub 源码地址](https://github.com/square/retrofit)
[官方文档地址](http://square.github.io/retrofit/)

## 导入源码

`Retrofit` 是 Maven 项目，对于经常使用 Android Studio 的我们还是希望在 AS 中看代码的，这样就需要将 Maven 项目转化成 AS 项目：
其实很简单，只需要在 Maven 根目录下运行：

```
gradle init --type pom
```

然后用 Android Studio 打开根目录即可倒入源码。

参考[链接](http://www.cnblogs.com/softidea/p/5631341.html)

## 源码结构

![效果图](/images/open-source-retrofit-source-code-analysis-start/retrofit-source-code-structure.png)

adapter-* 和 converter-* 这几个模块是官方提供的适配器和转化器，Retrofit 的主要代码在 retrofit 模块中。

![效果图](/images/open-source-retrofit-source-code-analysis-start/retrofit-source-code-classes.png)

可以看到 Retrofit 的源码其实比较简单的，后面我进一步进行分析。


<!-- 
https://www.jianshu.com/p/52f3ca09e2ed
https://www.jianshu.com/p/7148b70f923f
https://www.jianshu.com/p/c40267f8e9b8
-->
