---
title: Weex 源码研究 -- 序言
categories: Weex
comments: true
tags: [Weex]
description: Weex 的一些简单介绍和源码目录结构
date: 2018-3-2 10:00:00
---

## Weex 简介

一套构建高性能、可扩展的原生应用跨平台开发方案。
Weex V8 引擎现在也已经开源。
[Weex 官网地址](http://weex.apache.org/cn/index.html)
[Weex 源码](https://github.com/apache/incubator-weex/)
[V8 引擎库 Github 地址](https://github.com/alibaba/weex_js_engine)

## 导入源码

```
git clone https://github.com/apache/incubator-weex.git weex
```

### 目录结构

```
├── weex
    ├── android
        ├── sdk
        ├── commons
        └── playground
    ├── ios
    ├── runtime
    ├── packages
    ├── scripts
    ├── pre-build
    ├── bin
    ├── build
    ├── examples
    ├── test
    └── doc

```

打开 Android Studio，导入 android 目录下的源码，发现会有三个 Module：

 - weex_sdk：lib Module，顾名思义，就是 weex 的 Android sdk 源码。
 - commons：lib Module，在 playground Module 中用到的通用工具代码。
 - playground：app Module，Weex 官方的 Demo，展示了一些 Weex 教程、Weex 实例以及一些 Weex 资讯。


现在你就可以尽情地研究 Weex 源码了，后面我的一系列博客也会进行一些 Weex 源码（源码的 Weex SDK Version 是基于0.17.0，JS Framework 是基于 0.23.9， 平台是基于 Android）的分析，欢迎大家关注。




