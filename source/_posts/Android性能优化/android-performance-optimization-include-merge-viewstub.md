---
title: Android 性能优化理论篇 -- 布局优化三剑客include、merge、ViewStub
categories: Android性能优化
comments: true
tags: [稳定性]
description: 介绍include、merge、ViewStub 的使用
date: 2017-8-20 10:00:00
---

## 概述

Layout的性能也是影响Android应用用户体验的关键部分，Layout的优化是减少内存使用，构建流畅UI的关键。
Android为我们提供了include、merge、ViewStub三剑客来帮助我们优化Layout布局，使用好它们可以解决我们在开发过程中遇到的下面的问题：

 - 某个布局在多个地方会被重复使用，相同的代码在多个地方重复出现是我们开发要尽可能避免的，那么怎么才能解决这个问题呢？可以使用include。
 - 使用了include，我们可能会发现，复用布局的地方会额外地套了一层无用的布局，增加了布局的层级，这种情况是需要我们进行优化的。那么这种情况下，merge就可以出场了。
 - 在有些情况下，一些布局是在某些条件下才显示的，虽然我们可以设置布局的可见性为invisible或者none来解决，但是这样还是会占用一些内存。这种情况使用 ViewStub 可以解决。


## include

使用 include 标签可以实现重用我们自定义的 Layout 布局，提高了布局重用性，提升我们的开发效率。
首先我们定义一个通用布局：

```
<?xml version="1.0" encoding="utf-8"?>
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="include 测试">
</TextView>
```

这个布局比较简单，就一个 TextView，现在我们通过 include 标签把它添加到布局中：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/root_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
    <include
        android:id="@+id/include_view"
        layout="@layout/include_test1"/>
</LinearLayout>
```

这里简单使用了它的两个属性：

 - layout：必填属性，指定加入的布局的名称。
 - android:id：设置加入布局的id。

当然还有其他很多的属性，开发时可以按需填写。
这里就引申出一个问题，如果我们在include里面和布局的根节点同时设置了相同的属性，哪个优先级更高呢？
答案是 include 设置的会覆盖掉布局里面相同的设置。
上面的例子插入布局后，整体布局为：

```
<LinearLayout>
    <TextView/>
</LinearLayout>
```

由于插入布局很简单就一个 TextView，没有使用额外的根布局，因此，此时 include 的使用不会导致出现一层无用的额外布局。
下面我们在插入布局里面使用两个TextView组件：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:gravity="center"
    android:background="#ff0000"
    android:orientation="horizontal">
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Test1"/>
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Test2"/>
</LinearLayout>
```

同样在上面的例子中引入该布局，那么整体布局就变为：

```
<LinearLayout>
    <LinearLayout>
        <TextView/> <TextView/>
    </LinearLayout>
</LinearLayout>
```

这样就出现了我们前面提到的问题：多出了一层额外的无用布局。

## merge

为了避免出现出现上面介绍的include在某些情况下引入的一层额外无用的布局，我们在一些情况下可以使用 merge 标签，之所以是在一些情况下，是因为它的使用有一些局限性：即你需要明确将merge里面的布局和控件将要include到什么类型的布局中，才能提前设置好merge里面的布局和控件的位置等属性。
merge 标签必须是一个布局文件中的根节点，看起来跟其他布局没什么区别，但它的特别之处在于页面加载时它的不会绘制的。它不会占用任何空间，因此也就不会增加布局层级了。这正如它的名字一样，它只起“合并”作用。

```
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android">
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Test1"/>
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Test2"/>
</merge>
```

引用布局：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/root_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
    <include
        android:id="@+id/include_view"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        layout="@layout/include_test1"/>
    <include
        android:id="@+id/include_view"
        layout="@layout/include_test2"/>
    <include
        android:id="@+id/include_view"
        layout="@layout/include_test3"/>

</LinearLayout>
```

### merge的属性

在前面使用 include 时我们说过，include 中设置的属性，会覆盖掉被引用布局根布局的属性。由于 merge 标签并不会作为一个布局绘制出来，因此我们给 include 设置各种属性是不起作用的，比如id、merge的ID等等。

## ViewStub

我们在开发中一定遇到这样的情况：有些组件的出现是需要一定的条件的，即不是在初始化的时候需要显示出来的，但是我们又需要再布局中先设置好。虽然说过 visibility 的 invisible 和 gone 可以来实现这样的目的，但是这样的用法导致在初始化的时候该布局还是会被初始化和加载，我们使用布局查看工具发现，该布局存在于布局中。但是有可能该布局出现的条件根本不会出现。那么这样无疑会影响应用的性能，降低UI的加载速度。
这时候我们可以考虑使用 ViewStub。 它具有懒加载功能，在视图中具有占位功能，只有调用了 `inflate()` 或者 `setVisibility(View.VISIBLE)` 之后才真正的渲染视图，因此，它不会影响系统的初始化加载速度。

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/root_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
    <ViewStub
        android:id="@+id/view_stub"
        android:inflatedId="@id/root_view"
        android:layout="@layout/include_test2"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        />
    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="显示ViewStub"
        android:onClick="showViewStub"/>

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="隐藏ViewStub"
        android:onClick="hideViewStub"/>

</LinearLayout>
```

 - android:id：ViewStub 的id。
 - android:inflatedId：重写被填充的视图的父布局id。
 - android:layout：ViewStub 需要填充的布局的名称

ViewStub 还提供了 OnInflateListener接口，用于监听布局是否已经加载了。

```
        mViewStub = findViewById(R.id.view_stub);
        mViewStub.setOnInflateListener(new ViewStub.OnInflateListener() {
            @Override
            public void onInflate(ViewStub stub, View inflated) {
                Log.e("Test","onInflate");
            }
        });
```

显示视图：

```
mViewStub.inflate();
或者 mViewStub.setVisibility(View.VISIBLE);
```

隐藏视图：

```
mViewStub.setVisibility(View.GONE);
```

注意：`mViewStub.inflate();` 只能调用一次，如果重复调用会抛出 `java.lang.IllegalStateException: ViewStub must have a non-null ViewGroup viewParent` 异常，如果我们想在 ViewStub 加载后控制它的显示和隐藏，可以通过 `setVisibility` 方法来控制。

## ViewStub 源码分析

ViewStub 继承自 View，并重写了下面的方法：

 - onMeasure
 - draw
 - dispatchDraw
 - setVisibility

既然 ViewStub 继承自 View，那么它是如何进行布局优化的呢？
我们首先来看一下它的下面方法：

```
    public ViewStub(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context);
        // 获取 android:inflatedId 和 android:layout 属性
        final TypedArray a = context.obtainStyledAttributes(attrs,
                R.styleable.ViewStub, defStyleAttr, defStyleRes);
        mInflatedId = a.getResourceId(R.styleable.ViewStub_inflatedId, NO_ID);
        mLayoutResource = a.getResourceId(R.styleable.ViewStub_layout, 0);
        mID = a.getResourceId(R.styleable.ViewStub_id, NO_ID);
        a.recycle();
        // 设置view不可见
        setVisibility(GONE);
        // 设置不绘制视图
        setWillNotDraw(true);
    }
    
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(0, 0);
    }

    @Override
    public void draw(Canvas canvas) {
    }

    @Override
    protected void dispatchDraw(Canvas canvas) {
    }
```

此时就可以回答上面的问题了：ViewStub 设置了大小为0，设置了不绘制视图，因此，系统为ViewStub几乎没有付出什么代价，它也就只完成了占位的作用。等待调用 `inflate`或者`setVisibility` 来显示。

再来看一下 `inflate` 方法：

```
    public View inflate() {
        final ViewParent viewParent = getParent();

        if (viewParent != null && viewParent instanceof ViewGroup) {
            if (mLayoutResource != 0) {
                final ViewGroup parent = (ViewGroup) viewParent;
                final View view = inflateViewNoAdd(parent);
                // 把ViewStub自身从布局中移除，把view添加到布局中
                // 把自身异常后，如果继续调用 inflate()。会导致 viewParent为空，从而抛异常
                replaceSelfWithView(view, parent);

                mInflatedViewRef = new WeakReference<>(view);
                if (mInflateListener != null) {
                    mInflateListener.onInflate(this, view);
                }

                return view;
            } else {
                throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
            }
        } else {
            // 如果父组件为空或者不是ViewGroup，这里就抛出异常
            throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
        }
    }
    
    private void replaceSelfWithView(View view, ViewGroup parent) {
        final int index = parent.indexOfChild(this);
        // 在这里把ViewStub自身从布局中移除
        parent.removeViewInLayout(this);

        final ViewGroup.LayoutParams layoutParams = getLayoutParams();
        // 添加android:layout到当前布局中
        if (layoutParams != null) {
            parent.addView(view, index, layoutParams);
        } else {
            parent.addView(view, index);
        }
    }
```

再来看一下 setVisibility 方法：

```
    @Override
    @android.view.RemotableViewMethod(asyncImpl = "setVisibilityAsync")
    public void setVisibility(int visibility) {
        // 如果 mInflatedViewRef 不为空，就调用 setVisibility
        if (mInflatedViewRef != null) {
            View view = mInflatedViewRef.get();
            if (view != null) {
                view.setVisibility(visibility);
            } else {
                throw new IllegalStateException("setVisibility called on un-referenced view");
            }
        } else {
            // 如果 mInflatedViewRef 为空，就调用 inflate 方法
            super.setVisibility(visibility);
            if (visibility == VISIBLE || visibility == INVISIBLE) {
                inflate();
            }
        }
    }
```

可见，setVisibility 操作的是ViewStub要填充的视图的可见性，并不是 ViewStub本身。
那么 getVisibility 方法呢？ViewStub并没有重写该方法，因此调用还是View的方法，指的是ViewStub自身的属性。
如果我们调用的是 `inflate()` 方法显示布局，那么getVisibility获得的值永远是初始设置的 GONE，如果是调动的ViewStub.setVisibility(View.VISIBLE)显示，由于在 setVisibility 方法里面调用了`super.setVisibility(visibility)`，而且后面在mInflatedViewRef不为空的情况下没有设置该属性，那么获取的值永远为VISIBLE。

<!-- 
https://mp.weixin.qq.com/s/DaEukO0oNzvHij2XOknHTg
-->