---
title: Android 性能优化理论篇 -- 内存泄漏的几种情形
categories: Android性能优化
comments: true
tags: [稳定性]
description: 介绍使用 Android 导致内存泄漏的几种情形
date: 2016-2-28 10:00:00
---

## 集合类

集合类如果仅仅有添加元素的方法，而没有相应的删除机制，导致内存被占用。如果这个集合类是全局性的变量（比如类中的静态属性，全局性的map等即有静态引用或final一直指向它），那么没有相应的删除机制，很可能导致集合所占用的内存只增不减。

## 单例模式

不正确使用单例模式是引起内存泄漏的一个常见问题，单例对象在被初始化后将在JVM的整个生命周期中存在（以静态变量的方式），如果单例对象持有外部对象的引用，那么这个外部对象将不能被JVM正常回收，导致内存泄漏。
如果需要Context，尽量引用Application，而不用Activity。

## Android组件或特殊集合对象的使用

BroadcastReceiver，ContentObserver，FileObserver，Cursor，Callback等在Activity onDestory或者某类生命周期结束之后一定要unregister或者close掉，否则这个Activity类会被system强引用，不会被内存回收。不要直接对Activity进行直接引用作为成员变量，如果不得不这么做，请用private WeakReference mActivity来做，相同的，对于Service等其他有自己生命周期的对象来说，直接引用都需要谨慎考虑是否会存在内存泄漏的可能。

## Handler

### Handler的生命周期与Activity不一致

 1. 由于Handler属于TLS（Thread Local Storage）变量，生命周期和Activity是不一致的。
 2. 当Android应用启动的时候，会先创建一个UI主线程的Looper对象，Looper实现了一个简单的消息队列，一个一个的处理里面的Message对象。主线程Looper对象在整个应用生命周期中存在。
 3. 当在主线程中初始化Handler时，该Handler和Looper的消息队列关联（没有关联会报错的）。发送到消息队列的Message会引用发送该消息的Handler对象，这样系统可以调用 Handler#handleMessage(Message) 来分发处理该消息。
 4. 只要Handler发送的Message尚未被处理，则该Message及发送它的Handler对象将被线程MessageQueue一直持有。因此这种实现方式一般很难保证跟View或者Activity的生命周期保持一致，故很容易导致无法正确释放。

### handler引用Activity阻止了GC对Acivity的回收

 1. 在Java中，非静态(匿名)内部类会默认隐性引用外部类对象。而静态内部类不会引用外部类对象。
 2. 如果外部类是Activity，则会引起Activity泄露。
 3. 当Activity finish后，延时消息会继续存在主线程消息队列中1分钟，然后处理消息。而该消息引用了Activity的Handler对象，然后这个Handler又引用了这个Activity。这些引用对象会保持到该消息被处理完，这样就导致该Activity对象无法被回收，从而导致了上面说的 Activity泄露。

如上所述，Handler的使用要尤为小心，否则将很容易导致内存泄漏的发生。

错误示例：

```
    private final Handler mHandler = new Handler() {  
        @Override  
        public void handleMessage(Message msg) {  
            // ...  
        }  
    }; 
```

解决办法：
使用显形的引用，1.静态内部类。 2. 外部类
使用弱引用 2. WeakReference

正确示例：

```
    private static class MyHandler extends Handler {  
        private final WeakReference<HandlerActivity2> mActivity;  
  
        public MyHandler(HandlerActivity2 activity) {  
            mActivity = new WeakReference<HandlerActivity2>(activity);  
        }  
  
        @Override  
        public void handleMessage(Message msg) {  
            System.out.println(msg);  
            if (mActivity.get() == null) {  
                return;  
            }  
            mActivity.get().todo();  
        }  
    }

    @Override  
    public void onDestroy() {  
        //  If null, all callbacks and messages will be removed.  
        mHandler.removeCallbacksAndMessages(null);  
    }
```

http://blog.csdn.net/zhuanglonghai/article/details/38233069

## ThreadHandler

使用ThreadHandler步骤：
http://www.cnblogs.com/hnrainll/p/3597246.html
创建一个HandlerThread，即创建了一个包含Looper的线程

```
HandlerThread handlerThread = new HandlerThread("test");
handlerThread.start(); //创建HandlerThread后一定要记得start()
```

获取HandlerThread的Looper

```
Looper looper = handlerThread.getLooper();
```

创建Handler，通过Looper初始化

```
Handler handler = new Handler(looper);
```

通过以上三步我们就成功创建HandlerThread。通过handler发送消息，就会在子线程中执行。
如果我们在Activity的onCreate中进行上面的初始化，不再进行其他工作，那么就有可能造成内存泄漏。
因为不过Activity finish后，进程没有退出，那么创建的test会一直存在。

解决办法：
在onDestory中调用handlerThread.quit()或handlerThread.quitSafely()；

## Thread内存泄漏

线程也是造成内存泄漏的一个重要的源头。线程产生的内存泄漏主要原因在于线程生命周期的不可控。比如线程是Activity的内部类，则线程对象中保存了Activity的一个引用，当线程的run函数耗时较长没有结束时，线程对象是不会被销毁的，因此它引用的老的Activity也不会被销毁，因此就出现了内存泄漏的问题。

## 一些不良代码造成的内存压力

这些代码并不造成内存泄漏，但是它们，或是对没使用的内存没有进行有效及时的释放，或是没有有效的利用已有的对象而是频繁的申请新内存。

## Bitmap没调用recycle()

Bitmap对象在不使用时，我们应该先调用recycle()释放内存，然后将它设置为null。因为加载Bitmap对象的内存空间，一部分是java的，一部分C的（因为Bitmap分配的底层是通过JNI调用的）。而这个recycle()就是针对C部分的内存释放。

## 构造Adapter时，没有使用缓存的convertView。

http://blog.csdn.net/u010878994/article/details/51553415