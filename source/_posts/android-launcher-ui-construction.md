---
title: Android -- 页面结构
categories:  Android Launcher
comments: true
tags: [Launcher3]
description: 介绍 Launcher 的页面结构
date: 2021-1-20 10:00:00
---

## 页面布局

```
LauncherRootView:InsettableFrameLayout
    DragLayer:BaseDragLayer:InsettableFrameLayout
        Workspace:PagedView
            CellLayout:ViewGroup
                ShortcutAndWidgetContainer:ViewGroup
                    DoubleShadowBubbleTextView:BubbleTextView:TextView
                    DoubleShadowBubbleTextView:BubbleTextView:TextView
                    ......
            CellLayout:ViewGroup
                ShortcutAndWidgetContainer:ViewGroup
                    DoubleShadowBubbleTextView:BubbleTextView:TextView
                    DoubleShadowBubbleTextView:BubbleTextView:TextView
                    ......
            ......
        Hotseat:CellLayout:ViewGroup
            ShortcutAndWidgetContainer:ViewGroup
                DoubleShadowBubbleTextView:BubbleTextView:TextView
                DoubleShadowBubbleTextView:BubbleTextView:TextView
                DoubleShadowBubbleTextView:BubbleTextView:TextView
                DoubleShadowBubbleTextView:BubbleTextView:TextView
        WorkspacePageIndicator  // 页面切换指示器原点
        DropTargetBar:FrameLayout  //卸载面板Layout
        Folder:AbstractFloatingView //当显示文件夹时添加这个View
            FolderPagedView
                CellLayout
                    ShortcutAndWidgetContainer
                        BubbleTextView
                        BubbleTextView
        LauncherRecentsView:RecentsView:PagedView  //多任务页面
            TaskView:FrameLayout
            TaskView:FrameLayout
            TaskView:FrameLayout
            ......
        OverviewActionsView
        ClearAllButton:Button
```

桌面的最外层是一个FrameLayout，它包含了一个DragLayer，顾名思义就是它可以处理拖动事件，它包含了很多子View，它的第一个子View是Workspace，它其实是可以左右滑动的PagedView，它包含了很多CellLayout，桌面有几页数据它就包含了几个CellLayout。
DragLayer的第二个子View是Hotseat，它也继承自CellLayout，它主要是存放桌面下部的四个常用图标，它不会随页面切换而滑动。
CellLayout主要的作用是装在快捷方式或者小部件等。
DropTargetBar是当卸载桌面图标是弹出的面板。
Folder当点击文件夹时显示
LauncherRecentsView就是多任务的页面，显示桌面时它是隐藏的。
OverviewActionsView屏幕截图时显示，带分享图片按钮
ClearAllButton就是多任务页面的清除所有任务按钮。