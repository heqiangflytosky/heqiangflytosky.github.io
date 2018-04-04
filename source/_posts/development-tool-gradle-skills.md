---
title: Gradle 使用指南 -- 一些小技巧
categories: Gradle
comments: true
keywords: Gradle
tags: [Gradle, 开发工具]
description: 介绍在Android开发过程中Gradle的一些常见命令和配置
date: 2016-11-12 10:00:00
---

## 运行 Linux 命令行

方法 1:

```
task A << {
    def content = "ls -al".execute().text.trim()
    print content
}
```

通过调用 `ProcessGroovyMethods` 的 `execute()` 方法来执行命令行。

方法 2:

```
task C (type: Exec) {
    println 'Hello from C'

    workingDir '.'
    commandLine 'ls','-al'
}
```

通过 [Exec Task](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.Exec.html)来实现。
