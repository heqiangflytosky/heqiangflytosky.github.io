---
title: Android -- SystemUI -- QS Tile
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍 SystemUI -- QS Tile
date: 2021-11-20 12:00:00
---


## 概述

本文我们来介绍 SystemUI 下拉面板快捷开关的实现原理。
下拉面板中的 QQS 和 QS 分别可以承载 QS Tile ，进行一些快捷设置的功能。

## 模块划分

 - TileView：主要负责快捷开关的视图展示。
  - QSTileView：继承自 LinearLayout，实现 QS 视图的抽象类。
  - QSTileViewImpl：继承自 QSTileView，
  - CustomizeTileView：继承自 QSTileViewImpl，编辑页面中用到

![UML类图](http://www.plantuml.com/plantuml/svg/VP9FgnCn5CNt-HJXhKRa-VRYucdTF2Z8C4LncKckxSKq6PARaVfdfo0YwCvYDwxSYKX1n8gVnccNVWKpgQrZcRvDI3Y_UpWdoRaduIXCDHiXQU0d8zHm5-7HvikgUR7SB5VDukS9Or8Bx_aG3GWt53CRY8dIanEI-5RBYzqeVyAkyYgK6YNVGqa8bH84DwK4xx54ZJIxunIuBAcLWnfjyEtXIez5Nbn8Qn8w1chtMH_M1UuXJMu9-N3iR32gY--e0YAcX9iTzQijAeu6ATjMv1INew0r1Gc2mKIOCQWi7RYFQ-y86cf3t0OIgE_tvHkA4lJ0cWWOS9SsI6Wadh735xcOLZg-IkMmRkFKGdDfjHQL1yNFplXw_hbv-t7zQjwy-V3xn-iyBtvz-xHv_tmttxnSDAllwtTV8xYbeXZjB84aYAKhCFn1C3osLXh-ku7K_VhVCDUIr4RiVYL-u-dfQISk-xSza58JTb0i8OFWoKxnCUnDpi6e-B7_0000)

 - QSTile：主要负责快捷开关的逻辑处理。
  - QSTile
  - QSTileImpl
  - CustomTile：编辑页面中需要用到
  - WifiTile
  - BluetoothTile
  - State 
  - BooleanState
  - SignalState

![UML类图](http://www.plantuml.com/plantuml/svg/bLLTJXiz57ttAYo-b_0Ls02q8YHK825QEg7bhVBODubRZpsrxoH4bCUkegkeU4JgRX3Rg3rsfeu8FtsPZCyvzzztik-GMMmgHKOI5T2Dis8a5Tp_khtxSND_Stlty_S68L8Zs37ZG2DDhyWGa4dL5wps59q0hW2zJLAWbNeIc61HrC0zE3DIK9T1IONfd1GMnXRSj426qDlS9H5bGWoOKYNZNtlB5yhJYrSbqyhCu5zrBaGvAuwvqHybp2ctm112bk7O6nYDS8cKnz-K9LVIHzW_RPHYfGBQMixyewlJwmMpYy4pGcNqv3bDuy0MSubysfCQAM-i1Otd7ksYuObq2Dhti_Uk_08SGuPUj4LBOnGmJTLI4CR34C9qKfF9DM6bGyjQC-PS6lLnLIi_xjeQz1v1q56qhDgV31zBpPGtEZ8EQHMU3yWZZpCj55X2wgRq5TCkPdAhOHwYfgGo3vsmBNRQkeazophLYJgxN7vU2uR9TGhdmdEBfdOnwnkprhJ8cxkcczVvZdHtxehG_w8Nezju0-zj0gAN1T_CDLCpZslKnN0UoYKZCvaeo9rrqeVkEDClK_QrDetcK8763_2j0OTUxiri4KwUeIoJjFN6In8hyOM-eccYOOUez8_T32oQkbOIx5vWtxJq4ifbKcDCZhwcmyEt_VEJqUTYr1yTvgALj1GhGJtATXjuyfGihhN68JXkPOtIwCUaCRkN24N3L1jry9JpRZE5T67jnS7YlV7u8jUaRoHSramB47tEmJbPIYLnCGXJE6KLh1XuS9hkM7ZDkQ5byn29EYaWhCwWSMYg9Rmchgk1x_wT-47CE1_yQYapfXe8UfB-BFQ-x-naNvCb_F_sThm-6T3j3WbCqdJLHD3xysYU1ac7Z2xaH3ELkm-X8nShzjX3wfCSoxN4WMe0ZS7fMlRYeeNtGOkcKdy0)

 - QSHost
  - QSHost：主要负责快捷开关TileView和QSTile的构建，提供获取QSTile集合的接口，提供了展开和收起下拉面板的接口，是快捷开关对外沟通的桥梁。
  - QSTileHost：
  - QSFactory：
  - QSFactoryImpl：继承自 QSFactory

![UML类图](http://www.plantuml.com/plantuml/svg/XPBFJW8n4CRlynGJJhi6U049mMnYDF4WMUAzT0TixBQRjguIvEXJ-1I68ublGtWDjnJA7qFamao_txJVpCx0EcRSQobX99maR0tpEstTvdkr_bnkbglrrqyBG2X7Pi8uOP1n3jZyQYqAv5fALbbwP8gaJTA3Cj66KtPHrXMfarEF4dT2gzum7mb9VPoIyy597IkAZ4avPlmbsGV8Ty4HPwZKDVs7-kjpChNWPCDpXppvSvjNazmPeQReF5aHJm4M05moQp7uzYEQGsL4pHpXH2WcySOODdiGrZMztJBkj9driHpQ40koP_mMtNjko7reON7yFM9NSmX3LTjvPSzCJQb8qkjJFBxTyC4hSaFCNMWi84-2tY8MqYJJoj6xGnx-snZGRMqRnrfBOYhkHk5hKcy5TWlK--XsZwO5qLXGOdfVLXBW0E9LfnERFZ-FLJ0WLJHB_Gi0)


 - QSPanel：承载快捷开关的容器。
  - QuickQSPanel：继承自 QSPanel，显示 QQS 部分快捷开关。
  - QSPanel：继承自 LinearLayout，显示 QS 快捷开关。
  - QSTileLayout
  - TileLayout
  - PagedTileLayout
  - SideLabelTileLayout

![UML类图](http://www.plantuml.com/plantuml/svg/bPB1RjD048Rl-nIhdWngymGSgbOz80Lewi1zufrO8yOprkviA0AEF0LFWNginDjKUGtin8aTAGVmnVx__vlPyTZNoI1QRjWjoHZznQAy1q7U_dv--VcqVNf--VNdmZaImxY62lsYN1BZ7BvgDl_DRPI2Jx31jv8CfCBImB2uoH8OVSVizAiz5p6PnlDetoECoLW48VGd5iDWjoeMbHdZ3IISzT43LXg3D-Bne4Ot-3zb9EPhj0_hTK9RQlklTIjLHX2Vsm3MHKbph30Lmo09RKx9K4Zgui3omRdq7-bkWs9phMkCFa_Ls3kXlIDvq2-fwCTizi-dEJpUAkT61kGenpdk7bkGH2h5cXNCuq5V-htHHcqrOLZ6pcUmBZhkvNrkFb6Y5VTBtRtU3zTT9p_3c--pcH_rcV0UQWmJmfq0LrU-fD3f5V84ElKcy69HdkFTd0GXJidXMxhXFfYPgi-7v1Yztk6JTjSe8JVy5m00)

 - QS：QS面板的顶层容器,主要处理QS面板的展开/收起逻辑。
  - QSContainerImpl：下拉面板快捷开关部分的根布局，包含QQS，QS，QSDetail，QSCustomer 等。添加到 QSFragment。
  - QSFragment：包含 QSContainerImpl 的 QSFragment。
  - QS：主要定义了 QS 面板展开/收起相关的接口




## QS Tile 的创建

创建 QSTile，通过初始化 QsTileHost，进行 QsTile 的加载。在 QsTileHost 初始化的时候，增加 Tunable 监听。

```
QSTileHost()
    Settings.Secure.getStringForUser("sysui_qs_tiles") // 从 Secure.QS_TILES 获取QS配置
    TunerServiceImpl.addTunable()
        QSTileHost.onTuningChanged()
            QSTileHost.loadTileSpecs()
                QSTileHost.getDefaultSpecs() // 如果 Secure.QS_TILES 没有配置，就从R.string.quick_settings_tiles_default读取
            QSTileHost.createTile()
                QSFactoryImpl.createTile()
                    QSFactoryImpl.createTileInternal()
```

TunerServiceImpl 中注册对字段"sysui_qs_tiles"的系统数据库监听，当编辑模式添加或者删除 Tile 时，会触发重新加载 Tile。

```
// TunerServiceImpl.java
    private class Observer extends ContentObserver {
        ......
        @Override
        public void onChange(boolean selfChange, java.util.Collection<Uri> uris,
                int flags, int userId) {
            if (userId == mUserTracker.getUserId()) {
                for (Uri u : uris) {
                    reloadSetting(u);
                }
            }
        }
    }
```

根据配置文件生成对应的 QSTile。

```
// QSFactoryImpl.java
    private QSTileImpl createTileInternal(String tileSpec) {
        // Stock tiles.
        switch (tileSpec) {
            case "wifi":
                return mWifiTileProvider.get();
            case "internet":
                return mInternetTileProvider.get();
            case "bt":
                return mBluetoothTileProvider.get();
            ......
            case "alarm":
                return mAlarmTileProvider.get();
            case "wallet":
                return mQuickAccessWalletTileProvider.get();
        }
```

创建 QSTileView

```
QSFragment.onViewCreated()
    QSPanelController.init()
        OnAttachStateChangeListener.onViewAttachedToWindow()
            QSPanelController.onViewAttached()
                QSPanelControllerBase.onViewAttached()
                    QSPanelControllerBase.setTiles()
                        QSTileHost.getTiles()
                        QSPanelControllerBase.setTiles()
                            QSPanelControllerBase.addTile()
                                QSTileHost.createTileView()
                                    QSFactoryImpl.createTileView()
                                        new QSTileViewImpl()
                                QSPanel.addTile()
                                    QSTile.addCallback() // 向QSTile注册回调
                                        onStateChanged()
                                            drawTile()
                                    QSTileViewImpl.init() // 初始化点击事件
                                        setOnClickListener()
                                        setOnLongClickListener()
                                    QSTile.refreshState()
                                    TileLayout.addTile() // 添加到 TileLayout
```

从上面流程可以看出来，QSTileHost 对象在构建的时候就通过 QSFactoryImpl 创建好了各个开关的对应的 QSTile,而后 QSPanel 在初始化的过程中,再次利用 QSTileHost 去构建各个开关的视图对象 QSTileView。

## 编辑页面 QS Tile 的创建

进入 QS 编辑模式之后，会出现更多 Tile，并且可以拖动到上方供用户平时下拉后直接使用，或者将不常用的放到待选区域。这里从UI上可以看出待选区域由两部分组成，一部分是 SystemUI 内部的 Tile（StockTiles），另一部分则是第三应用自己注册的 Tile（PackageTiles）。

```
QSPanelController.showEdit()
    QsCustomizerController.show()
        TileQueryHelper.queryTiles()
            TileQueryHelper.addCurrentAndStockTiles()
                getString(R.string.quick_settings_tiles_stock)
                QSTileHost.createTile()
                    QSFactoryImpl.createTile()
                        QSFactoryImpl.createTileInternal()
```

获取 R.string.quick_settings_tiles_stock 对应的就是SystemUI中可加载的所有Tile，如果希望客制化也可以在此处进行修改。

接下来，再看一下如何加载第三方 Tile。

```
TileQueryHelper.TileCollector.onStateChanged()
    TileQueryHelper.TileCollector.finished()
        TileAdapter.onTilesChanged()
            TileAdapter.recalcSpecs()
        TileQueryHelper.addPackageTiles() // 获取带有 TileService.ACTION_QS_TILE 的应用信息
            TileQueryHelper.createStateAndAddTile()
                TileQueryHelper.addTile()
```

## 关联 QS Tile 和 QS View


它们关联的建立主要时通过 QSTile.Callback 和 QSTile.State 来实现。

```
    public interface Callback {
        public static final int VERSION = 1;
        void onStateChanged(State state);
        void onShowDetail(boolean show);
        void onToggleStateChanged(boolean state);
        void onScanStateChanged(boolean state);
        void onAnnouncementRequested(CharSequence announcement);
    }
```

在 QSPanelControllerBase.addTile() 中创建 QSTileView 时调用 QSPanel.addTile() 为它们建立联系：

```
// QSPanel.java
    void addTile(QSPanelControllerBase.TileRecord tileRecord) {
        final QSTile.Callback callback = new QSTile.Callback() {
            @Override
            public void onStateChanged(QSTile.State state) {
                drawTile(tileRecord, state);
            }

            @Override
            public void onShowDetail(boolean show) {
                // Both the collapsed and full QS panels get this callback, this check determines
                // which one should handle showing the detail.
                if (shouldShowDetail()) {
                    QSPanel.this.showDetail(show, tileRecord);
                }
            }

            @Override
            public void onToggleStateChanged(boolean state) {
                if (mDetailRecord == tileRecord) {
                    fireToggleStateChanged(state);
                }
            }

            @Override
            public void onScanStateChanged(boolean state) {
                tileRecord.scanState = state;
                if (mDetailRecord == tileRecord) {
                    fireScanStateChanged(tileRecord.scanState);
                }
            }

            @Override
            public void onAnnouncementRequested(CharSequence announcement) {
                if (announcement != null) {
                    mHandler.obtainMessage(H.ANNOUNCE_FOR_ACCESSIBILITY, announcement)
                            .sendToTarget();
                }
            }
        };

        tileRecord.tile.addCallback(callback); // 向QSTile注册回调
        tileRecord.callback = callback;
        tileRecord.tileView.init(tileRecord.tile); // 初始化点击事件
        tileRecord.tile.refreshState();

        if (mTileLayout != null) {
            mTileLayout.addTile(tileRecord);
        }
    }
```

下面一次点击事件来介绍一下它们之间的关联。
BluetoothTile 继承自 QSTileImpl，实现了抽象方法 handleClick(),handleUpdateState()
点击事件触发后，先把 State 发送给 QSTileImpl 来更新逻辑，然后再通过回调函数通知 QSTileView 更新 UI。

```
QSTileViewImpl.onClick() // 在init()方法中注册
    QSTileImpl.click()
        Handler.obtainMessage(H.CLICK, view).sendToTarget()
            QSTileImpl.H.handleMessage()
                BluetoothTile.handleClick()
                    QSTileImpl.refreshState()
                        Handler.obtainMessage(H.REFRESH_STATE, arg).sendToTarget()
                             QSTileImpl.H.handleMessage()
                                QSTileImpl.handleRefreshState()
                                    BluetoothTile.handleUpdateState()
                                        BooleanState.value // 更新State
                                    QSTileImpl.handleStateChanged()
                                        QSTile.Callback().onStateChanged(mState) // 在 QSPanel.addTile() 中注册
                                            QSPanel.drawTile()
                                                QSTileViewImpl.onStateChanged()
                                                    QSTileViewImpl.handleStateChanged() // 更新 UI
                    BluetoothController.setBluetoothEnabled() // 更新蓝牙状态
```




<!--
@startuml
Title "TileView类关系图"

class LinearLayout
abstract class QSTileView {
+ public abstract View updateAccessibilityOrder(View previousView)
+ public abstract QSIconView getIcon()
+ public abstract View getIconWithBackground()
+ public View getSecondaryIcon()
+ public abstract void init(QSTile tile)
+ public abstract void onStateChanged(State state)
+ public abstract int getDetailY()
+ public View getLabelContainer()
+ public View getSecondaryLabel()
}
class QSTileViewImpl {
private val collapsed // 是否折叠，QQS中的TileView创建时是折叠的
protected open fun handleStateChanged(state: QSTile.State)
}
LinearLayout <|-- QSTileView
QSTileView <|-- QSTileViewImpl
QSTileViewImpl <|-- CustomizeTileView
@enduml
-->

<!--
@startuml
Title "QSTile类关系图"

interface QSTile {
DetailAdapter getDetailAdapter()
String getTileSpec()
void setTileSpec(String tileSpec)
void refreshState();

void addCallback(Callback callback);
void removeCallback(Callback callback);
void removeCallbacks();
QSIconView createTileView(Context context)
void click(@Nullable View view)
void secondaryClick(@Nullable View view)
void longClick(@Nullable View view)
void userSwitch(int currentUser)
void setListening(Object client, boolean listening)
State getState()
}
abstract class QSTileImpl {
protected TState mState
protected final QSHost mHost
protected final H mHandler

public abstract TState newTileState()
protected abstract void handleClick(View view)
abstract protected void handleUpdateState(TState state, Object arg)
protected void handleSetListening(boolean listening)
}

interface Callback {
 # void onStateChanged(State state)
 # void onShowDetail(boolean show)
 # void onToggleStateChanged(boolean state)
 # void onScanStateChanged(boolean state)
 # void onAnnouncementRequested(CharSequence announcement)
}

class State {
 + public Icon icon;
 + public Supplier<Icon> iconSupplier;
 + public int state = DEFAULT_STATE;
 + public CharSequence label;
 + public CharSequence secondaryLabel;
 + public CharSequence contentDescription;
 + public CharSequence stateDescription;
 + public CharSequence dualLabelContentDescription;
 + public boolean disabledByPolicy;
 + public String expandedAccessibilityClassName;
 + public SlashState slash;
 + public Drawable sideViewCustomDrawable;
 + public String spec;
}

class BooleanState {
 + public boolean value
}


QSTile <|.. QSTileImpl
QSTile *-- State
QSTile *-- Callback
State *-- SlashState
State <|-- BooleanState
BooleanState <|-- SignalState
QSTileImpl <|-- CustomTile
QSTileImpl <|-- WifiTile
QSTileImpl <|-- BluetoothTile
QSTileImpl <|-- XXTile
@enduml
-->

<!-- 
@startuml
Title "QSHost类关系图"

interface QSHost {
    void collapsePanels()
    void forceCollapsePanels()
    void openPanels()
    Collection<QSTile> getTiles()
    void addCallback(Callback callback)
    void removeCallback(Callback callback)
    TileServices getTileServices()
    void removeTile(String tileSpec)

}

class QSTileHost{
private final ArrayList<QSFactory> mQsFactories
private final List<Callback> mCallbacks
}

interface Callback {
void onTilesChanged();
}

interface QSFactory {
QSTile createTile(String tileSpec)
QSTileView createTileView(Context context, QSTile tile, boolean collapsedView)
}


QSHost <|.. QSTileHost
QSFactory <|.. QSFactoryImpl

QSTileHost *-- Callback
QSTileHost *-- QSFactory
@enduml
-->

<!-- 
@startuml
Title "QSPanel类关系图"

interface QSTileLayout {
void saveInstanceState(Bundle outState)
void restoreInstanceState(Bundle savedInstanceState)
void addTile(QSPanelControllerBase.TileRecord tile)
void removeTile(QSPanelControllerBase.TileRecord tile)
int getOffsetTop(QSPanelControllerBase.TileRecord tile)
boolean updateResources()
void setListening(boolean listening, UiEventLogger uiEventLogger)
boolean setMinRows(int minRows)
boolean setMaxColumns(int maxColumns)
void setExpansion(float expansion, float proposedTranslation)
int getNumVisibleTiles()
}



QSTileLayout <|.. TileLayout
QSTileLayout <|.. PagedTileLayout

TileLayout <|-- SideLabelTileLayout
SideLabelTileLayout <|-- QQSSideLabelTileLayout

LinearLayout <|-- QSPanel
QSPanel <|-- QuickQSPanel

QSPanel *-- QSTileLayout
@enduml
-->