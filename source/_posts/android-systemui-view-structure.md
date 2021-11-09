---
title: SystemUI -- 下拉通知面板页面结构和重要的类
categories:  SystemUI
comments: true
tags: [SystemUI]
description: 介绍 SystemUI 下拉通知面板页面结构和重要的类
date: 2021-11-5 10:00:00
---


## 页面结构

```
NotificationShadeWindowView //R.layout.super_notification_shade
    ScrimView(R.id.scrim_notification) // 通知栏背景
    NotificationPanelView(R.id.notification_panel) //R.layout.status_bar_expanded 下拉通知栏 
        NotificationsQuickSettingsContainer(R.id.notification_container_parent) //下拉通知栏快捷设置以及通知中心部分
            KeyguardStatusBarView  // 锁屏状态栏
            KeyguardStatusView // 锁屏界面显示时间和日期界面
            FrameLayout(R.id.qs_frame) //R.layout.qs_panel 顶部快捷开关部分 QSFragment
                QSContainerImpl(R.id.quick_settings_container)
                    QuickStatusBarHeader(R.id.header)//显示顶部的状态栏、时间和顶部常显示按钮
                        LinearLayout(R.id.quick_status_bar_date_privacy)
                        RelativeLayout
                            LinearLayout(R.id.quick_qs_status_icons)
                            QuickQSPanel // 顶部常显示部分
                                QuickQSPanel.QQSSideLabelTileLayout
                                    QSTileViewImpl
                                    QSTileViewImpl
                                    QSTileViewImpl
                                    QSTileViewImpl
                    NonInterceptingScrollView(R.id.expanded_qs_scroll_view)
                        QSPanel(R.id.quick_settings_panel) //顶部常显示按钮和下部的折叠部分快捷开关按钮
                            BrightnessSliderView  // 顶部亮度条
                                ToggleSeekBar
                            PagedTileLayout
                                SideLabelTileLayout
                                    QSTileViewImpl
                                    QSTileViewImpl
                                    ......
                                SideLabelTileLayout
                                    QSTileViewImpl
                                    QSTileViewImpl
                                    ......
                            QSFooterView // 编辑，关机，管理等按钮
                    QSDetail
                    QSCustomer // 编辑
            NotificationStackScrollLayout(R.id.notification_stack_scroller) //通知栏面板
                MediaHeaderView
                ExpandableNotificationRow  //通知
                    NotificationBackgroundView(R.id.backgroundNormal)
                    NotificationBackgroundView(R.id.backgroundDimmend)
                    NotificationContentView(R.id.expanded)
                ExpandableNotificationRow //通知
                SectionHeaderView
                NotificationShelf // 通过过多时显示隐藏通知的界面
                ......
```

通知栏下拉，可以看到的UI有下面几部分：

QS面板：
Quick Quick Settings (QQS) : 即初级展开面板,是一次下拉面板看到的简版QS面板,包含少量的开关
Quick Settings (QS) : 完整QS面板,是二次下拉面板看到的完成QS面板,其包含更多的开关
通知中心：
NotificationStackScrollLayout
编辑页面：
QSCustomer

下图中左边是 QQS，右边为QS区域。

<img src="/images/android-systemui-view-structure/QS-QQS.png" width="480" height="360"/>


## 类介绍

1.StatusBar:

collapseShade()
collapsePanel()
makeExpandedInvisible()
makeExpandedVisible()

### QS

1.NotificationShadeWindowView：承载锁屏和下拉通知的界面
2.NotificationShadeWindowViewController：
3.NotificationShadeWindowControllerImpl：
5.ShadeControllerImpl：ShadeController 的实现类。Shade 可以理解为状态栏的概念，可以有多种状态，比如：dozing, locked, showing the bouncer, occluded 等。ShadeControllerImpl 被 StatusBar 类用来控制 shade 的状态，它提供了下面的集中方法。
animateCollapsePanels()：以动画形式收起面板，如果在KEYGUARD状态就显示bouncer view，如果是在 SHADE 就隐藏shade view。
collapsePanel()：收起面板。
closeShadeIfOpen()：
addPostCollapseAction(Runnable action)：如果需要在面板收起时做一些工作，可以用这个方法添加一个Runnable，它只会在下次面板收起时执行一次。
runPostCollapseRunnables()：收起面板时执行。
6.QSPanelController:
7.QuickQSPanelController
8.QSFragment 承载 QSPanel的界面。它实现了 QS 接口，提供了折叠和扩展QS中界面的接口。
9.QSContainerImpl quick_settings_container，承载 QuickStatusBarHeader 和 QSPanel。
setFancyClipping，updateClippingPath()：通过切图方式来设置 QSContainerImpl 的显示大小。



通过 LOCK_SCREEN_SHOW_SILENT_NOTIFICATIONS 和 LOCK_SCREEN_SHOW_NOTIFICATIONS 来设定锁屏通知的显示规则。


mExpandedHeight 的计算方法

https://www.shangmayuan.com/a/00c1613a561946829b6a2846.html