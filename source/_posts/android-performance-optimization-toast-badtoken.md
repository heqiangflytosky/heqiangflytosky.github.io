---
title: Android 性能优化实战篇 -- 解决 Toast BadTokenException 异常
categories: Android性能优化
comments: true
tags: [崩溃优化]
description: 解决 Toast BadTokenException 异常
date: 2018-10-15 10:00:00
---

## 概述

Toast 以其简单的交互和便捷的使用深受大家的喜爱，那么我们在使用过程中肯定也经常碰到下面的崩溃异常：

```
android.view.WindowManager$BadTokenException: Unable to add window -- token android.os.BinderProxy@2b6ef40 is not valid; is your activity running?
	at android.view.ViewRootImpl.setView(ViewRootImpl.java:714)
	at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:353)
	at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:94)
	at android.widget.Toast$TN.handleShow(Toast.java:509)
	at android.widget.Toast$TN$2.handleMessage(Toast.java:374)
	at android.os.Handler.dispatchMessage(Handler.java:102)
	at android.os.Looper.loop(Looper.java:154)
	at android.app.ActivityThread.main(ActivityThread.java:6435)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:939)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:829)
```

而且奇怪的是该问题集中出现在 Android 的某些版本上面，基于该问题的分析，需要我们通读一下 Android Toast 的源码，具体请参考我的博客[Toast 源码分析]()。
本文只简单分析该问题的原因和解决方法。

## 问题原因

这个问题目前只在Android 7以及以下的版本中出现，为什么会这样呢？我们从源码中一探究竟。
先来看一下Android 7中 `TN.handleShow` 这个方法：

```
        public void handleShow(IBinder windowToken) {
            if (mView != mNextView) {
                ......
                mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
                ......
                mParams.token = windowToken;
                ......
                mWM.addView(mView, mParams);
                trySendAccessibilityEvent();
            }
        }
```

再来看一下 Android 8 版本中的代码：

```
        public void handleShow(IBinder windowToken) {
            if (mView != mNextView) {
                ......
                mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
                ......
                mParams.token = windowToken;
                ......
                try {
                    mWM.addView(mView, mParams);
                    trySendAccessibilityEvent();
                } catch (WindowManager.BadTokenException e) {
                    /* ignore */
                }
            }
        }
```

看到没有，在 Android 8以及以上的版本中，在 `mWM.addView()` 代码添加了try catch 代码块，把可能抛出的 `WindowManager.BadTokenException` 捕获了，因此，当出现异常时，仅仅是 Toast 不会显示出来，而不会导致应用崩溃。
那么为什么会出现 `WindowManager.BadTokenException` 的异常呢？

我们知道，Toast 显示的时序控制是由 NotificationManagerService 来控制的，在显示 Toast 部分的代码中：

```
    void showNextToastLocked() {
            ...
            try {
                record.callback.show();//通知进程显示
                scheduleTimeoutLocked(record);//超时监听消息
                return;
            } catch (RemoteException e) {
                ....
            }
        }
    }
```

有个超时监听的代码：

```
    private void scheduleTimeoutLocked(ToastRecord r)
    {
        mHandler.removeCallbacksAndMessages(r);
        Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
        long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
        mHandler.sendMessageDelayed(m, delay);
    }
```

当某个 Toast 通知显示进程显示后，就会发一个超时监听的消息，在规定时间到达后就会隐藏 Toast，并且把窗口 token 删除。

```
//NotificationManagerService.java
    void cancelToastLocked(int index) {
        ToastRecord record = mToastQueue.get(index);
        try {
            record.callback.hide();
        } catch (RemoteException e) {
            Slog.w(TAG, "Object died trying to hide notification " + record.callback
                    + " in package " + record.pkg);
            // don't worry about this, we're about to remove it from
            // the list anyway
        }

        ToastRecord lastToast = mToastQueue.remove(index);
        mWindowManagerInternal.removeWindowToken(lastToast.token, true);

        keepProcessAliveIfNeededLocked(record.pid);
        if (mToastQueue.size() > 0) {
            // Show the next one. If the callback fails, this will remove
            // it from the list, so don't assume that the list hasn't changed
            // after this point.
            showNextToastLocked();
        }
    }
```

如果这是碰巧 UI 系统比较卡顿，在 WMS 进程删除 token 后，显示进程才去显示，那么就会抛出 BadTokenException 异常，只不过 Android 8以及以上系统会把这个异常捕获。

## 验证

我们通过下面的代码来验证一下这个问题。

```
        Toast.makeText(this，"test"，Toast.LENGTH_SHORT).show();
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

在 Android 7和以下的版本版本中会出现崩溃，而在 Android 7 以上版本中，Toast 没有显示，但是没有出现崩溃。

## 解决方案

如果我们在 `Toast.show` 调用的地方加个 try catch 模块可以解决这个问题吗？因为 `Toast.show` 仅仅是发了个通知消息给 NotificationManagerService，真正需要显示时是 NotificationManagerService 通知 TN 对象来显示的。因此这个方法是不起作用的。
`TN.mHandler` 对象来处理 NotificationManagerService 发出的消失消息并且调用 `handleShow(token)` 方法来显示。
根据 [Android 消息机制 -- 源码分析 ](http://www.heqiangfly.com/2016/10/10/android-knowledge-message-system-source-code/) 的分析我们知道，`Hanlder` 处理消息的方法 `handleMessage` 也是在 `dispatchMessage` 方法中执行的，因此，我们只需要在 `Hanlder` 方法的 `dispatchMessage` 加上 try catch 块即可。
因此我们定义一个 `Handler` 装饰器，参考[Android  Toast问题深度剖析(二)] (https://cloud.tencent.com/developer/article/1034223) 给出的解决方案。

```
public class ToastUtils {
    private static Field sField_TN ;
    private static Field sField_TN_Handler ;
    static {
        try {
            sField_TN = Toast.class.getDeclaredField("mTN");
            sField_TN.setAccessible(true);
            sField_TN_Handler = sField_TN.getType().getDeclaredField("mHandler");
            sField_TN_Handler.setAccessible(true);
        } catch (Exception e) {}
    }

    private static void hook(Toast toast) {
        try {
            Object tn = sField_TN.get(toast);
            Handler preHandler = (Handler)sField_TN_Handler.get(tn);
            sField_TN_Handler.set(tn,new SafelyHandlerWarpper(preHandler));
        } catch (Exception e) {}
    }

    public static void showToast(Context context, CharSequence cs, int length) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            Toast.makeText(context,cs,length).show();
        } else {
            Toast toast = Toast.makeText(context, cs, length);
            hook(toast);
            toast.show();
        }
    }

    private static class SafelyHandlerWarpper extends Handler {

        private Handler impl;

        public SafelyHandlerWarpper(Handler impl) {
            this.impl = impl;
        }

        @Override
        public void dispatchMessage(Message msg) {
            try {
                super.dispatchMessage(msg);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        @Override
        public void handleMessage(Message msg) {
            impl.handleMessage(msg);//需要委托给原Handler执行
        }
    }
}
```

运行一下测试代码：

```
        ToastUtils.showToast(this,"hello", Toast.LENGTH_LONG);
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {}
```

在 Android 7 版本中也没有出现崩溃。

## 参考文章

[Android] Toast问题深度剖析(一)：https://cloud.tencent.com/developer/article/1034225
[Android] Toast问题深度剖析(二)：https://cloud.tencent.com/developer/article/1034223