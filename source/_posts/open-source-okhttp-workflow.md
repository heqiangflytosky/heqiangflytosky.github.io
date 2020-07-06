---
title: OkHttp 源码分析 -- 基本流程
categories: Android开源项目
comments: true
tags: [OkHttp]
description: 介绍 OkHttp 的工作流程
date: 2019-9-15 10:00:00
---

## 概述

从本文开始会通过一系列文章来简单分析 OkHttp 的源码，来掌握其工作原理，本系列文章分析 OkHttp 是基于 3.11.0 版本为基础的，后面有版本更新会尽可能同步更新博客的内容。

## 代码导入

首先从 [GitHub](https://github.com/square/okhttp) 上 clone 源码到本地，因为它是一个 maven 项目，我们首先要将其转为 Android Studio项目，这样才方便通过 Android Studio 打开。
转换的方法：在根目录执行：`gradle init --type pom`。
也可以通过远程依赖的方法直接分析代码。

## OkHttpClient

通过前面的 OkHtttp 的基本使用我们知道，需要先创建一个 OkHttpClient 对象。OkHttpClient 相当于 OkHttp 的客户端，通过它可以设置一些网络配置，比如是否失败重试，连接时间，是否缓存等。每个 OkHttpClient 内部都维护了属于自己的任务队列，连接池，Cache，拦截器等，所以在使用 OkHttp 作为网络框架时应该全局共享一个 OkHttpClient 实例。
创建 OkHttpClient 有两种方法：

```
        mOkHttpClient = new OkHttpClient();
```

和

```
        mOkHttpClient = new OkHttpClient.Builder()
                .retryOnConnectionFailure(true)
                .connectTimeout(10, TimeUnit.SECONDS)
                .cache(cache)
                .build();
```

后面一种方式是使用建造者模式来创建的，可以方便地为 OkHttpClient 添加一些相关配置。
第一种方式内部其实也是通过 `this(new Builder());` 来实现的，网络请求的各个配置参数都是 Builder 里面默认配置的。

## Request

使用 OkHttp 进行网络请求，都要创建一个 Request，它封装了网络请求的一些信息，比如：地址、http方法、请求头等。

## Call

OkHttpClient 的 newCall 方法会根据 Request 信息生成一个 Call 对象。Call 描述一个实际的访问请求，用户的每一个网络请求都是一个Call实例。Call本身只是一个接口，定义了Call的接口方法，实际执行过程中，OkHttp会为每一个请求创建一个RealCall。

## Dispather

Dispather 内部维护了三个任务队列和一个线程池，当有接收到一个 Call 时，Dispatcher负责在线程池中找到空闲的线程并执行其 execute 方法。

## Interceptor

Interceptor（拦截器）是 OkHttp 工作流程的重点，我们专门有一篇文章来介绍。

## 底层实现

