---
title: Android View -- RecycleView 的基本特点和使用方法
categories: Android
comments: true
tags: [RecycleView]
description: 介绍 RecycleView 的基本特点和使用方法
date: 2017-8-26 10:00:00
---

## 概述

RecycleView 是 Android support-v7 包中提供的组件，其灵活易用的特点使其在一些列表场景中深得我们的喜爱。
RecycleView 的特点：数据绑定，Item的创建和View的回收复用机制等。
本文简单介绍一下 RecycleView 的一些使用方法。

## 基础用法

### 引入依赖

```
com.android.support:recyclerview-v7:26.1.0
```

另外在 `com.android.support:design` 包中默认引入了 recyclerview-v7 包。如果使用了 `com.android.support:design` 就不需手动添加。

### 添加 RecycleView 布局

```
    <android.support.v7.widget.RecyclerView
        android:id="@+id/recycle_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
```

### 初始化 RecycleView

```
        mRecycleView = findViewById(R.id.recycle_view);
        // 设置元素之间的间隔
        mRecycleView.addItemDecoration(new DividerItemDecoration(this,
                DividerItemDecoration.VERTICAL));
        // 设置 Item 动画，为增加和删除 Item 时添加动画
        mRecycleView.setItemAnimator(new DefaultItemAnimator());
        // 设置布局管理器
        mRecycleView.setLayoutManager(new LinearLayoutManager(this,LinearLayoutManager.VERTICAL,false));

        mAdapter = new ListAdapter();
        // 设置Adapter
        mRecycleView.setAdapter(mAdapter);
```

### Adapter

我们需要继承 `RecyclerView.Adapter` 来实现 Adapter。

 - onCreateViewHolder(ViewGroup parent, int viewType)：生成 ViewHolder，viewType 为 ViewHolder 的类型，根据 getItemViewType(int position) 的返回值来确定。如果不重新实现 getItemViewType，将默认返回 0。
 - onBindViewHolder(RecyclerView.ViewHolder holder, int position)：为当前元素绑定 ViewHolder。这里的 ViewHolder 有可能是复用以前使用过的。
 - getItemCount()：获取元素的数量。
 - getItemViewType(int position)：获取当前元素的 ViewHolder 类型。

下面看一个简单的实现：

```
public class ListAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    private static final int TYPE_A = 1;
    private static final int TYPE_B = 2;

    private List<Data.DataHolder> mData;
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        switch (viewType) {
            case TYPE_A:
                return new ViewHolder.ViewHolderA(parent);
            case TYPE_B:
                return new ViewHolder.ViewHolderB(parent);
        }
        return null;
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
        if (holder instanceof ViewHolder.ViewHolderA) {
            ViewHolder.ViewHolderA viewHolderA = (ViewHolder.ViewHolderA) holder;
            Data.DataHolderA dataHolderA = (Data.DataHolderA)mData.get(position);
            viewHolderA.mTextView.setText(dataHolderA.text);
        } else if (holder instanceof ViewHolder.ViewHolderB) {
            ViewHolder.ViewHolderB viewHolderB = (ViewHolder.ViewHolderB) holder;
            Data.DataHolderB dataHolderB = (Data.DataHolderB)mData.get(position);
            viewHolderB.mImageView.setImageResource(dataHolderB.imageID);
        }
    }

    @Override
    public int getItemCount() {
        return mData != null ? mData.size() : 0;
    }

    @Override
    public int getItemViewType(int position) {
        Data.DataHolder data = mData.get(position);
        if (data instanceof Data.DataHolderA) {
            return TYPE_A;
        } else if (data instanceof Data.DataHolderB) {
            return TYPE_B;
        }
        return super.getItemViewType(position);
    }

    public void setData(List<Data.DataHolder> data) {
        mData = data;
        notifyDataSetChanged();
    }

    public void addData(List<Data.DataHolder> data ,int position) {
        mData.addAll(position,data);
        notifyItemRangeInserted(position, data.size());
    }

    public void deleteData(int position) {
        mData.remove(position);
        notifyItemRemoved(position);
    }
}
```

## 布局管理

 - 线性布局：LinearLayoutManager
 - 网格布局：GridLayoutManager
 - 瀑布流布局：StaggeredGridLayoutManager

## 元素间隔

`DividerItemDecoration` 是 support 包为我们提供的为元素添加分割线的一个实现，另外我们还可以通过继承 `RecyclerView.ItemDecoration` 自定义元素的间隔样式。
下面是一个网格列表的间隔自定义实现：

```
    private static class GridDecoration extends RecyclerView.ItemDecoration{
        private int margin;
        public GridDecoration(Context context) {
            margin = DisplayUtil.dip2Pixel(context,12);
        }
        @Override
        public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
            outRect.set(0, margin, margin, 0);
        }
    }
```

## 动画

我们可以为元素设置动画，当增加、删除或者更新元素时就会以某种动画形式呈现。
`DefaultItemAnimator()` 是 support 包为我们提供的一种动画实现，我们也可以继承 `ItemAnimator` 来实现自定义的动画。

## 局部刷新

`RecyclerView.Adapter` 为我们提供了下面方法来实现局部刷新：

 - notifyItemChanged(int position)
 - notifyItemChanged(int position, Object payload)
 - notifyItemRangeChanged(int positionStart, int itemCount)
 - notifyItemRangeChanged(int positionStart, int itemCount, Object payload)
 - notifyItemInserted(int position)
 - notifyItemMoved(int fromPosition, int toPosition)
 - notifyItemRangeInserted(int positionStart, int itemCount)
 - notifyItemRemoved(int position)
 - notifyItemRangeRemoved(int positionStart, int itemCount)

相对于 `notifyDataSetChanged()` 会刷新所有数据，局部刷新能为我们代码更经济实惠的刷新体验。

## 缓存机制

在RecyclerView中，并不是每次绘制 Item 项，都会重新创建 ViewHolder 对象，也不是每次都会重新绑定 ViewHolder 数据。
RecycleView 提供了强大的5级缓存机制。

 - mChangedScrap：表示数据已经改变的 ViewHolder 列表。复用时不需要重新绑定数据。
 - mAttachedScrap：未与RecyclerView分离的 ViewHolder 列表。复用时不需要重新绑定数据。
 - mCachedViews：其大小由mViewCacheMax决定，默认DEFAULT_CACHE_SIZE为2，可动态设置。
 - ViewCacheExtension：开发者可自定义的一层缓存。复用时不需要重新绑定数据。
 - mRecyclerPool：在有限的mCachedViews中如果存不下ViewHolder时，就会把ViewHolder存入RecyclerViewPool中。复用时需要重新绑定数据。

在上述缓存都没有命中的情况下，就要重新创建 ViewHolder，并且重新绑定数据。

具体的分析我们在后面的源码分析章节详细介绍。

## MergeAdapter

androidx 扩展包下的 recyclerview 1.2 以上版本新增了一个 MergeAdapter。它可以顺序组合多个 adapter，以在单个 RecyclerView 中显示。 这使得我们可以更好地封装 adapter，而不必将许多数据源组合到单个 adapter 中，从而使它们集中并复用。
这样，如果我们的列表中有多重类型的 item 时，我们就可以不再使用前面推荐的多种 itemtype 的方式。
这样做的优点是，解除了不同类型 type 之间的耦合，便于扩展不同的item类型。

```
 MyAdapter adapter1 = ...;
 AnotherAdapter adapter2 = ...;
 MergeAdapter merged = new MergeAdapter(adapter1, adapter2);
 recyclerView.setAdapter(mergedAdapter);
```

## 个性化

### 下拉刷新

我们可以使用 `support.v4` 包中提供的 `SwipeRefreshLayout` 组件来实现下拉刷新功能。
添加布局：

```
    <android.support.v4.widget.SwipeRefreshLayout
        android:id="@+id/refresh"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <android.support.v7.widget.RecyclerView
            android:id="@+id/recycle_view"
            android:layout_width="match_parent"
            android:layout_height="match_parent">

        </android.support.v7.widget.RecyclerView>
    </android.support.v4.widget.SwipeRefreshLayout>
```

初始化：

```
        mRefreshLayout = findViewById(R.id.refresh);
        // 设置下拉时圆圈的颜色（可以由多种颜色拼成）
        mRefreshLayout.setColorSchemeResources(R.color.blue, R.color.green, R.color.orange);
        // 设置触发刷新监听
        mRefreshLayout.setOnRefreshListener(mOnRefreshListener);
```

设置刷新监听：

```
    private SwipeRefreshLayout.OnRefreshListener mOnRefreshListener = new SwipeRefreshLayout.OnRefreshListener() {
        @Override
        public void onRefresh() {
            //updateData();
            Log.e("Test","startLoadMore");
            mRecycleView.postDelayed(new Runnable() {
                @Override
                public void run() {
                    // 停止刷新
                    mRefreshLayout.setRefreshing(false);
                }
            },5000);
        }
    };
```

更新数据完毕把数据加到 Adapter中。

### 加载更多

#### 触发加载更多事件

为 RecycleView 添加 OnScrollListener：

```
    mRecycleView.addOnScrollListener(mOnScrollListener);
```

```
    private RecyclerView.OnScrollListener mOnScrollListener = new RecyclerView.OnScrollListener() {
        @Override
        public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
            super.onScrollStateChanged(recyclerView, newState);
        }

        @Override
        public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
            super.onScrolled(recyclerView, dx, dy);
            boolean isScrollUp = dy > 0 ? true : false;
            int totalItemCount = mLinearLayoutManager.getItemCount();
            int lastVisibleItemPosition = mLinearLayoutManager.findLastCompletelyVisibleItemPosition();
            if(!mLoadingMore && isScrollUp && lastVisibleItemPosition >=totalItemCount -1){
                // 触发加载更多事件
                //startLoadMore();
                Log.e("Test","startLoadMore");
            }
        }
    };
```

#### 添加加载更多View

在 Adapter 中数据列表末尾添加一个 Item 来显示加载更多：

```
    public void onLoadMore(){
        int size = mList.size();
        mList.add(new GankFooterItem());
        notifyItemRangeInserted(size,1);
    }
```

#### 删除加载更多View

加载完毕后删除这个 Item，并添加新的数据：

```
    public void loadMoreData(List<GankItem> list){
        mList.remove(mList.size() - 1);
        notifyItemRangeRemoved(mList.size(),1);
        if(list != null) {
            int size = mList.size();
            mList.addAll(size,list);
            notifyItemRangeInserted(size,list.size());
        }
    }
```

## 与 ListView 对比

