---
title: EventBus -- 基本用法
categories: Android开源项目
comments: true
tags: [EventBus]
description: 介绍EventBus的用法
date: 2017-12-10 10:00:00
---

## 概述

EventBus 是有 [greenrobot](http://greenrobot.org/) 贡献的一个Android事件发布/订阅轻量级框架。使用它可以方便的实现 Android 端的 publish/subscribe 消息总线，在某些场景来代替 Intent、Handler、BroadCast 等实现在Activity、Fragment或者不同线程间的事件通信。

![效果图](https://raw.githubusercontent.com/greenrobot/EventBus/master/EventBus-Publish-Subscribe.png)

作为一个 Android 开发者，我们平时都会用到或者了解它的用法，本文就将它的用法来做一个总结，已备查用。后面还会写一篇 EventBus 源码分析的文章。

[官方使用文档](http://greenrobot.org/eventbus/documentation/)
[github 源码地址](https://github.com/greenrobot/EventBus)

## EventBus 使用

### 添加依赖

添加 EventBus 依赖：

```
compile 'org.greenrobot:eventbus:3.1.1'
```

### 基本使用

先来了解 EventBus 的三要素：

 - 消息事件
 - 事件订阅者
 - 事件发布者

EventBus的使用很简单，只需要下面几个步骤就可以轻松实现事件的订阅和发布：

 - 声明消息事件对象
 - 需要接受事件的页面注册EventBus
 - 通过EventBus发送事件，相关页面接收事件并处理
 - 页面销毁，反注册EventBus

声明消息事件承载类，消息事件发送后，会根据不同的消息事件对象来寻找合适的接收消息事件的方法：

```
public class Event {
    public String msg;
}
```

在一个 Activity 里面订阅消息事件：

```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ......
        EventBus.getDefault().register(this);

    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        EventBus.getDefault().unregister(this);
        ......
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onReceiveEvent(Event event) {
        Log.e("Test","Receive event = "+event.msg);
    }
```

使用 Subscribe 注解订阅方法，使用 Event 作为参数来接受 Event 类型的消息事件。threadMode 用来指定该方法允许的线程。onReceiveEvent 这个方法的名字可以随意命名。

然后在另外一个 Activity 中发送消息事件：

```
    public void testEvent(View view) {
        Event event = new Event();
        event.msg = "Test Event";
        EventBus.getDefault().post(event);
    }
```

另外注意要配置 proguard：

```
-keepattributes *Annotation*
-keepclassmembers class * {
    @org.greenrobot.eventbus.Subscribe <methods>;
}
-keep enum org.greenrobot.eventbus.ThreadMode { *; }
 
# Only required if you use AsyncExecutor
-keepclassmembers class * extends org.greenrobot.eventbus.util.ThrowableFailureEvent {
    <init>(java.lang.Throwable);
}
```

## 进阶用法

现在来介绍一下 EventBus 的一些高级用法。

### 线程模型 threadMode

在使用 Subscribe 注解订阅方法时，可以加上  threadMode 参数来指定订阅方法运行的线程。
threadMode 有下面几种：

```
public enum ThreadMode {
    POSTING,
    MAIN,
    MAIN_ORDERED,
    BACKGROUND,
    ASYNC
}
```

 - POSTING：接收消息事件的方法和发送消息事件的方法运行在同一个线程。这个也是 threadMode 的默认模式。
 - MAIN：接收消息事件的方法运行在Android主线程。如果发送消息事件方法也运行在主线程，那么会直接调用接收方法。如果发送方法不在主线程，会把消息加到主线程消息队列中。这种模式下不能执行一些耗时的操作。
 - MAIN_ORDERED：接收消息事件的方法也是运行在Android主线程。和 MAIN 不同的时，无论发送方法运行在什么线程，消息事件总会加入到主线程消息队列中，排队执行。
 - BACKGROUND：接收消息事件的方法运行在后台线程。如果发送方不是运行在主线程，那么会直接在发送方线程运行接收方法。如果发送方法运行在主线程，那么就会启动线程池里面的一个后台线程来运行接收方法。
 - ASYNC：接收消息事件的方法运行在后台线程。无法发送方运行在什么线程，都会重新创建线程来运行接收方法。
 

### 粘性事件

一般的事件在发布之后，后来注册的订阅者就无法再收到这些事件，但是粘性事件可以解决这个问题。粘性事件发出之后，EventBus 会把它保存下来，知道有新的同类型的粘性事件发出，才会将就旧的事件覆盖。后面注册的订阅者只要使用 sticky 模式，就依然可以收到以前发出的粘性事件。
事件订阅：

```
    @Subscribe(sticky = true)
    public void onReceiveEvent(Event event) {
        Log.e("Test","event = "+event.msg);
    }
```

事件发送：

```
EventBus.getDefault().postSticky(event);
```

### 高级配置

我们一般获取默认配置的 EventBus 方式是这样的：

```
EventBus.getDefault().register(this);
```

我们也可以自定义一些自己的配置：

```
        EventBus eventBus = EventBus.builder().eventInheritance(true)
                .executorService(executorService)
                .ignoreGeneratedIndex(true)
                .logNoSubscriberMessages(false)
                .sendNoSubscriberEvent(false)
                .build();
```

可以配置的项很多，这里就不一一介绍了。
另外我们还可以在默认配置的基础上添加我们的个性化配置：

```
        EventBus eventBus = EventBus.builder()
                .throwSubscriberException(BuildConfig.DEBUG)
                .installDefaultEventBus();
```

### 订阅事件优先级和取消事件

可以使用 Subscribe 的 priority 参数来制定接收相同事件的优先级。
数值越大，优先级越高，就越先收到消息。

```
    @Subscribe(priority = 1)
    public void onReceiveEvent(Event event) {
        Log.e("Test"," event = "+event.msg);
    }
```

先收到消息的订阅者还可以拦截该消息事件的继续发送。那么低优先级的订阅者就不会收到消息事件了。

```
    @Subscribe(priority = 5)
    public void onReceiveEvent(Event event) {
        Log.e("Test"," event = "+event.msg);
        EventBus.getDefault().cancelEventDelivery(event);
    }
```

### 订阅者索引 Subscriber Index

Subscriber Index是一个可选的优化技术，用来加速初始化订阅者注册。

### AsyncExecutor

AsyncExecutor 就像是一个线程池，它添加了对异常的处理。如果在发送事件过程中出现了异常，AsyncExecutor 会捕获异常并封装在消息事件内，并自动的发布出去。

```
AsyncExecutor.create().execute(
    new AsyncExecutor.RunnableEx() {
        @Override
        public void run() throws LoginException {
            // No need to catch any Exception (here: LoginException)
            remote.login();
            EventBus.getDefault().postSticky(new LoggedInEvent());
        }
    }
);
```

```
@Subscribe(threadMode = ThreadMode.MAIN)
public void handleLoginEvent(LoggedInEvent event) {
    // do something
}
 
@Subscribe(threadMode = ThreadMode.MAIN)
public void handleFailureEvent(ThrowableFailureEvent event) {
    // do something
}
```

通过上面的方式，如果相关代码出现了异常，则会将异常封装成 ThrowableFailureEvent，自动发布出去，只要订阅者定义了接收 ThrowableFailureEvent 的方法，就可以拿到异常信息，后续的消息事件也不会再发布，如果没有出现异常，则正常发布事件。
