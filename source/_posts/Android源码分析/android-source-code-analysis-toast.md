---
title: Android Toast 源码分析
categories: Android源码分析
comments: true
tags: [Android源码分析, Toast]
description: Android Toast 源码分析
date: 2018-10-18 10:00:00
---

## 概述

Toast 以其简单的交互和便捷的使用方式深受大家的喜爱，在使用过程中不知大家是否有这样的思考：

 - 为什么多个 Toast 同时请求显示时，它们会有序地进行显示
 - 为什么应用退出到后台时，未显示的 Toast 仍然会继续显示
 - 如何解决同一个 Toast 多次触发会重复显示多次的问题
 - Toast 可以在子线程中调用吗

带着这些问题，我们从源码里面一探究竟，代码基于 Android O。

## 源码分析

一般我们使用 Toast 都是这样的：

```
Toast.makeText(this, "test", Toast.LENGTH_LONG).show();;
```

我们先来看一下 `Toast.makeText` 方法：
`makeText` 有三个重载的方法：

 - makeText(Context context, CharSequence text, @Duration int duration)：显示内容参数直接传入字符串
 - makeText(Context context, @StringRes int resId, @Duration int duration)：显示内容参数传入资源ID
 - makeText(@NonNull Context context, @Nullable Looper looper,@NonNull CharSequence text, @Duration int duration)：这个方法是Android O添加的方法，可以传入一个 looper 参数，然后在子线程调用 Toast 显示，否则会有 `java.lang.RuntimeException: Can't toast on a thread that has not called Looper.prepare()` 错误。但是这个是一个 hide 方法，可以通过反射调用。

```
    public static Toast makeText(@NonNull Context context, @Nullable Looper looper,
            @NonNull CharSequence text, @Duration int duration) {
        Toast result = new Toast(context, looper);

        LayoutInflater inflate = (LayoutInflater)
                context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
        TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
        tv.setText(text);

        result.mNextView = v;
        result.mDuration = duration;

        return result;
    }
```

这个方法就是新建了一个 Toast 对象，然后为 Toast 设置布局和显示时长等参数。
再来看一下 `show` 方法：

```
    public void show() {
        if (mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }

        INotificationManager service = getService();//调用系统的notification服务
        String pkg = mContext.getOpPackageName();
        TN tn = mTN;//本地binder
        tn.mNextView = mNextView;

        try {
            service.enqueueToast(pkg, tn, mDuration);
        } catch (RemoteException e) {
            // Empty
        }
    }
```

从上面代码可以看出，当调用 show 方法时，会远程调用 NotificationManagerService 的 enqueueToast 方法放到一个队列中，为了方便本地和 NotificationManagerService 进行通讯，传递了一个 TN 对象到 NotificationManagerService。
为什么要通过 NotificationManagerService 来管理呢？其中一个原因是 NotificationManagerService 是系统服务，具有较高的权限，由它来生成的窗口 token 就可以显示在所有应用的上面。二是可以对所有的 Toast 进行统一的管理来控制它们的显示顺序。

```
//NotificationManagerService.java
        @Override
        public void enqueueToast(String pkg, ITransientNotification callback, int duration)
        {
                ...
            if (pkg == null || callback == null) {
                Slog.e(TAG, "Not doing toast. pkg=" + pkg + " callback=" + callback);
                return ;
            }

            final boolean isSystemToast = isCallerSystemOrPhone() || ("android".equals(pkg));
            final boolean isPackageSuspended =
                    isPackageSuspendedForUser(pkg, Binder.getCallingUid());

            if (ENABLE_BLOCKED_TOASTS && !isSystemToast &&
                    (!areNotificationsEnabledForPackage(pkg, Binder.getCallingUid())
                            || isPackageSuspended)) {
                ...
                return;
            }

            synchronized (mToastQueue) {
                int callingPid = Binder.getCallingPid();
                long callingId = Binder.clearCallingIdentity();
                try {
                    ToastRecord record;
                    int index = indexOfToastLocked(pkg, callback);
                    // If it's already in the queue, we update it in place, we don't
                    // move it to the end of the queue.
                    if (index >= 0) {
                        record = mToastQueue.get(index);
                        record.update(duration);
                    } else {
                        // Limit the number of toasts that any given package except the android
                        // package can enqueue.  Prevents DOS attacks and deals with leaks.
                        if (!isSystemToast) {
                            int count = 0;
                            final int N = mToastQueue.size();
                            for (int i=0; i<N; i++) {
                                 final ToastRecord r = mToastQueue.get(i);
                                 if (r.pkg.equals(pkg)) {
                                     count++;
                                     if (count >= MAX_PACKAGE_NOTIFICATIONS) {
                                         // //上限判断
                                         return;
                                     }
                                 }
                            }
                        }

                        Binder token = new Binder();
                        //生成一个Toast窗口，指定窗口类型为 TYPE_TOAST
                        mWindowManagerInternal.addWindowToken(token, TYPE_TOAST, DEFAULT_DISPLAY);
                        record = new ToastRecord(callingPid, pkg, callback, duration, token);
                        mToastQueue.add(record);
                        index = mToastQueue.size() - 1;
                        keepProcessAliveIfNeededLocked(callingPid);
                    }
                    // 如果队列中没有其他Toast，则显示当前Toast
                    if (index == 0) {
                        showNextToastLocked();
                    }
                } finally {
                    Binder.restoreCallingIdentity(callingId);
                }
            }
        }
```

该方法会为每个队列中不存在的 Toast 生成一个 ToastRecord，并生成一个 Toast 窗口。如果该 Toast 已经存在于队列中，那么仅仅更新参数即可。如果队列中没有其他Toast，则立即显示当前Toast。

```
    void showNextToastLocked() {
        ToastRecord record = mToastQueue.get(0);
        while (record != null) {
            try {
                // 远程调用远程 TN 的show 方法
                record.callback.show();
                // 触发超时监听
                scheduleTimeoutLocked(record);
                return;
            } catch (RemoteException e) {
                // remove it from the list and let the process die
                int index = mToastQueue.indexOf(record);
                if (index >= 0) {
                    mToastQueue.remove(index);
                }
                keepProcessAliveLocked(record.pid);
                if (mToastQueue.size() > 0) {
                    record = mToastQueue.get(0);
                } else {
                    record = null;
                }
            }
        }
    }
```

先来看一下本地端 TN 是如何显示 Toast 的。

```
// Toast.java
        @Override
        public void show() {
            if (localLOGV) Log.v(TAG, "SHOW: " + this);
            mHandler.post(mShow);
        }

        final Runnable mShow = new Runnable() {
            @Override
            public void run() {
                handleShow();
            }
        };
        
        public void handleShow(IBinder windowToken) {
            // If a cancel/hide is pending - no need to show - at this point
            // the window token is already invalid and no need to do any work.
            if (mHandler.hasMessages(CANCEL) || mHandler.hasMessages(HIDE)) {
                return;
            }
            if (mView != mNextView) {
                // remove the old view if necessary
                handleHide();
                mView = mNextView;
                Context context = mView.getContext().getApplicationContext();
                String packageName = mView.getContext().getOpPackageName();
                if (context == null) {
                    context = mView.getContext();
                }
                mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
                // We can resolve the Gravity here by using the Locale for getting
                // the layout direction
                final Configuration config = mView.getContext().getResources().getConfiguration();
                final int gravity = Gravity.getAbsoluteGravity(mGravity, config.getLayoutDirection());
                mParams.gravity = gravity;
                if ((gravity & Gravity.HORIZONTAL_GRAVITY_MASK) == Gravity.FILL_HORIZONTAL) {
                    mParams.horizontalWeight = 1.0f;
                }
                if ((gravity & Gravity.VERTICAL_GRAVITY_MASK) == Gravity.FILL_VERTICAL) {
                    mParams.verticalWeight = 1.0f;
                }
                mParams.x = mX;
                mParams.y = mY;
                mParams.verticalMargin = mVerticalMargin;
                mParams.horizontalMargin = mHorizontalMargin;
                mParams.packageName = packageName;
                mParams.hideTimeoutMilliseconds = mDuration ==
                    Toast.LENGTH_LONG ? LONG_DURATION_TIMEOUT : SHORT_DURATION_TIMEOUT;
                mParams.token = windowToken;
                if (mView.getParent() != null) {
                    if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                    mWM.removeView(mView);
                }
                
                try { // Android 8 开始加了 try catch
                    mWM.addView(mView, mParams);
                    trySendAccessibilityEvent();
                } catch (WindowManager.BadTokenException e) {
                    /* ignore */
                }
            }
        }
```

这个方法的主要任务就是通过 WindowManager.addView 把 Toast 窗口加入 WindowManager 进行显示。

那么队列中的 Toast 是怎么按顺序来显示的呢？还记得刚才的超时监听嘛？
这里会按照 Toast 的显示时长参数设置一个超时监听，这个 `scheduleTimeoutLocked` 就是用于管理 Toast 时序的。

```
    private void scheduleTimeoutLocked(ToastRecord r)
    {
        mHandler.removeCallbacksAndMessages(r);
        Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
        long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
        mHandler.sendMessageDelayed(m, delay);
    }
    
    private void handleTimeout(ToastRecord record)
    {
        synchronized (mToastQueue) {
            int index = indexOfToastLocked(record.pkg, record.callback);
            if (index >= 0) {
                cancelToastLocked(index);
            }
        }
    }
    
    void cancelToastLocked(int index) {
        ToastRecord record = mToastQueue.get(index);
        try {
            record.callback.hide();
        } catch (RemoteException e) {
        }

        ToastRecord lastToast = mToastQueue.remove(index);
        mWindowManagerInternal.removeWindowToken(lastToast.token, true, DEFAULT_DISPLAY);

        keepProcessAliveIfNeededLocked(record.pid);
        if (mToastQueue.size() > 0) {
            showNextToastLocked();
        }
    }
```

当超时时间到，也就是前一个 Toast 显示时间到，就会调用远程 TN 的 hide 方法隐藏 Toast，并把 token 从 WMS 中删除，并且调用 showNextToastLocked 方法显示下一个 Toast。

## 总结

 - Toast 的显示是在一个系统窗口上面显示的，窗口的类型是 `public static final int TYPE_TOAST = FIRST_SYSTEM_WINDOW+5;//系统窗口`，而且这个窗口的 token 是由系统内服务 NotificationManagerService 来创建的，具有最高的权限，因此可以显示在所有应用的最上层。这也就是为什么应用退出到后台时，未显示的 Toast 仍然会继续显示的原因。
 - 至于为什么 Toast 会有序地进行显示，是因为 Toast 的管理是由 NotificationManagerService 来进行的，并通过一个 Binder 来控制 Toast 的显示和隐藏。
 - Toast 可以在子线程中调用，要么通过调用`Looper.prepare();Looper.loop();`。或者调用带 Looper 参数的 makeText 方法，该方法在 Android 8添加，是隐藏方法，可以通过反射调用。

## 解决Toast重复显示多次问题

https://blog.csdn.net/justinnick/article/details/78971627
https://blog.csdn.net/qq_33650641/article/details/81185409
https://www.cnblogs.com/laoyimou/p/8108053.html
https://www.jianshu.com/p/923ffe58ec69

## 参考

[Android] Toast问题深度剖析(一)：https://cloud.tencent.com/developer/article/1034225