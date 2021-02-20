---
title: Android RecyclerView -- StaggeredGridLayoutManager 多样式布局
categories: Android
comments: true
tags: [RecyclerView]
description: 介绍 StaggeredGridLayoutManager 多样式布局
date: 2017-8-26 12:00:00
---

## 概述

StaggeredGridLayoutManager 可以帮助我们使用 RecyclerView 来实现瀑布流布局，但是如果我们想实现下面的效果，使上面的 BannerView 和下面的瀑布流一起滑动。

<img src="/images/android-view-recycleview-staggeredgridlayoutmanager/1.png" width="343" height="679"/>

## 实现

我们是可以通过 ScrollerView + RecyclerView 来实现，但是这样就需要处理滑动冲突以及事件管理问题。那么能不能把 BannerView 也作为 RecyclerView 的一个item，让 RecyclerView 统一来管理呢？答案是可以的。

重写 Adpater 中的 `onViewAttachedToWindow()` 方法：

```
    override fun onViewAttachedToWindow(holder: RecyclerView.ViewHolder) {
        super.onViewAttachedToWindow(holder)

        var type = holder.itemViewType
        when (type) {
            TYPE_BANNER -> {
                val lp = holder.itemView.layoutParams
                if (lp is StaggeredGridLayoutManager.LayoutParams) {
                    lp.isFullSpan = true
                }
            }
        }
    }
```

主要是调用 `setFullSpan(true)` 方法。


## 推荐阅读

[RecyclerView 多布局实现、动态设置布局管理器、StaggeredGridLayoutManager占满一行](https://blog.csdn.net/qq_38287890/article/details/108268994)