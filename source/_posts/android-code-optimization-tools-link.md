---
title: Android 代码优化工具篇 -- 静态代码检测工具
categories: Android
comments: true
tags: [Lint]
description: 介绍使用 Lint 工具检测代码规范的使用方法
date: 2018-8-18 10:00:00
---

## 概述

Android 代码检测是我们开发工程中必不可少的步骤，高质量的代码检测可以提前发现问题，有效地避免线上bug的出现，是质量保证的关键。
Lint 是Android提供的代码检测工具，除了检查 Android 项目源码中潜在的错误，对于代码的正确性、安全性、性能、易用性、便利性和国际化方面也会作出检查。Lint 还可自定义一些检测约束规则。Lint 除了可以使用命令行外，已经集成Android Studio中使用方便。
另外pmd和阿里开源的代码检测工具p3c的使用也较普遍，我们可以把以上工具结合运用到我们的项目中去。
Lint 侧重进行 Android 项目规范检测，pmd和p3c侧重于Java规范检测。
对于Android和Java 的开发规范，建议大家看看阿里开源的《阿里巴巴Java开发手册》和《阿里巴巴Android开发手册》。

[Lint Android官方文档](https://developer.android.com/studio/write/lint?hl=zh-CN)

## Lint 命令行

### 配置

```
  ...
  android {
      lintOptions {
          // 遇到lint错误时不中断
          abortOnError false
          // 配置lint规则，拷贝配置文件lint.xml文件到build.gradle所在目录
          lintConfig file('lint.xml')
      }
  }
  ...
```

./gradlew lintRelease

运行结果会生成 html 和 xml 格式的报告。
相关的任务：

 - lint - Runs lint on all variants.
 - lintDebug - Runs lint on the Debug build.
 - lintRelease - Runs lint on the Release build.
 - lintVitalRelease - Runs lint on just the fatal issues in the release build.

如果遇到下面错误：

```
> Task :app:lintRelease FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:lintRelease'.
> Could not resolve all files for configuration ':app:lintClassPath'.
   > Could not resolve com.android.tools.lint:lint-gradle:26.3.2.
     Required by:
         project :app
      > Could not resolve com.android.tools.lint:lint-gradle:26.3.2.
         > Could not get resource 'https://maven.google.com/com/android/tools/lint/lint-gradle/26.3.2/lint-gradle-26.3.2.pom'.
            > Could not GET 'https://maven.google.com/com/android/tools/lint/lint-gradle/26.3.2/lint-gradle-26.3.2.pom'.
               > Connect to maven.google.com:443 [maven.google.com/172.217.25.3, maven.google.com/2404:6800:4005:809:0:0:0:2003] failed: No route to host (connect failed)

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 34s
```

在 build.gradle 中添加：

```
repositories {
    ...
    google()
}
```

## 使用Android Studio检测

菜单栏选择 Analyze > Inspect Code，在弹出的对话框中进行配置：

<img src="/images/android-code-optimization-tools-link/dialog.png" width="505" height="271"/>

点击 OK 按钮后开始进行Lint检查，运行完成后显示运行结果：

<img src="/images/android-code-optimization-tools-link/result.png" width="725" height="401"/>

检测结果分为几类，可以根据提示进行修复。

## p3c

## pmd