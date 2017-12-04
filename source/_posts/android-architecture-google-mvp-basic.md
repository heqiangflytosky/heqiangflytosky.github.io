
---
title: Android 架构 android-architecture 之 todo-mvp 介绍
categories: Android 架构
comments: true
tags: [Android 架构, android-architecture]
description: 介绍 Google 开源的 Android 架构中的 MVP 架构
date: 2017-9-6 10:00:00
---

## 概述

todo-mvp 是对 MVP 的一种基础的实现，里面没有用到其他的框架，下面来具体分析一下。
[Github地址](https://github.com/googlesamples/android-architecture/tree/todo-mvp/)

## 源码简介

按功能划分模块，包结构如图：

![效果图](/images/android-architecture-google-mvp-basic/image1.png)

 1. tasks、addedittask、taskdetail和statistics：四个功能界面
 2. data：数据层的代码，即 MVP 中的 Model 层
 3. util：一些工具类
 4. BasePresenter和BaseView：Presenter和View的基类

各个包中的类：

![效果图](/images/android-architecture-google-mvp-basic/image2.png)

### 具体实现

 1. 定义一个协议类 `Contract`，它定义了两个接口 `View` 和 `Presenter`，分别继承自 V 和 P 的基类 `BaseView` 和 `BasePresenter`。
 2. 创建一个 `Presenter`，并实现协议类中的 `Presenter` 接口，实现`Presenter` 类的两个参数分别是 `TasksRepository` （M）和 `View` （V）实现类，建立 M 和 V 联系的桥梁。
 3. 在 `Activity` 中实例化 `Presenter`，建立 M 和 V 的联系，实例化 `Fragment`。
 4. 在 `Fragment` 中实现了 `View` 接口，作为 `View` （V）实现类。

V 层监听用户的操作，并把用户的操作传递到P层，并把P层中的指令转化为UI操作，并呈现出来。
P 层建立 M 和 V 的联系，响应 V 层的操作并向 M 层获取数据，然后传递到 V 层。 
M 层负责数据的存储与查询。

以 taskdetail 为例，用一个类图来展示它们的关系：

![效果图](http://www.plantuml.com/plantuml/svg/XLHBJiCm4Dtx55PNYL1xWYugg92Gg5JHgdjZ3wr5OaVsj1M2xhW5Kk_0XWqIjw5S0pjfVeQqMJbltlER6OyziiWChjE4p2KcG7kJnVIm_pZiNt_UFx_Vldg4I8LW7XW7EcVsqOuPifbU6mw4a4jcOI5XIuSl_NuU7mCocLnfXOPn7FWWwS2TQ31eYAuDMwQWa9IBSDUAaFjE3LZmkNMQLmnoAXYcKQj8K73Dj7UGQIjHcwSZgmRei9NDoIIADdJm6vsl-lnCBgW5h4ZHd6RbEYQxK5CNcGlzMKS1hIihBzXeARpThMP2gkMD4f8pLsDqhtK2J577bXk8A-fARoVIMiVrsqOnzLyPNKa1-PH5BK41pT0u5KN_4pSLOx3So0obLcrTCt1KYnewNsMxD_cs82GMYPU8W0GGsXZNQVONpLl1AjIvyuHP-y_uEh_JuqhJUWZE77VeqIog7uugJSFPxRop1RK8v7TuHqQWX7ieVW40)

### V 和 P 的基类

先来看一下 V 的基类 `BaseView<T>`：
```java
public interface BaseView<T> {
    void setPresenter(T presenter);
}
```

`BaseView` 是一个泛型接口，里面只有一个抽象方法 `setPresenter(T presenter)` ，用来设置 Presenter 。
P 的基类 `BasePresenter`：

```java
public interface BasePresenter {
    void start();
}
```

它也只定义来一个方法 `start()`，用来在 `Activity` 或者 Fragm`ent` 的 `onResume()` 方法中调用，来进行数据的查询。

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

在 `TaskDetailActivity` 中实例化 `TaskDetailFragment`、`TasksRepository` 和 `TaskDetailPresenter`，其中 `TaskDetailFragment` 和 `TasksRepository` 作为 `TaskDetailPresenter` 的参数。
来看一下 `TaskDetailPresenter` 的构造函数：

```java
    public TaskDetailPresenter(@Nullable String taskId,
                               @NonNull TasksRepository tasksRepository,
                               @NonNull TaskDetailContract.View taskDetailView) {
        mTaskId = taskId;
        mTasksRepository = checkNotNull(tasksRepository, "tasksRepository cannot be null!");
        mTaskDetailView = checkNotNull(taskDetailView, "taskDetailView cannot be null!");

        mTaskDetailView.setPresenter(this);
    }
```

在这里调用了 `TaskDetailFragment` 的 `setPresenter` 方法来设置 P 模块。

分别调用 `TasksRepository`  和 `TaskDetailFragment` 的相应的方法来实现数据的操作以及视图的更新。

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

`TaskDetailFragment` 中调用 `TaskDetailPresenter` 的对应方法来实现数据的增删改查。

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

App需要的数据都是通过Data模块来提供，由它去完成访问网络和数据库等。

![效果图](http://www.plantuml.com/plantuml/svg/dPB1IWCn48RlUOhGKqNR9-XXfO88RH5Rl8_9j0sTtPHaTa7KcsyXz1syU15y6-jh6BVhrYvn1M_9Fr--6S8adi5ndfAO6IQKdV7rvNRpijqyZgr6-dX-VNzwwmWn0x_oPy0mjRbJA0Vt_Ruimv5LGFjA2tc5Q-iDMtVR2gMMyOUlwjre8mUzNjpQ54H9OJ96DuTGRevo9uxb0hcCkyd4PfESI8uiw38Q0j4Dg9LKrU4ey8Kr-llH_isKdSaMaaDueKzadP_lmDzD7WeyL7tTIb7DA9kk2VcVtC5eDGkAJG5_E-DStAa8mGsh8NPVsAsB3kSE_RAHXhqBx2bHD6zj-Y0Ip7HOvqy0)

 - 接口 `TasksDataSource` 定义了操作数据的各种方法。
 - 类 `TasksLocalDataSource` 实现了`TasksDataSource` 接口，可以操作本地数据库来实现数据的存储和读取。 
 - 类 `TasksRemoteDataSource` 实现了`TasksDataSource` 接口，用来模拟网络数据的读取上上传。
 - 类 `TasksRepository` 实现了`TasksDataSource` 接口，提供来对数据访问的能力，它持有 `TasksLocalDataSource` 和 `TasksRemoteDataSource` 的对象，由它们真正实现对本地数据和网络数据的操作。

### util工具类

这部分不再介绍，想了解详情的看代码即可。
