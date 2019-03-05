---
title: Android实用技巧之adb命令：logcat命令的使用 
categories: Android实用技巧
comments: true
tags: [Android, logcat]
description: 介绍logcat的使用
date: 2014-10-10 10:00:00
---

## 概述

`adb logcat` 是作为一个 Android 开发者最经常接触到的一个命令，我们在程序中打印的各种日志都可以通过它来呈现出来，对于我们查看程序运行情况和解决定位问题非常有帮助。
下面介绍的日志的命令行可以自由组合使用。
比如，输出指定等级日志到文件：
`adb logcat *:S ActivityManager:D KeyguardUpdateMonitor:D -f /sdcard/log.txt`

## 基本用法

`adb logcat`：就可以把 main log 和 system log打印出来，输出完成后阻塞终端，后面输出的log会及时更新到终端。
`adb logcat -d`：输出完日之后退出，不会阻塞。

## 帮助信息

`adb logcat -h` 可以打印用户帮助信息 

## 日志过滤

### 输出指定标签日志

`adb logcat -s ActivityManager`
`adb logcat *:S ActivityManager`

### 过滤指定等级日志

`adb logcat *:D`：只输出Debug等级（包含）以上的日志

### 过滤指定等级指定标签日志

`adb logcat *:S ActivityManager:D`：只输出 ActivityManager 标签的Debug等级（包含）以上的日志

### 过滤多个标签的指定等级日志

`adb logcat *:S ActivityManager:D KeyguardUpdateMonitor:D`：输出 ActivityManager 和 KeyguardUpdateMonitor  标签的Debug等级（包含）以上的日志

### 使用管道过滤日志

`adb logcat |grep ActivityManager`：输出包含指定字符串的行

### 正则表达式匹配

这个可以使用管道自由发挥，属于 linux 命令行知识范畴。

## 指定输出日志数量

`adb logcat -m 8`：输出 8 条日志后退出，只输出缓冲区中最开始的8条记录
`adb logcat -t 8`：输出最近的８条日之后退出。

## 清空日志缓冲区

`adb logcat -c`

## 输出日志到指定文件

`adb logcat -f /sdcard/log.txt`：将日志输出到手机的/sdcard/log.txt，注意是手机上。
`adb logcat > ~/log.txt`：将日志输出到终端所在电脑的 ~/log.txt 文件。

## 指定日志输出格式

主要介绍 `adb logcat -v <format>` 选项的使用。

##　指定缓冲区

`adb logcat -b <system, radio, events, main(default)>`，默认输出main buffer里面的日志
`adb logcat -b events`：输出 event log。

## 查看日志缓冲区

`adb logcat -g`