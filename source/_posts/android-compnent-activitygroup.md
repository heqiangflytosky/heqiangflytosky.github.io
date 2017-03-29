---
title: Android ActivityGroup简介
categories: Android
comments: true
tags: [Android, ActivityGroup]
description: 介绍ActivityGroup的使用
date: 2015-1-8 10:00:00
---

## ActivityGroup简介
`ActivityGroup`可以看作是`Activity`的容器，可以包含多个嵌套进来的`Activity`。
我们只需要知道有这个类就可以了，现在这个类现在已经不推荐使用了，是`deprecated`状态的，一般情况下是可以被`Fragment`取代的，但是在一些特殊场合下仍然是被需要的。
下面通过一个Demo来介绍`ActivityGroup`的使用。
## 示例
### 代码

MainActivity.java
```
public class MainActivity extends ActivityGroup {

    private ViewGroup mLeft;
    private ViewGroup mRight;
    private LocalActivityManager manager;
    private View mRoot;
    private static WeakReference<MainActivity> mInstance;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mInstance = new WeakReference<MainActivity>(this);
        setContentView(R.layout.activity_main);
        manager = getLocalActivityManager();
        mRoot = findViewById(R.id.container_main);
        mLeft = (ViewGroup) findViewById(R.id.container_left);
        mRight = (ViewGroup) findViewById(R.id.container_right);

        startLeftActivity();
    }

    public static MainActivity getInstance(){
        return mInstance != null ? mInstance.get() : null;
    }

    public void startLeftActivity(){
        mLeft.removeAllViews();
        Intent i = new Intent(this, LeftActivity.class);
        mLeft.addView(manager.startActivity("LeftActivity", i).getDecorView());
    }

    public void startRightActivity(){
        mRight.removeAllViews();
        Intent i = new Intent(this, RightActivity.class);
        mRight.addView(manager.startActivity("RightActivity", i).getDecorView());
    }
}
```
activity_main.xml
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    android:id="@+id/container_main"
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal"
    android:showDividers="middle"
    android:divider="?android:attr/dividerHorizontal"
    tools:context="com.android.hq.testapplication.MainActivity">

    <FrameLayout
        android:id="@+id/container_left"
        android:layout_width="150dp"
        android:layout_height="match_parent"
        android:layout_weight="0">

    </FrameLayout>

    <FrameLayout
        android:id="@+id/container_right"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1">
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:text="Empty"/>
    </FrameLayout>
</LinearLayout>
```
LeftActivity.java
```
public class LeftActivity extends Activity{
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_left);
    }

    public void startRightActivity(View v){
        MainActivity.getInstance().startRightActivity();
    }
}
```

### 效果

在启动右边的`Activity`之前，右边区域是空的：
<img src="/images/android-compnent-activitygroup/image1.png" width="360" height="620"/>
点击按钮启动右边`Activity`：
<img src="/images/android-compnent-activitygroup/image2.png" width="360" height="620"/>

## 源码分析
从源码中我们可以看到`ActivityGroup`类的父类是`Activity`，也就是说二者具有相同的方法和生命周期。在`ActivityGroup`有成员变量`protected LocalActivityManager mLocalActivityManager`，那么`ActivityGroup`对`Activity`的管理就要以来这个变量来管理了。
在`ActivityGroup`的`onCreate``onResume``onPause`等生命周期函数里面分别调用
```
        Bundle states = savedInstanceState != null
                ? (Bundle) savedInstanceState.getBundle(STATES_KEY) : null;
        mLocalActivityManager.dispatchCreate(states);
```
```
mLocalActivityManager.dispatchResume();
```
```
mLocalActivityManager.dispatchPause(isFinishing());
```
来保证子`Activity`的生命周期与`ActivityGroup`一致。
`LocalActivityManager`通过`startActivity(String id,Intent intent)`这个方法获取当前`Window`对象，再然后调用`getDecorView()`方法获取当前`Activity`对应的`View`，这个就把多个`Activity`添加到一个`RootView`里面了。
具体是怎么获得我们需要`Activity`的`DecorView`的呢？我们就要看一下`LocalActivityManager`的`startActivity`这个方法了。
首先要获取主线程的`mActivityThread`：
```
mActivityThread = ActivityThread.currentActivityThread();
```
```
            r.activity = mActivityThread.startActivityNow(
                    mParent, r.id, r.intent, r.activityInfo, r, r.instanceState, instance);
            if (r.activity == null) {
                return;
            }
            r.window = r.activity.getWindow();
```
这里调用了`mActivityThread.startActivityNow`方法，在`startActivityNow`内部会调用`ActivityThread.performLaunchActivity`来装载这个`Activity`，然后通过`activity.getWindow()`获得`Window`对象，在通过`Window`获得`DecorView`。
