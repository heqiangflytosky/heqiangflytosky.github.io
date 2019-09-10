---
title: RxJava 使用指南 -- PublishSubject 实现事件总线
categories: Android开源项目
comments: true
tags: [Android开源项目, RxJava]
description: 本文介绍 RxJava 实现事件总线
date: 2017-10-20 10:00:00
---

## 概述

关于 Android 中实现事件总线，我们可能接触过 EventBus，可以很方便的在Activity和Fragment和View间传递消息，今天我们来介绍一下使用 RxJava 来实现事件总线。

## 代码实现

实现事件发布，使用 PublishSubject 来实现：

```
public class EventPublisher {
    private final Subject<Object> mSubject;
    private EventPublisher() {
        mSubject = PublishSubject.create().toSerialized();
    }

    private static class ThemePublisherHolder{
        private static EventPublisher INSTANCE = new EventPublisher();
    }

    public static EventPublisher getInstance() {
        return ThemePublisherHolder.INSTANCE;
    }

    public void post(Object obj) {
        mSubject.onNext(obj);
    }

    public Observable<Event> toObservable(Class<Event> type) {
        return mSubject.ofType(type);
    }

    public Flowable<Event> toFlowable(Class<Event> type) {
        return toObservable(type).toFlowable(BackpressureStrategy.BUFFER);
    }

    public Disposable toDisposable(Class<Event> type, Consumer<Event> consumer) {
        return toFlowable(type).observeOn(AndroidSchedulers.mainThread()).subscribe(consumer);
    }
}
```

实现事件观察者，订阅事件监听。

```
public class EventObserver {

    private WeakReference<ITarget> mTarget;
    private Disposable mDisposable;

    public EventObserver(@NonNull ITarget target) {
        mTarget = new WeakReference<>(target);
    }

    public static EventObserver createObserver(@NonNull ITarget target) {
        return new EventObserver(target);
    }

    public void register() {
        mDisposable = EventPublisher.getInstance().toDisposable(Event.class, new Consumer<Event>() {
            @Override
            public void accept(Event event) throws Exception {
                ITarget target = mTarget.get();
                if (target != null) {
                    target.update(event);
                }
            }
        });
    }

    public void unRegister() {
        if (mDisposable != null) {
            mDisposable.dispose();
            mDisposable = null;
        }
    }
}
```

实现一个接口，声明一个方法来更新事件。

```
public interface ITarget {
    void update(Event event);
}
```

```
public class Event {
    public String msg;
}
```

自定义一个 TextView 来监听事件：

```
public class MyTextView extends TextView implements ITarget {
    private EventObserver mObserver;
    public MyTextView(Context context) {
        this(context, null);
    }

    public MyTextView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public MyTextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr,0);
    }

    public MyTextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
        mObserver = EventObserver.createObserver(this);
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        if (mObserver != null) {
            mObserver.register();
        }
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        if (mObserver != null) {
            mObserver.unRegister();
        }
    }

    @Override
    public void update(Event event) {
        Log.e("Test",event != null ? event.msg:"");
    }
}
```

发布消息：

```
        Event event = new Event();
        event.msg = "发布消息";
        EventPublisher.getInstance().post(event);
```

