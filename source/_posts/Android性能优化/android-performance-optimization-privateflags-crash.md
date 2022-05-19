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

再看一下我们 `removeView` 的时机，

<img src="/images/android-performance-optimization-privateflags-crash/1.png" width="479" height="568"/>

为了方便调试，我们模拟了一个环境，布局如下：

```
<com.example.heqiang.testsomething.event.LinearLayoutA xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center"
    android:id="@+id/view_a"
    android:background="#B9D3EE">
    <com.example.heqiang.testsomething.event.LinearLayoutB
        android:id="@+id/view_b"
        android:layout_width="500dp"
        android:layout_height="300dp"
        android:background="#CD8162">
        <com.example.heqiang.testsomething.event.ViewC
            android:id="@+id/view_c"
            android:layout_width="200dp"
            android:layout_height="200dp"
            android:background="#9F79EE"/>

    </com.example.heqiang.testsomething.event.LinearLayoutB>
    <com.example.heqiang.testsomething.event.ViewD
        android:id="@+id/view_d"
        android:layout_width="200dp"
        android:layout_height="200dp"
        android:background="#9F79EE"/>
</com.example.heqiang.testsomething.event.LinearLayoutA>
```

然后在 ViewC 和 ViewD 的 onTouchEvent 中收到 MotionEvent.ACTION_DOWN 时返回 true，表示可以处理事件。在 ViewC 的 收到 MotionEvent.ACTION_UP 时把根布局从根布局的父布局中 remove 掉。 

```
// ViewC
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        if(ev.getAction() != MotionEvent.ACTION_MOVE) {
            Log.e("Event", "ViewC onTouchEvent " + EventActivity.getActionString(ev));
        }
        if(ev.getActionMasked() == MotionEvent.ACTION_UP){
            Log.e("Event","RRRRRRRRRRRR ");
            ((ViewGroup)(mRoot.getParent())).removeView(mRoot);
        } else if (ev.getActionMasked() == MotionEvent.ACTION_DOWN){
            return true;
        }
        return super.onTouchEvent(ev);
    }
```

```
// ViewD
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        if(ev.getAction() != MotionEvent.ACTION_MOVE) {
            Log.e("Event", "ViewD onTouchEvent " + EventActivity.getActionString(ev));
        }
        if (ev.getActionMasked() == MotionEvent.ACTION_DOWN){
            return true;
        }
        return super.onTouchEvent(ev);
    }
```

然后用一根手指按住 ViewD，此时用另外一根手指点击 ViewC，ViewC 上的手指抬起时发生崩溃。日志和上面是一样一样的。

前面的博客中我们介绍过，当当调用 ViewGroup#removeView 移除某个子 View 时，ViewGroup 会调用 cancelTouchTarget，该方法不仅从链表中删除了该 View 对应的 TouchTarget，调用其 recycle 方法，还给它对应的 View 发了一个 ACTION_CANCEL 事件，使得View能清理各类状态。

```
        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
```

那么主要问题就出现在这，LinearLayoutA 向 LinearLayoutB 和 ViewD 分发 ACTION_CANCEL 事件，那么就会调用它们的 TouchTarget 链表里面 TouchTarget 的 recycle 操作，当然就包括了 ViewD：

ViewGroup.dispatchTouchEvent() -> ViewGroup.resetTouchState() -> ViewGroup.clearTouchTargets()

```
    private void clearTouchTargets() {
        TouchTarget target = mFirstTouchTarget;
        if (target != null) {
            do {
                TouchTarget next = target.next;
                target.recycle();
                target = next;
            } while (target != null);
            mFirstTouchTarget = null;
        }
    }

```

看一下下面的调用日志：
RRRRRRRRRRRR 这里表示 removeView 操作。

```
Event   : Activity dispatchTouchEvent ACTION_DOWN
Event   : LinearLayoutA dispatchTouchEvent ACTION_DOWN
Event   : LinearLayoutA onInterceptTouchEvent ACTION_DOWN
Event   : ViewD dispatchTouchEvent ACTION_DOWN
Event   : ViewD onTouchEvent ACTION_DOWN
Event   : Activity dispatchTouchEvent ACTION_POINTER_DOWN
Event   : LinearLayoutA dispatchTouchEvent ACTION_POINTER_DOWN
Event   : LinearLayoutA onInterceptTouchEvent ACTION_POINTER_DOWN
Event   : LinearLayoutB dispatchTouchEvent ACTION_DOWN
Event   : LinearLayoutB onInterceptTouchEvent ACTION_DOWN
Event   : ViewC dispatchTouchEvent ACTION_DOWN
Event   : ViewC onTouchEvent ACTION_DOWN
Event   : Activity dispatchTouchEvent ACTION_POINTER_UP
Event   : LinearLayoutA dispatchTouchEvent ACTION_POINTER_UP
Event   : LinearLayoutA onInterceptTouchEvent ACTION_POINTER_UP
Event   : LinearLayoutB dispatchTouchEvent ACTION_UP
Event   : LinearLayoutB onInterceptTouchEvent ACTION_UP
Event   : ViewC dispatchTouchEvent ACTION_UP
Event   : ViewC onTouchEvent ACTION_UP
Event   : RRRRRRRRRRRR 
Event   : LinearLayoutA dispatchTouchEvent ACTION_CANCEL
Event   : LinearLayoutA onInterceptTouchEvent ACTION_CANCEL
Event   : LinearLayoutB dispatchTouchEvent ACTION_CANCEL
Event   : LinearLayoutB onInterceptTouchEvent ACTION_CANCEL
Event   : ViewC dispatchTouchEvent ACTION_CANCEL
Event   : ViewC onTouchEvent ACTION_CANCEL
Event   : ViewD dispatchTouchEvent ACTION_CANCEL
Event   : ViewD onTouchEvent ACTION_CANCEL

```

LinearLayoutA 在处理 ACTION_POINTER_UP 过程中，首先向它的子 View LinearLayoutB 派发事件，派发前，先把该事件转换为 ACTION_UP，当该事件分发到 ViewC 时，触发 removeView LinearLayoutA 操作，然后 LinearLayoutA 就会向它的所有子 View 分发 ACTION_CANCEL，这样就会触发所有的 TouchTarget 链表的 recycle 操作。
然后 LinearLayoutA 的 dispatchTouchEvent 方法会继续处理 ACTION_POINTER_UP 事件，当它继续分发给 LinearLayoutB 所在的 TouchTarget 的下一个 TouchTarget 也就是 ViewD 时，由于 ViewD 所在的 TouchTarget 已经执行了 recycle 操作，因此就会出现调用 resetCancelNextUpFlag(target.child) 时传入了 null 参数。
这里有个问题，为什么 ACTION_POINTER_UP 是先分发给 LinearLayoutB ，然后分发给 ViewD 呢？这是因为我们是最后触摸 LinearLayoutB 上的 ViewC，那么它所对应的 TouchTarget 就位于 TouchTarget 链表的表头。

来看一下到 LinearLayoutA 开始分发ACTION_CANCEL 之前的流程。

```
LinearLayoutA.dispatchTouchEvent ACTION_POINTER_UP
  ViewGroup.dispatchTouchEvent ACTION_POINTER_UP
```

```
            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        // 调用这里的 dispatchTransformedTouchEvent
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
```

```
    // while循环，向 LinearLayoutB 分发ACTION_POINTER_UP
    ViewGroup.dispatchTransformedTouchEvent ACTION_POINTER_UP
      ViewGroup.dispatchTouchEvent ACTION_POINTER_UP
        LinearLayoutB.dispatchTouchEvent ACTION_UP
          ViewGroup.dispatchTransformedTouchEvent ACTION_UP
            ViewC.dispatchTouchEvent ACTION_UP
              View.dispatchTouchEvent ACTION_UP
                ViewC.onTouchEvent ACTION_UP
                  ViewGroup.removeView
                    ViewGroup.removeViewInternal
                      ViewGroup.cancelTouchTarget
                        LinearLayoutA dispatchTouchEvent ACTION_CANCEL
                          ViewGroup.dispatchTouchEvent ACTION_CANCEL
                            ViewGroup.dispatchTransformedTouchEvent ACTION_CANCEL
......
                               ViewD.dispatchTouchEvent ACTION_CANCEL
    // while 循环继续向　ViewD 分发 ACTION_POINTER_UP
    ViewGroup.resetCancelNextUpFlag ACTION_POINTER_UP 引发null
```

所以我们看到，派发 ACTION_POINTER_UP 事件的事件循环中，在传递给 LinearLayoutB 时，先转换 ACTION_UP 事件，然后执行 removeView 操作，之后派发 ACTION_CANCEL 操作，然后再派发给 ViewD ACTION_POINTER_UP 事件， 这些都是在一次事件派发循环中进行的，于是就出现了逻辑上的问题导致了崩溃，因此，稳妥的做法是在这次事件循环进行完毕后再进行 removeView 操作，这样派发 ACTION_CANCEL 操作以及EventTarget的recycle都是在另外的事件循环中进行的了，这样就能避免这个逻辑错误。

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