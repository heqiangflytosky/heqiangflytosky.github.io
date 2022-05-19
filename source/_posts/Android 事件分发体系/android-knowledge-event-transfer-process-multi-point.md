---
title: Android View 事件处理流程 -- 多点触控
categories: Android 事件分发体系
comments: true
tags: [Android 事件分发体系]
description: 介绍 Android 多点触控事件处理流程
date: 2015-11-15 10:00:00
---

## 概述

前面的几篇关于 Android 事件分发的文章我们简要的介绍过关于多点触控的知识，本章我们就从源码角度进行一些分析。
Android 也是支持多点触控的，当已经有手指接触屏幕的情况下，当再有其他触摸点出现时，会触发 `ACTION_POINTER_DOWN` 事件(可能被拆分成 ACTION_DOWN 事件)，当有手指离开屏幕时会触发 `ACTION_POINTER_UP` 事件（当然这个事件在某个View上还可能转换为`ACTION_UP ` 事件），最后一根手指离开屏幕是触发 `ACTION_UP ` 事件，因此多点触控的事件可能是下面的流程：

```
ACTION_DOWN -> ACTION_POINTER_DOWN -> ACTION_POINTER_UP -> ACTION_UP
```

获取多点触控获取事件类型请使用 `event.getAction() & MotionEvent.ACTION_MASK` 或者 `getActionMasked()`。追踪事件流可以使用 PointId。

## 多点触控事件传递流程

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

然后在 ViewC 和 ViewD 的 onTouchEvent 中收到 MotionEvent.ACTION_DOWN 时返回 true，表示可以处理事件，保证后续的事件可以继续分发下来。

```
// ViewC
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        if(ev.getAction() != MotionEvent.ACTION_MOVE) {
            Log.e("Event", "ViewC onTouchEvent " + EventActivity.getActionString(ev));
        }
        if (ev.getActionMasked() == MotionEvent.ACTION_DOWN){
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

然后用一根手指按住 ViewD，此时用另外一根手指点击 ViewC。这样就模拟出了一个多点触控的场景。
下面来看一下事件的传递，这个过程可以分为４个阶段：

```
//ViewD被按压
Event   : Activity dispatchTouchEvent ACTION_DOWN
Event   : LinearLayoutA dispatchTouchEvent ACTION_DOWN
Event   : LinearLayoutA onInterceptTouchEvent ACTION_DOWN
Event   : ViewD dispatchTouchEvent ACTION_DOWN
Event   : ViewD onTouchEvent ACTION_DOWN

//ViewC被按压
Event   : Activity dispatchTouchEvent ACTION_POINTER_DOWN
Event   : LinearLayoutA dispatchTouchEvent ACTION_POINTER_DOWN
Event   : LinearLayoutA onInterceptTouchEvent ACTION_POINTER_DOWN
Event   : LinearLayoutB dispatchTouchEvent ACTION_DOWN
Event   : LinearLayoutB onInterceptTouchEvent ACTION_DOWN
Event   : ViewC dispatchTouchEvent ACTION_DOWN
Event   : ViewC onTouchEvent ACTION_DOWN

//ViewC抬起
Event   : Activity dispatchTouchEvent ACTION_POINTER_UP
Event   : LinearLayoutA dispatchTouchEvent ACTION_POINTER_UP
Event   : LinearLayoutA onInterceptTouchEvent ACTION_POINTER_UP
Event   : LinearLayoutB dispatchTouchEvent ACTION_UP
Event   : LinearLayoutB onInterceptTouchEvent ACTION_UP
Event   : ViewC dispatchTouchEvent ACTION_UP
Event   : ViewC onTouchEvent ACTION_UP
Event   : Activity onTouchEvent ACTION_POINTER_UP

//ViewD抬起
Event   : Activity dispatchTouchEvent ACTION_UP
Event   : LinearLayoutA dispatchTouchEvent ACTION_UP
Event   : LinearLayoutA onInterceptTouchEvent ACTION_UP
Event   : ViewD dispatchTouchEvent ACTION_UP
Event   : ViewD onTouchEvent ACTION_UP
Event   : Activity onTouchEvent ACTION_UP
```

我们可以看到，在 LinearLayoutA 分发 ACTION_POINTER_DOWN 和 ACTION_POINTER_UP 之前，会把事件拆分成 ACTION_DOWN 和 ACTION_UP 事件，然后分发给它的子 View。

## 源码分析

dispatchTransformedTouchEvent 的调用有三处，第一处是：

```
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;
                    ......
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            ......

                            resetCancelNextUpFlag(child);
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                ......
                            }
```

当事件为ACTION_DOWN、ACTION_POINTER_DOWN、ACTION_HOVER_MOVE事件时会走这里，表示一个新的事件序列，寻找可以接收事件的目标。

第二处和第三处是下面：

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
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
```

在这个阶段是在事件的派发阶段，第二次调用是因为事件没有找到 mFirstTouchTarget 对象，可以认为是没有子 View 消耗事件，最终将事件交给父控件本身处理。
第三次调用是在父控件没有拦截事件的情况下,会将事件分发到子 View 中。

```
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // oldPointerIdBits 表示的是当前屏幕所有 pointer id
        // desiredPointerIdBits 表示的是当前时间的 pointer id
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // If for some reason we ended up in an inconsistent state where it looks like we
        // might produce a motion event with no pointers in it, then drop the event.
        if (newPointerIdBits == 0) {
            return false;
        }

        // 如果是单点触摸，newPointerIdBits == oldPointerIdBits
        // 如果是多点触摸，newPointerIdBits != oldPointerIdBits
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    // 向子 View 分发事件
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);

                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            // 执行事件转换
            transformedEvent = event.split(newPointerIdBits);
        }

        // 分发事件
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }
```

`event.getPointerIdBits()` 获取的是当前屏幕所有 pointer id，如果是一个触摸点就是 1，2个触摸点就是3，3个触摸点就是7。
前面的文章也介绍过，ViewGroup 是没有重写 onTouchEvent 的，而是在 View.dispatchTouchEvent 调用的，从 dispatchTransformedTouchEvent 可以看到，分发给 子 View 的事件都是经过转换的，因此 onTouchEvent 方法是不会接收多点触控事件，在传入给View的dispatchTouchEvent之前，就已经进行转换。


## 参考文章

https://blog.csdn.net/dehang0/article/details/104317611
https://juejin.im/post/6844904145749704712
https://www.google.com/search?source=hp&ei=0CtwX4HvDseRr7wPoNuruAI&q=newPointerIdBits+%3D%3D+oldPointerIdBits&oq=newPointerIdBits+%3D%3D+oldPointerIdBits&gs_lcp=CgZwc3ktYWIQAzIHCCEQChCgAVDBCVjBCWCuD2gAcAB4AIAB8gGIAfIBkgEDMi0xmAEAoAECoAEBqgEHZ3dzLXdpeg&sclient=psy-ab&ved=0ahUKEwiBl6nE1YjsAhXHyIsBHaDtCicQ4dUDCAY&uact=5
https://www.jishuwen.com/d/pKCJ/zh-hk
https://wang-zoo.github.io/post/view_event/
https://juejin.im/post/6844904065617362952
https://blog.csdn.net/MoLiao2046/article/details/103737611
http://gityuan.com/2015/09/19/android-touch/