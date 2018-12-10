---
title: Gradle 使用指南 -- Plugin 断点调试
categories: Gradle
comments: true
keywords: Gradle
tags: [Gradle, 开发工具]
description: 介绍如何在 Gradle Plugin 开发过程中进行断点调试
date: 2018-8-2 10:00:00
---

## 概述

大家都知道 Gradle 插件的开发是基于Groovy，而Groovy 是一种基于 Java 平台的语言。平时我们用 Android Studio 开发 Android 时，可以使用到它强大的调试功能。那么插件开发过程中是不是可以调试呢？答案是可以的。下面就来简单介绍一下。

## 调试方法

### 创建 Remote 调试任务

选择 Edit Configurations ...，点击添加，在弹出的对话框中选择 Remote。

![效果图](/images/development-tool-gradle-plugin-debug/edig-config.png)

![效果图](/images/development-tool-gradle-plugin-debug/create-remote-debug.png)

这时会生成一个 Remote debug 配置，名字可以随意制定，其他按照默认配置就可以，点击 OK 按钮。

![效果图](/images/development-tool-gradle-plugin-debug/remote-debug-config.png)

### 配置调试环境

在终端中输入：

```
export GRADLE_OPTS="-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=5005"
```

### 开始调试

首先在 Groovy 文件中添加断点。
**注意**：.gradle 文件是无法断点调试的。
然后在终端中输入下面命令：

```
./gradlew small  -Dorg.gradle.debug=true
```

这时会打印：

```
Listening for transport dt_socket at address: 5005
```

表示在等待 attach 调试器了。

点击图中 Debug 按钮：

![效果图](/images/development-tool-gradle-plugin-debug/debug-button.png)

这时在 Android Studio 的调试器终端中会打印：

```
Connected to the target VM, address: 'localhost:5005', transport: 'socket'
```

在刚才输入执行 task 任务的终端中打印：

```
Starting a Gradle Daemon, 1 incompatible and 1 stopped Daemons could not be reused, use --status for details

> Starting Daemon
```

这时需要再点击一下 Debug 按钮，这时就可以断点调试了。

![效果图](/images/development-tool-gradle-plugin-debug/debug-result.png)

## 断开调试

如果想断开调试，点击 Debug 面板中的小红叉❌，在弹出对话框中点击 Disconnect 按钮就行了。

![效果图](/images/development-tool-gradle-plugin-debug/debug-cancel.png)

![效果图](/images/development-tool-gradle-plugin-debug/debug-cancel-dialog.png)

## 继续调试

如果想继续调试，首先要点击工具栏中的 Debug 按钮，然后在终端中输入 Gradle 命令：

```
./gradlew small  -Dorg.gradle.debug=true
```

然后再点击 Debug 按钮，才能继续断点调试。
如果不点击 Debug 按钮，直接输入 Gradle 命令，会有下面报错：

```
ERROR: transport error 202: bind failed: Address already in use
ERROR: JDWP Transport dt_socket failed to initialize, TRANSPORT_INIT(510)
JDWP exit error AGENT_ERROR_TRANSPORT_INIT(197): No transports initialized [debugInit.c:750]
```


<!-- 
https://segmentfault.com/a/1190000008266525
https://fucknmb.com/2017/04/07/Intellij-IDEA%E8%BF%9C%E7%A8%8B%E8%B0%83%E8%AF%95/
https://blog.csdn.net/ceabie/article/details/55271161
-->
