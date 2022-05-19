---
title: Android 消息机制 -- IdleHandler 的用法
categories: Android 消息机制
comments: true
tags: [Android 消息机制]
description: 本文介绍 IdleHandler 的用法
date: 2016-10-12 10:00:00
---

在上一篇博客[Android 消息机制 -- 源码分析](http://www.heqiangfly.com/2016/10/10/android-knowledge-message-system-source-code/)中我们在 `MessageQueue.next()` 中分析了 `IdleHandler`，这是一种在只有当消息队列没有消息时或者是队列中的消息还有到执行时间时才会执行的 `IdleHandler`，它存放在 `mPendingIdleHandlers` 队列中。

## 源码

```
    /**
     * Callback interface for discovering when a thread is going to block
     * waiting for more messages.
     */
    public static interface IdleHandler {
        /**
         * Called when the message queue has run out of messages and will now
         * wait for more.  Return true to keep your idle handler active, false
         * to have it removed.  This may be called if there are still messages
         * pending in the queue, but they are all scheduled to be dispatched
         * after the current time.
         */
        boolean queueIdle();
    }
```

`queueIdle()` 方法如果返回 false，那么这个 `IdleHandler` 只会执行一次，执行完后会从队列中删除，如果返回 true，那么执行完后不会被删除，只要执行 `MessageQueue.next` 时消息队列中没有可执行的消息，即为空闲时间，那么 `IdleHandler` 队列中的 `IdleHandler` 还会继续被执行。

## 示例

```
    public void clickTestIdleHandler(View view) {
        // 添加第一个 IdleHandler
        Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
            @Override
            public boolean queueIdle() {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                Log.e("Test","IdleHandler1 queueIdle");
                return false;
            }
        });
        // 添加第二个 IdleHandler
        Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
            @Override
            public boolean queueIdle() {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                Log.e("Test","IdleHandler2 queueIdle");
                return false;
            }
        });

        Handler handler = new Handler();
        // 添加第一个Message
        Message message1 = Message.obtain(handler, new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                Log.e("Test","message1");
            }
        });
        message1.sendToTarget();

        // 添加第二个Message
        Message message2 = Message.obtain(handler, new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                Log.e("Test","message2");
            }
        });
        message2.sendToTarget();
    }
```

上面的例子中分别向消息队列中添加来两个 `IdleHandler` 和两个 `Message`，其中 `IdleHandler` 的 `queueIdle()` 方法返回 false，下面来看一下结果：

```
16:23:07.726 E/Test: message1
16:23:09.727 E/Test: message2
16:23:11.732 E/Test: IdleHandler1 queueIdle
16:23:13.733 E/Test: IdleHandler2 queueIdle
```

可以看到 `IdleHandler` 是在消息队列的其它消息执行完后才执行的，而且只执行来一次。
再来看一下 `queueIdle()` 方法返回 true 的情况：

```
07-10 16:27:02.083 23560-23560/com.example.heqiang.testsomething E/Test: message1
07-10 16:27:04.084 23560-23560/com.example.heqiang.testsomething E/Test: message2

07-10 16:27:06.090 23560-23560/com.example.heqiang.testsomething E/Test: IdleHandler1 queueIdle
07-10 16:27:08.090 23560-23560/com.example.heqiang.testsomething E/Test: IdleHandler2 queueIdle

07-10 16:27:10.095 23560-23560/com.example.heqiang.testsomething E/Test: IdleHandler1 queueIdle
07-10 16:27:12.096 23560-23560/com.example.heqiang.testsomething E/Test: IdleHandler2 queueIdle

07-10 16:27:14.099 23560-23560/com.example.heqiang.testsomething E/Test: IdleHandler1 queueIdle
07-10 16:27:16.099 23560-23560/com.example.heqiang.testsomething E/Test: IdleHandler2 queueIdle

07-10 16:27:43.788 23560-23560/com.example.heqiang.testsomething E/Test: IdleHandler1 queueIdle
07-10 16:27:45.788 23560-23560/com.example.heqiang.testsomething E/Test: IdleHandler2 queueIdle

07-10 16:27:47.792 23560-23560/com.example.heqiang.testsomething E/Test: IdleHandler1 queueIdle
07-10 16:27:49.793 23560-23560/com.example.heqiang.testsomething E/Test: IdleHandler2 queueIdle
```

可以看到，当 `queueIdle()` 方法返回 true时会多次执行，即 `IdleHandler` 执行一次后不会从 `IdleHandler` 的队列中删除，等下次空闲时间到来时还会继续执行。关于这点在上一篇博客[Android 消息机制 -- 源码分析](http://www.heqiangfly.com/2016/10/10/android-knowledge-message-system-source-code/)中有源码分析。

## 使用场景

比如我们想实现一个 Android 绘制完成的回调方法，Android本身提供的 `Activity` 框架和 `Fragment` 框架并没有提供绘制完成的回调，如果我们自己实现一个框架，就可以使用 `IdleHandler` 来实现一个 `onRenderFinished` 这种回调了。
下面链接中实现的『结合HandlerThread, 用于单线程消息通知器』，这个也挺有意思，可以看一下。

## 参考文章

https://cloud.tencent.com/developer/article/1006147
