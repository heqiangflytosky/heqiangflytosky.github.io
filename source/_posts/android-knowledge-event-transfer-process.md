---
title: Android View 事件处理流程
categories: Android 事件分发体系
comments: true
tags: [Android 事件分发体系]
description: 介绍 Android 事件处理流程
date: 2015-11-2 10:00:00
---

网上流传着一篇被誉为[可能是讲解Android事件分发最好的文章](http://balpha.de/2013/07/android-development-what-i-wish-i-had-known-earlier/)，我看了之后也觉得受益匪浅，文章虽然很短，但是却明了的阐述了Android事件分发机制。本文是在研读这篇文章的基础上做了一些自己理解，在此记录下来，以备后面查阅。
首先我们先来熟悉一下几个在处理Android事件分发过程中可能用到的几个方法：
`View` 中的方法：

 - boolean dispatchTouchEvent(MotionEvent event)：事件分发。返回true表示处理该事件，中断分发，false表示会继续分发。
 - boolean onTouchEvent(MotionEvent event)：事件处理。返回true表示处理该事件，false表示不处理事件。该方法在 dispatchTouchEvent 中调用。
 - void setOnTouchListener(OnTouchListener l)
 - void setTouchDelegate(TouchDelegate delegate)

`ViewGroup` 中的方法：我们知道 `ViewGroup` 是继承了 `View` 类的，因此也继承了上面的几个方法，另外还有：

 - boolean onInterceptTouchEvent(MotionEvent ev)：事件拦截。它只存在于 `ViewGroup` 中，普通的 `View` 中没有这个方法。在任何一个 `View` 的 `onTouchEvent` 被调用之前，它的父辈们将先获得拦截这个事件的一次机会。换句话说，它们可以窃取该事件。返回 true 表示拦截，false 表示不拦截。该方法在 dispatchTouchEvent 中调用。
 - requestDisallowInterceptTouchEvent(boolean disallowIntercept)：阻止事件传递：当前 View 可以调用父View的 `requestDisallowInterceptTouchEvent` 方法来阻止父View对事件的拦截，`getParent().requestDisallowInterceptTouchEvent(true)`。那么父 View 将无法通过 `onInterceptTouchEvent` 来拦截事件。


## 一些假设

我们只考虑最重要的四个触摸事件，即：DOWN,MOVE,UP和CANCEL。一个手势（gesture）是一个事件列，以一个DOWN事件开始（当用户触摸屏幕时产生），后跟0个或多个MOVE事件（当用户四处移动手指时产生），最后跟一个单独的UP或CANCEL事件（当用户手指离开屏幕或者系统告诉你手势（gesture）由于其他原因结束时产生）。当我们说到“手势剩余部分”时指的是手势后续的MOVE事件和最后的UP或CANCEL事件。
在这里我也不考虑多点触摸手势（我们只假设用一个手指）并且忽略多个MOVE事件可以被归为一组这一实际情况。最后，我们假设文中的view都没有注册onTouchListener。
我们将要讨论的视图层次是这样的：最外层是一个ViewGroup A，包含一个或多个子view（children），其中一个子view是ViewGroup B，ViewGroupB中又包含一个或多个子view，其中一个子view是 View C,C不是一个ViewGroup。这里我们忽略同层级view之间可能的交叉叠加。

![效果图](/images/android-knowledge-event-transfer-process/android-touch.png)

假设用户首先触摸到的屏幕上的点是C上的某个点，该点被标记为触摸点（touch point），DOWN事件就在该点产生。然后用户移动手指并最后离开屏幕，此过程中手指是否离开C的区域无关紧要，关键是手势（gesture）是从哪里开始的。

## 默认情况

假设上面的A,B,C都没有覆写默认的事件传播行为，那么下面就是事件传播的过程：

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN
B onTouchEvent ACTION_DOWN
A onTouchEvent ACTION_DOWN
Activity onTouchEvent ACTION_DOWN
Activity dispatchTouchEvent ACTION_MOVE
Activity onTouchEvent ACTION_UP
```

 1. DOWN事件首先由 Activity 进程分发。
 2. 首先传递给 A，由 A 进行分发，A 没有进行时间拦截。
 3. 事件传递给 B，由 B 进行分发，B 没有进行时间拦截。
 4. 事件传递给 C，C 没有处理这个事件。
 5. 再把事件传递给 B，B 没有处理这个事件。
 6. 再把事件传递给 A，A 没有处理这个事件。
 7. 最终把 DOWN 事件传递给 Activity。
 8. Activity 处理后面的 MOVE 和 UP 事件。

由于没有 View 对这个事件感兴趣。后面的 MOVE 和 UP 事件不再进行逐级分发和传递，直接由 Activity 进行处理。
从这个流程我们可以得到下面的结论：

 - 假如 DOWN 事件传给 C 的 `onTouchEvent` 方法时，它返回了 false，DOWN 事件会继续向上传递给 B 和 A 的 `onTouchEvent`，即使它们在 `onInterceptTouchEvent` 方法中说它们不想拦截这个 DOWN 事件，但没办法，没有子 `View` 愿意处理该事件。
 - 如果所有的 `View` 都不愿意处理事件，那么最后只能交给 Activity 来处理了。

## 处理事件

现在我们假设 C 对 DOWN 事件感兴趣，我们可以通过下面方法实现：

 - 覆写了 C 的 `onTouchEvent` 方法，处理 DOWN 事件，并返回 true 。
 - 调用 `setOnClickListener` 设置监听。
 - 调用 `setOnTouchListener` 设置监听，并且 `onTouch` 方法返回true。
 - 设置 `setClickable(true)`。

先来看一下通过 C 的 `onTouchEvent` 方法处理 DOWN 事件来看整个事件传递流程：

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onInterceptTouchEvent ACTION_UP
C dispatchTouchEvent ACTION_UP
C onTouchEvent ACTION_UP
```

 - 前面几步的流程和前面一样。
 - DOWN 事件传递到 C 的 onTouchEvent 方法时，C 需要对这个事件做处理，即消费了这个事件。
 - DOWN事件不会继续传递，将不再被传递给 B 和 A 的 onTouchEvent 方法。
 - 因为 C 对当前的事件感兴趣，所以剩余的手势事件（MOVE 和 UP）也将传递给 C 的 onTouchEvent 方法，此时该方法返回 true 或 false 都无关紧要了，但是为保持一致最好还是返回 true。

从这里可以看出，各个 `View` 的 `onTouchEvent` 方法对 DOWN 事件的处理，代表了该 `View` 对以此 DOWN 开始的整个手势（gesture）的处理意愿，返回 true 代表愿意处理该 gesture，返回 false 代表不愿意处理该 gesture。

从这个流程我们可以得到下面的结论：

 - 虽然 A 和 B 的 `onInterceptTouchEvent` 方法对 DOWN 事件返回了 false，后续的事件依然会传递给它们的 `onInterceptTouchEvent` 方法，这一点与 `onTouchEvent` 的行为是不一样的。

下面把 B 接受该事件的流程也列出来：

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN
B onTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onTouchEvent ACTION_UP
```

再来看一下通过设置 C setOnTouchListener 回调来处理事件：

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouch ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onInterceptTouchEvent ACTION_MOVE
C dispatchTouchEvent ACTION_MOVE
onTouch ACTION_MOVE

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onInterceptTouchEvent ACTION_UP
C dispatchTouchEvent ACTION_UP
onTouch ACTION_UP
```

可以看到，C 设置了 setOnTouchListener 后，就不会在调用 onTouchEvent 了，在 dispatchTouchEvent 中，会先去判断是否设置 OnTouchListener，如果设置就不会调用 onTouchEvent。 具体原因在下一篇博客中会分析。

下面再来看一下设置 setOnClickListener：

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN
Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onInterceptTouchEvent ACTION_MOVE
C dispatchTouchEvent ACTION_MOVE
C onTouchEvent ACTION_MOVE
Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onInterceptTouchEvent ACTION_UP
C dispatchTouchEvent ACTION_UP
C onTouchEvent ACTION_UP
onClick
```

可以到是在 C 的 onTouchEvent 后面调用了 onClick，其实 onClick 的调用时在 onTouchEvent 中对 ACTION_UP 时调用的，具体在下一篇文章中详细介绍。
那么如果 onTouchEvent 中处理了 ACTION_DOWN 事件呢？onClick 还会不会调用呢？

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN
Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onInterceptTouchEvent ACTION_UP
C dispatchTouchEvent ACTION_UP
C onTouchEvent ACTION_UP
```

不会再调用 onClick 了。

综上所述：onTouchEvent、setOnTouchListener 和 setOnClickListener 处理事件的优先级为：setTouchListener > onTouchEvent > setClickListener 

下面再来看一下 C 只设置 `setClickable(true)` 后的传递流程：

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onInterceptTouchEvent ACTION_MOVE
C dispatchTouchEvent ACTION_MOVE
C onTouchEvent ACTION_MOVE

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onInterceptTouchEvent ACTION_UP
C dispatchTouchEvent ACTION_UP
C onTouchEvent ACTION_UP
```

由此可见 `setClickable(true)` 只是说 C 可以消费事件，阻止了事件的继续传递。

上面都是介绍 C 消费事件，下面再来介绍一下通过 B 消费事件：

onTouchEvent

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN
B onTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onTouchEvent ACTION_MOVE

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onTouchEvent ACTION_UP

```

setOnTouchListener

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN
onTouch ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
onTouch ACTION_MOVE

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
onTouch ACTION_UP

```

setOnClickListener

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN
B onTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onTouchEvent ACTION_MOVE

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onTouchEvent ACTION_UP
onClick

```


setClickable

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN
B onTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onTouchEvent ACTION_MOVE

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onTouchEvent ACTION_UP

```

很有意思的事情：设置 B 的onTouchEvent、 setOnTouchListener 、 setOnClickListener 和 setClickable 并没有阻止 ACTION_DOWN 事件向 C 传递。因为 setOnTouchListener 、 setOnClickListener 和 setClickable 都是在 onTouchEvent 中处理的。只是处理的优先级不一样。

如何才能实现不把事件传递到 C 呢？只能使用 onInterceptTouchEvent 了，在下面一节中会介绍。

## 拦截事件

上面的事件处理的例子中，DOWN 事件被 C 消费了，那么后面的 MOVE 和 UP 事件默认都会给 C 来处理。但是如果后面的 MOVE 事件 B 又想处理了怎么办呢？这个时候就需要 `onInterceptTouchEvent` 上场了。
前面也讲过，`onInterceptTouchEvent` 只存在于 `ViewGroup` 中，普通的 `View` 中没有这个方法。在任何一个 `View` 的 `onTouchEvent` 被调用之前，它的父辈们将先获得拦截这个事件的一次机会，换句话说，它们可以窃取该事件。
从上面的流程中也可以看出，事件在交给 C 的 `onTouchEvent` 处理之间，都会经过 A 和 B 的 `onInterceptTouchEvent` 方法。
那么如果 B 想拦截这个事件，只需要 `onInterceptTouchEvent` 中处理并返回 true 就行了。
先来看一下事件传递流程：

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onInterceptTouchEvent ACTION_MOVE

C dispatchTouchEvent ACTION_CANCEL
C onTouchEvent ACTION_CANCEL

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onTouchEvent ACTION_MOVE
（Activity onTouchEvent ACTION_MOVE）（如果 B onTouchEvent返回false，表示只拦截，不消费，那么还会传递给 A 和 Activity）

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onTouchEvent ACTION_UP
（Activity onTouchEvent ACTION_UP）（如果 B onTouchEvent返回false，表示只拦截，不消费，那么还会传递给 A 和 Activity）
```
 - 前面几步的流程和前面一样。C 处理了 DOWN 事件。
 - MOVE 事件传递到 B 的 `onInterceptTouchEvent` 方法时，B 想对这个事件做处理，那么就返回了 true。
 - MOVE 事件中断传递，这时 C 分发了 CANCEL 事件，`onTouchEvent` 也将会接收到 CANCEL 事件。
 - 那么后面的 MOVE 和 UP 事件不会再通过 B 的 `onInterceptTouchEvent`，将会直接分发到 B 的 `onTouchEvent` 进行处理。
 - 后面的 MOVE 和 UP 事件如果 B 的 `onTouchEvent` 做了处理并返回true，那么 UP 事件将被消费，不再继续传递，如果 B 的 `onTouchEvent` 不处理返回 false，那么这个事件将直接返回给 `Activity` 进行处理。
 - 事件被拦截后，除了 CANCEL 事件，C 将不会再收到任何事件。

从这个流程我们可以得到下面的结论：

 - DOWN 事件的处理实际上经历了一下一上两个过程，下是指 A->B 的 `onInterceptTouchEvent`，上是指 C->B->A 的 `onTouchEvent`，当然，任意一步的方法中返回 true,都能阻止它继续传播。
 - 如果 ViewGroup 拦截了一个半路的事件（比如 MOVE），这个事件将会被系统变成一个 CANCEL 事件，并传递给之前处理该手势（gesture）的子 View，而且不会再传递（无论是被拦截的 MOVE 还是系统生成的 CANCEL）给 `ViewGroup` 的 `onTouchEvent` 方法。只有再到来的事件才会传递到 `ViewGroup` 的 `onTouchEvent` 方法中。
 - 事件被拦截后，如果本身的 `onTouchEvent` 也不做处理，那么直接返回给 `Activity` 处理。

下面我们再来做一个实验，来回答上一节的问题：B 在 `onInterceptTouchEvent` 中拦截了最初的 DOWN 事件。
先看第一种情况：DOWN 事件只在 `onInterceptTouchEvent` 中拦截，`onTouchEvent` 不做处理。

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
B onTouchEvent ACTION_DOWN
A onTouchEvent ACTION_DOWN
Activity onTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
Activity onTouchEvent ACTION_MOVE

Activity dispatchTouchEvent ACTION_UP
Activity onTouchEvent ACTION_UP
```

可以看到，只是拦截了事件的传输，没有消费的话还是最终由 Activity 来处理。

第二种情况：DOWN 事件在 `onInterceptTouchEvent` 中拦截，`onTouchEvent` 中处理并返回true。

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
B onTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onTouchEvent ACTION_MOVE

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onTouchEvent ACTION_UP
```

 - 如果一个 `ViewGroup` 拦截了最初的 DOWN 事件，该事件仍然会传递到该 `ViewGroup` 的 `onTouchEvent` 方法中。
 - `onInterceptTouchEvent` 只会拦截事件的传递，具体事件如何传递还要看 `onTouchEvent` 的处理。
 - 如果只拦截 DOWM 事件而不做处理，那么认为没有 View 对这组手势事件感兴趣，那么后面直接交由 Activity 处理。
 - 如果对 DOWN 事件处理，那么后续事件都会分发给 ViewGroup 的 `onTouchEvent` ，如果不处理再直接传递给 Activity 处理。

## 分发事件

如果你仔细看了上面几节，你会发现这样的情况，onInterceptTouchEvent 只管事件拦截，具体是否消费事件由 onTouchEvent 来决定，如果只拦截不消费，那么事件还是会回传到父 View 或者 Activity 的。
那么如果我们阻止了事件的分发呢？
我们在 B 的 dispatchTouchEvent 方法中针对 ACTION_DOWN 事件返回true：

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onTouchEvent ACTION_MOVE
Activity onTouchEvent ACTION_MOVE

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onTouchEvent ACTION_UP
Activity onTouchEvent ACTION_UP
```

dispatchTouchEvent 只管当前事件的分发，阻止了 onTouchEvent 事件的分发并不能阻止随后事件的分发。

## 阻止拦截事件

子 View 是可以阻止父 View 对事件拦截的，这就要用到 `ViewGroup` 的 `requestDisallowInterceptTouchEvent`，子 `View` 中调用 `getParent().requestDisallowInterceptTouchEvent(true)` 即可。
但是这个调用却要在合适的位置才能生效，具体原因在后面的源码分析中会介绍。
在 `onTouchEvent` 的处理 `DOWN` 事件时设置是合适的位置，因为在事件处理的开始(ACTION_DOWN)时会重新清理这个标志位，因此，设置过早是不生效的。
下面是 C 处理了事件，并且在 `onTouchEvent` 处理 `DOWN` 事件时设置了 `getParent().requestDisallowInterceptTouchEvent(true)` 的流程。

```
Activity dispatchTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN

Activity dispatchTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
C dispatchTouchEvent ACTION_MOVE
C onTouchEvent ACTION_MOVE

Activity dispatchTouchEvent ACTION_UP
A dispatchTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
C dispatchTouchEvent ACTION_UP
C onTouchEvent ACTION_UP
```

可以看出，除了 DOWN 事件 C 的父元素们将无法通过 `onInterceptTouchEvent` 拦截 MOVE 和 UP 事件。

## 多点触控

前面我们介绍的都是在单点触摸的情况下，Android 也是支持多点触控的，当已经有手指接触屏幕的情况下，当再有其他触摸点出现是，会触发 `ACTION_POINTER_DOWN` 事件，当有手指离开屏幕时会触发 `ACTION_POINTER_UP` 事件，最后一根手指离开屏幕是触发 `ACTION_UP ` 事件，因此多点触控的事件可能是下面的流程：

```
ACTION_DOWN -> ACTION_POINTER_DOWN -> ACTION_POINTER_UP -> ACTION_UP
```

获取多点触控获取事件类型请使用 `event.getAction() & MotionEvent.ACTION_MASK` 或者 `getActionMasked()`。追踪事件流可以使用 PointId。

## 几个问题

在总结几个关于 Android 事件分发经常遇到的几个问题：

 - 如果B、C都没有处理ACTION_DOWN事件，包括没有注册touch和click监听，那么B的onInterceptTouchEvent执行几次？
  - 处理一次ACTION_DOWN，因为所有的View对此次时间都没有兴趣，那以后的Move和Up就不传回来了。
 - 如果B、C没有注册touch和click监听，B的onTouchEvent对MotionEvent.ACTION_DOWN返回true，那么B的onInterceptTouchEvent执行几次？
  - 处理一次ACTION_DOWN，因为B表示对这个时间感兴趣，以后的move和up就没必要再调用onInterceptTouchEvent了。B的onTouchEvent会对这些事件响应。
 - 如果B、C都没有拦截和处理ACTION_DOWN事件，C注册了click监听，那么B的onInterceptTouchEvent和onTouchEvent会执行几次？ 
  - B的onInterceptTouchEvent会对ACTION_DOWN、ACTION_MOVE和ACTION_UP都会响应，但onTouchEvent对此都不响应，因为子View已经表示对此次事件感兴趣了。但是父View仍然有权利对事件拦截。
 - ACTION_CANCEL事件如何被触发
  - 当C响应了ACTION_DOWN事件后（比如ACTION_CANCEL对down事件返回true），B突然插手把ACTION_MOVE事件拦截了，那么以后的move和up事件都交由B的onTouchEvent来处理，C的onTouchEvent就会收到ACTION_CANCEL事件。

## 参考文章

http://balpha.de/2013/07/android-development-what-i-wish-i-had-known-earlier/
https://mp.weixin.qq.com/s?__biz=MzI0MjE3OTYwMg==&mid=2649547708&idx=1&sn=143add3bcceba00c4e3292b49557d4bc#rd
