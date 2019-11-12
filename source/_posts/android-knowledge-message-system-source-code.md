---
title: Android 消息机制 -- 源码分析
categories: Android 消息机制
comments: true
tags: [Android 消息机制]
description: 本文介绍 Android 消息机制，并分析卡顿监测工具BlockCanary的实现原理
date: 2016-10-10 10:00:00
---

## 概述

本文介绍的消息机制主要指消息在进程内的传递机制，Android 内的消息传递无处不在，主要涉及到下面的四个类：

 - Handler：用于发送消息 `Handler.sendMessage` 和处理消息 `Handler.handleMessage` 的类。
 - Message：用于传输的消息体的类。
 - MessageQueue：消息队列，主要用来存放消息 `MessageQueue.enqueueMessage` 和发出消息 `MessageQueue.next()` 。`MessageQueue` 是一个单向链表，`Message` 对象有个 `next` 变量保存列表中的下一个，`MessageQueue` 中的 `mMessages` 保存链表的第一个元素。
 - Looper：不断循环执行 `Looper.loop`，按分发机制将消息分发给目标处理者。

Handler 是我们在 Android 线程间通信的主要手段。
看一下它们之间的关系图：

![效果图](http://www.plantuml.com/plantuml/svg/VP2zIWD1483xUOeXLKJo1XQnKr1GIISMUtV3ShW_nyvk8OS1kxo3leFOf53mRNA-XcFSFQEYrGpVV3i_E-UeGapMmAADXd1ow9hWsmQ7zMguUnmUdZUhzTlJo-R-TG9G6yMCHyerXWBsiCGJxpj9xMSKS4hCIjDveaHejm7sqLTHjIxNfdj2EiznUf6SKvMC3H-8oJL5oH4jwtzBjsMfGdkn9PS7cW86wipDmiDoN5hErHG5ZBDhPKobwkiVlSeDFSpmM9xc1fTNQCzaczRf7SfVbwFD2SFOiNnJ_yS7YGTPEPZDogwmgtuhvBXbMVebvvc5ZsvnO2vN96lU0G00)

<!-- 
@startuml
Title "Android 消息机制类图"

class Handler {
~ Looper mLooper
~ MessageQueue mQueue
+ obtainMessage()
+ post(Runnable r)
+ sendMessage(Message msg)
+ sendMessage(Message msg)
}

class Message {
+ Messenger replyTo
~ Handler target
~ Runnable callback
}

class MessageQueue {
- IdleHandler[] mPendingIdleHandlers
~ Message mMessages

}
-->

在进行消息处理的代码编写中，我们是直接和 Handler 和 Message 打交道的。

 1. 首先我们需要创建一个 Handler，Handler 内部持有 Looper 和 MessageQueue 对象，我们可以在创建 Handler 时为其指定 Looper，也可以不指定。如果指定了，后面的消息执行就在该 Looper 中执行，如果不指定，就会与Handler 创建时所在的线程的 Looper 指定。`mainHandler = new Handler()` 等价于 `new Handler(Looper.myLooper())`， `Looper.myLooper()` 用来获取当前进程的 Looper 对象。还有一个类似的 `Looper.getMainLooper()` 用于获取主线程的 Looper 对象。
 2. Looper 类别用来为一个线程开启一个消息循环。默认情况下 Android 中新诞生的线程是没有开启消息循环的。（主线程除外，主线程系统会自动为其创建 Looper 对象，开启消息循环）。
 在非主线程中直接 new Handler() 会报如下的错误:`java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()`，原因是非主线程中默认没有创建 Looper 对象，需要先调用 Looper.prepare() 启用 Looper。
 需要先调用 Looper.prepare() 启用 Looper。
 调用 Looper.loop() 让 Looper 开始工作，从消息队列里取消息，处理消息。
 **注意**：写在 Looper.loop() 之后的代码不会被执行，这个函数内部应该是一个循环，当调用 mHandler.getLooper().quit() 后，loop 才会中止，其后的代码才能得以运行。
 Looper 对象通过 MessageQueue 来存放消息和事件。一个线程只能有一个 Looper，对应一个 MessageQueue。
 通常是通过 Handler 对象来与 Looper 交互的。Handler 可看做是 Looper 的一个接口，用来向指定的 Looper 发送消息及定义处理方法。
 3. Handler 和 Looper 准备好以后，就可以准备消息进行发送了，然后Handler接收到消息以后进行消息的响应和处理。可以参考下面的对应章节。
 
分析过了源码，相信大家会对消息机制有更好的领悟，那么使用起来也就更加的得心应手。

## 创建 Handler

`Handler` 一共有7个构造方法：

 - Handler()
 - Handler(Callback callback)
 - Handler(Looper looper)
 - Handler(Looper looper, Callback callback)
 - Handler(boolean async)
 - Handler(Callback callback, boolean async)
 - Handler(Looper looper, Callback callback, boolean async)

下面对这些参数做一下讲解

前面5个构造方法最终调用最后两个构造方法其中一个来完成初始化工作。
下面来分析一下这两个构造方法的源码：

```
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            // 这里如果打开了FIND_POTENTIAL_LEAKS开关
            // 会判断是否是非静态的匿名内部类、非静态成员内部类或非静态局部内部类
            // 如果是的话就给出警告
            // 够体贴了吧？
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        // 这个构造函数没有指定 Looper 对象，就获取当前线程的 Looper 对象
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        // 从Looper对象中获取消息队列
        mQueue = mLooper.mQueue;
        // 设置 handler 的 callback
        mCallback = callback;
        // 异步消息，后面详细介绍
        mAsynchronous = async;
    }
```

从这里我们可以看出，如果没有为 `Handler` 指定 `Looper`，那么就会获取当前创建 `Handler` 的 `Looper` 对象，那么后面对消息的处理也就在这个线程来处理了。

```
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;//设置Looper
        mQueue = looper.mQueue;// 从Looper对象中获取消息队列
        mCallback = callback;// 设置 handler 的 callback
        mAsynchronous = async;// 异步消息，后面详细介绍
    }
```

这个构造方法和前面一个不一样的地方就在于指定了 `Looper` 对象，那么后面对消息的处理也就在该 `Looper` 对应的线程来处理了。

## 创建消息

创建消息一般用下的方法：

 - new Message
 - Handler.obtainMessage()
 - Handler.obtainMessage(int what)
 - Handler.obtainMessage(int what, Object obj)
 - Handler.obtainMessage(int what, int arg1, int arg2)
 - Handler.obtainMessage(int what, int arg1, int arg2, Object obj)
 - Message.obtain()
 - Message.obtain(Message orig)
 - Message.obtain(Handler h)
 - Message.obtain(Handler h, Runnable callback)
 - Message.obtain(Handler h, int what)
 - Message.obtain(Handler h, int what, Object obj)
 - Message.obtain(Handler h, int what, int arg1, int arg2)
 - Message.obtain(Handler h, int what, int arg1, int arg2, Object obj)

`new Message` 这种方式没什么需要解释的，直接调用相关构造函数创建一个 `Message` 实例出来。

```
    Message message = new Message();
    message.what = result;
    message.obj = obj;
    sendMessage(message);
```


再来看一下 `obtainMessage` 方法：
使用方法：

```
    obtainMessage(result, obj).sendToTarget();
```

```
    public final Message obtainMessage(int what, int arg1, int arg2, Object obj)
    {
        return Message.obtain(this, what, arg1, arg2, obj);
    }
```

`obtainMessage` 的几个相关方法都调用了 `Message.obtain` 方法：

```
    public static Message obtain(Handler h, int what, int arg1, int arg2) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;

        return m;
    }

    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

`obtainMessage` 是从整个 `Messge` 池中返回一个新的 `Message` 实例，通过 `obtainMessage` 能避免重复 `Message` 创建对象，比 `new Message` 效率更高。
从上面代码可以看到 `Handler.obtainMessage` 内部还是调用 `Message.obtain` 来实现的，没什么本质的区别。
不管上面以哪种方式创建 Message ，都要为其设定 target，也就是处理该 Message 的Handler，从而与 Handler 发生绑定。

## 发送消息

可以使用下面的方法来发送消息：

 - Handler.sendMessage(Message msg)
 - Handler.sendMessageDelayed(Message msg, long delayMillis)
 - Handler.sendMessageAtTime(Message msg, long uptimeMillis)
 - Handler.sendMessageAtFrontOfQueue(Message msg)
 - Handler.sendEmptyMessage(int what)
 - Handler.sendEmptyMessageDelayed(int what, long delayMillis)
 - Handler.sendEmptyMessageAtTime(int what, long uptimeMillis)
 - Handler.post(Runnable r)
 - Handler.postAtTime(Runnable r, long uptimeMillis)
 - Handler.postAtTime(Runnable r, Object token, long uptimeMillis)
 - Handler.postDelayed(Runnable r, long delayMillis)
 - Handler.postAtFrontOfQueue(Runnable r)
 - Message.sendToTarget()


我们可以把它们归结为三种方法：`sendMessage`、`sendEmptyMessage`、`Handler.post` 和 `Message.sendToTarget()`。
先来看一下 `sendMessage` 相关的方法：

```
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

这里代码比较简单，把 `Message` 对象放到消息队列中去，等待执行。
再来看一下 `sendEmptyMessage` 相关方法：

```
    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }
```

可以看到，`sendEmptyMessage` 最终是调用了 `sendMessage` 相关方法，`Message` 是通过 `Message.obtain()`，只设置了 `what` 属性。
再来看一下 `Handler.post` 相关方法：

```
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
```

这个方法最终调用的还是 `Handler.sendMessage` 的相关方法，知识这里要关注一下 `Handler.getPostMessage` 方法：

```
    private static Message getPostMessage(Runnable r, Object token) {
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r;
        return m;
    }
```

同样也是通过 `Message.obtain()` 获取 `Message` 对象，并把参数 `Runnable` 作为 `Message` 的回调方法，该回调的执行后面会介绍，是优先级最高的。
再来看一下 `Message.sendToTarget()` 的源码：

```
    public void sendToTarget() {
        target.sendMessage(this);
    }
```

其实还是调用 `Handler.sendMessage` 来完成 `Message` 的发送。

## 消息队列 MessageQueue

`MessageQueue` 的实例化实在 Looper 构造方法中进行的。
上面介绍到消息的发送，其实最终都是通过 `MessageQueue.enqueueMessage` 来将消息放入消息队列，然后等待消息的分发。

```
    boolean enqueueMessage(Message msg, long when) {
        // 一个 message 必须有 target（Handler），否则为无效消息
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        // 如果该消息正在使用，则抛出异常
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            //正在退出时，回收msg，加入到消息池
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }
            // 标记该消息正在使用   
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // p 为null（队列中没有消息）或者消息执行时间是最早的，就把该消息作为队列的头部
                // 如果队列再阻塞状态，则唤醒队列
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // 将消息按照执行时间顺序插入到消息队列中去
                // 如果在消息头部是 barrier （p.target == null）或者该消息是执行时间最早的异步消息
                // 那么就唤醒消息队列
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    // 如果不是消息头部的异步消息，不唤醒队列
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

消息的提取：

```
    Message next() {
        // ptr == 0 表明当前队列已经销毁
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            // 调用native方法来阻塞nextPollTimeoutMillis秒的时间
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // 如果当前消息是 SyncBarrier，那么找到下一个异步消息，
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    // 通过前面的一步找到了消息
                    if (now < msg.when) {
                        // 该消息还没有到达执行时间，则设置下一次轮询的超时时长
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 如果到了执行时间，就取出这条消息返回
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // 通过前面的一步没有找到消息，说明消息队列已经空了
                    // 先设置为-1,如果后面的没有可以执行的 IdleHandler，那么就一直阻塞下去
                    // 直到有新的消息被唤醒
                    // 如果后面有 可以执行的 IdleHandler，nextPollTimeoutMillis被赋值为 0
                    nextPollTimeoutMillis = -1;
                }

                // 消息队列正在退出时返回null
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // 如果是第一次循环进入空闲时间，那么 pendingIdleHandlerCount 是为-1的，
                // 这个时候获取一下当前 IdleHandler 的个数
                // 只有当消息队列么有消息时或者是队列中的消息还有到执行时间是才会执行 IdleHandler
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // 如果没有 IdleHandler 需要执行，那么就等待下一个消息到来
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // 当有idle handlers时，运行 idle handlers
            // 执行一次next方法只有第一次轮询能执行这里的操作
            // 执行后 pendingIdleHandlerCount 会被设置为0
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    // 执行 IdleHandler
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    // 如果不需要保留，那么移除这个 IdleHandler
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // 设置 pendingIdleHandlerCount 0 保证在再次执行next方法之前不会再执行idle handler
            pendingIdleHandlerCount = 0;

            // 当调用一个空闲IdleHandler时，相当于分发了一个新message，因此无需等待可以直接查询pending message.
            nextPollTimeoutMillis = 0;
        }
    }
```

关于 `nativePollOnce(ptr, nextPollTimeoutMillis)` 方法，这里做一下介绍。
循环体内首先调用 `nativePollOnce(ptr, nextPollTimeoutMillis)`，这是一个Native方法，实际作用就是通过Native层的 `MessageQueue` 阻塞 `nextPollTimeoutMillis` 毫秒的时间。

 - 如果nextPollTimeoutMillis=-1，一直阻塞不会超时。
 - 如果nextPollTimeoutMillis=0，不会阻塞，立即返回。
 - 如果nextPollTimeoutMillis>0，最长阻塞nextPollTimeoutMillis毫秒(超时)，如果期间有程序唤醒会立即返回。

## 同步消息、异步消息和同步栅栏

我们在构造 `Handler` 时有个参数是给变量 `mAsynchronous` 赋值，那这个变量代表什么意义呢？
在`Handler.enqueueMessage` 中通过 `Message.setAsynchronous` 来设置 `flags`。

```
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        ...
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        ...
    }
```

默认情况下如果我们不去设置 `Message.setAsynchronous` 都是同步消息，那么同步消息和异步消息有什么区别呢？
接下来我们就要看一下哪些地方有调用 `Message.isAsynchronous` 方法。
一个是在 `MessageQueue.enqueueMessage` 方法：

```
    boolean enqueueMessage(Message msg, long when) {
                ......
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                ......
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                ......
    }
```

另外一个是在 `MessageQueue.next` 方法中：

```
    Message next() {
                ......
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                ......
    }
```

可以看到，使用 `Message.isAsynchronous` 方法的地方都和 `msg.target == null` 有关，我们都知道，一般情况下 `Message` 的属性 `target` 是不能为空的，但有一种消息是例外的，就是 SyncBarrier，我们暂且管它叫同步栅栏。
SyncBarrier 是起什么作用的呢？它就像一个卡子，卡在消息链表中的某个位置，当消息循环不断从消息链表中分发消息并进行处理时，一旦遇到这种 SyncBarrier，那么即使在 SyncBarrier 之后还有若干已经到时的同步 `Message`，也不会分发这些消息了。请注意，此时只是不会分发同步 `Message`，如果队列中还有异步 `Message`，那么还是会分发已到时的异步 `Message` 的。
如果消息队列中根本没有设置 SyncBarrier 的话，那么同步 `Message` 和异步 `Message` 的处理就没什么大的不同了。
那么如果设置 SyncBarrier 呢？
可以通过 `MessageQueue.postSyncBarrier` 方法实现：

```
    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```

当然，SyncBarrier 也是可以清除的，通过 `MessageQueue.removeSyncBarrier` 来实现。

## 消息循环 Looper

说到消息循环就和 `Looper` 相关了，在 `Looper` 中创建了一个 `MessageQueue` 对象，并通过 `loop` 方法不断的从当前线程的 `MessageQueue` 中取消息数据，调用该消息的 `Handler` 对象 `target` 的 `dispatchMessage()` 方法来分发消息。

```
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

```
    public static void loop() {
        final Looper me = myLooper();//获取ThreadLocal的Looper对象
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
        // 进入循环
        for (;;) {
            Message msg = queue.next(); // 获取消息队列中的消息，可能会阻塞
            if (msg == null) {
                // 如果没有消息，说明消息队列已经退出
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            // 打印开始分发消息 Log，可以通过 Looper.getMainLooper().setMessageLogging(printer) 为主线程设置 printer，
            // 那么每次主线程处理消息都会触发开始和结束的打印，可以通过这个办法来监测系统的卡顿问题，
            // 著名的卡顿监测框架BlockCanary就是用的这个原理
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }

            try {
                // 分发消息
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            // 打印分发结束的 Log
            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            ......
            //将Message放入消息池
            msg.recycleUnchecked();
        }
    }
```

## 消息分发和执行

消息的分发和执行都是由 Handler 来进行的。
来看一下 `Handler.dispatchMessage` 源码：

```
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

 1. 当 `Message` 的回调方法不为空的时候，优先执行该回调方法来响应消息。这种情况是 `Message` 设置了 callback 时的情况，对应的 `Handler.post(Runnable r)` 这类发送消息的方法，这是就直接执行 post 的 `Runnable` 方法。
 2. 如果没有设置 `Message` 的回调方法，再看来 `Handler` 有没有设置 `Callback`，如果设置就执行该回调的 `handleMessage` 方法。对应的 `Handler(Callback callback)` 构造方法，这是直接执行 `Callback.handleMessage`。
 3. 如果上面两个回调方法都没有设置，那么就执行 `Handler.handleMessage` 方法。该方法默认为空实现，`Handler` 子类通过 override 该方法来完成具体的实现逻辑。

消息队列中的消息都是同步执行的，一个任务执行完才会去消息队列中获取下一个消息进行执行。

## 关于内存泄漏

我们知道，Handler 使用不当会很容易造成内存泄漏，一般原因就是 Handler 的生命周期与 Activity 不一致，handler 引用 Activity 阻止了 GC 对 Acivity 的回收。
比如，Handler 以非静态(匿名)内部类的形式存在于 Activity 中，则该 Handler 会持有外部类 Activity 的引用。
我们知道，Message 一旦创建，就会与target Handler 发送绑定，持有 Handler 的引用，如果该 Message 未被处理，那么就会被 MessageQueue 一直持有，从而导致 Hander 对象也被 MessageQueue 持有，那么 Activity 也会被 MessageQueue 持有，如果 Activity 对象如果此时声明周期结束，则 Activity 由于被 MessageQueue 持有而无法被回收导致内存泄漏。

## 卡顿监测的应用

主线程的所有工作，做种都会回到 MainLooper 中来处理，都会通过 dispatchMessage 来执行，因此如果主线程卡住了，就是在dispatchMessage这里卡住了。
上面代码分析中我们也提到了，在 loop 方法中，dispatchMessage 前后都有 Printer 来调用打印Log的接口，因此我们只要在应用程序中为主线程通过 `Looper.getMainLooper().setMessageLogging(printer)` 设置一个自定义的 Printer，在 Printer 方法中判断start和end，来获取主线程dispatch该message的开始和结束时间，并判定该时间超过阈值(如2000毫秒)为主线程卡慢发生，并dump出各种信息，提供开发者分析性能瓶颈。
著名的卡顿监测框架BlockCanary就是用的这个原理。
