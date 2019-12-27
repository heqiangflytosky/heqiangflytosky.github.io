---
title: Android 性能优化理论篇 -- SharedPreference 使用优化
categories: Android性能优化
comments: true
tags: [SharedPreference]
description: 介绍使用 SharedPreference 时需要注意的问题
date: 2017-2-26 10:00:00
---

## 概述

SharedPreference 是我们在 Android 开发中经常使用的一种轻量级存储方式，但是，如果使用的方法不对，也会带来一些意想不到的结果。本文就介绍一下我们日常使用过程中需要注意的问题。
相关文章：[Android SharedPreferences 源码分析以及跨进程读写问题](http://www.heqiangfly.com/2016/09/12/android-source-code-analysis-sharedpreferences/)

## 谨慎使用多进程

SharedPreferences 提供了 `MODE_MULTI_PROCESS` 模式来支持跨进程读写数据问题，关于使用这种模式需要注意的问题在 [Android SharedPreferences 源码分析以及跨进程读写问题](http://www.heqiangfly.com/2016/09/12/android-source-code-analysis-sharedpreferences/) 一文中做了详细的分析，可以参考一下。
如果实现了跨进程读写，那么我们每次调用 `getSharedPreferences` 时都会重新从把磁盘中保存在xml中的数据重新加载到内存中一次，这就涉及到了IO操作，虽然 `startLoadFromDisk()` 方法中新开启了一个子线程来读取数据，但是我们进行 SharedPreferences 的get和put操作时，都会通过锁的方式来等待`startLoadFromDisk()`操作完成后才进行，如果多进程进行频繁的 SharedPreferences 操作，非常容易造成 ANR。

## 避免存储超大的数据

前面我们说过，SharedPreferences 是一种轻量级的存储方式，它的设计原理在 [Android SharedPreferences 源码分析以及跨进程读写问题](http://www.heqiangfly.com/2016/09/12/android-source-code-analysis-sharedpreferences/) 一文中也介绍过：

 - 它设计了对象缓存，SharedPreferencesImpl 实例对象在一个进程中只有一份，只会实例化一次
 - 而且该类还设计了一个二级缓存，在实例化的时候将磁盘上的数据全部读到内存中
 - 虽然加载xml数据是在子线程中进行，但是通过锁机制控制全部数据读完才能进行正常的读写操作

基于以上的设计，如果我们在 SharedPreferences 里面存储比较大的数据，会带来一下问题：

 - 第一次操作 SharedPreferences 的时候，有可能阻塞主线程，使界面卡顿、掉帧。
 - 这些值会用于存在于内存中，占用大量内存

另外，我们可以采取以下措施来解决第一次操作 SharedPreferences 时，由于需要加载数据造成的等待问题：既然 SharedPreferences 的写入和读取之类的操作会等待 SharedPreferences 加载完成，而加载是在另外一个线程执行的，我们可以让 SharedPreferences 先去加载，做一堆事情，然后再进行写入和读取之类的操作。

```
// 先让sp去另外一个线程加载
SharedPreferences sp = getSharedPreferences("test", MODE_PRIVATE);
// 做一堆别的事情
// OK,这时候估计已经加载完了吧,就算没完,我们在原本应该等待的时间也做了一些事!
String testValue = sp.getString("testKey", null);
```

但是，内存占用问题目前是没有什么好的办法来优化的。

## 避免直接存储JSON或者HTML格式的问题

SharedPreferences 是为存储一些键值对的，像一些网络数据或者配置表尽量不要直接存储在 SharedPreferences 中，这种数据会有大量的特殊字符串，SharedPreferences 在解析碰到这个特殊符号的时候会进行特殊的处理，引发额外的字符串拼接以及函数调用开销。

## 合理使用 apply 和 commit

 - apply 是没有返回值的，commit 有返回值
 - apply 写入文件的操作是异步的，而commit 的写入文件的操作是在当前线程同步执行的

综合性能考虑，如果在主线程操作且不需要返回值的情况下，优先使用 apply 来提交修改。

## 多次edit多次apply

先看下面的代码：

```
SharedPreferences sp = getSharedPreferences("test", MODE_PRIVATE);
sp.edit().putString("test1", "sss").apply();
sp.edit().putString("test2", "sss").apply();
sp.edit().putString("test3", "sss").apply();
sp.edit().putString("test4", "sss").apply();
```

每次edit都会创建一个Editor对象，额外占用内存；当然多创建几个对象也影响不了多少；但是，多次apply也会卡界面你造吗？
有童鞋会说，apply不是在别的线程些磁盘的吗，怎么可能卡界面？我们再来仔细看一下源码。

```
    public void apply() {
        // 写入到内存Map中
        final MemoryCommitResult mcr = commitToMemory();
        // awaitCommit 和 postWriteRunnable 用来做一些线程间的同步操作
        final Runnable awaitCommit = new Runnable() {
                public void run() {
                    try {
                        // 等待写入到本地xml文件结束
                        mcr.writtenToDiskLatch.await();
                    } catch (InterruptedException ignored) {
                    }
                }
            };
        QueuedWork.add(awaitCommit);
        Runnable postWriteRunnable = new Runnable() {
                public void run() {
                    awaitCommit.run();
                    QueuedWork.remove(awaitCommit);
                }
            };
        // 添加到队列写入到本地xml文件
        SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
        // 通知通过 registerOnSharedPreferenceChangeListener 注册的监听器
        notifyListeners(mcr);
    }
    
    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr);
                    }
                    synchronized (SharedPreferencesImpl.this) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };

        final boolean isFromSyncCommit = (postWriteRunnable == null);

        // Typical #commit() path with fewer allocations, doing a write on
        // the current thread.
        if (isFromSyncCommit) {
            boolean wasEmpty = false;
            synchronized (SharedPreferencesImpl.this) {
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) {
                writeToDiskRunnable.run();
                return;
            }
        }

        QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
    }
```

注意两点，第一，把一个带有 await 的 runnable 添加进了 QueueWork 类的一个队列；第二，把这个写入任务通过 enqueueDiskWrite 丢给了一个只有单个线程的线程池执行。
到这里一切都OK，在子线程里面写入不会卡UI。但是，我们再来看一下 `ActivityThread` 类的 `handlePauseActivity ` 和 `handleStopActivity` 方法：

```
@Override
public void handlePauseActivity(IBinder token, boolean show, int configChanges, PendingTransactionActions pendingactions, boolean finalStateRequest, String reason){
    //... 
    if(!r.isPreHoneycomb()){
      //这里检查，异步提交的SharedPreferences任务是否已经完成
      //否则一直等到执行完成
      QueuedWork.waitToFinish();
    }
    //...
 }

private void handleStopActivity(IBinder token, boolean show, int configChanges, int seq) {

    // ...
    // Make sure any pending writes are now committed.
    if (!r.isPreHoneycomb()) {
        QueuedWork.waitToFinish();
    }

    // ...
}

public static void waitToFinish() {
    Runnable toFinish;
    while ((toFinish = sPendingWorkFinishers.poll()) != null) {
        toFinish.run();
    }
}
```

在 Activity pause 和 stop 时会等待所有的任务操作完之后才能继续往下走，包括上面的写入 xml 的操作。如果碰巧在 pause 或者 stop 时有很多的写入 xml 任务在等待执行，那么这里就会出现卡顿。
因此，虽然apply是在子线程执行的，但是请不要无节制地使用。
所以针对上面代码的优雅的做法应该是：

```
        SharedPreferences sp = getSharedPreferences("test", MODE_PRIVATE);
        sp.edit().putString("test1", "sss")
                .putString("test2", "sss")
                .putString("test3", "sss")
                .putString("test4", "sss")
                .apply();
```

## 相关阅读推荐

http://weishu.me/2016/10/13/sharedpreference-advices/
https://www.jianshu.com/p/5fcef7f68341
https://www.jianshu.com/p/f5a29bce2e6f