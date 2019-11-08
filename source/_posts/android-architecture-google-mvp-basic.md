
---
title: Android 架构 android-architecture 之 mvp 介绍
categories: Android 架构
comments: true
tags: [Android 架构, android-architecture]
description: 介绍 Google 开源的 Android 架构中的 MVP 架构
date: 2017-9-6 10:00:00
---

## 概述

Android 官方给出了在 Android 开发中使用 MVP 架构的范例，我们可以在我们的项目中做为参考。
todo-mvp 是对 MVP 的一种基础的实现，里面没有用到其他的框架。
todo-mvp-rxjava 是使用 RxJava 搭建的响应式MVP架构。
先来看一下 MVP 整个项目的逻辑图：

![效果图](/images/android-architecture-google-mvp-basic/mvp.png)

下面来具体分析一下。
[todo-mvp Github地址](https://github.com/googlesamples/android-architecture/tree/todo-mvp/)
[todo-mvp-rxjava Github地址](https://github.com/android/architecture-samples/tree/todo-mvp-rxjava)

## todo-mvp 源码简介

按功能划分模块，包结构如图：

![效果图](/images/android-architecture-google-mvp-basic/image1.png)

 1. tasks、addedittask、taskdetail和statistics：四个功能界面
 2. data：数据层的代码，即 MVP 中的 Model 层
 3. util：一些工具类
 4. BasePresenter和BaseView：Presenter和View的基类

各个包中的类：

![效果图](/images/android-architecture-google-mvp-basic/image2.png)

### 具体实现

 1. 定义一个协议类 `Contract`，它定义了两个接口 `View` 和 `Presenter`，分别继承自 V 和 P 的基类 `BaseView` 和 `BasePresenter`。把 V 和 P 的接口统一写在这个协议类里面，能够更清晰的看到在 V 层和 P 层有哪些功能，方便我们以后的维护。这是其他MVP框架没有的类。
 2. 创建一个 `Presenter`，并实现协议类中的 `Presenter` 接口，实现`Presenter` 类的两个参数分别是 `TasksRepository` （M）和 `View` （V）实现类，建立 M 和 V 联系的桥梁，将 M 层和 V 层联系在一起。
 3. 在 `Activity` 中实例化 `Presenter`，建立 M 和 V 的联系，实例化 `Fragment`。在这里的 MVP 架构例子中，Activity 没有负责任何的 View 层功能，仅仅是负责对 P 层 和 M/V 层的绑定。
 4. 在 `Fragment` 中实现了 `View` 接口，作为 `View` （V）实现类。

V 层监听用户的操作，并把用户的操作传递到P层，并把P层中的指令转化为UI操作，并呈现出来。
P 层建立 M 和 V 的联系，响应 V 层的操作并向 M 层获取数据，然后传递到 V 层。 
M 层负责数据的存储与查询。
可以看出，V 层和 M 层不再存在直接耦合，P 层是它们之间联系的桥梁。

以 taskdetail 为例，用一个类图来展示它们的关系：

![效果图](http://www.plantuml.com/plantuml/svg/XLHBJiCm4Dtx55PNYL1xWYugg92Gg5JHgdjZ3wr5OaVsj1M2xhW5Kk_0XWqIjw5S0pjfVeQqMJbltlER6OyziiWChjE4p2KcG7kJnVIm_pZiNt_UFx_Vldg4I8LW7XW7EcVsqOuPifbU6mw4a4jcOI5XIuSl_NuU7mCocLnfXOPn7FWWwS2TQ31eYAuDMwQWa9IBSDUAaFjE3LZmkNMQLmnoAXYcKQj8K73Dj7UGQIjHcwSZgmRei9NDoIIADdJm6vsl-lnCBgW5h4ZHd6RbEYQxK5CNcGlzMKS1hIihBzXeARpThMP2gkMD4f8pLsDqhtK2J577bXk8A-fARoVIMiVrsqOnzLyPNKa1-PH5BK41pT0u5KN_4pSLOx3So0obLcrTCt1KYnewNsMxD_cs82GMYPU8W0GGsXZNQVONpLl1AjIvyuHP-y_uEh_JuqhJUWZE77VeqIog7uugJSFPxRop1RK8v7TuHqQWX7ieVW40)

<!--  
@startuml
Title "MVP架构类图"

interface TasksDataSource


BaseView <|-- TaskDetailContract.View
TaskDetailContract.View <|.. TaskDetailFragment
Fragment <|-- TaskDetailFragment

BasePresenter <|-- TaskDetailContract.Presenter
TaskDetailContract.Presenter <|.. TaskDetailPresenter

TasksDataSource <|.. TasksRepository

TasksRepository <-- TaskDetailPresenter
TaskDetailContract.View <-- TaskDetailPresenter

interface BaseView {
+ setPresenter(T presenter)
}

interface BasePresenter {
+ start()
}

interface TaskDetailContract.View {
+ void setLoadingIndicator(boolean active)
+ void showMissingTask()     
+ void hideTitle()
+ void showTitle(String title)
+ void hideDescription()
+ void showDescription(String description)
}

interface TaskDetailContract.Presenter {
+ void editTask()
+ void deleteTask()
+ void completeTask()
+ void activateTask()
}

class TaskDetailFragment {
- TaskDetailContract.Presenter mPresenter
+ setPresenter(T presenter)
}

class TaskDetailPresenter {
- TasksRepository mTasksRepository
- TaskDetailContract.View mTaskDetailView
}
@enduml
-->

### V 和 P 的基类

先来看一下 V 的基类 `BaseView<T>`：
```java
public interface BaseView<T> {
    void setPresenter(T presenter);
}
```

`BaseView` 是一个泛型接口，里面只有一个抽象方法 `setPresenter(T presenter)` ，用来设置 Presenter ，实现 P 层的注入。设置的时机是在 Presenter 实现类的构造方法中。
P 的基类 `BasePresenter`：

```java
public interface BasePresenter {
    void start();
}
```

它也只定义来一个方法 `start()`，用来在 `Activity` 或者 `Fragment` 的 `onResume()` 方法中调用，来进行数据的查询。

### 功能模块

 - tasks：包括菜单抽屉界面，主界面显示任务列表
 - addedittask：点击添加任务界面
 - taskdetail：点击任务后的详情界面
 - statistics：任务统计界面，点击抽屉菜单可以弹出

这四个包对应了四个功能界面，它们内部的结构大同小异，包括了：

 - Activity
 - Fragment
 - Contract
 - Presenter

那么我们就选取其中的 taskdetail 模块来进行介绍。

#### Contract

`TaskDetailContract` 类功能比较简单，定义了 View 和 Presenter 接口，通过这个类，我们可以大概了解 V 层和 P 层具体实现的逻辑功能。

```
public interface TaskDetailContract {
    interface View extends BaseView<Presenter> {
        void setLoadingIndicator(boolean active);
        void showMissingTask();
        void hideTitle();
        void showTitle(String title);
        void hideDescription();
        void showDescription(String description);
        void showCompletionStatus(boolean complete);
        void showEditTask(String taskId);
        void showTaskDeleted();
        void showTaskMarkedComplete();
        void showTaskMarkedActive();
        boolean isActive();
    }

    interface Presenter extends BasePresenter {
        void editTask();
        void deleteTask();
        void completeTask();
        void activateTask();
    }
}
```

#### Presenter

在 `TaskDetailActivity` 中实例化 `TaskDetailFragment`、`TasksRepository` 和 `TaskDetailPresenter`，其中 `TaskDetailFragment` 和 `TasksRepository` 作为 `TaskDetailPresenter` 的参数。

```
        TaskDetailFragment taskDetailFragment = (TaskDetailFragment) getSupportFragmentManager()
                .findFragmentById(R.id.contentFrame);

        ......

        // Create the presenter
        new TaskDetailPresenter(
                taskId,
                Injection.provideTasksRepository(getApplicationContext()),
                taskDetailFragment);
```

来看一下 `TaskDetailPresenter` 的构造函数：

```java
    public TaskDetailPresenter(@Nullable String taskId,
                               @NonNull TasksRepository tasksRepository,
                               @NonNull TaskDetailContract.View taskDetailView) {
        mTaskId = taskId;
        mTasksRepository = checkNotNull(tasksRepository, "tasksRepository cannot be null!");
        mTaskDetailView = checkNotNull(taskDetailView, "taskDetailView cannot be null!");

        mTaskDetailView.setPresenter(this); // 为 V 层 注入 P 层
    }
```

在这里调用了 `TaskDetailFragment` 的 `setPresenter` 方法来设置 P 模块。`TaskDetailView` 作为参数传递到 `TaskDetailPresenter`。那么，这里的 V 和 P 是互相注入的。

在这里可以分别调用 `TasksRepository`（M）  和 `TaskDetailFragment`（V） 的相应的方法来实现数据的操作以及视图的更新。比如从 M 层得到数据，进行逻辑操作后分发给 V 层用来 UI 显示，或者 V 层响应用户操作后，把进行逻辑操作后的数据发给 M 层进行数据的保存。

```java

    @Override
    public void start() {
        openTask();
    }

    private void openTask() {
        if (Strings.isNullOrEmpty(mTaskId)) {
            mTaskDetailView.showMissingTask();
            return;
        }

        mTaskDetailView.setLoadingIndicator(true);
        mTasksRepository.getTask(mTaskId, new TasksDataSource.GetTaskCallback() {
            @Override
            public void onTaskLoaded(Task task) {
                // The view may not be able to handle UI updates anymore
                if (!mTaskDetailView.isActive()) {
                    return;
                }
                mTaskDetailView.setLoadingIndicator(false);
                if (null == task) {
                    mTaskDetailView.showMissingTask();
                } else {
                    showTask(task);
                }
            }

            @Override
            public void onDataNotAvailable() {
                // The view may not be able to handle UI updates anymore
                if (!mTaskDetailView.isActive()) {
                    return;
                }
                mTaskDetailView.showMissingTask();
            }
        });
    }
    
    private void showTask(@NonNull Task task) {
        String title = task.getTitle();
        String description = task.getDescription();

        if (Strings.isNullOrEmpty(title)) {
            mTaskDetailView.hideTitle();
        } else {
            mTaskDetailView.showTitle(title);
        }

        if (Strings.isNullOrEmpty(description)) {
            mTaskDetailView.hideDescription();
        } else {
            mTaskDetailView.showDescription(description);
        }
        mTaskDetailView.showCompletionStatus(task.isCompleted());
    }
```

#### View 

这里把 Fragment 来作为 View 层的实现类，`TaskDetailFragment` 中调用 `TaskDetailPresenter` 的对应方法来针对UI上面的操作进行相应数据的变更。

```java
    @Override
    public void onResume() {
        super.onResume();
        mPresenter.start();
    }
    
    ...
    
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mPresenter.editTask();
            }
        });
    ...
    
        @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            case R.id.menu_delete:
                mPresenter.deleteTask();
                return true;
        }
        return false;
    }
    
        @Override
    public void showCompletionStatus(final boolean complete) {
        Preconditions.checkNotNull(mDetailCompleteStatus);

        mDetailCompleteStatus.setChecked(complete);
        mDetailCompleteStatus.setOnCheckedChangeListener(
                new CompoundButton.OnCheckedChangeListener() {
                    @Override
                    public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
                        if (isChecked) {
                            mPresenter.completeTask();
                        } else {
                            mPresenter.activateTask();
                        }
                    }
                });
    }
```
### Data Model层

Data Model层是业务逻辑处理层，App需要的数据都是通过Data模块来提供，由它去完成访问网络或者数据库等。

![效果图](http://www.plantuml.com/plantuml/svg/dPB1IWCn48RlUOhGKqNR9-XXfO88RH5Rl8_9j0sTtPHaTa7KcsyXz1syU15y6-jh6BVhrYvn1M_9Fr--6S8adi5ndfAO6IQKdV7rvNRpijqyZgr6-dX-VNzwwmWn0x_oPy0mjRbJA0Vt_Ruimv5LGFjA2tc5Q-iDMtVR2gMMyOUlwjre8mUzNjpQ54H9OJ96DuTGRevo9uxb0hcCkyd4PfESI8uiw38Q0j4Dg9LKrU4ey8Kr-llH_isKdSaMaaDueKzadP_lmDzD7WeyL7tTIb7DA9kk2VcVtC5eDGkAJG5_E-DStAa8mGsh8NPVsAsB3kSE_RAHXhqBx2bHD6zj-Y0Ip7HOvqy0)

 - 接口 `TasksDataSource` 定义了操作数据的各种方法。
 - 类 `TasksLocalDataSource` 实现了`TasksDataSource` 接口，可以操作本地数据库来实现数据的存储和读取。 
 - 类 `TasksRemoteDataSource` 实现了`TasksDataSource` 接口，用来模拟网络数据的读取上上传。
 - 类 `TasksRepository` 实现了`TasksDataSource` 接口，提供来对数据访问的能力，它持有 `TasksLocalDataSource` 和 `TasksRemoteDataSource` 的对象，来决定调用哪个对象类完成数据请求，再由它们真正实现对本地数据或者网络数据的操作。

### util工具类

这部分不再介绍，想了解详情的看代码即可。

## todo-mvp-rxjava

在 mvp-rxjava 中，P 层的基类的方法有所不同：

```
public interface BasePresenter {
    void subscribe(); //开启订阅
    void unsubscribe(); //取消订阅
}
```
这么做是由RxJava 的特点决定的，在需要数据的时候开始订阅数据，接收数据。不再需要数据就取消订阅数据，让数据不再发送。
然后在 View 层的具体实现类Fragment中就做订阅和取消订阅两步操作，同步P层和V层的生命周期：

```
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
```

主要区别在 Presenter 的实现类 TaskDetailPresenter 中：

```
public class TaskDetailPresenter implements TaskDetailContract.Presenter {

    @NonNull
    private final TasksRepository mTasksRepository;

    @NonNull
    private final TaskDetailContract.View mTaskDetailView;

    @NonNull
    private final BaseSchedulerProvider mSchedulerProvider;

    @Nullable
    private String mTaskId;

    @NonNull
    private CompositeDisposable mCompositeDisposable;

    public TaskDetailPresenter(@Nullable String taskId,
                               @NonNull TasksRepository tasksRepository,
                               @NonNull TaskDetailContract.View taskDetailView,
                               @NonNull BaseSchedulerProvider schedulerProvider) {
        this.mTaskId = taskId;
        mTasksRepository = checkNotNull(tasksRepository, "tasksRepository cannot be null!");
        mTaskDetailView = checkNotNull(taskDetailView, "taskDetailView cannot be null!");
        mSchedulerProvider = checkNotNull(schedulerProvider, "schedulerProvider cannot be null");

        mCompositeDisposable = new CompositeDisposable();
        mTaskDetailView.setPresenter(this);
    }

    @Override
    public void subscribe() {
        openTask();
    }

    @Override
    public void unsubscribe() {
        mCompositeDisposable.clear();
    }

    ....
}
```

使用 CompositeDisposable 来管理 Disposable。

## 总结

通过上面的 MVP 结构可以看出，MVP 的设计使得程序的各个层次之间的职责更加单一，很大程度上降低了代码的耦合度。
