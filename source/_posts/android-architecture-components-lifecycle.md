---
title: Android Architecture Components -- LifeCycle
categories: Android 架构
comments: true
tags: [Architecture Components, LifeCycle]
description: 本文介绍 Android 生命周期架构组件 LifeCycle 的基本使用
date: 2018-9-10 10:00:00
---

## 概述

[LifeCycle 官方文档](https://developer.android.com/topic/libraries/architecture/lifecycle#java)
Google 提供的Android Architecture Components中包含了 LifeCycle，Lifecycle 实现的一个重要目的，是实现 Android 中与 Activity 和 Fragment 生命周期相关的逻辑控制进一步的解耦。
以前我们写和 Android 生命周期相关的逻辑时，会把相关的代码放在对应的生命周期方法中。这样做的后果时生命周期的代码很臃肿，耦合程度很高。比如在前面介绍的[Android 架构 android-architecture 之 todo-mvp 介绍 ](http://www.heqiangfly.com/2017/09/06/android-architecture-google-mvp-basic/)中需要在 Presenter 中实现生命周期逻辑时是这样做的：

```
public class TaskDetailFragment extends Fragment implements TaskDetailContract.View {
    ......
    @Override
    public void onResume() {
        super.onResume();
        mPresenter.subscribe();
    }

    @Override
    public void onPause() {
        super.onPause();
        mPresenter.unsubscribe();
    }
    ......
}
```

LifeCycle 它可以有效避免内存泄漏和解决 Android 生命周期的常见难题。LifeCycle 已经更新到 2.0 版，现已纳入 Jetpack 并包含数据绑定库的新集成。
LifeCycle 组件也是 LiveData 和 ViewModel 的基础组件。LifeCycle 简单独立，可以单独使用，也可以配合上述组件使用。

 - LifeCycleOwner：Lifecycle 持有者，LifecycleOwner 时一个接口，它仅有一个方法 `getLifecycle()` 用来表明它持有Lifecycle 对象。它一般是具有生命周期的 Activity 或者 Fragment 组件。
 - LifecycleObserver：Lifecycle 的观察者，当它通过 Lifecycle 的 `addObserver` 方法注册后，它便可以观察 LifeCycleOwner 的生命周期事件。
 - State：生命周期状态，当 LifeCycleOwner 生命周期状态改变时，LifecycleRegistry 通过 `markState` 方法标记 Lifecycle 进入的状态，并向 LifecycleObserver 分发消息。
 - Event：生命周期事件，可以在LifecycleObserver中通过注解标记接受某个生命周期状态的方法。


## 使用方法

Lifecycle 已经是稳定版，它包含在support library 26.1.0 及之后的依赖包中，如果我们的项目基于这些依赖包，那么不需要额外的引用。如果是之前的版本，则要额外添加 LifeCycle 的依赖。
LifeCycle 是 ViewModel 和 LiveData 的基础构件，它们的依赖包中也都包含 Lifecycle。

### 实现 LifecycleOwner

#### 26.1.0之前的AppCompatActivity

添加依赖：

```
implementation "android.arch.lifecycle:runtime:1.1.1"
```

添加 LifecycleOwner：

```
public class MainActivity extends AppCompatActivity implements LifecycleOwner{
    private LifecycleRegistry mLifecycleRegistry;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mLifecycleRegistry = new LifecycleRegistry(this);
        mLifecycleRegistry.addObserver(new MyObserver());
        mLifecycleRegistry.markState(Lifecycle.State.CREATED);
    }


    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }

    @Override
    protected void onStart() {
        super.onStart();
        mLifecycleRegistry.markState(Lifecycle.State.STARTED);
    }

    @Override
    protected void onResume() {
        super.onResume();
        mLifecycleRegistry.markState(Lifecycle.State.RESUMED);
    }

    //...... 省略其他生命周期方法
}
```

实现LifecycleObserver接口，参考下节。

#### 26.1.0和之后的AppCompatActivity

26.1.0以及以后的 `AppCompatActivity` 类的父类已经实现了 `LifecycleOwner` 接口。

```
public class SupportActivity extends Activity implements LifecycleOwner {
}
```

```
public class RoomActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_room);
        getLifecycle().addObserver(new MyObserver());
    }
}
```

实现LifecycleObserver接口，参考下节。

### 实现 LifecycleObserver

关于实现 LifecycleObserver 我们介绍下面两种方法：

 - 一个是直接实现 LifecycleObserver 接口，然后通过为方法添加注解的方式来接收生命周期变化的事件。
 - 一个是实现 DefaultLifecycleObserver 接口，重写我们感兴趣的生命回调周期方法。


```
public class MyObserver implements LifecycleObserver {
    private final static String TAG = "MyObserver";
    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    public void onCreate(){
        Log.e(TAG,"MyObserver onCreate");
    }
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void onResume(){
        Log.e(TAG,"MyObserver onResume");
    }
    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    public void onStart(){
        Log.e(TAG,"MyObserver onStart");
    }
    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void onPause(){
        Log.e(TAG,"MyObserver onPause");
    }

    // 省略其他声明周期方法
}
```

要想使用 DefaultLifecycleObserver 需要添加下面的支持 java8 的依赖：


```
implementation 'android.arch.lifecycle:common-java8:1.1.1'
```


```
public class MyObserver implements DefaultLifecycleObserver {
    @Override
    public void onCreate(@NonNull LifecycleOwner owner) {
        Log.e("Test","onCreate");
    }

    @Override
    public void onStart(@NonNull LifecycleOwner owner) {
        Log.e("Test","onStart");
    }

    @Override
    public void onResume(@NonNull LifecycleOwner owner) {
        Log.e("Test","onResume");
    }

    @Override
    public void onPause(@NonNull LifecycleOwner owner) {

    }

    @Override
    public void onStop(@NonNull LifecycleOwner owner) {

    }

    @Override
    public void onDestroy(@NonNull LifecycleOwner owner) {

    }
}
```

DefaultLifecycleObserver 类中的文档提到，

```
/**
 * Callback interface for listening to {@link LifecycleOwner} state changes.
 * <p>
 * If you use Java 8 language, <b>always</b> prefer it over annotations.
 */
```

如果你使用了 Java8，那么就推荐使用 DefaultLifecycleObserver。

## 源码分析

顺便瞅瞅源码是如何实现的。

### 生命周期方法注册

LifecycleObserver 的注册是通过 LifecycleRegistry 来完成的， 它实现了 Lifecycle 接口，并持有了 LifecycleOwner 的一个弱引用，可以避免导致 Fragment / Activity 的内存泄漏。

主要来看一下 LifecycleRegistry.addObserver 方法：

```
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        // 生成一个 ObserverWithState 对象
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        // 把 ObserverWithState 对象放到 mObserverMap中，由此可见，可以注册多个 LifecycleObserver
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        // 这里是看看有没有未分发的事件
        // 如果 addObserver 用在 markState 之后，那么就存在未分发的事件
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }
```

ObserverWithState 封装了 State 和 GenericLifecycleObserver，后面事件的分发是通过 GenericLifecycleObserver 来进行的。GenericLifecycleObserver 同样实现了 LifecycleObserver 并增加了一个方法 onStateChanged。

```
    static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            // 根据 LifecycleObserver 生成 GenericLifecycleObserver 对象
            mLifecycleObserver = Lifecycling.getCallback(observer);
            mState = initialState;
        }
        // 触发生命周期回调时会走这个方法
        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
```

来看一下 Lifecycling.getCallback 方法，这个方法的作用就是根据 LifecycleObserver 的不同实现来生成不同的事件分发方法，比如我们前面介绍的可以直接实现 LifecycleObserver 或者是 实现 DefaultLifecycleObserver。

```
    static GenericLifecycleObserver getCallback(Object object) {
        // DefaultLifecycleObserver 继承了 DefaultLifecycleObserver
        // 如果 LifecycleObserver 继承 DefaultLifecycleObserver 那么就
        // 返回 FullLifecycleObserverAdapter
        if (object instanceof FullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object);
        }

        if (object instanceof GenericLifecycleObserver) {
            return (GenericLifecycleObserver) object;
        }

        final Class<?> klass = object.getClass();
        int type = getObserverConstructorType(klass);
        if (type == GENERATED_CALLBACK) {
            List<Constructor<? extends GeneratedAdapter>> constructors =
                    sClassToAdapters.get(klass);
            if (constructors.size() == 1) {
                GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                        constructors.get(0), object);
                return new SingleGeneratedAdapterObserver(generatedAdapter);
            }
            GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
            for (int i = 0; i < constructors.size(); i++) {
                adapters[i] = createGeneratedAdapter(constructors.get(i), object);
            }
            return new CompositeGeneratedAdaptersObserver(adapters);
        }
        // 如果直接继承 LifecycleObserver 通过注解方式注册生命周期回调方法
        // 就返回 ReflectiveGenericLifecycleObserver
        return new ReflectiveGenericLifecycleObserver(object);
    }
```

### 生命周期状态分发

先来看一下事件分发流程的初始流程，因为如果 LifecycleOwner 的使用方式不一样，初始流程也是不一样的。
先来看使用 26.1.0和之后的AppCompatActivity的情况：

```
├── ReportFragment.ReportFragment
    └── LifecycleRegistry.handleLifecycleEvent
        └── LifecycleRegistry.sync()
```

使用 26.1.0 之后Activity实现LifecycleOwner的情况：

```
├── LifecycleRegistry.markState
    └── LifecycleRegistry.moveToState
        └── LifecycleRegistry.sync()
```

这里没什么好介绍的，大家自行了解即可。

```
    private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }
```

其实事件的分发主要就是 ObserverWithState.dispatchEvent 的调用，然后调用生成的 GenericLifecycleObserver 对象的 onStateChanged 方法。

#### ReflectiveGenericLifecycleObserver.onStateChanged

```
class ReflectiveGenericLifecycleObserver implements GenericLifecycleObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Event event) {
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```

在添加 LifecycleObserver 时会根据 LifecycleObserver 对象生成 ReflectiveGenericLifecycleObserver，根据 LifecycleObserver 生成一个 CallbackInfo 对象。
CallbackInfo 保存了添加了 OnLifecycleEvent 注解的方法以及它对应的生命周期事件。
分发事件通过 

```
        void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
            invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
            invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
                    target);
        }
```

通过反射来调用注册的回调方法。

#### FullLifecycleObserverAdapter.onStateChanged

这个方法其实很简单，就是直接调用 FullLifecycleObserver 的几个生命周期回调方法。

```
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        switch (event) {
            case ON_CREATE:
                mObserver.onCreate(source);
                break;
            case ON_START:
                mObserver.onStart(source);
                break;
            case ON_RESUME:
                mObserver.onResume(source);
                break;
            case ON_PAUSE:
                mObserver.onPause(source);
                break;
            case ON_STOP:
                mObserver.onStop(source);
                break;
            case ON_DESTROY:
                mObserver.onDestroy(source);
                break;
            case ON_ANY:
                throw new IllegalArgumentException("ON_ANY must not been send by anybody");
        }
    }
```