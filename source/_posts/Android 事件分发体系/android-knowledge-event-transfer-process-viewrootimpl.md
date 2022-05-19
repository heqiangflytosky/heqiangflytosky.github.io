---
title: Android View 事件处理流程 -- ViewRootImpl
categories: Android 事件分发体系
comments: true
tags: [Android 事件分发体系]
description: 介绍 Android 事件处理中 ViewRootImpl 部分流程
date: 2015-11-7 10:00:00
---


## 概述

在上一篇文章中，我们对于事件在Activity之前的传递我们并没有详细的介绍，那么本文就详细来介绍一下这部分事件传递流程。
Activity的事件来源是和 ViewRootImpl 密切相关的。

## 事件传递流程

当用户点击屏幕产生一个触摸行为，这个触摸行为则是通过底层硬件来传递捕获，然后交给ViewRootImpl，接着将事件传递给DecorView，而DecorView再交给PhoneWindow，PhoneWindow再交给Activity，然后接下来就是我们前面介绍的常见的View事件分发了。
我们用下面的流程来简单描述一下：

```
硬件
    InputEventReceiver.dispatchInputEvent
        ViewRootImpl.deliverInputEvent
            DecorView.dispatchTouchEvent
                Activity.dispatchTouchEvent
                    PhoneWindow.dispatchTouchEvent
                        DecorView.superDispatchTouchEvent
                            View/ViewGroup.dispatchTouchEvent

```

## InputEventReceiver

ViewRootImpl 中实例化了一个 InputEventReceiver 对象来接收输入事件，然后交给ViewRootImpl来进行处理。

```
// ViewRootImpl.java
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        ......
                if (mInputChannel != null) {
                    if (mInputQueueCallback != null) {
                        mInputQueue = new InputQueue();
                        mInputQueueCallback.onInputQueueCreated(mInputQueue);
                    }
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                }
```

```
    final class WindowInputEventReceiver extends InputEventReceiver {
        public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
            super(inputChannel, looper);
        }

        @Override
        public void onInputEvent(InputEvent event) {
            List<InputEvent> processedEvents;
            try {
                processedEvents =
                    mInputCompatProcessor.processInputEventForCompatibility(event);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
            if (processedEvents != null) {
                if (processedEvents.isEmpty()) {
                    // InputEvent consumed by mInputCompatProcessor
                    finishInputEvent(event, true);
                } else {
                    for (int i = 0; i < processedEvents.size(); i++) {
                        enqueueInputEvent(
                                processedEvents.get(i), this,
                                QueuedInputEvent.FLAG_MODIFIED_FOR_COMPATIBILITY, true);
                    }
                }
            } else {
                enqueueInputEvent(event, this, 0, true);
            }
        }

        ......
    }
```

```
    void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {
        QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);

        // Always enqueue the input event in order, regardless of its time stamp.
        // We do this because the application or the IME may inject key events
        // in response to touch events and we want to ensure that the injected keys
        // are processed in the order they were received and we cannot trust that
        // the time stamp of injected events are monotonic.
        QueuedInputEvent last = mPendingInputEventTail;
        if (last == null) {
            mPendingInputEventHead = q;
            mPendingInputEventTail = q;
        } else {
            last.mNext = q;
            mPendingInputEventTail = q;
        }
        mPendingInputEventCount += 1;
        Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                mPendingInputEventCount);
        //看是同步处理还是异步处理
        if (processImmediately) {
            doProcessInputEvents();
        } else {
            scheduleProcessInputEvents();
        }
    }
```

```
    void doProcessInputEvents() {
        // 循环获取队列中的输入事件
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            mPendingInputEventHead = q.mNext;
            if (mPendingInputEventHead == null) {
                mPendingInputEventTail = null;
            }
            q.mNext = null;

            mPendingInputEventCount -= 1;
            Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                    mPendingInputEventCount);

            long eventTime = q.mEvent.getEventTimeNano();
            long oldestEventTime = eventTime;
            if (q.mEvent instanceof MotionEvent) {
                MotionEvent me = (MotionEvent)q.mEvent;
                if (me.getHistorySize() > 0) {
                    oldestEventTime = me.getHistoricalEventTimeNano(0);
                }
            }
            mChoreographer.mFrameInfo.updateInputEventTime(eventTime, oldestEventTime);

            deliverInputEvent(q);
        }

        // We are done processing all input events that we can process right now
        // so we can clear the pending flag immediately.
        if (mProcessInputEventsScheduled) {
            mProcessInputEventsScheduled = false;
            mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
        }
    }
```

```
    private void deliverInputEvent(QueuedInputEvent q) {
        Trace.asyncTraceBegin(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
                q.mEvent.getSequenceNumber());
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
        }

        InputStage stage;
        if (q.shouldSendToSynthesizer()) {
            stage = mSyntheticInputStage;
        } else {
            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
        }

        if (q.mEvent instanceof KeyEvent) {
            mUnhandledKeyManager.preDispatch((KeyEvent) q.mEvent);
        }

        if (stage != null) {
            handleWindowFocusChanged();
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
    }
```

这里InputStage则是一个实现处理输入事件责任的阶段，它是一个基类，也就是说InputStage提供一系列处理输入事件的方法，也可以转发给其他事件处理，而具体的处理则是看它的实现类。每种InputStage可以处理一定的事件类型，比如AsyncInputStage、ViewPreImeInputStage、ViewPostImeInputStage等。当一个InputEvent到来时，ViewRootImpl会寻找合适它的InputStage来处理。
InputStage的处理情况为，会先调用deliver开始处理。

```
       public final void deliver(QueuedInputEvent q) {
            if ((q.mFlags & QueuedInputEvent.FLAG_FINISHED) != 0) {
                forward(q);
            } else if (shouldDropInputEvent(q)) {
                finish(q, false);
            } else {
                apply(q, onProcess(q));
            }
        }
```

最终的事件分发处理则是在apply方法里的onProcess方法。
对于点击事件来说，InputState的子类ViewPostImeInputStage可以处理它，我们看下ViewPostImeInputStage的onProcess方法。

```
    final class ViewPostImeInputStage extends InputStage {
        public ViewPostImeInputStage(InputStage next) {
            super(next);
        }

        @Override
        protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                return processKeyEvent(q);
            } else {
                final int source = q.mEvent.getSource();
                // 判断是触摸事件还是鼠标事件
                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                    return processPointerEvent(q);
                } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
                    return processTrackballEvent(q);
                } else {
                    return processGenericMotionEvent(q);
                }
            }
        }
```

ViewPostImeInputStage.processPointerEvent -> DecorView.dispatchPointerEvent -> DecorView.dispatchTouchEvent

```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        final Window.Callback cb = mWindow.getCallback();
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
    }
```

这里的 cb 其实就是 Activity，在 attach() 方法中注册的回调：

```
    final void attach(......) {
        ......

        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
```

那么接下来就由 Activity 来处理了。