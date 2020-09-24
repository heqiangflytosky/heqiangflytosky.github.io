---
title: Android 性能优化实战篇 -- 解决 'int android.view.View.mPrivateFlags' NullPointerException
categories: Android性能优化
comments: true
tags: [崩溃优化]
description: 解决 'int android.view.View.mPrivateFlags' NullPointerException 异常
date: 2018-10-22 10:00:00
---

## 概述

最近发现线上会报下面的 Crash：

```
E AndroidRuntime: FATAL EXCEPTION: main
E AndroidRuntime: Process: com.*.*.*, PID: 9363
E AndroidRuntime: java.lang.NullPointerException: Attempt to read from field 'int android.view.View.mPrivateFlags' on a null object reference
E AndroidRuntime: 	at android.view.ViewGroup.resetCancelNextUpFlag(ViewGroup.java:2414)
E AndroidRuntime: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2319)
E AndroidRuntime: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2632)
E AndroidRuntime: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2321)
E AndroidRuntime: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2632)
E AndroidRuntime: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2321)
E AndroidRuntime: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2632)
E AndroidRuntime: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2321)
E AndroidRuntime: 	at android.view.View.dispatchPointerEvent(View.java:10317)
E AndroidRuntime: 	at android.view.ViewRootImpl$ViewPostImeInputStage.processPointerEvent(ViewRootImpl.java:4685)
E AndroidRuntime: 	at android.view.ViewRootImpl$ViewPostImeInputStage.onProcess(ViewRootImpl.java:4519)
E AndroidRuntime: 	at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4027)
E AndroidRuntime: 	at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:4080)
E AndroidRuntime: 	at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:4046)
E AndroidRuntime: 	at android.view.ViewRootImpl$AsyncInputStage.forward(ViewRootImpl.java:4173)
E AndroidRuntime: 	at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:4054)
E AndroidRuntime: 	at android.view.ViewRootImpl$AsyncInputStage.apply(ViewRootImpl.java:4230)
E AndroidRuntime: 	at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4027)
E AndroidRuntime: 	at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:4080)
E AndroidRuntime: 	at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:4046)
E AndroidRuntime: 	at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:4054)
E AndroidRuntime: 	at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4027)
E AndroidRuntime: 	at android.view.ViewRootImpl.deliverInputEvent(ViewRootImpl.java:6520)
E AndroidRuntime: 	at android.view.ViewRootImpl.doProcessInputEvents(ViewRootImpl.java:6494)
E AndroidRuntime: 	at android.view.ViewRootImpl.enqueueInputEvent(ViewRootImpl.java:6455)
E AndroidRuntime: 	at android.view.ViewRootImpl$WindowInputEventReceiver.onInputEvent(ViewRootImpl.java:6632)
E AndroidRuntime: 	at android.view.InputEventReceiver.dispatchInputEvent(InputEventReceiver.java:191)
E AndroidRuntime: 	at android.os.MessageQueue.nativePollOnce(Native Method)
E AndroidRuntime: 	at android.os.MessageQueue.next(MessageQueue.java:323)
E AndroidRuntime: 	at android.os.Looper.loop(Looper.java:136)
E AndroidRuntime: 	at android.app.ActivityThread.main(ActivityThread.java:6435)
E AndroidRuntime: 	at java.lang.reflect.Method.invoke(Native Method)
E AndroidRuntime: 	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:939)
E AndroidRuntime: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:829)
```

但是在发布测试的时候没有测出来这样的问题，对于这样的偶现问题，我的一贯认为是：偶现bug一定有其必现的路径，为了找到问题的原因，对于上面的问题，就有必要从源码来分析了。
其实对于类似的问题，以前在实现动画过程中如果遇到 `View.mViewFlags on null object refrence` 的问题，后面发现是在 `onAnimationEnd`里面做了一些View操作，比如 remove view，导致的，这个问题应该比较类似。

## 源码分析

这个问题在 Android 7 机型上报的比较多，那么就从 Android 7 的源码入手。
Crash 发生的 `resetCancelNextUpFlag` 如下：

```
    private static boolean resetCancelNextUpFlag(@NonNull View view) {
        if ((view.mPrivateFlags & PFLAG_CANCEL_NEXT_UP_EVENT) != 0) {
            view.mPrivateFlags &= ~PFLAG_CANCEL_NEXT_UP_EVENT;
            return true;
        }
        return false;
    }
```

很明显，发生崩溃的原因是传入的 view 参数为 null 导致的。
虽然这里有调用的堆栈和对应的行数，但是我这里的源码和崩溃机型的定制过的源码肯定是不一样的，所以要先从 `resetCancelNextUpFlag` 入手找出实际的调用堆栈。
`resetCancelNextUpFlag` 在 `dispatchTouchEvent` 的调用有三处，我们逐一来分析一下。
第一处：

```
            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;
```

这里传入的是 this，是肯定不为空的。
第二处是

```
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            ......
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            ......

                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            ......

                            resetCancelNextUpFlag(child);
```

如果这里的 child 为空的话，在执行 `canViewReceivePointerEvents` 方法时早就奔溃了，因此也不是这里的问题。
那么就是下面的第三处调用了：

```
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
```

`target.child` 为 null，造成了崩溃。


## 原因分析

在 ContainerView 中有两个 RecycleView，在每个 RecycleView 的 `onItemClick` 方法中写入了对该 ContainerView 的 `mWindowManager.removeViewImmediate(mContainerView)` 操作。
当长按其中一个 RecycleView 的 ItemView 时，点击另外一个 RecycleView 的 ItemView，就必现这个崩溃。
为什么会出现崩溃呢？
在前面的博客中我们也介绍过，在`ViewGroup#removeView` 移除某个子 `View` 时，会调用 `ViewGroup#cancelTouchTarget` 这里面会调用 TouchTarget 的 `recycle()` 方法把 view 设置为 null。

再看一下我们 `removeView` 的时机，

<img src="/images/android-performance-optimization-privateflags-crash/1.png" width="479" height="568"/>

具体原因解析请参考 https://www.jianshu.com/p/ab1fb00fcb90

## 解决办法

不要在事件分发时进行 `removeView` 操作，把这些 `removeView` 的操作都放 Handler 中执行，在下一次的 Loop 循环中处理 `removeView` 操作，这样就可以计算出正确的事件分发逻辑，可以避免这个问题。
或者是把上面的 `mWindowManager.removeViewImmediate(mContainerView)`  改成 `mWindowManager.removeView(mContainerView)` 应该也能避免这个问题，因为 `mWindowManager.removeView(mContainerView)` 也是通过 Handler 在下一次消息循环执行的 removeView 操作。

```
        recyclerView.setOnItemClickListener(new RecyclerView.OnItemClickListener() {
            @Override
            public void onItemClick(RecyclerView parent, View view, int position, long id) {
                hidePanelView();
            }
        });
        
    public void hidePanelView() {
        mHandler.removeMessages(PanelViewHandler.MESSAGE_HIDE);
        mHandler.sendEmptyMessage(PanelViewHandler.MESSAGE_HIDE);
    }
    
    public void handleHidePanelView() {
        if (isPanelShowing()) {
            mWindowManager.removeViewImmediate(mContainerView);
        }
    }
    
    private static class PanelViewHandler extends Handler{
        public final static int MESSAGE_HIDE = 1;
        private WeakReference<PanelViewController> mRef;
        public PanelViewHandler(PanelViewController controller) {
            mRef = new WeakReference<>(controller);
        }

        @Override
        public void handleMessage(Message msg) {
            if (mRef == null || mRef.get() == null) {
                return;
            }
            switch (msg.what) {
                case MESSAGE_HIDE:
                    mRef.get().handleHidePanelView();
                    break;
            }
        }
    }
```

## 参考文章

https://www.jianshu.com/p/ab1fb00fcb90