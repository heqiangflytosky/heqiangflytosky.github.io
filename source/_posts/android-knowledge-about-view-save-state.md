---
title: Android -- View 之 onSaveInstanceState 深入理解
categories: Android
comments: true
tags: [onSaveInstanceState]
description: 基于 Android6.0 来分析 View  onSaveInstanceState 数据保存源码
date: 2018-8-15 10:00:00
---

## 概述

前面的文章[Android -- Activity 之 onSaveInstanceState 深入理解 ](http://www.heqiangfly.com/2018/08/12/android-knowledge-about-activity-save-state/) 介绍了 Activity 状态保存机制，本文介绍一下 View 的状态保存机制。
View 和 Activity 类似，也有两个状态保存的方法：

 - public Parcelable onSaveInstanceState()
 - public void onRestoreInstanceState(Parcelable state)

当 View 因意外销毁或者因旋转屏幕导致的销毁时，可以通过 onSaveInstanceState 方法保存我们需要保存的数据，当重新创建 View 时，可以通过 onRestoreInstanceState 方法来恢复我们的数据。

## 使用方法

在 View 类中，状态保存的相关方法有：

 - dispatchSaveInstanceState
 - onSaveInstanceState
 - restoreHierarchyState
 - dispatchRestoreInstanceState
 - onRestoreInstanceState

我们只要重新实现 onSaveInstanceState 和 onRestoreInstanceState 方法就能实现状态保存的需求。
一些 View 的派生类比如 TextView 也是覆盖了这些方法来实现状态保存的，比如文本是否选择，以及选择的起始和结束位置等。在状态保存之前可以先看父类是否已经为我们完成了我们的需求，如果没有，再自己实现既可。

```
public class CustomTextView extends EditText {
    ......
    @Override
    public Parcelable onSaveInstanceState() {
        Parcelable state = super.onSaveInstanceState();
        Bundle bundle = new Bundle();
        bundle.putParcelable("origin", state);
        bundle.putString("Test","Test");
        Log.e("Test","CustomTextView onSaveInstanceState "+state.toString());
        return bundle;
    }

    @Override
    public void onRestoreInstanceState(Parcelable state) {
        Bundle bundle = (Bundle) state;
        Parcelable superData = bundle.getParcelable("origin");
        String test = bundle.getString("Test");
        Log.e("Test","CustomTextView onRestoreInstanceState Test = "+test);
        super.onRestoreInstanceState(superData);
    }
    ......
}
```

## 源码分析

在前面的文章[Android -- Activity 之 onSaveInstanceState 深入理解 ](http://www.heqiangfly.com/2018/08/12/android-knowledge-about-activity-save-state/) 曾经提到过 Activity 的 onSaveInstanceState 方法会保存 View 的一些状态：

```
    protected void onSaveInstanceState(Bundle outState) {
        outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
        ......
    }
```

那么我们就接着 View 保存这部分接着分析。

`mWindow` 是 `PhoneWindow` 的实例，我们来看一下 `PhoneWindow` 的 `saveHierarchyState` 方法：

```
        Bundle outState = new Bundle();
        if (mContentParent == null) {
            return outState;
        }

        SparseArray<Parcelable> states = new SparseArray<Parcelable>();
        mContentParent.saveHierarchyState(states);
        outState.putSparseParcelableArray(VIEWS_TAG, states);

        // Save the focused view ID.
        // 保存当前获得焦点的 View 的ID
        final View focusedView = mContentParent.findFocus();
        // 如果 focusedView 不为空，且设置了id则会保存状态为 android:focusedViewId
        if (focusedView != null && focusedView.getId() != View.NO_ID) {
            outState.putInt(FOCUSED_ID_TAG, focusedView.getId());
        }

        // Save the accessibility focused view ID.
        if (mDecor != null) {
            final ViewRootImpl viewRootImpl = mDecor.getViewRootImpl();
            if (viewRootImpl != null) {
                final View accessFocusHost = viewRootImpl.getAccessibilityFocusedHost();
                if (accessFocusHost != null && accessFocusHost.getId() != View.NO_ID) {
                    outState.putInt(ACCESSIBILITY_FOCUSED_ID_TAG, accessFocusHost.getId());

                    // If we have a focused virtual node ID, save that too.
                    final AccessibilityNodeInfo accessFocusedNode =
                            viewRootImpl.getAccessibilityFocusedVirtualView();
                    if (accessFocusedNode != null) {
                        final int virtualNodeId = AccessibilityNodeInfo.getVirtualDescendantId(
                                accessFocusedNode.getSourceNodeId());
                        outState.putInt(ACCESSIBILITY_FOCUSED_VIRTUAL_ID_TAG, virtualNodeId);
                    }
                }
            }
        }

        // save the panels
        SparseArray<Parcelable> panelStates = new SparseArray<Parcelable>();
        savePanelState(panelStates);
        if (panelStates.size() > 0) {
            outState.putSparseParcelableArray(PANELS_TAG, panelStates);
        }

        if (mDecorContentParent != null) {
            SparseArray<Parcelable> actionBarStates = new SparseArray<Parcelable>();
            mDecorContentParent.saveToolbarHierarchyState(actionBarStates);
            outState.putSparseParcelableArray(ACTION_BAR_TAG, actionBarStates);
        }

        return outState;
    }
```

mContentParent 其实就是 id 为 `com.android.internal.R.id.content` 的 ViewGroup，在 ViewGroup 中没有重写 saveHierarchyState 方法，因此 `mContentParent.saveHierarchyState(states)` 调用的是 View 的 saveHierarchyState 方法。

View.saveHierarchyState

```
    public void saveHierarchyState(SparseArray<Parcelable> container) {
        dispatchSaveInstanceState(container);
    }
    
    protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
        // 是否设置id以及 saveEnabled 属性是否打开
        if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
            mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
            Parcelable state = onSaveInstanceState();
            if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
                throw new IllegalStateException(
                        "Derived class did not call super.onSaveInstanceState()");
            }
            if (state != null) {
                // 将当前 View id 和 state 键值对保存到 container 中
                container.put(mID, state);
            }
        }
    }
```

从上面的 dispatchSaveInstanceState 可以看出，View 必须设置 id，否则是不会调用 onSaveInstanceState 来保存状态的。

## 和 android:id 的关系

结合上面的 dispatchSaveInstanceState 方法还有下面的 dispatchRestoreInstanceState 方法，可以得出结论，View 的 onSaveInstanceState 和 onRestoreInstanceState 方法的调用，都必须为这个 View 设置 id。

```
    protected void dispatchRestoreInstanceState(SparseArray<Parcelable> container) {
        if (mID != NO_ID) {
            // 从 SparseArray 中根据 id 去除 state
            Parcelable state = container.get(mID);
            if (state != null) {
                mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
                onRestoreInstanceState(state);
                if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
                    throw new IllegalStateException(
                            "Derived class did not call super.onRestoreInstanceState()");
                }
            }
        }
    }
```

在上面方法中，状态的恢复是从 SparseArray 中根据 View 的 id 来去除状态的来进行保存并调用 onRestoreInstanceState 方法，这里就要注意，在当前 View 树中不要设置两个相同的 id。否则这里的状态可能会发生覆盖的现象。这种情况很容易出现在 ViewPager 这种多页面的布局中。

其实Android官方的说法是在整个view树中id不一定非要唯一，但你至少要
保证在你搜索的这部分view树中是唯一的（局部唯一）。因为很显然，如果同一个 layout 文件中有2个 id 都是 "android:id="@+id/button"
的 Button，那你通过 findViewById 的时候只能找到前面的 button，后面的那个就没机会被找到了，所以 Android 的说法是合理的。

## 和 android:saveEnabled 的关系

前面的 View 的 dispatchSaveInstanceState 方法中，

```
    protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
        // 是否设置id以及 saveEnabled 属性是否打开
        if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
            ......
        }
    }
```

`(mViewFlags & SAVE_DISABLED_MASK) == 0 `满足条件才会调用 onSaveInstanceState 保存状态，SAVE_DISABLED_MASK 是通过 android:saveEnabled 来设置的，默认是 true。