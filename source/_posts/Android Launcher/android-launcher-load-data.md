---
title: Android -- 数据加载
categories:  Android Launcher
comments: true
tags: [Launcher3]
description: 介绍 Launcher 的数据加载流程
date: 2021-1-26 10:00:00
---

## 概述

本文涉及的到几个重要的类：

 - LauncherModel：Launcher 数据处理核心类
 - LoaderTask:主要任务负责加载数据（workspace、all apps）和绑定数据（workspace、all apps）
 - AutoInstallsLayout:
 - InvariantDeviceProfile: 桌面的一些配置，比如默认布局文件
 - BgDataModel: 用来放布局相关的集合
 - WorkspaceItemInfo，FolderInfo，LauncherAppWidgetInfo：都是继承自ItemInfo，保存桌面图标的一些信息：图标位置，宽高，类型等信息

## 数据加载流程

```
Launcher.onCreate
    LauncherModel.startLoader
        // 处理重新绑定逻辑，此时桌面数据已经加载，只执行重新绑定
        LoaderResults.bindWorkspace
            Launcher.bindScreens  // 绑定屏幕
                Launcher.bindAddScreens //根据实际计算所得的屏幕个数，创建屏幕的View，launcher中每一屏对应的View是一个CellLayout。
        LoaderResults.bindAllApps
        LoaderResults.bindDeepShortcuts
        LoaderResults.bindWidgets
        // 处理首次加载逻辑
        LauncherModel.startLoaderForResults
            LoaderTask.run
                LoaderTask.loadWorkspace()
                    LauncherSettings.Settings.call(contentResolver,LauncherSettings.Settings.METHOD_LOAD_DEFAULT_FAVORITES);
                        LauncherProvider.loadDefaultFavoritesIfNecessary()
                mResults.bindWorkspace(); //绑定数据
                    WorkspaceBinder.bind()
                        WorkspaceBinder.bindWorkspaceItems
                            Launcher.bindItems //创建对应快捷图标
                                Workspace.addInScreenFromBind
                                    Workspace.addInScreen
                                        CellLayout.addViewToCellLayout //添加到CellLayout
                        WorkspaceBinder.bindAppWidgets
                sendFirstScreenActiveInstallsBroadcast()
                waitForIdle()
                loadAllApps()
                mResults.bindAllApps()
                updateHandler.updateIcons
                waitForIdle()
                loadDeepShortcuts()
                mResults.bindDeepShortcuts()
                waitForIdle()
                mBgDataModel.widgetsModel.update
                mResults.bindWidgets()
                updateHandler.updateIcons
```

数据加载主要在 LoaderTask.run() 中进行，这部分代码注释非常清楚，主要分为下面5个步骤：

 - step 1: workspace 
  - step 1.1: loading workspace
  - step 1.2: bind workspace
  - step 1.3: send first screen broadcast
  - step 1 completed, wait for idle
 - step 2: all apps
  - step 2.1: loading all apps
  - step 2.2: Binding all apps
  - step 2.3: Update icon cache
  - step 2 completed, wait for idle
 - step 3: deep shortcuts
  - step 3.1: loading deep shortcuts
  - step 3.2: bind deep shortcuts
  - step 3 completed, wait for idle
 - step 4: widgets
  - step 4.1: loading widgets
  - step 4.2: Binding widgets
  - step 4.3: Update icon cache
 - step 5
  - step 5: Finish icon cache update

## Launcher 

Launcher实现了LauncherModel.Callbacks接口，用于回调界面显示更新。

```
    public interface Callbacks {
        ......
        //这边只列出了这边分析涉及的接口，便于查看
        public void bindItems(List<ItemInfo> shortcuts, boolean forceAnimateIcons);//绑定items
        public void bindScreens(ArrayList<Long> orderedScreenIds);//绑定screen
        public void bindAllApplications(ArrayList<AppInfo> apps);//绑定所有应用

    }
```

## WorkSpace

### loadWorkspace

先来分析一下 `LoaderTask.loadWorkspace()` 的几个关键步骤。
loadWorkspace 也可以分两小步，第1小步是获取数据库，如果有数据库则直接进入第2小步，如果数据库为空则从默认xml布局文件生成数据库。第2小步，从数据库读取信息存放到 `sBgDataModel` 中。loadWorkspace 方法的结果就是 `sBgDataModel`。
读取数据库的操作也比较简单，根据item的类别进行一些合法性的校验，如果不合法就标记删除，全部加载完毕后统计一些删除操作，保留数据库中的合法的数据。

开始分析 loadWorkspace 源码。

#### 准备工作

```
// LoaderTask.loadWorkspace()
    private void loadWorkspace() {
        final Context context = mApp.getContext();
        final ContentResolver contentResolver = context.getContentResolver();
        final PackageManagerHelper pmHelper = new PackageManagerHelper(context);
        final boolean isSafeMode = pmHelper.isSafeMode();
        final boolean isSdCardReady = Utilities.isBootCompleted();
        final MultiHashMap<UserHandle, String> pendingPackages = new MultiHashMap<>();

        SharedPreferences sp = Utilities.getPrefs(context);
        if (sp.getBoolean(LOAD_WORKSPACE_FIRST, true)) {
            // 第一次启动时创建临时数据库，进行数据迁移
            LauncherSettings.Settings.call(contentResolver, LauncherSettings.Settings.METHOD_CREATE_TEMP_FAVORITES_TABLE);
            LauncherSettings.Settings.call(contentResolver, LauncherSettings.Settings.METHOD_DATA_TRANSFER);
            sp.edit().putBoolean(LOAD_WORKSPACE_FIRST, false).commit();
        } else {
            Log.i(TAG, "Not first load workspace");
        }

        // 进行数据迁移
        boolean clearDb = false;
        try {
            ImportDataTask.performImportIfPossible(context);
        } catch (Exception e) {
            // Migration failed. Clear workspace.
            clearDb = true;
        }

        if (!clearDb && !GridSizeMigrationTask.migrateGridIfNeeded(context)) {
            // Migration failed. Clear workspace.
            clearDb = true;
        }
        // 清除数据库
        if (clearDb) {
            Log.d(TAG, "loadWorkspace: resetting launcher database");
            LauncherSettings.Settings.call(contentResolver,
                    LauncherSettings.Settings.METHOD_CREATE_EMPTY_DB);
        }
        // 加载默认桌面布局
        Log.d(TAG, "loadWorkspace: loading default favorites");
        LauncherSettings.Settings.call(contentResolver,
                LauncherSettings.Settings.METHOD_LOAD_DEFAULT_FAVORITES);
```

`LauncherSettings.Settings.call(contentResolver,LauncherSettings.Settings.METHOD_LOAD_DEFAULT_FAVORITES)` 会调用到 `LauncherProvider` 的 `loadDefaultFavoritesIfNecessary()` 方法。
这个方法来判断是否需要加载默认的布局，如果清除了桌面数据或者手机恢复了出厂设置，就会加载默认的布局。

```
// LauncherProvider.loadDefaultFavoritesIfNecessary
    synchronized private void loadDefaultFavoritesIfNecessary() {
        SharedPreferences sp = Utilities.getPrefs(getContext());

        if (sp.getBoolean(EMPTY_DATABASE_CREATED, false)) {
            Log.d(TAG, "loading default workspace");

            AppWidgetHost widgetHost = mOpenHelper.newLauncherWidgetHost();
            // 获取loader，下面介绍 AutoInstallsLayout
            // 1. 尝试从系统配置的 launcher3.layout.provider 提供的 provider 里面获取布局资源
            AutoInstallsLayout loader = createWorkspaceLoaderFromAppRestriction(widgetHost);
            if (loader == null) {
                // 2. 尝试从带有 android.autoinstalls.config.action.PLAY_AUTO_INSTALL 的apk里面获取布局资源
                loader = AutoInstallsLayout.get(getContext(),widgetHost, mOpenHelper);
            }
            if (loader == null) {
                // 3. 尝试从系统内置的partner应用里获取workspace默认配置,
                // 这个功能是提供给提供布局的第三方应用
                final Partner partner = Partner.get(getContext().getPackageManager());
                if (partner != null && partner.hasDefaultLayout()) {
                    final Resources partnerRes = partner.getResources();
                    int workspaceResId = partnerRes.getIdentifier(Partner.RES_DEFAULT_LAYOUT,
                            "xml", partner.getPackageName());
                    if (workspaceResId != 0) {
                        loader = new DefaultLayoutParser(getContext(), widgetHost,
                                mOpenHelper, partnerRes, workspaceResId);
                    }
                }
            }

            final boolean usingExternallyProvidedLayout = loader != null;
            if (loader == null) {
                // 4. 尝试获取Launcher里的默认资源，比如@xml/default_workspace_5x5
                // 看下面对这个方法的详细分析
                loader = getDefaultLayoutParser(widgetHost);
            }

            // There might be some partially restored DB items, due to buggy restore logic in
            // previous versions of launcher.
            mOpenHelper.createEmptyDB(mOpenHelper.getWritableDatabase());

            // Populate favorites table with initial favorites
            // 解析布局文件并写入数据库
            // 首先时使用上面得到的loader获取，如果该loader提供的方式里面为空的布局
            // 那么还是使用上面第四部的 loader。
            if ((mOpenHelper.loadFavorites(mOpenHelper.getWritableDatabase(), loader) <= 0)
                    && usingExternallyProvidedLayout) {
                // Unable to load external layout. Cleanup and load the internal layout.
                mOpenHelper.createEmptyDB(mOpenHelper.getWritableDatabase());
                mOpenHelper.loadFavorites(mOpenHelper.getWritableDatabase(),
                        getDefaultLayoutParser(widgetHost));
            }
            
            clearFlagEmptyDbCreated();
        }
```

```
// LauncherProvider.getDefaultLayoutParser
    private DefaultLayoutParser getDefaultLayoutParser(AppWidgetHost widgetHost) {
        // InvariantDeviceProfile 的配置文件在 R.xml.device_profiles
        //  从 InvariantDeviceProfil.getPredefinedDeviceProfiles 中获取
        InvariantDeviceProfile idp = LauncherAppState.getIDP(getContext());
        // 获取布局资源id，比如：@xml/default_workspace_5x5
        int defaultLayout = idp.defaultLayoutId;

        UserManagerCompat um = UserManagerCompat.getInstance(getContext());
        if (um.isDemoUser() && idp.demoModeLayoutId != 0) {
            defaultLayout = idp.demoModeLayoutId;
        }

        return new DefaultLayoutParser(getContext(), widgetHost,
                mOpenHelper, getContext().getResources(), defaultLayout);
    }

```

如果不是第一次开机，那么上面的这一步时不需要进行的，直接进行下一步。
接下来继续分析 `LoaderTask.loadWorkspace()`。
先来简单介绍一下 BgDataModel 这个类，用来放布局相关的集合：

 - workspaceScreens是保存屏幕的
 - workspaceItems是桌面的图标
 - appWidgets是桌面的小部件
 - Folders是文件夹

下面开始读取数据库以及对从数据库里面读取的数据进行分析。

#### 加载数据库

```
// LoaderTask.loadWorkspace()
        synchronized (mBgDataModel) {
            mBgDataModel.clear();

            final HashMap<String, SessionInfo> installingPkgs =
                    mPackageInstaller.updateAndGetActiveSessionCache();
            mFirstScreenBroadcast = new FirstScreenBroadcast(installingPkgs);

            Map<ShortcutKey, ShortcutInfo> shortcutKeyToPinnedShortcuts = new HashMap<>();
            // 查询数据库
            final LoaderCursor c = new LoaderCursor(contentResolver.query(
                    LauncherSettings.Favorites.CONTENT_URI, null, null, null, null), mApp);

            HashMap<ComponentKey, AppWidgetProviderInfo> widgetProvidersMap = null;

            try {
                ...... 
                for (UserHandle user : mUserManager.getUserProfiles()) {
                    long serialNo = mUserManager.getSerialNumberForUser(user);
                    allUsers.put(serialNo, user);
                    quietMode.put(serialNo, mUserManager.isQuietModeEnabled(user));

                    boolean userUnlocked = mUserManager.isUserUnlocked(user);

                    // We can only query for shortcuts when the user is unlocked.
                    // 向 LauncherAppsService 获取 Pinned Shortcuts
                    if (userUnlocked) {
                        List<ShortcutInfo> pinnedShortcuts =
                                mShortcutManager.queryForPinnedShortcuts(null, user);
                        if (mShortcutManager.wasLastCallSuccess()) {
                            for (ShortcutInfo shortcut : pinnedShortcuts) {
                                shortcutKeyToPinnedShortcuts.put(ShortcutKey.fromInfo(shortcut),
                                        shortcut);
                            }
                        } else {
                            userUnlocked = false;
                        }
                    }
                    unlockedUsers.put(serialNo, userUnlocked);
                }
                
                WorkspaceItemInfo info;
                LauncherAppWidgetInfo appWidgetInfo;
                Intent intent;
                String targetPkg;
                // 开始遍历数据库，一次读取数据库内容
                while (!mStopped && c.moveToNext()) {
                    try {
                        ......
                        switch (c.itemType) {
```

下面开始对数据库每一条数据根据类型的不同进行不同的处理。

##### 快捷方式数据

这里的快捷方式分为三种：

 - ITEM_TYPE_APPLICATION：应用图标
 - ITEM_TYPE_SHORTCUT：应用创建的快捷图标
 - ITEM_TYPE_DEEP_SHORTCUT：Pinned Shortcuts，长按应用图标弹出菜单，从菜单里面拖出的快捷方式。

这里会根据不同的类型生成不同的对象，保存在 BgDataModel 中。

````
                        case LauncherSettings.Favorites.ITEM_TYPE_SHORTCUT:
                        case LauncherSettings.Favorites.ITEM_TYPE_APPLICATION:
                        case LauncherSettings.Favorites.ITEM_TYPE_DEEP_SHORTCUT:
                            // 解析出数据库中的 intent 字段
                            intent = c.parseIntent();
                            if (intent == null) {
                                c.markDeleted("Invalid or null intent");
                                continue;
                            }

                            int disabledState = quietMode.get(c.serialNumber) ?
                                    WorkspaceItemInfo.FLAG_DISABLED_QUIET_USER : 0;
                            ComponentName cn = intent.getComponent();
                            targetPkg = cn == null ? intent.getPackage() : cn.getPackageName();

                            if (allUsers.indexOfValue(c.user) < 0) {
                                if (c.itemType == LauncherSettings.Favorites.ITEM_TYPE_SHORTCUT) {
                                    c.markDeleted("Legacy shortcuts are only allowed for current users");
                                    continue;
                                } else if (c.restoreFlag != 0) {
                                    // Don't restore items for other profiles.
                                    c.markDeleted("Restore from other profiles not supported");
                                    continue;
                                }
                            }
                            // 如果 targetPkg为空 且不是 ITEM_TYPE_SHORTCUT，则标记删除
                            if (TextUtils.isEmpty(targetPkg) &&
                                    c.itemType != LauncherSettings.Favorites.ITEM_TYPE_SHORTCUT) {
                                c.markDeleted("Only legacy shortcuts can have null package");
                                continue;
                            }

                            // If there is no target package, its an implicit intent
                            // (legacy shortcut) which is always valid
                            boolean validTarget = TextUtils.isEmpty(targetPkg) ||
                                    mLauncherApps.isPackageEnabledForProfile(targetPkg, c.user);
                            //检查对应的应用是否在系统中为disable状态，如果为disable状态，则不显示。
                            // 通过isActivityEnabledForProfile来判断。 
                            // 当用户在设置里面对某个应用设置为disable，回到Launcher的时候，
                            //Launcher的数据库里面还是保留着该应用。
                            // 这里会进行一个判断，当数据库有，但手机不支持的时候，不显示。
                            if (cn != null && validTarget) {
                                // If the apk is present and the shortcut points to a specific
                                // component.

                                // If the component is already present
                                if (mLauncherApps.isActivityEnabledForProfile(cn, c.user)) {
                                    // no special handling necessary for this item
                                    c.markRestored();
                                } else {
                                    // Gracefully try to find a fallback activity.
                                    intent = pmHelper.getAppLaunchIntent(targetPkg, c.user);
                                    if (intent != null) {
                                        c.restoreFlag = 0;
                                        c.updater().put(
                                                LauncherSettings.Favorites.INTENT,
                                                intent.toUri(0)).commit();
                                        cn = intent.getComponent();
                                    } else {
                                        c.markDeleted("Unable to find a launch target");
                                        continue;
                                    }
                                }
                            }
                            // else if cn == null => can't infer much, leave it
                            // else if !validPkg => could be restored icon or missing sd-card

                            if (!TextUtils.isEmpty(targetPkg) && !validTarget) {
                                // Points to a valid app (superset of cn != null) but the apk
                                // is not available.

                                if (c.restoreFlag != 0) {
                                    // Package is not yet available but might be
                                    // installed later.
                                    FileLog.d(TAG, "package not yet restored: " + targetPkg);

                                    if (c.hasRestoreFlag(WorkspaceItemInfo.FLAG_RESTORE_STARTED)) {
                                        // Restore has started once.
                                    } else if (installingPkgs.containsKey(targetPkg)) {
                                        // App restore has started. Update the flag
                                        c.restoreFlag |= WorkspaceItemInfo.FLAG_RESTORE_STARTED;
                                        c.updater().put(LauncherSettings.Favorites.RESTORED,
                                                c.restoreFlag).commit();
                                    } else {
                                        c.markDeleted("Unrestored app removed: " + targetPkg);
                                        continue;
                                    }
                                } else if (pmHelper.isAppOnSdcard(targetPkg, c.user)) {
                                    // Package is present but not available.
                                    disabledState |= WorkspaceItemInfo.FLAG_DISABLED_NOT_AVAILABLE;
                                    // Add the icon on the workspace anyway.
                                    allowMissingTarget = true;
                                } else if (!isSdCardReady) {
                                    // SdCard is not ready yet. Package might get available,
                                    // once it is ready.
                                    Log.d(TAG, "Missing pkg, will check later: " + targetPkg);
                                    pendingPackages.addToList(c.user, targetPkg);
                                    // Add the icon on the workspace anyway.
                                    allowMissingTarget = true;
                                } else {
                                    // Do not wait for external media load anymore.
                                    c.markDeleted("Invalid package removed: " + targetPkg);
                                    continue;
                                }
                            }

                            if ((c.restoreFlag & WorkspaceItemInfo.FLAG_SUPPORTS_WEB_UI) != 0) {
                                validTarget = false;
                            }

                            if (validTarget) {
                                // The shortcut points to a valid target (either no target
                                // or something which is ready to be used)
                                c.markRestored();
                            }

                            boolean useLowResIcon = !c.isOnWorkspaceOrHotseat();

                            if (c.restoreFlag != 0) {
                                // Already verified above that user is same as default user
                                info = c.getRestoredItemInfo(intent);
                            } else if (c.itemType ==
                                    LauncherSettings.Favorites.ITEM_TYPE_APPLICATION) {
                                info = c.getAppShortcutInfo(
                                        intent, allowMissingTarget, useLowResIcon);
                            } else if (c.itemType ==
                                    LauncherSettings.Favorites.ITEM_TYPE_DEEP_SHORTCUT) {

                                ShortcutKey key = ShortcutKey.fromIntent(intent, c.user);
                                if (unlockedUsers.get(c.serialNumber)) {
                                    ShortcutInfo pinnedShortcut =
                                            shortcutKeyToPinnedShortcuts.get(key);
                                    if (pinnedShortcut == null) {
                                        // The shortcut is no longer valid.
                                        c.markDeleted("Pinned shortcut not found");
                                        continue;
                                    }
                                    // 根据pinnedShortcut生成 WorkspaceItemInfo 
                                    info = new WorkspaceItemInfo(pinnedShortcut, context);
                                    final WorkspaceItemInfo finalInfo = info;

                                    LauncherIcons li = LauncherIcons.obtain(context);
                                    // If the pinned deep shortcut is no longer published,
                                    // use the last saved icon instead of the default.
                                    Supplier<ItemInfoWithIcon> fallbackIconProvider = () ->
                                            c.loadIcon(finalInfo, li) ? finalInfo : null;
                                    info.applyFrom(li.createShortcutIcon(pinnedShortcut,
                                            true /* badged */, fallbackIconProvider));
                                    li.recycle();
                                    if (pmHelper.isAppSuspended(
                                            pinnedShortcut.getPackage(), info.user)) {
                                        info.runtimeStatusFlags |= FLAG_DISABLED_SUSPENDED;
                                    }
                                    intent = info.intent;
                                } else {
                                    // Create a shortcut info in disabled mode for now.
                                    info = c.loadSimpleWorkspaceItem();
                                    info.runtimeStatusFlags |= FLAG_DISABLED_LOCKED_USER;
                                }
                            } else { // item type == ITEM_TYPE_SHORTCUT
                                info = c.loadSimpleWorkspaceItem();

                                // Shortcuts are only available on the primary profile
                                if (!TextUtils.isEmpty(targetPkg)
                                        && pmHelper.isAppSuspended(targetPkg, c.user)) {
                                    disabledState |= FLAG_DISABLED_SUSPENDED;
                                }

                                // App shortcuts that used to be automatically added to Launcher
                                // didn't always have the correct intent flags set, so do that
                                // here
                                if (intent.getAction() != null &&
                                    intent.getCategories() != null &&
                                    intent.getAction().equals(Intent.ACTION_MAIN) &&
                                    intent.getCategories().contains(Intent.CATEGORY_LAUNCHER)) {
                                    intent.addFlags(
                                        Intent.FLAG_ACTIVITY_NEW_TASK |
                                        Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED);
                                }
                            }

                            if (info != null) {
                                c.applyCommonProperties(info);
                                // 快捷方式的宽高都是1
                                info.intent = intent;
                                info.rank = c.getInt(rankIndex);
                                info.spanX = 1;
                                info.spanY = 1;
                                info.runtimeStatusFlags |= disabledState;
                                if (isSafeMode && !Utilities.isSystemApp(context, intent)) {
                                    info.runtimeStatusFlags |= FLAG_DISABLED_SAFEMODE;
                                }

                                if (c.restoreFlag != 0 && !TextUtils.isEmpty(targetPkg)) {
                                    SessionInfo si = installingPkgs.get(targetPkg);
                                    if (si == null) {
                                        info.status &= ~WorkspaceItemInfo.FLAG_INSTALL_SESSION_ACTIVE;
                                    } else {
                                        info.setInstallProgress((int) (si.getProgress() * 100));
                                    }
                                }
                                // 将 WorkspaceItemInfo 添加到 mBgDataModel
                                c.checkAndAddItem(info, mBgDataModel);
                            } else {
                                throw new RuntimeException("Unexpected null WorkspaceItemInfo");
                            }
                            break;
```


#####  文件夹类型数据


```
//LoaderTask.loadWorkspace()
                        case LauncherSettings.Favorites.ITEM_TYPE_FOLDER:
                            FolderInfo folderInfo = mBgDataModel.findOrMakeFolder(c.id);
                            c.applyCommonProperties(folderInfo);

                            // Do not trim the folder label, as is was set by the user.
                            folderInfo.title = c.getString(c.titleIndex);
                            // 文件夹的宽高都是1
                            folderInfo.spanX = 1;
                            folderInfo.spanY = 1;
                            folderInfo.options = c.getInt(optionsIndex);

                            // no special handling required for restored folders
                            c.markRestored();
                            // 将 folderInfo 放入 mBgDataModel
                            c.checkAndAddItem(folderInfo, mBgDataModel);
                            break;
```

##### 桌面插件类型数据

```
//LoaderTask.loadWorkspace()
                        case LauncherSettings.Favorites.ITEM_TYPE_APPWIDGET:
                            if (FeatureFlags.GO_DISABLE_WIDGETS) {
                                c.markDeleted("Only legacy shortcuts can have null package");
                                continue;
                            }
                            // Follow through
                        case LauncherSettings.Favorites.ITEM_TYPE_CUSTOM_APPWIDGET:
                            ......
                            final AppWidgetProviderInfo provider = widgetProvidersMap.get(
                                    new ComponentKey(
                                            ComponentName.unflattenFromString(savedProvider),
                                            c.user));
                            // 判断该 widget 是否存在 provider
                            final boolean isProviderReady = isValidProvider(provider);
                            ......
                            break;
                        }
                    } catch (Exception e) {
                        Log.e(TAG, "Desktop items loading interrupted", e);
                    }
                }
            } finally {
                Utilities.closeSilently(c);
            }
```

##### 数据的最终处理

对数据进一步处理，如果前面标记了清除，那么这里就从数据库中进行删除对应数据。

```
//LoaderTask.loadWorkspace()
            // Break early if we've stopped loading
            if (mStopped) {
                mBgDataModel.clear();
                return;
            }

            // Remove dead items
            // 删除前面标记delete的数据
            if (c.commitDeleted()) {
                // Remove any empty folder
                int[] deletedFolderIds = LauncherSettings.Settings
                        .call(contentResolver,
                                LauncherSettings.Settings.METHOD_DELETE_EMPTY_FOLDERS)
                        .getIntArray(LauncherSettings.Settings.EXTRA_VALUE);
                for (int folderId : deletedFolderIds) {
                    mBgDataModel.workspaceItems.remove(mBgDataModel.folders.get(folderId));
                    mBgDataModel.folders.remove(folderId);
                    mBgDataModel.itemsIdMap.remove(folderId);
                }

                // Remove any ghost widgets
                LauncherSettings.Settings.call(contentResolver,
                        LauncherSettings.Settings.METHOD_REMOVE_GHOST_WIDGETS);
            }

            // Unpin shortcuts that don't exist on the workspace.
            HashSet<ShortcutKey> pendingShortcuts =
                    InstallShortcutReceiver.getPendingShortcuts(context);
            for (ShortcutKey key : shortcutKeyToPinnedShortcuts.keySet()) {
                MutableInt numTimesPinned = mBgDataModel.pinnedShortcutCounts.get(key);
                if ((numTimesPinned == null || numTimesPinned.value == 0)
                        && !pendingShortcuts.contains(key)) {
                    // Shortcut is pinned but doesn't exist on the workspace; unpin it.
                    mShortcutManager.unpinShortcut(key);
                }
            }

            // Sort the folder items, update ranks, and make sure all preview items are high res.
            FolderIconPreviewVerifier verifier =
                    new FolderIconPreviewVerifier(mApp.getInvariantDeviceProfile());
            for (FolderInfo folder : mBgDataModel.folders) {
                Collections.sort(folder.contents, Folder.ITEM_POS_COMPARATOR);
                verifier.setFolderInfo(folder);
                int size = folder.contents.size();

                // Update ranks here to ensure there are no gaps caused by removed folder items.
                // Ranks are the source of truth for folder items, so cellX and cellY can be ignored
                // for now. Database will be updated once user manually modifies folder.
                for (int rank = 0; rank < size; ++rank) {
                    WorkspaceItemInfo info = folder.contents.get(rank);
                    info.rank = rank;

                    if (info.usingLowResIcon()
                            && info.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPLICATION
                            && verifier.isItemInPreview(info.rank)) {
                        mIconCache.getTitleAndIcon(info, false);
                    }
                }
            }

            c.commitRestoredItems();
            if (!isSdCardReady && !pendingPackages.isEmpty()) {
                context.registerReceiver(
                        new SdCardAvailableReceiver(mApp, pendingPackages),
                        new IntentFilter(Intent.ACTION_BOOT_COMPLETED),
                        null,
                        new Handler(LauncherModel.getWorkerLooper()));
            }
        }
    }
```


### bindWorkspace

```
// BaseLoaderResults.java

    public void bindWorkspace() {
        Callbacks callbacks = mCallbacks.get();
        // Launcher 实现了 callback
        if (callbacks == null) {
            // 如果没有callback，就没有bind的必要了
            Log.w(TAG, "LoaderTask running with no launcher");
            return;
        }

        // Save a copy of all the bg-thread collections
        ArrayList<ItemInfo> workspaceItems = new ArrayList<>();
        ArrayList<LauncherAppWidgetInfo> appWidgets = new ArrayList<>();
        final IntArray orderedScreenIds = new IntArray();

        synchronized (mBgDataModel) {
            // 从 mBgDataModel 获取数据
            workspaceItems.addAll(mBgDataModel.workspaceItems);
            appWidgets.addAll(mBgDataModel.appWidgets);
            orderedScreenIds.addAll(mBgDataModel.collectWorkspaceScreens());
            mBgDataModel.lastBindId++;
            mMyBindingId = mBgDataModel.lastBindId;
        }

        final int currentScreen;
        {
            int currScreen = mPageToBindFirst != PagedView.INVALID_RESTORE_PAGE
                    ? mPageToBindFirst : callbacks.getCurrentWorkspaceScreen();
            if (currScreen >= orderedScreenIds.size()) {
                // There may be no workspace screens (just hotseat items and an empty page).
                currScreen = PagedView.INVALID_RESTORE_PAGE;
            }
            currentScreen = currScreen;
        }
        final boolean validFirstPage = currentScreen >= 0;
        final int currentScreenId =
                validFirstPage ? orderedScreenIds.get(currentScreen) : INVALID_SCREEN_ID;

        // 过滤数据，将要绑定的内容分为当前屏幕的和非当前的数据
        ArrayList<ItemInfo> currentWorkspaceItems = new ArrayList<>();
        ArrayList<ItemInfo> otherWorkspaceItems = new ArrayList<>();
        ArrayList<LauncherAppWidgetInfo> currentAppWidgets = new ArrayList<>();
        ArrayList<LauncherAppWidgetInfo> otherAppWidgets = new ArrayList<>();

        filterCurrentWorkspaceItems(currentScreenId, workspaceItems, currentWorkspaceItems,
                otherWorkspaceItems);
        filterCurrentWorkspaceItems(currentScreenId, appWidgets, currentAppWidgets,
                otherAppWidgets);
        sortWorkspaceItemsSpatially(currentWorkspaceItems);
        sortWorkspaceItemsSpatially(otherWorkspaceItems);

        // 告诉 Launcher 开始 bind
        executeCallbacksTask(c -> {
            c.clearPendingBinds();
            c.startBinding();
        }, mUiExecutor);

        // 绑定 workspace screens
        executeCallbacksTask(c -> c.bindScreens(orderedScreenIds), mUiExecutor);

        Executor mainExecutor = mUiExecutor;
        // Load items on the current page.
        bindWorkspaceItems(currentWorkspaceItems, mainExecutor);
        bindAppWidgets(currentAppWidgets, mainExecutor);
        // In case of validFirstPage, only bind the first screen, and defer binding the
        // remaining screens after first onDraw (and an optional the fade animation whichever
        // happens later).
        // This ensures that the first screen is immediately visible (eg. during rotation)
        // In case of !validFirstPage, bind all pages one after other.
        // Launcher为了让默认页尽快显示，自定义了一个ViewOnDrawExecutor
        // 绑定完第一页的元素并且第一次onDraw执行完之后才执行绑定其他页的操作
        final Executor deferredExecutor =
                validFirstPage ? new ViewOnDrawExecutor() : mainExecutor;

        executeCallbacksTask(c -> c.finishFirstPageBind(
                validFirstPage ? (ViewOnDrawExecutor) deferredExecutor : null), mainExecutor);

        bindWorkspaceItems(otherWorkspaceItems, deferredExecutor);
        bindAppWidgets(otherAppWidgets, deferredExecutor);
        // 通知 Launcher bind 工作已经完成
        executeCallbacksTask(c -> c.finishBindingItems(mPageToBindFirst), deferredExecutor);

        if (validFirstPage) {
            executeCallbacksTask(c -> {
                // We are loading synchronously, which means, some of the pages will be
                // bound after first draw. Inform the callbacks that page binding is
                // not complete, and schedule the remaining pages.
                if (currentScreen != PagedView.INVALID_RESTORE_PAGE) {
                    c.onPageBoundSynchronously(currentScreen);
                }
                c.executeOnNextDraw((ViewOnDrawExecutor) deferredExecutor);

            }, mUiExecutor);
        }
    }
```

最终的数据绑定和快捷图标的生成在Launcher.bindItems中：

```
    public void bindItems(final List<ItemInfo> items, final boolean forceAnimateIcons) {
        // Get the list of added items and intersect them with the set of items here
        final Collection<Animator> bounceAnims = new ArrayList<>();
        final boolean animateIcons = forceAnimateIcons && canRunNewAppsAnimation();
        Workspace workspace = mWorkspace;
        int newItemsScreenId = -1;
        int end = items.size();

        for (int i = 0; i < end; i++) {
            final ItemInfo item = items.get(i);

            // Short circuit if we are loading dock items for a configuration which has no dock
            if (item.container == LauncherSettings.Favorites.CONTAINER_HOTSEAT &&
                    mHotseat == null) {
                continue;
            }

            final View view;
            switch (item.itemType) {
                case LauncherSettings.Favorites.ITEM_TYPE_APPLICATION:
                case LauncherSettings.Favorites.ITEM_TYPE_SHORTCUT:
                case LauncherSettings.Favorites.ITEM_TYPE_DEEP_SHORTCUT: {
                    WorkspaceItemInfo info = (WorkspaceItemInfo) item;
                    view = createShortcut(info);
                    break;
                }
                case LauncherSettings.Favorites.ITEM_TYPE_FOLDER: {
                    view = FolderIcon.inflateFolderAndIcon(R.layout.folder_icon, this,
                            (ViewGroup) workspace.getChildAt(workspace.getCurrentPage()),
                            (FolderInfo) item);
                    break;
                }
                case LauncherSettings.Favorites.ITEM_TYPE_APPWIDGET:
                case LauncherSettings.Favorites.ITEM_TYPE_CUSTOM_APPWIDGET: {
                    view = inflateAppWidget((LauncherAppWidgetInfo) item);
                    if (view == null) {
                        continue;
                    }
                    break;
                }
                default:
                    throw new RuntimeException("Invalid Item Type");
            }

            /*
             * Remove colliding items.
             */
            if (item.container == LauncherSettings.Favorites.CONTAINER_DESKTOP) {
                CellLayout cl = mWorkspace.getScreenWithId(item.screenId);
                if (cl != null && cl.isOccupied(item.cellX, item.cellY)) {
                    View v = cl.getChildAt(item.cellX, item.cellY);
                    Object tag = v.getTag();
                    String desc = "Collision while binding workspace item: " + item
                            + ". Collides with " + tag;
                    if (FeatureFlags.IS_STUDIO_BUILD) {
                        throw (new RuntimeException(desc));
                    } else {
                        getModelWriter().deleteItemFromDatabase(item);
                        continue;
                    }
                }
            }

            workspace.addInScreenFromBind(view, item); // 添加到CellLayout中
            if (animateIcons) {
                // Animate all the applications up now
                view.setAlpha(0f);
                view.setScaleX(0f);
                view.setScaleY(0f);
                bounceAnims.add(createNewAppBounceAnimation(view, i));
                newItemsScreenId = item.screenId;
            }
        }

        // Animate to the correct page
        if (animateIcons && newItemsScreenId > -1) {
            AnimatorSet anim = new AnimatorSet();
            anim.playTogether(bounceAnims);

            int currentScreenId = mWorkspace.getScreenIdForPageIndex(mWorkspace.getNextPage());
            final int newScreenIndex = mWorkspace.getPageIndexForScreenId(newItemsScreenId);
            final Runnable startBounceAnimRunnable = anim::start;

            if (newItemsScreenId != currentScreenId) {
                // We post the animation slightly delayed to prevent slowdowns
                // when we are loading right after we return to launcher.
                mWorkspace.postDelayed(new Runnable() {
                    public void run() {
                        if (mWorkspace != null) {
                            closeOpenViews(false);
                            mWorkspace.snapToPage(newScreenIndex);
                            mWorkspace.postDelayed(startBounceAnimRunnable,
                                    NEW_APPS_ANIMATION_DELAY);
                        }
                    }
                }, NEW_APPS_PAGE_MOVE_DELAY);
            } else {
                mWorkspace.postDelayed(startBounceAnimRunnable, NEW_APPS_ANIMATION_DELAY);
            }
        }

        workspace.requestLayout();
    }
```

### loadAllApps

加载系统中已经安装的应用

### bindAllApps

### bindDeepShortcuts