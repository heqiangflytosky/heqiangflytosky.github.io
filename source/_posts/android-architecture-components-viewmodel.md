---
title: Android Architecture Components -- ViewModel
categories: Android 架构
comments: true
tags: [Architecture Components, ViewModel]
description: 本文介绍 Android 生命周期架构组件 ViewModel 的基本使用
date: 2018-9-20 10:00:00
---

[官方文档](https://developer.android.google.cn/topic/libraries/architecture/viewmodel)

## 概述

ViewModel 我们在 MVC、MVP 和 MVVM 架构中经常见到这个概念。
本文来介绍 LifeCycle 库中的 ViewModel 库。
ViewModel 库是用来保存应用UI数据的类，它可以在配置变更（比如屏幕选择导致的Activit重建）后继续存在，可以避免因数据Activit重建需要数据保存和恢复问题而导致的各种错误。
因此，Android 推荐使用这样的架构设计，将应用中所有的UI数据保存在ViewModel中，而不是Activity中，这样能保证数据不受比如configuration change 带来的Activity重建而带来的影响。
Android 开发中一个常见的坑是很多的变量、逻辑和数据放在Activity和Fragment中，这样的代码比较混乱和难以维护，这种开发模式违反了单一职责的原则，ViewModel可以有效地划分责任，具体地，它可以用来保存 Activity 的所有UI数据，然后Activity仅负责了解如何在屏幕上显示该数据和接受用户互动，但是它不会处理这些互动，如果应用要加载和存储数据，建议创建一个 Repository 的存储区类。
另外还应该确保 ViewModel 不会因为承担过多的责任而变的臃肿，要避免这种情况，可以创建 Presenter 类，或者实现一种更成熟的架构。

## 使用

要创建一个 ViewModel，首先要扩展ViewModel类，然后将Activity中之前与UI相关的实例变量摆放在这个ViewModel中，然后在 Activity 中的 onCreate 中从 ViewModel Provider 的框架实用类再获取 ViewModel。
请注意 ViewModel Provider 将获取一个 Activity 的实例，它可以确保Activity重建后获取一个新的Activity实例，不过，请确保它始终和同一个ViewModel关联。
对于ViewModel实例，你可以通过getter函数在Activity中直接获取数据。
ViewModel可以单独使用，当然可以和 LiveData 相结合，ViewModel 返回一个 LiveData 对象，然后在 Activity中创建监听，然后更新数据。
使用 ViewModel 和 LiveData 可以创建反应式界面。也就是说当底层数据更新时，UI界面会相应自动更新。
当然还可以和 Data Binding 结合使用。

```
public class LiveDataTimerViewModel extends ViewModel {

    private static final int ONE_SECOND = 1000;

    private MutableLiveData<Long> mElapsedTime = new MutableLiveData<>();

    private long mInitialTime;

    public LiveDataTimerViewModel() {
        mInitialTime = SystemClock.elapsedRealtime();
        Timer timer = new Timer();

        // Update the elapsed time every second.
        timer.scheduleAtFixedRate(new TimerTask() {
            @Override
            public void run() {
                final long newValue = (SystemClock.elapsedRealtime() - mInitialTime) / 1000;
                mElapsedTime.postValue(newValue);
            }
        }, ONE_SECOND, ONE_SECOND);

    }

    public LiveData<Long> getElapsedTime() {
        return mElapsedTime;
    }
}
```

```
    private LiveDataTimerViewModel mLiveDataTimerViewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.chrono_activity_3);

        mLiveDataTimerViewModel = ViewModelProviders.of(this).get(LiveDataTimerViewModel.class);

        final Observer<Long> elapsedTimeObserver = new Observer<Long>() {
            @Override
            public void onChanged(@Nullable final Long aLong) {
                String newText = ChronoActivity3.this.getResources().getString(
                        R.string.seconds, aLong);
                ((TextView) findViewById(R.id.timer_textview)).setText(newText);
            }
        };

        mLiveDataTimerViewModel.getElapsedTime().observe(this, elapsedTimeObserver);
    }
```

ViewModel的默认构造函数是没有任何参数的，如果你想要修改，可以使用ViewModelFactory类创建一个自定义的构造函数，

## ViewModel 的生命周期

ViewModel 的生命周期是由传递给 `ViewModelProviders.of(this)` 的生命周期决定的，ViewModel 初始化后保留在内存中，直到 Activity 永久消失，比如退出并销毁。
下图展示了 ViewModel 的生命周期：

![效果图](https://developer.android.google.cn/images/topic/libraries/architecture/viewmodel-lifecycle.png)

在系统第一次调用 Activity 的 onCreate() 方法时初始化 ViewModel，系统可能会在Activity的整个生命周期内多次调用 onCreate()，例如当设备屏幕旋转时。 ViewModel 的生命周期从第一次请求 ViewModel 开始，直到 Activity 被 FINISHED 并销毁。

## Fragment 之间共享数据

我们经常遇到在一个Activity中有多个 Fragment 需要互相通信的情况，一般的做法可能就是两个 Fragment 都需要定义一些接口，而它们的 Activity 必须将两者联系在一起。 此外，Fragment 必须处理另一个 Fragment 尚未创建或可见的情况。
使用 ViewModel 可以解决这个问题， Fragment 可以通过一个在 Activity 范围内共享的 ViewModel 来处理彼此的通信。

```
public class SharedViewModel extends ViewModel {
    private final MutableLiveData<Item> selected = new MutableLiveData<Item>();

    public void select(Item item) {
        selected.setValue(item);
    }

    public LiveData<Item> getSelected() {
        return selected;
    }
}


public class MasterFragment extends Fragment {
    private SharedViewModel model;
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        itemSelector.setOnClickListener(item -> {
            model.select(item);
        });
    }
}

public class DetailFragment extends Fragment {
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        SharedViewModel model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        model.getSelected().observe(this, { item ->
           // Update the UI.
        });
    }
}
```

请注意，在获取ViewModel时，两个Fragment都要使用getActivity()。这样，这两个Fragment会收到同一个SharedViewModel实例，该实例的作用域为Activity。
这样做有以下好处：

 - 这个 Activity 不需要做任何事情，也不需要知道 Fragment 之间的交流。
 - 除了 SharedViewModel 的接口之外，Fragment不需要了解彼此。 如果其中一个 Fragment 消失，另一个 Fragment 继续照常工作。
 - 每个 Fragment 都有自己的生命周期，不受其他生命周期的影响。 一个 Fragment 替换成另一个 Fragment，UI继续工作也没有任何问题。


## 用 ViewModel 来代替 Loaders

在之前我们会经常使用 CursorLoader 来保持应用程序中的数据与数据库同步。那么现在就可以用 ViewModel 来代替 Loader 的使用了。使用 ViewModel 可以将 UI controller 与数据加载操作分开，这意味着在类之间的强引用减少了。
Loader 通用的做法是使用 CursorLoader 来观察数据库中的数据，当数据库的数据变化时，Loader 会自动触发一次数据加载并更新UI，如下图：

![效果图](https://developer.android.google.cn/images/topic/libraries/architecture/viewmodel-loader.png)

现在我们可以使用 ViewModel + LiveData + Room 的组合来代替 Loader，ViewModel 确保设备配置改变后数据仍然存在，当数据发生变化时，Room 会通知 LiveData，而 LiveData 可以来更新 UI。如下图：

![效果图](https://developer.android.google.cn/images/topic/libraries/architecture/viewmodel-replace-loader.png)

## 注意事项

### 不要将 Context 传入 ViewModel

也就是说 Activity、Fragment和View都不能被传入，正如之前介绍，ViewModel 可以比相联的 Activity 和Fragment 的生命周期都要长。假设你在 ViewModel 中存储了一个Activity，当旋转屏幕导致Activity销毁时，但是 ViewModel 还保存着已经被销毁的 Activity 的引用，这样就会造成内存泄漏。
如果你需要比ViewModel生命周期长的Application类，可以使用 AndroidViewModel的子类，通过这个子类就可以直接使用 Application 的引用了。

### ViewModel 和 onSaveInstanceState 并用

ViewModel 不应该取代 onSaveInstanceState 的使用，他们两者是相辅相成的。当进程被关闭时，ViewModel将被销毁，但是 onSaveInstanceState 将不会受到影响。
另外，ViewModel可以用来存储大量数据，而 onSaveInstanceState 就只可以用来存储有限的数据了。我们尽可能把多一点的UI数据存储到ViewModel中。在Activity重新创建是不需要重新加载或者生成数据。
如果进程被关闭，我们应该用 onSaveInstanceState 来存储足够还原 UI 状态的最少数据。

## 源码分析

照例来看看源码实现。

### 创建实例

`ViewModelProviders.of(this).get(TimerViewModel.class)` 这个方法会返回一个 `TimerViewModel` 实例对象。
先来看一下 ViewModelProvider 的 get 方法：

```
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        // 首先从 ViewModelStore 根据key中获取缓存的实例
        // key 的形式 android.arch.lifecycle.ViewModelProvider.DefaultKey:com.example.heqiang.testsomething.jetpack.TimerViewModel
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            // 如果有缓存，那么返回缓存对象
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        // 如果没有，生成一个对象，并放入到 ViewModelStore
        viewModel = mFactory.create(modelClass);
        mViewModelStore.put(key, viewModel);
        //noinspection unchecked
        return (T) viewModel;
    }
```

ViewModelStore 其实就是内部维护了一个 HashMap 来存放 ViewModel 对象的。
ViewModelStore 是在 ViewModelProviders.of 方法中创建 ViewModelProvider 

```
    public static ViewModelProvider of(@NonNull FragmentActivity activity,
            @Nullable Factory factory) {
        Application application = checkApplication(activity);
        if (factory == null) {
            factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
        }
        return new ViewModelProvider(ViewModelStores.of(activity), factory);
    }
```

```
    public static ViewModelStore of(@NonNull FragmentActivity activity) {
        // 如果实现了 ViewModelStoreOwner 接口，那么直接调用个方法获取
        if (activity instanceof ViewModelStoreOwner) {
            return ((ViewModelStoreOwner) activity).getViewModelStore();
        }
        // 获取 HolderFragment 实例来获取 ViewModelStore
        return holderFragmentFor(activity).getViewModelStore();
    }
```

HolderFragment.holderFragmentFor()

```
        HolderFragment holderFragmentFor(FragmentActivity activity) {
            FragmentManager fm = activity.getSupportFragmentManager();
            // 获取 HolderFragment
            HolderFragment holder = findHolderFragment(fm);
            // 如果已经创建，返回
            if (holder != null) {
                return holder;
            }
            // 从缓存列表中获取
            holder = mNotCommittedActivityHolders.get(activity);
            if (holder != null) {
                return holder;
            }

            if (!mActivityCallbacksIsAdded) {
                mActivityCallbacksIsAdded = true;
                // 注册生命周期回调，当生命周期结束时remove缓存的 HolderFragment
                activity.getApplication().registerActivityLifecycleCallbacks(mActivityCallbacks);
            }
            // 创建 HolderFragment 并添加到 HolderFragment
            holder = createHolderFragment(fm);
            mNotCommittedActivityHolders.put(activity, holder);
            return holder;
        }
```

从上面这系列分析可以看到，ViewModel 生命周期和 ViewModelStore 相关，而 ViewModelStore 又和 HolderFragment 相关。
那么 HolderFragment 是什么呢？接下来分析。

### HolderFragment

```
public class HolderFragment extends Fragment implements ViewModelStoreOwner {
    private static final HolderFragmentManager sHolderFragmentManager = new HolderFragmentManager();

    private ViewModelStore mViewModelStore = new ViewModelStore();

    public HolderFragment() {
        setRetainInstance(true);
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        sHolderFragmentManager.holderFragmentCreated(this);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        mViewModelStore.clear();
    }

    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        return mViewModelStore;
    }
    ......
}
```

HolderFragment 继承自 Fragment 实现了 ViewModelStoreOwner 接口，在生命周期结束的 onDestroy() 执行 ViewModelStore.clear()，然后这个方法会调用 ViewModel 的 onCleared() 方法。
那么 HolderFragment 是怎么做到在 Activity 配置变更导致 Activity 重建的情况下而不销毁呢？
玄机就在 `setRetainInstance(true)` 这里。
我们称之为 Fragment 的非中断保存，设置了这个属性的 Fragment 当它依附的 Activity 由于配置变更导致的重建时，该 Fragment 不会销毁。
ViewModel 维持它的生命周期正式巧用了 Fragment 的这一特性。

### 总结

综合上面的源码分析，我们可以得到下面的结论：

 - ViewModel 的生命周期和 HolderFragment 保持同步。
 - HolderFragment 持有 ViewModelStore，ViewModelStore 持有 ViewModel，ViewModel 以键值对缓存在 ViewModelStore 的 HashMap 中。
 - 一个 Activity 或者 Fragment 只会有一个HolderFragment。
 - 一个 Activity 或者 Fragment 可以有多个 ViewModel。