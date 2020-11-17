---
title: Kotlin -- 协程
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin 协程的使用
date: 2019-12-22 10:00:00
---

## 概述

学习 Kotlin，就必须提一下协程，它是 Kotlin 中非常重要的一部分，如果你不知道协程，那就说明你还没有学习过  Kotlin。
在 Kotlin 1.1 版本中引入了协程（Coroutines），目前协程还是实验性功能，因此使用前还要在 gradle 里面加上依赖：

```
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:1.3.2"
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.3.2'
```

它是将复杂的异步操作放入底层库中，程序逻辑可以在协程中按照顺序来表达，以此简化异步编程，该底层库将用户代码包装为回调/订阅事件，在不同线程或者不同机器调度执行。

## 协程的使用

### suspend

先来介绍一下 suspend 关键字，用该关键字修饰的方法可以被协程挂起，如果在协程外部，只有在该关键字修饰的方法中才可以使用 `delay` `Deferred.await` 等方法。

### 创建协程

可以通过下面的方法创建一个协程

 - GlobalScope.launch
 - GlobalScope.async
 - runBlocking

#### GlobalScope.launch

我们可以通过 `GlobalScope.launch` 启动一个协程。

```
        GlobalScope.launch {
            Log.e("Test","Coroutine start")
            delay(1000)
            Log.e("Test","Coroutines end")
        }
        Log.e("Test","continue")
```

 `delay` 是一个特殊的挂起函数，它不会造成线程阻塞，但是会挂起协程，并且只能在协程或者 suspend 方法修饰的方法中使用。
 
 ```
 public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {

}
 ```
 
`launch` 方法接受三个参数：

 - context: CoroutineContext：协程的上下文，可以设置协程运行的线程调度器，有 4种模式可以选择：
  - Dispatchers.Default：默认为子线程运行
  - Dispatchers.IO：子线程运行
  - Dispatchers.Main：主线程运行
  - Dispatchers.Unconfined：没指定的情况下就是运行在当前创建协程的线程
 
 ```
        GlobalScope.launch(Dispatchers.Main) {
            Log.e("Test","Coroutine start ${Thread.currentThread()}")
            delay(1000)
            Log.e("Test","Coroutines end")
        }
 ```
 或者我们可以自己创建线程池，通过 `newSingleThreadContext` 或者 `newFixedThreadPoolContext` 来创建。

 - start: CoroutineStart：启动模式，可以有下面四种模式：
  - DEAFAULT：默认模式，也就是创建就启动
  - ATOMIC：
  - UNDISPATCHED：
  - LAZY：懒加载模式，你需要它的时候再调用启动
 
 ```
        var job:Job = GlobalScope.launch( start = CoroutineStart.LAZY) {
            Log.e("Test","Coroutine start ${Thread.currentThread()}")
            delay(1000)
            Log.e("Test","Coroutines end")
        }
        Log.e("Test","continue")
        Thread.sleep(1000);
        job.start()
 ```
 - block：闭包方法体，定义协程内需要执行的操作

launch 方法返回一个 Job 类型的对象，Job 类提供了下面的常用方法：

 - start()：启动协程，除了 lazy 模式之外都不需要手动启动
 - cancel()：取消协程
 - join()：等待协程执行完毕

#### GlobalScope.async

`async` 方法和 `launch` 方法使用类似，唯一的区别就是 async 是有返回值的。

```
        GlobalScope.launch {
            var deffer = GlobalScope.async {
                Log.e("Test","Coroutine start ${Thread.currentThread()}")
                delay(1000)
                Log.e("Test","Coroutines end")
                return@async 200;
            }
            Log.e("Test","continue ")
            Log.e("Test","result = ${deffer.await()} ")
        }
```

```
17:25:19.388 16973 17003 E Test    : continue 
17:25:19.388 16973 17004 E Test    : Coroutine start Thread[DefaultDispatcher-worker-4,5,main]
17:25:20.393 16973 17002 E Test    : Coroutines end
17:25:20.398 16973 17008 E Test    : result = 200
```

async 返回的是 Deferred 类型，Deferred 继承自 Job 接口，增加了一个 await 方法，这个方法只能在协程内部使用。async 的特点是不会阻塞当前线程，但会阻塞所在协程，也就是挂起。

#### runBlocking

runBlocking 方法和 launch 方法的区别是 delay 方法是可以阻塞当前的线程的。

```
        runBlocking {
            Log.e("Test","Coroutine start ${Thread.currentThread()}")
            delay(1000)
            Log.e("Test","Coroutines end")
        }
        Log.e("Test","continue ")
```

```
17:37:01.513 17388 17388 E Test    : Coroutine start Thread[main,5,main]
17:37:02.516 17388 17388 E Test    : Coroutines end
17:37:02.518 17388 17388 E Test    : continue 
```

## 协程的特点

协程主要有以下特点：

 - 协程挂起：协程提供了一种使程序避免阻塞且更廉价可控的操作，叫作协程挂起（coroutines suspension），协程挂起不阻塞线程。
 - 简化代码：协程让原来使用“异步+回调”方式写出来的复杂代码，简化成可以用看似同步的方式表达。
 - 节约资源：唤醒一个线程需要巨大的资源开销。在一个现代化系统上，一个线程非常容易就能吃掉 1M 多的内存。在当前的上下文中，我们可以通过调用协程（根据文档）来作为“轻量级”的线程。通常，一个协程坐落在一个实际的线程池当中，专门用于后台任务的执行操作，这也就是协程为什么如此高效的原因。它只会在需要的时候才会使用系统资源。

协程是一种轻量级线程，主要还是运行在线程中的，因此协程不能代替线程。