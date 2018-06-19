---
title: Android View 事件处理流程源码解析
categories: Android 事件分发体系
comments: true
tags: [Android 事件分发体系]
description: 介绍 Android 事件处理流程
date: 2015-11-6 10:00:00
---

上一篇博客介绍了事件传递的基本流程，为了做到知其然并知其所以然，本文从源码的角度来分析一下Android事件传的流程。
再分析代码之前，我们先用 Android Studio 来看一下事件传过程中的调用栈。
我们在前文中的默认情况下（不拦截和处理事件）在 ViewC 的 `onTouchEvent` 中添加断点。
调用栈如图所示：

![效果图](/images/android-knowledge-event-transfer-process-source-code/android-event-process.png)

看了上篇博客我们也许会有疑问，事件的传递流程源头是不是从 `Activity` 开的，看了上图就有答案了，源头是从系统的事件分发系统开始，首先传递给 `DecorView` 开始的。
下面我们源码的分析的思路就按照上面的事件传递的流程顺序来进行。
调用流程先用下面的流程来表示一下：

```

```

## 事件起源

`DecorView` 接收事件之前的流程我们这不做详细介绍，只介绍 `View` 部分的事件传递。在 `Activit` 接收到事件之前的传递流程是：

```
├── View.dispatchPointerEvent
    ├── DecorView.dispatchTouchEvent
        ├── Activity.dispatchTouchEvent
```

先来看一下 `View.dispatchPointerEvent` ，这里其实执行的是 `DecorView` 对象的 `dispatchPointerEvent`：

```
    public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }
```

`View.dispatchPointerEvent` 方法是final类型的，是不可以被 `Override` 的。
如果是触摸事件，则调用 `DecorView` 的 `dispatchTouchEvent` 方法。

```
        @Override
        public boolean dispatchTouchEvent(MotionEvent ev) {
            final Callback cb = getCallback();
            ......
            // 这调用 Callback 的 dispatchTouchEvent 方法。这个 cb 其实就是 Activity
            return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
                    : super.dispatchTouchEvent(ev);
        }
```

## Activity 的事件分发

上一篇博客中介绍事件传递的开是 Activit，Activity 首先进行事的分发，那么下面先来看一下 `Activity.dispatchTouchEvent(MotionEvent ev)` 方法。在此之的流程这里先不涉及介绍。

```
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

这个方法主是调用了 `getWindow().superDispatchTouchEvent(ev)` 方法，如果该方法返回false，那么再调用 `Activity.onTouchEvent(ev)` 来处理事件。
Activity 中的 `mWindow` 在 `Activity.attach()` 中被实例化，是一个 `PhoneWindow` 对象。

```
        mWindow = new PhoneWindow(this);
        // 设置callback，上面一章的流程中涉及
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
```

那么我们就再来看一下 `PhoneWindow.superDispatchTouchEvent` 方法。

```
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        boolean handled = mDecor.superDispatchTouchEvent(event);
        ...
        return handled;
    }
```

这里调用了 `DecorView` 的 `superDispatchTouchEvent` 方法。

```
        public boolean superDispatchTouchEvent(MotionEvent event) {
            return super.dispatchTouchEvent(event);
        }
```

`DecorView.superDispatchTouchEvent` 直接调了它的 `dispatchTouchEvent` 方法，我们知，`DecorView` 是继承了 `FrameLayout` 的，那么其实就是调了 `ViewGroup` 的 `dispatchTouchEvent` 方法。

## ViewGroup 的事件分发

来看一下 `ViewGroup.dispatchTouchEvent` 方法，这个方法比较长，我们只选择事件传递的关键代码进行分析。
先来看下面一段代码：

```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
            ....
            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                // 清除 mFirstTouchTarget
                cancelAndClearTouchTargets(ev);
                // 这个方法将会对 FLAG_DISALLOW_INTERCEPT 标志进行重置
                resetTouchState();
            }
            // Check for interception.
            final boolean intercepted;
            // mFirstTouchTarget != null 表示有子View消费事件或者事件没有被拦截
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                // 看当前是否设置了FLAG_DISALLOW_INTERCEPT标志
                // 通过requestDisallowInterceptTouchEvent设置
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    // 如果允许被拦，则调用 onInterceptTouchEvent 方法，再次判断是否拦截
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }
            ....
    }
```

上面的一段代码主要判断是否对当前事件进行拦截。
首先第一个条件是当前事件是否是 `ACTION_DOWN` 或者 `mFirstTouchTarget != null`，判断当前是否是 `ACTION_DOWN` 事件我们容易理解，上一篇博客的例子我们也知道，`ACTION_DOWN` 事件是一个手势事件系列的开始，是会调用 `onInterceptTouchEvent` 的。那么 `mFirstTouchTarget != null` 是什么呢？从 `dispatchTouchEvent` 后面的代码我们就可以看出来，当事件被 `ViewGroup` 的子元素处理时，`mFirstTouchTarget` 就会指向这个子元素。也就是说，如果当前 `ViewGroup` 拦截该事件时，`mFirstTouchTarget != null`这个条件是不成立的，那么当后面的 `ACTION_MOVE` 和 `ACTION_UP` 事件到来时，将不会执行这段代码，那么 `onInterceptTouchEvent` 方法就不会被调用。如果有子元素处理事件，那么后面的 `ACTION_MOVE` 和 `ACTION_UP` 事件到来时，将会执行这段代码，那么 `ViewGroup` 就会有机会拦截事件的传递。
这里还有另外一个判断条件，就是是否设置了 `FLAG_DISALLOW_INTERCEPT` 这个标志位，这个标志位一般是子 `View` 通过 `ViewGroup.requestDisallowInterceptTouchEvent` 方法来设置的，一旦设置，`ViewGroup` 将无法拦截除了 DOWN 事件的其他一切事件。
为什么说可以拦截 DOWN 事件呢？因为 `ViewGroup` 在事件分发时，如果是 DOWN 事件就重置 `FLAG_DISALLOW_INTERCEPT` 标志位，这一点从上面的源码中也可以看到，这个操作将会导致子 View 设置的该标志位无效，因此当分发 DOWN 事件时，总是会调用 `onInterceptTouchEvent` 来询问是否要拦截该事件。
从上面的代码可以得到下面几点结论：

 - 当 ViewGroup 决定拦截事件后，那么后续的事件将会默认交给它来处理而不再调用 `onInterceptTouchEvent` 方法。
 - `FLAG_DISALLOW_INTERCEPT` 这个标志位的作用是不让它的父View们中途来拦截事件，当然父View没有拦截DOWN事件，这个标志位无法操控父View对DOWN事件的拦截。
 - `onInterceptTouchEvent` 不是每次都调用，如果我们想操控所有的点击事件，只能在 `dispatchTouchEvent` 方法中处理。只有这个方法保证每次都会被调用（前提是事件能传递到当前的 `ViewGroup`）。
 - 子 View 调用父 View 的 `requestDisallowInterceptTouchEvent` 一定要注意调用时机，否则是不会生效的。

下面我们接着看当 `ViewGrou` 不拦截事件时的处理。

```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
                        ...
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }
                            // 判断子View是否可、是否在做动画或者点击事件的坐标是否在子View范围之内
                            // 以此来作为是否可以接收点击事件的先决条件
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                ...
                                continue;
                            }
                            // 满足接收条件，看当前子View是否正在处理事件
                            newTouchTarget = getTouchTarget(child);
                            ...
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            // 在dispatchTransformedTouchEvent中调用子 View的dispatchTouchEvent
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        ...
    }
```

从上面源码可以看到，当 `ViewGroup` 不拦截事件时，事件会向下分发交由它的子 View 处理。
首先会遍历一遍所有的子元素，判断元素是否能接收点击事件，条件是判断子View是否可、是否在做动画或者点击事件的坐标是否在子View范围之内来作为依据，如果可以接收事件，则调用 `dispatchTransformedTouchEvent` 方法。

那么再来看一下这个方法：

```
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        ...
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
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
            transformedEvent = event.split(newPointerIdBits);
        }

        ...
    }
```

如果子元素为null则调用父类的 `dispatchTouchEvent`，如果子元素不为null，则调用子元素的 `dispatchTouchEvent`。

再来回到 `dispatchTouchEvent` 方法，如果遍历子元素后事件都没有被处理，这包含两种情况：一是 `ViewGroup` 没有子元素，二是子元素处理了点击事件，但是子元素 `dispatchTouchEvent` 返回了 false，这时  `ViewGroup`会自己处理事件。这时还是会调用 `dispatchTransformedTouchEvent`，只是 child 参数为  null，这是根据上面的的代码，会调用 `super.dispatchTouchEvent`，即 `View.dispatchTouchEvent` 来处理。

```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
            ...
            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } 
            ...
    }
```

我们可以看到 `ViewGroup` 是没有覆盖父类的 `onTouchEvent ` 方法的。也没有发现调用 `onTouchEvent`。
对 `ViewGroup` 的 `onTouchEvent` 调用，都是通过其父类 `View` 的 `dispatchTouchEvent` 来调用的。

## View 的事件分发

先来看一下 `View` 的 `dispatchTouchEvent` 方法。

```
    public boolean dispatchTouchEvent(MotionEvent event) {
        ...
        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
        ...
    }
```

`View` 这里对事件的处理比较简单，这里的处理包含两种类型：一类是由父元素调用子元素的 `dispatchTouchEvent` 分发而来，第二类是 `ViewGroup` 没有子元素，调用 `super.dispatchTouchEvent` 而来。
这里对事件的处理也比较简单，如果没有满足下面的两个条件就会调用 `onTouchEvent` 来处理事件，即：该 `View` 是使能状态；设置了 `OnTouchListener` 且 `onTouch` 方法返回 true。否则就会调用 `onTouchEvent`。
从这里可以看到 `OnTouchListener` 的优先级是高于 `onTouchEvent` 的，这样做的好处是方便在外界处理事件。

接下来再看一下 `View` 的 `onTouchEvent` 方法。

```
    public boolean onTouchEvent(MotionEvent event) {
        ...
        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }
        ...
    }
```

上面一段代码是处理 `View` 在不可用的情况下对事件的处理。通过阅读代码我们可以得到下面的结论：

 - 不可用状态下的 `View` 在设置可点击（`CLICKABLE`、`LONG_CLICKABLE`或`CONTEXT_CLICKABLE`）的情况下仍然会消费事件。

下面再来看一下 `onTouchEvent` 对具体事件的处理：

```
    public boolean onTouchEvent(MotionEvent event) {
        ...
            if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            if (!focusTaken) {
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
                    mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);

                        checkForLongClick(0, x, y);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    setPressed(false);
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_MOVE:
                    drawableHotspotChanged(x, y);

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
                        removeTapCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            // Remove any future long press/tap checks
                            removeLongPressCallback();

                            setPressed(false);
                        }
                    }
                    break;
            }

            return true;
        }
        ...
    }
```

```
    public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }
```

从上面的代码可以得到如下结论：

 - 设置可点击（`CLICKABLE`、`LONG_CLICKABLE`或`CONTEXT_CLICKABLE`）满足任意一个条件，`View`就会消费这个事件，`onTouchEvent` 就会返回 true，不管是否是 DISABLE 状态。
 - 当 `ACTION_UP` 事件触发时，会调用 `performClick()` 方法。`performClick()` 会调用 `OnClickListener` 的  `onClick` 方法。

## 事件处理流程图

下图表示了在事件不被消费的情况下 `DOWN` 事件的处理流程：

![效果图](http://www.plantuml.com/plantuml/svg/dP912i8m44NtSufUm0kuaBeGSI_kOpAIW6P2-bFrzbR1WIffnTsGl3VyyDDsC1dbSYOV73Sd4HpbHhHODMkBq0VSbovqoS3wlHJhDpr7a7dU6R12z1wQmJm4lcwpb3IfAaKwZMM9kmZEbXEcVSU_B9qDyrAKbbZb75SyZJwIF_6_Ng1jr5Oh_LsEOdDdBKSt_8K7)

<!--  
@startuml
hide footbox

-> Activity:dispatchTouchEvent
activate Activity
Activity -> ViewGroup:dispatchTouchEvent
activate ViewGroup
ViewGroup -> ViewGroup:onInterceptTouchEvent
activate ViewGroup
deactivate ViewGroup
ViewGroup -> View:dispatchTouchEvent
activate View
View -> View:onTouchEvent
activate View
deactivate View
View -> ViewGroup:onTouchEvent
deactivate View
activate ViewGroup
deactivate ViewGroup
ViewGroup -> Activity:onTouchEvent
deactivate ViewGroup
activate Activity
deactivate Activity
deactivate Activity
@enduml
-->
