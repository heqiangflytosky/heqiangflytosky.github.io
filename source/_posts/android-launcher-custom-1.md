---
title: Android -- Launcher 定制
categories:  Android Launcher
comments: true
tags: [Launcher3]
description: 定制 Launcher，去掉搜索栏和抽屉式页面
date: 2021-1-10 10:00:00
---

## 概述

Android 原生桌面不太符合我们的使用习惯，因此，本文就一步步的去定制我们熟悉的Launcher交互风格。
源码请参考[GitHub](https://github.com/heqiangflytosky/QLauncher3/tree/android11)

## 去掉首屏搜索栏

```
//FeatureFlags.java
    // QLauncher modify 去掉qsb搜索栏@{
    //public static final boolean QSB_ON_FIRST_SCREEN = true;
    public static final boolean QSB_ON_FIRST_SCREEN = false;
    // @}
```

## 去掉抽屉式

Launcher模式是把所有的应用显示在抽屉式的AllAppsView里面，主页上显示的是可以从所有应用页面拖动出来的收藏应用，现在我们做一下改造，把所有的应用都显示在主页。
可以分为一下步骤：

 1. 增加一个全局变量来控制是否显示抽屉。
 2. 增加一个default_workspace_5x5_no_all_apps.xml来作为主页图标布局。
 3. 将所有应用图标显示在WorkSpace。
 4. 屏蔽上拉显示抽屉操作。

### 增加全局变量

```
//FeatureFlags.java
    // QLauncher add 去掉抽屉@{
    public static final boolean REMOVE_DRAWER = true;
    // @}
```

### 增加一套布局

```
<?xml version="1.0" encoding="utf-8"?>

<favorites xmlns:launcher="http://schemas.android.com/apk/res-auto/com.android.launcher3">

    <!-- appwidget-->
        <appwidget launcher:className="com.android.weather.WeatherWidgetTimeProvider"
            launcher:packageName="com.android.weather"
            launcher:screen="0"
            launcher:x="0"
            launcher:y="0"/>

    <!-- appwidget end-->

    <!-- Hotseat (We use the screen as the position of the item in the hotseat) -->
    <!-- Dialer, Messaging, [Maps/Music], Browser, Camera -->
    <resolve
        launcher:container="-101"
        launcher:screen="0"
        launcher:x="0"
        launcher:y="0" >
        <favorite launcher:uri="#Intent;action=android.intent.action.DIAL;end" />
        <favorite launcher:uri="tel:123" />
        <favorite launcher:uri="#Intent;action=android.intent.action.CALL_BUTTON;end" />
    </resolve>

    <resolve
        launcher:container="-101"
        launcher:screen="1"
        launcher:x="1"
        launcher:y="0" >
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_MESSAGING;end" />
        <favorite launcher:uri="sms:" />
        <favorite launcher:uri="smsto:" />
        <favorite launcher:uri="mms:" />
        <favorite launcher:uri="mmsto:" />
    </resolve>

    <resolve
        launcher:container="-101"
        launcher:screen="2"
        launcher:x="2"
        launcher:y="0" >
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_MAPS;end" />
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_MUSIC;end" />
    </resolve>

    <resolve
        launcher:container="-101"
        launcher:screen="3"
        launcher:x="3"
        launcher:y="0" >
        <favorite
            launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_BROWSER;end" />
        <favorite launcher:uri="http://www.example.com/" />
    </resolve>

    <resolve
        launcher:container="-101"
        launcher:screen="4"
        launcher:x="4"
        launcher:y="0" >
        <favorite launcher:uri="#Intent;action=android.media.action.STILL_IMAGE_CAMERA;end" />
        <favorite launcher:uri="#Intent;action=android.intent.action.CAMERA_BUTTON;end" />
    </resolve>

    <!-- row 4-->
    <resolve
        launcher:screen="0"
        launcher:x="1"
        launcher:y="3" >
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_GALLERY;end" />
        <favorite launcher:uri="#Intent;type=images/*;end" />

    </resolve>
    <!-- row 4 end -->
    <!-- row 5-->
    <resolve
        launcher:screen="0"
        launcher:x="0"
        launcher:y="-1" >
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_MARKET;end" />
        <favorite launcher:uri="market://details?id=com.android.launcher" />
    </resolve>

    <resolve
        launcher:screen="0"
        launcher:x="1"
        launcher:y="4" >
        <favorite launcher:className="com.meizu.flyme.gamecenter.activity.GameMainActivity"
            launcher:packageName="com.meizu.flyme.gamecenter"/>
    </resolve>

    <resolve
        launcher:screen="0"
        launcher:x="4"
        launcher:y="-1" >
	    <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_EMAIL;end" />
	    <favorite launcher:uri="mailto:" />

    </resolve>
    <!-- row 5 end-->

    <!-- screen 2-->
    <!-- row 1-->
    <resolve
        launcher:screen="1"
        launcher:x="0"
        launcher:y="0" >
        <favorite launcher:className="com.meizu.flyme.alarmclock.DeskClock"
            launcher:packageName="com.android.alarmclock"/>

    </resolve>

    <favorite launcher:className="com.android.settings.Settings"
        launcher:packageName="com.android.settings"
        launcher:screen="1"
        launcher:x="1"
        launcher:y="0"/>
    <!-- row 1 end-->
    <!-- screen 2 end-->


</favorites>
```

修改 device_profiles.xml

```
//launcher:defaultLayoutId="@xml/default_workspace_5x5" >
launcher:defaultLayoutId="@xml/default_workspace_5x5_no_all_apps"
```

添加默认 AppWidget 需要 `<uses-permission android:name="android.permission.BIND_APPWIDGET"/>`权限，而且需要系统签名。

### 所有应用添加到 WorkSpace

LauncherProvider中加载数据库时添加所有应用到launcher.db。

```
//LauncherProvider.java
    synchronized private void loadDefaultFavoritesIfNecessary() {
        ......
                mOpenHelper.createEmptyDB(mOpenHelper.getWritableDatabase());
                mOpenHelper.loadFavorites(mOpenHelper.getWritableDatabase(),
                        getDefaultLayoutParser(widgetHost));
            }
            // QLauncher add 去掉抽屉@{
            if (FeatureFlags.REMOVE_DRAWER) {
                //或者这里定制一个parser
                loadCustomFavourite();
            }
            // @}
            clearFlagEmptyDbCreated();
        }
    }
    // QLauncher add 去掉抽屉@{
    synchronized private void loadCustomFavourite(){
        final Context context = getContext();
        ArrayList<Pair<ItemInfo, Object>> installQueue = new ArrayList<>();
        final List<UserHandle> profiles = context.getSystemService(UserManager.class).getUserProfiles();
        LauncherApps launcherApps = context.getSystemService(LauncherApps.class);
        for (UserHandle user : profiles) {
            final List<LauncherActivityInfo> apps = launcherApps.getActivityList(null, user);
            ArrayList<InstallShortcutReceiver.PendingInstallShortcutInfo> added = new ArrayList<InstallShortcutReceiver.PendingInstallShortcutInfo>();
            synchronized (this) {
                for (LauncherActivityInfo app : apps) {

                    // 剔除掉默认的favourite
                    boolean exists = false;
                    for(ComponentName componentName : mOpenHelper.mCompoentList){
                        if (componentName.getPackageName().equals(app.getComponentName().getPackageName())
                        && componentName.getClassName().equals(app.getComponentName().getClassName())) {
                            exists = true;
                            break;
                        }
                    }
                    if(exists){
                        continue;
                    }

                    InstallShortcutReceiver.PendingInstallShortcutInfo pendingInstallShortcutInfo = new InstallShortcutReceiver.PendingInstallShortcutInfo(app, context);
                    added.add(pendingInstallShortcutInfo);
                    installQueue.add(pendingInstallShortcutInfo.getItemInfo());
                }
            }
            Log.e("Test",""+added.size());
            if (!added.isEmpty()) {
                LauncherAppState.getInstance(context).getModel()
                        .addAndBindAddedWorkspaceItems(installQueue);
            }
        }
    }
    // @}
```

记录一下默认布局中写入数据库的app，后面遍历所有app添加到数据库时去掉这些app，防止重复添加。

```
//LauncherProvider.java
        // QLauncher add @{
        public ArrayList<ComponentName> mCompoentList = new ArrayList<>();
        // @}
        @Override
        public int insertAndCheck(SQLiteDatabase db, ContentValues values) {
            // QLauncher add @{
            try {
                String str = values.getAsString(Favorites.INTENT);
                Intent intent = Intent.parseUri(str,0);
                mCompoentList.add(intent.getComponent());
            } catch (Exception e) {
                e.printStackTrace();
            }
            // @}

            return dbInsertAndCheck(this, db, Favorites.TABLE_NAME, null, values);
        }
```

```
//AddWorkspaceItemsTask.java
    @Override
    public void execute(LauncherAppState app, BgDataModel dataModel, AllAppsList apps) {
        ......

                    // QLauncher modified 去掉抽屉,允许系统应用添加图标@{
//                    if (PackageManagerHelper.isSystemApp(app.getContext(), item.getIntent())) {
//                        continue;
//                    }
                    // @}
                }

                ......
    }
```

```
//BaseModelUpdateTask.java
    @Override
    public final void run() {
        if (!mModel.isModelLoaded()) {
            ...
            // Loader has not yet run.
            // QLauncher add 去掉抽屉@{
            //return;
            if (!FeatureFlags.REMOVE_DRAWER){
                return;
            }
            //@}
        }
        execute(mApp, mDataModel, mAllAppsList);
    }
```

安装或者更新应用时添加到WorkSpace中：

```
//PackageUpdatedTask.java
    @Override
    public void execute(LauncherAppState app, BgDataModel dataModel, AllAppsList appsList) {
        ......
        }

        // QLauncher add 去掉抽屉,添加或者更新应以时更新到WorkSpace@{
        updateToWorkSpace(context,app,dataModel,appsList,matcher);
        //@}

        bindApplicationsIfNeeded();
        ......
    }
    // QLauncher add 去掉抽屉,添加或者更新应以时更新到WorkSpace@{
    public void updateToWorkSpace(Context context, LauncherAppState app ,BgDataModel dataModel, AllAppsList appsList,ItemInfoMatcher matcher){
        if (FeatureFlags.REMOVE_DRAWER) {
            boolean intentChangedUpdate = false;
            if (mOp == OP_UPDATE) {
                WorkspaceItemInfo si1 = null;
                String packageName1 = null;
                synchronized (dataModel) {
                    for (ItemInfo info : dataModel.itemsIdMap) {
                        if (info instanceof WorkspaceItemInfo && mUser.equals(info.user)) {
                            si1 = (WorkspaceItemInfo) info;
                            ComponentName cn = si1.getTargetComponent();
                            if (cn != null && matcher.matches(si1, cn)) {
                                packageName1 = cn.getPackageName();

                                if (null != packageName1) {
                                    Intent newIntent = new PackageManagerHelper(context).getAppLaunchIntent(packageName1, mUser, cn);
                                    String newClassName = null;
                                    if (null != newIntent && null != newIntent.getComponent()) {
                                        newClassName = newIntent.getComponent().getClassName();
                                    }
                                    String oldClassName = cn.getClassName();

                                    Log.d(TAG, "update old intent toString=" + (null != si1.intent ? si1.intent.toString() : null));
                                    Log.d(TAG, "update new intent toString=" + (null != newIntent ? newIntent.toString() : null));
                                    Log.d(TAG, "update oldClassName=" + oldClassName);
                                    Log.d(TAG, "update newClassName=" + newClassName);

                                    // 只有intent class变化时更新
                                    if (null != newClassName && !newClassName.equals(oldClassName)) {
                                        Log.d(TAG, "update when intent changed.");

                                        ArrayList<WorkspaceItemInfo> updatedWorkspaceItems = new ArrayList<>();
                                        updateWorkspaceItemIntent(context, si1, packageName1);
                                        updatedWorkspaceItems.add(si1);
                                        getModelWriter().updateItemInDatabase(si1);
                                        bindUpdatedWorkspaceItems(updatedWorkspaceItems);
                                        intentChangedUpdate = true;
                                    }
                                }
                            }
                        }
                    }
                }
            }
            Log.d(TAG, "appsList.added=" + appsList.data.size());
            Log.d(TAG, "classe changed update=" + intentChangedUpdate);
            if (!intentChangedUpdate) {
                updateToWorkSpace(context, app, appsList);
            }
        }
    }

    private void updateToWorkSpace(Context context, LauncherAppState app, AllAppsList appsList) {
        if (FeatureFlags.REMOVE_DRAWER) {
            ArrayList<Pair<ItemInfo, Object>> installQueue = new ArrayList<>();
            UserManager userManager = context.getSystemService(UserManager.class);
            final List<UserHandle> profiles = userManager.getUserProfiles();
            for (UserHandle user : profiles) {
                final List<LauncherActivityInfo> apps = context.getSystemService(LauncherApps.class)
                        .getActivityList(null, user);
                synchronized (this) {
                    for (LauncherActivityInfo info : apps) {
                        for (AppInfo appInfo : appsList.data) {
                            if (info.getComponentName().equals(appInfo.componentName) && null != info.getUser() && info.getUser().equals(appInfo.user)) {
                                InstallShortcutReceiver.PendingInstallShortcutInfo pendingInstallShortcutInfo = new InstallShortcutReceiver.PendingInstallShortcutInfo(info, context);
                                installQueue.add(pendingInstallShortcutInfo.getItemInfo());
                            }
                        }
                    }
                }
            }

            if (!installQueue.isEmpty()) {
                for (Pair<ItemInfo, Object> item : installQueue) {
                    Log.d(TAG, "installQueue, user=" + item.first.user.toString() + ",title=" + item.first.title);
                }
                app.getModel().addAndBindAddedWorkspaceItems(installQueue);
            }
        }
    }
    //@}
```

```
//PackageManagerHelper.java
    // QLauncher add 去掉抽屉@{
    public Intent getAppLaunchIntent(String pkg, UserHandle user, ComponentName cn) {
        List<LauncherActivityInfo> activities = mLauncherApps.getActivityList(pkg, user);
        Intent result = null;
        if (!activities.isEmpty()) {
            for (LauncherActivityInfo info : activities) {
                if (null != info.getComponentName() && info.getComponentName().equals(cn)) {
                    result = AppInfo.makeLaunchIntent(info);
                    break;
                }
            }
            if (null == result) {
                result = AppInfo.makeLaunchIntent(activities.get(0));
            }
        }
        return result;
    }
    //@}
```

这里添加的应用都是从添加了数据库中的默认应用后的下一页开始的，后期还需要再改造。

### 去掉上滑呼出所有应用

```
    public TouchController[] createTouchControllers() {
        //QLauncher add 去掉抽屉@{
        //return new TouchController[]{getDragController(), new AllAppsSwipeController(this)};
        if (!FeatureFlags.REMOVE_DRAWER) {
            return new TouchController[]{getDragController(), new AllAppsSwipeController(this)};
        } else {
            return new TouchController[]{getDragController()};
        }
        //@}
    }
```

## 修改字体大小

```
        <display-option
            ......
            launcher:iconTextSize="12"
            ....../>
```