---
title: Android View 事件处理流程 -- TouchTarget
categories: Android 事件分发体系
comments: true
tags: [Android 事件分发体系]
description: 介绍 TouchTarget
date: 2015-11-10 10:00:00
---

## 概述

在前面的Android 事件分发体系系列文章中大概分析了一下 Andriod 对事件的大概的分发流程，本文着重介绍一下在事件分发中扮演重要角色的 `TouchTarget` 类。
TouchTarget的作用场景在事件派发流程中，用于记录派发目标，即消费了事件的子view。在 `ViewGroup` 中有一个成员变量 `mFirstTouchTarget`，它会持有 `TouchTarget`，并且作为 `TouchTarget` 链表的头节点。
由于我们的手机屏幕都支持多点触控，那么当用户同时用多个手指触摸屏幕的时候，产生的TouchEvent中就包含了多个触摸点的信息，一般我们把单个的触摸点就叫pointer，每个pointer都有它的pointer id。
TouchTarget用于记录一个被触摸的View，以及它所捕获的全部pointer。就是一个能处理TouchEvent的View，加上它处理的TouchEvent所属pointer的id。一个View能处理多个pointer产生的TouchEvent。
同时，在有多个pointer的情况下，不同的pointer产生的TouchEvent可能需要给不同的View处理，因此需要多个TouchTarget来记录这些信息，这些TouchTarget以链表的形式组织，每个TouchTarget都有一个next变量，指向另一个TouchTarget，链表尾的指向null。而TouchTarget的child变量，就是处理TouchEvent的View。

## 源码分析

```
    private static final class TouchTarget {
        private static final int MAX_RECYCLED = 32;
        private static final Object sRecycleLock = new Object[0];
        private static TouchTarget sRecycleBin;
        private static int sRecycledCount;

        public static final int ALL_POINTER_IDS = -1; // all ones

        // 消费事件的子view
        public View child;

        // child接收的触摸点的ID集合
        public int pointerIdBits;

        // 指向链表下一个节点
        public TouchTarget next;

        private TouchTarget() {
        }

        public static TouchTarget obtain(@NonNull View child, int pointerIdBits) {
            if (child == null) {
                throw new IllegalArgumentException("child must be non-null");
            }

            final TouchTarget target;
            synchronized (sRecycleLock) {
                if (sRecycleBin == null) {
                    target = new TouchTarget();
                } else {
                    target = sRecycleBin;
                    sRecycleBin = target.next;
                     sRecycledCount--;
                    target.next = null;
                }
            }
            target.child = child;
            target.pointerIdBits = pointerIdBits;
            return target;
        }

        public void recycle() {
            if (child == null) {
                throw new IllegalStateException("already recycled once");
            }

            synchronized (sRecycleLock) {
                if (sRecycledCount < MAX_RECYCLED) {
                    next = sRecycleBin;
                    sRecycleBin = this;
                    sRecycledCount += 1;
                } else {
                    next = null;
                }
                child = null;
            }
        }
    }
```

TouchTarget保存了响应触摸事件的子view和该子view上的触摸点ID集合，表示一个触摸事件派发目标。通过next成员可以看出，它支持作为一个链表节点储存。

### 触摸点ID存储

成员pointerIdBits用于存储多点触摸的这些触摸点的ID。pointerIdBits为int型，有32bit位，每一bit位可以表示一个触摸点ID，最多可存储32个触摸点ID。
pointerIdBits是如何做到在bit位上存储ID呢？假设触摸点ID取值为x（x的范围可从0～31），存储时先将1左移x位，然后pointerIdBits与之执行|=操作，从而设置到pointerIdBits的对应bit位上。
pointerIdBits的存在意义是记录TouchTarget接收的触摸点ID，在这个TouchTarget上可能只落下一个触摸点，也可能同时落下多个。当所有触摸点都离开时，pointerIdBits就已被清0，那么TouchTarget自身也将被从mFirstTouchTarget中移除。

### 对象获取和回收

TouchTarget的构造函数为私有，不允许直接创建。因为应用在使用过程中会涉及到大量的TouchTarget创建和销毁，因此TouchTarget封装了一个对象缓存池，通过TouchTarget.obtain方法获取，TouchTarget.recycle方法回收。

### ViewGroup 事件分发过程中对 TouchTarget 的操作

具体调用流程参考前面文章。

#### 查找 TouchTarget

`getTouchTarget` 方法说根据child查找对应的 TouchTarget

```
private TouchTarget getTouchTarget(@NonNull View child) {
    // 遍历链表
    for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
        // 比较child成员
        if (target.child == child) {
            return target;
        }
    }
    return null;
}
```

#### 添加 TouchTarget

在 `ViewGroup` 的 `dispatchTouchEvent` 方法中，会通过 `dispatchTransformedTouchEvent` 将调整后的`TouchEvent` 派发给子 `View`，如果子 `View` 感兴趣，会返回 true，此时就会把该子 `View` 和它感兴趣的` TouchEvent` 的 pointer 存储到 `TouchTarget` 中，加入链表作为表头存储，mFirstTouchTarget指向表头。这个添加操作时调用 `addTouchTarget` 实现的，它会将 child 和 pointerIdBits 保存到 TouchTarget 链表中，并且把新添加的 `TouchTarget` 作为表头。

```
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    // 通过对象缓存池获取可用的TouchTarget实例，同时保存child和pointerIdBits。
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    // 添加到链表中，并设置成新的头节点。
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

#### 删除 TouchTarget

当一个 `TouchTarget` 不捕获任何 pointer 的时候，如按在该 `View` 上的所有手指抬起时，该 `TouchTarget` 就会从链表中删除，并且执行 recycle 操作。
当调用 `ViewGroup#removeView` 移除某个子 `View` 时，`ViewGroup` 会调用下面的方法，该方法不仅从链表中删除了 `TouchTarget`，调用其 recycle 方法，还给它保存的 `View` 发了一个 ACTION_CANCEL 事件，使得View能清理各类状态。


```
private void cancelTouchTarget(View view) {
    TouchTarget predecessor = null;
    TouchTarget target = mFirstTouchTarget;
    while (target != null) {
        final TouchTarget next = target.next;
        if (target.child == view) {
            if (predecessor == null) {
                mFirstTouchTarget = next;
            } else {
                predecessor.next = next;
            }
            target.recycle();
            final long now = SystemClock.uptimeMillis();
            MotionEvent event = MotionEvent.obtain(now, now,
                    MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
            event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
            view.dispatchTouchEvent(event);
            event.recycle();
            return;
        }
        predecessor = target;
        target = next;
    }
}

```

#### 关于 mFirstTouchTarget

`ViewGroup` 不用单个 `TouchTarget` 保存消费了事件的 child，而是通过 `mFirstTouchTarget` 链表保存多个 `TouchTarget`，是因为存在多点触摸情况下，需要将事件拆分后派发给不同的child。
加入一个 `ViewGroup` 有两个子 `View` childA 和 childB，他们是兄弟 `View` 的关系，假设childA、childB都能响应事件：

 - 当触摸点1落于childA时，产生事件ACTION_DOWN，ViewGroup会为childA生成一个TouchTarget，后续滑动事件将派发给它。
 - 当触摸点2落于childA时，产生ACTION_POINTER_DOWN事件，此时可以复用TouchTarget，并给它添加触摸点2的ID。
 - 当触摸点3落于childB时，产生ACTION_POINTER_DOWN事件，ViewGroup会再生成一个TouchTarget，此时ViewGroup中有两个TouchTarget，后续产生滑动事件，将根据触摸点信息对事件进行拆分，之后再将拆分事件派发给对应的child。

## 参考文章

https://juejin.im/post/6844904065613201421
https://www.jianshu.com/p/ab1fb00fcb90
