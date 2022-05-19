
---
title: Android 架构 android-architecture 简介
categories: Android 架构
comments: true
tags: [Android 架构, android-architecture]
description: 介绍 Google 开源的 Android 架构 android-architecture 中的各个项目
date: 2017-9-5 10:00:00
---
## 概述

开发框架对于每个 Android 开发者来说也是必须要面对的话题，从 MVC 到 MVP，再到 MVVM。并不能说哪种框架说最好的，他们各有自己的特点，那么我们就需要从自己的项目实际出发来选择合适的架构。
Google 也给出了 Android 架构的示例，即 android-architecture。目前已经开源在 Github 上面供开发者学习使用。
[GitHub 地址(旧)](https://github.com/googlesamples/android-architecture/) = [GitHub 地址](https://github.com/android/architecture-samples)
本文作为 android-architecture 开篇之作来简单介绍一下这个项目。
## 源码简介

`git clone` 之后，用 `git branch -a` 来查看分支，每个分支代表一个简单的架构示例。

```
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/deprecated-todo-databinding
  remotes/origin/deprecated-todo-mvp-contentproviders
  remotes/origin/deprecated-todo-mvp-loaders
  remotes/origin/dev-todo-mvp-clean-fix-memory-leak
  remotes/origin/dev-todo-mvp-kotlin
  remotes/origin/dev-todo-mvp-tablet
  remotes/origin/dev-todo-mvvm-live
  remotes/origin/dev-todo-mvvm-live-kotlin
  remotes/origin/dev-todo-mvvm-rxjava
  remotes/origin/master
  remotes/origin/revert-375-feature/mikeBreaksThenHeFixes
  remotes/origin/todo-mvp
  remotes/origin/todo-mvp-clean
  remotes/origin/todo-mvp-dagger
  remotes/origin/todo-mvp-rxjava
  remotes/origin/todo-mvvm-databinding
  remotes/origin/todo-mvvm-live

```
`deprecated` 表示废弃的项目，`dev` 表示仍在完善中的项目。
