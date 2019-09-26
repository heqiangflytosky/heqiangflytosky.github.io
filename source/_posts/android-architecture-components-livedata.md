---
title: Android Architecture Components -- LiveData
categories: Android 架构
comments: true
tags: [Architecture Components, LiveData]
description: 本文介绍 Android 生命周期架构组件 LiveData 的基本使用
date: 2018-9-16 10:00:00
---


## 概述

LiveData 也是属于 LifeCycle 库的一个组件。
LiveData 是一种具有生命周期感知能力的可观察数据持有类。可以理解为它是一种可观测数据容器，它会在数据变化是通知观测器，以便更新界面。
假设有一个UI界面和一个数据对象，这个数据对象保存了你想要在UI界面上显示的的数据，那么这个数据对象就可以用 LiveData 实现，UI界面可以设置为“观察” LiveData 对象。那么界面就可以在数据有更新时收到通知，随后，UI界面就可以使用新数据进行重新绘制。
简而言之，LiveData 可以使UI界面上显示的内容和数据随时保持同步。
LiveData 对象通常保存在 ViewModel 类中，当然也可以单独使用它，它还可以结合 Room 组件一起使用来监控数据库数据的变化，还可以和 DataBinding 组件一起使用，这个我们后面再介绍。
总结起来，LiveData 有一下特点：

 - 保证数据与界面的同步更新：LiveData采用了观察者模式设计，其中LiveData是被观察者，当数据发生变化时会通知观察者进行数据更新。通过这点，可以确保数据和界面的实时性。
 - 避免内存泄漏：观察者被绑定在 Lifecycle 对象上，当 Lifecycle 销毁时会自动移除这些观察者。
 - Activity和Fragment停止活动时不会引起崩溃：当观察者处于非活动状态时，比如Activity或Fragment处在后台时，即使数据有更新，观察者也不会收到数据变化通知。这样可以避免在这个时候更新UI导致Crash。
 - 不需要手动处理生命周期事件：LiveData 时可以感知生命周期的组件，UI 组件仅在活动状态时收到数据更新通知。
 - 始终能够保持最新数据：如果被观察者在非活动状态下有数据更新，那么在转为活动状态时，可以收到最新一次的数据更新通知，以便于来更新界面保持和数据同步。
 - 适应配置的变化：由于数据和组件的分离，当Activity或者Fragment由于配置改变而重新创建，比如旋转屏幕，因为数据是保存在LiveData中，它们可以马上获取到最新的数据。
 - 数据共享：我们可以把LiveData扩展成单例模式，并包装成系统服务，那么就可以在应用程序中共享数据，需要这些数据的观察者只需要观察该LiveData对象即可。

## LiveData 类介绍

介绍它的方法的使用：

 - observer：添加观察LiveData数据变化的观察者。会传递一个 LifecycleOwner 对象，当 Lifecycle 处于激活状态时才会通知数据变化，如果不在激活状态，即使数据有变化也不会通知。当 Lifecycle 对象被销毁时， 观察者 observer 会被自动 remove。
 - onActive：当LiveData有激活状态的观察者时被调用。可以有下面的情况：当调用 observer 使观察者由0变为1时；有设置观察者时，当应用从后台转为前台处于活动状态时。
 - onInactive：当LiveData没有激活状态的观察者时被调用。可以有下面的情况：当调用 removeObserver 使观察者由1变为0时；有设置观察者时，当应用从前台转为后台处于活动状态时。
 - setValue：可以更新LiveData对象持有的数据并通知观察LiveData的所有观察者。只能在主线程中调用。
 - postValue：和 setValue 功能一样，不同的是只能在后台线程中调用。
 - Observer：观察者，当LiveData数据变化时会调用它的onChanged方法。

LiveData 是可以感知生命周期的组件，它可以在多个 Activity、Fragment 中使用，你可以单例的形式来使用它。

## 基本使用

### 添加依赖

```
implementation "android.arch.lifecycle:livedata:1.1.1"
```

### 直接使用 LiveData 对象

```
        Observer<Integer> observer = new Observer<Integer>() {
            @Override
            public void onChanged(@Nullable Integer integer) {
                Log.e("Test","onChanged "+ integer);
            }
        };

        final MutableLiveData<Integer> liveData = new MutableLiveData<>();
        liveData.observe(this, observer);

        Timer timer = new Timer();
        TimerTask task = new TimerTask() {
            @Override
            public void run() {
                liveData.postValue(mTime);
                mTime ++;
            }
        };
        timer.schedule(task, 0, 1000);
```

在 Activity 的onCreate 方法中生成一个 LiveData 对象并为它设置观察者对象。
用定时器来定时更新LiveData的值。

### 使用扩展 LiveData 对象

```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_room);

        Observer<Integer> observer = new Observer<Integer>() {
            @Override
            public void onChanged(@Nullable Integer integer) {
                Log.e("Test","onChanged "+ integer);
            }
        };
        TimeLiveData.getInstance().observe(this, observer);
    }
}
```

```
public class TimeLiveData extends LiveData<Integer> {
    private volatile static TimeLiveData mInstance;
    private int mTime;

    public static TimeLiveData getInstance() {
        if (mInstance == null) {
            synchronized (TimeLiveData.class){
                if(mInstance == null) {
                    mInstance = new TimeLiveData();
                }
            }
        }
        return mInstance;
    }
    private TimeLiveData() {
        super();
        Timer timer = new Timer();
        TimerTask task = new TimerTask() {
            @Override
            public void run() {
                postValue(mTime);
                mTime ++;
            }
        };
        timer.schedule(task, 0, 1000);
    }

    @Override
    protected void setValue(Integer value) {
        super.setValue(value);
        Log.e("Test","setValue");
    }

    @Override
    protected void onActive() {
        super.onActive();
        Log.e("Test","onActive");
    }

    @Override
    protected void onInactive() {
        super.onInactive();
        Log.e("Test","onInactive");
    }
}
```

TimeLiveData 对象内用定时器来模拟时间变化。

## 数据转换

类似于 RxJava 的map操作符，LiveData 也可以做数据转换的工作。
LifeCycle 提供了 Transformations 工具类的 map 和 switchMap 来实现数据转化：

 - Transformations.map()：提供 LiveData 持有的数据的转换。map 方法返回转换后的一个 LiveData 对象。
 - Transformations.switchMap()：源 LiveData 持有数据源到目标 LiveData 的转化，switchMap 的 Function 对象的 apply 方法参数为源 LiveData 的数据，返回一个新的 LiveData 对象，switchMap 返回为转换后的 LiveData。


### Transformations.map

```
        LiveData<String> liveData1 = Transformations.map(liveData, new Function<Integer, String>() {
            @Override
            public String apply(Integer input) {
                return String.valueOf(input);
            }
        });
        liveData1.observe(this, new Observer<String>() {
            @Override
            public void onChanged(@Nullable String s) {
                Log.e("Test","onChanged1 "+ s);
            }
        });
```

### Transformations.switchMap

```
        mMutableLiveData = new MutableLiveData<>();
        LiveData liveData2= Transformations.switchMap(liveData, new Function<Integer, LiveData<String>>() {
            @Override
            public LiveData<String> apply(Integer input) {
                mMutableLiveData.setValue(String.valueOf(input));
                return mMutableLiveData;
            }
        });

        liveData2.observe(this, new Observer<String>() {
            @Override
            public void onChanged(@Nullable String s) {
                Log.e("Test","onChanged2 "+ s);
            }
        });
```

## 和 Room 结合

Room 的使用可以参考前面的博客[Android Architecture Components -- Room ](http://www.heqiangfly.com/2017/12/02/android-architecture-components-room/)。
我们现在在前面的基础上结合 LiveData 来监听数据的更新。
Room 支持查询对象返回 LiveData 对象，这样我们就可以创建对该 LiveData 对象的监听，当更新数据库时，观察者会收到数据的更新的通知，这时我们就可以进行更新界面操作。

### LiveData 结合 Room 实践

在数据访问对象 UserDao 中添加添加下面的方法，返回一个 LiveData 对象。

```
@Dao
public interface UserDao {
    ......

    @Query("SELECT * FROM user")
    LiveData<List<User>> getAllData();

    ......
}
```

然后注册监听：

```
                DBManager.getInstance(RoomActivity.this).getAllData().observe(RoomActivity.this, new Observer<List<User>>() {
                    @Override
                    public void onChanged(@Nullable List<User> users) {
                        for( User user : users) {
                            Log.e("Test", "getDatas: "+ user.getFirstName()+", "+user.getLastName());
                        }
                    }
                });
```

后面当数据库有数据更新时就可以收到通知了。


## 使用 LiveData 构建事件总线

关于 Android 中事件总线的使用，我们可能使用过 EventBus、RxBus等。学习了上面的关于 LiveData 的特效，LiveData 也可以方便的构建事件总线。
使用LiveData构建事件总线有什么优点呢？

 - 实现简单
 - 官方支持，减少依赖	
 - 避免内存泄露
 - 免除Activity生命周期问题带来的困扰

## 和 RxJava 区别

LiveData 与 RxJava 都是基于观察者模式, 功能上也有重合, Google 在官方文档上也明确表示, 如果你正在使用 RxJava, Agera 等类似功能的库, 只要你能正确的处理数据流的生命周期, 就完全可以继续使用它们来替代 LiveData。

## 源码解析

### 注册监听

```
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        // 实例化LifecycleBoundObserver，它继承 ObserverWrapper 并实现 GenericLifecycleObserver
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        // 将Observer放入mObservers列表
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        // 这册 LifeCycle 监听
        owner.getLifecycle().addObserver(wrapper);
    }
```

可以看到 observe 主要做了两件事情：

 - 把数据观察者放到观察者列表中
 - 注册 LifeCycle 监听，前面说 LiveData 可以感知生命周期变化，就是通过它来实现的。

```
    @MainThread
    public void removeObserver(@NonNull final Observer<T> observer) {
        assertMainThread("removeObserver");
        ObserverWrapper removed = mObservers.remove(observer);
        if (removed == null) {
            return;
        }
        // 删除LifeCycle监听
        removed.detachObserver();
        // 看是否需要调用 onInactive，如果此时观察者数量为0，则调用 onInactive
        removed.activeStateChanged(false);
    }
```

### onChange 数据变化通知

```
├── LiveData.setValue
    └── LiveData.dispatchingValue(ObserverWrapper observer)
        └── Observer.onChanged
```

观察者模式，不多解释。

```
    private void considerNotify(ObserverWrapper observer) {
        // 如果观察者已经处于非活动状态，直接返回
        if (!observer.mActive) {
            return;
        }
        // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
        //
        // we still first check observer.active to keep it as the entrance for events. So even if
        // the observer moved to an active state, if we've not received that event, we better not
        // notify for a more predictable notification order.
        // 如果观察者此次为非活动状态，尝试上报 onInactive
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        observer.mLastVersion = mVersion;
        //noinspection unchecked
        observer.mObserver.onChanged((T) mData);
    }
```

### onActive 和 onInactive 通知

先来看一下 LifecycleBoundObserver.onStateChanged 方法：

```
        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            // 当 LifeCycle 对象销毁时，清除Observer
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            activeStateChanged(shouldBeActive());
        }
```

```
        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }
```

必须时 STARTED 或者 RESUMED 状态，shouldBeActive() 才会返回true。

```
        void activeStateChanged(boolean newActive) {
            if (newActive == mActive) {
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            mActive = newActive;
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += mActive ? 1 : -1;
            // 从非活动状态转为活动状态时调用 onActive
            if (wasInactive && mActive) {
                onActive();
            }
            // 从活动状态转为非活动状态时调用 onInactive
            if (LiveData.this.mActiveCount == 0 && !mActive) {
                onInactive();
            }
            // 活动状态时发送数据更新事件
            if (mActive) {
                dispatchingValue(this);
            }
        }
```
