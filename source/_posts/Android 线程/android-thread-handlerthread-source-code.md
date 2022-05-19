---
title: Android 线程 -- HandlerThread 使用及源码分析
categories: Android 线程
comments: true
tags: [Android 线程]
description: 本文介绍 HandlerThread 的使用及源码分析
date: 2015-10-10 10:00:00
---

本文是介绍 `HandlerThread` 这个类的使用以及源码分析，单从这个类的名称上看，我们就知道这个类可能会和 `Handler` 和 `Thread` 有关系。
进入正题以前，我们先来看一下 `Handler`、`Thread` 和 `Looper` 的关系。

## Handler、Thread 和 Looper 的关系

 1. `Looper` 类别用来为一个线程开启一个消息循环。默认情况下 Android 中新诞生的线程是没有开启消息循环的。（主线程除外，主线程系统会自动为其创建 `Looper` 对象，开启消息循环）。
`Looper` 对象通过 `MessageQueue` 来存放消息和事件。一个线程只能有一个 `Looper`，对应一个 `MessageQueue`。
通常是通过 `Handler` 对象来与 `Looper` 交互的。`Handler` 可看做是 `Looper` 的一个接口，用来向指定的 `Looper` 发送消息及定义处理方法。
 2. 默认情况下 `Handler` 会与其被定义时所在线程的 `Looper` 绑定，比如，在主线程中定义，其是与主线程的 `Looper` 绑定。
`mainHandler = new Handler()` 等价于 `new Handler(Looper.myLooper())`
`Looper.myLooper()` 用来获取当前进程的 `Looper` 对象。
还有一个类似的 `Looper.getMainLooper()` 用于获取主线程的 `Looper` 对象。
 3. 在非主线程中直接 `new Handler()` 会报如下的错误:

 ```
java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()
 ```

 原因是非主线程中默认没有创建 `Looper` 对象，需要先调用 `Looper.prepare()` 启用 `Looper`。
 4. 需要先调用 `Looper.prepare()` 启用 `Looper`。
调用 `Looper.loop()` 让 `Looper` 开始工作，从消息队列里取消息，处理消息。
注意：写在 `Looper.loop()` 之后的代码不会被执行，这个函数内部应该是一个循环，当调用 `mHandler.getLooper().quit()` 后，loop 才会中止，其后的代码才能得以运行。

## Handler和Thread结合

了解了上面的知识，如果我们想在一个子线程里面处理消息，就需要实现下面的代码：

```
    class LooperThread extends Thread {
        public Handler mHandler;

        public void run() {
            // 初始化Looper
            Looper.prepare();
            // 必须在初始化Looper之后创建Handler，否则会报错
            mHandler = new Handler() {
                public void handleMessage(Message msg) {
                    // 这里处理消息
                }
            };
            // 开启消息循环
            Looper.loop();
        }
    }
```

可以看到实现起来比较麻烦。
那么在这种情况下，`HandlerThread` 应运而生。

## HandlerThread 的用法

`HandlerThread` 是一个包含 `Looper` 的 `Thread`，我们可以直接使用这个 `Looper` 创建 `Handler`。

```
    private HandlerThread mUpdateStatusThread;
    
    {
        // 初始化 HandlerThread
        mUpdateStatusThread = new HandlerThread("update_download_status_thread");
        mUpdateStatusThread.start();
        // 初始化 Handler
        mUpdateStatusHandler = new UpdateStatusHandler(mUpdateStatusThread.getLooper(), this);
    }

    private static class UpdateStatusHandler extends Handler {
        static final int MSG_UPDATE_PROGRESS = 0;
        private WeakReference<DownloadObserver> mWeakReference;

        UpdateStatusHandler(Looper looper) {
            super(looper);
            mWeakReference = new WeakReference<>(downloadObserver);
        }

        @Override
        public void handleMessage(Message msg) {
            ....
            switch (msg.what) {
                case MSG_UPDATE_PROGRESS:
                    ......
                    break;
            }
        }
    }

    {
           // 发送消息
           mUpdateStatusHandler.obtainMessage(UpdateStatusHandler.MSG_UPDATE_PROGRESS, downloadId)
                    .sendToTarget();
    }
    
    {
            // 退出
            mUpdateStatusHandler.removeCallbacksAndMessages(null);
            mUpdateStatusThread.quitSafely();
    }
```

`HandlerThread` 相当于给 `Handler` 提供了一个 `Looper`，那么多个 `Handler` 共用一个 `HandlerThread`，显然也是可以的。
比如我们需要在后台处理多中任务，可能需要建多个 `Handler`（当然在一个 `Handler` 处理也是可行的），那么我们就可以只创建一个 `HandlerThread`，多个 `Handler` 通过 `HandlerThread.getLooper()` 来获取 `Looper` 对象来创建 `Handler`。

```
public class LooperUtils {

    public static HandlerThread mThread;

    private static final String BACKGROUND_THREAD="background_thread";

    static {
        mThread = new HandlerThread(BACKGROUND_THREAD);
        mThread.start();
    }

    public static Looper getThreadLooper(){
        return mThread.getLooper();
    }

    public static Looper getMainThreadLooper(){
        return Looper.getMainLooper();
    }


}
```

```
    mUpdateStatusHandler = new UpdateStatusHandler(LooperUtils.getThreadLooper(), this);
```

## 源码分析

```
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    // 构造方法，设置线程的名称，默认的优先级
    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    // 构造方法，设置线程的名称以及优先级
    // 注意使用的是 android.os.Process 而不是 java.lang.Thread 的优先级
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    // 回调方法
    protected void onLooperPrepared() {
    }
    // 调用Thread start 方法，就会执行 run方法，开启循环
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    
    // 获取当前 Thread 的Looper
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    // 退出 Looper，清空所有的消息队列，包含非延迟消息和延迟消息
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    // 退出 Looper，退出之前会派发所有的非延迟消息。
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    /**
     * Returns the identifier of this thread. See Process.myTid().
     */
    public int getThreadId() {
        return mTid;
    }
}
```
