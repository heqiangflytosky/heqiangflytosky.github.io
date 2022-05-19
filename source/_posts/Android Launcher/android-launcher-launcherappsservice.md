---
title: Android -- LauncherAppsService
categories:  Android Launcher
comments: true
tags: [Launcher3]
description: 介绍 LauncherAppsService
date: 2021-2-1 10:00:00
---


## 概述

Launcher 是管理系统安装应用的快捷方式的，那么如果应用发生了变更行为，比如安装，卸载，应用创建的快捷方式变更等，Luancher 都要做出对应的变化。这部分可以通过 `mLauncherApps.registerCallback` 来实现，当有变更发生时，系统服务会通过回调来通知 Launcher。
这里的快捷方式变化只的是，应用通过静态或者动态方式创建的快捷方式的属性发生了变化，比如：应用删除静态快捷方式、disable 快捷方式等。
LauncherAppsService 继承自 SystemService，是系统的服务之一，主要是协助 Launcher 管理本地的 package 的。在 LauncherAppsService 中会实例化 LauncherAppsImpl 类。
那么 Launcher 时如何和 LauncherAppsService 进行通讯的呢？在客户端是通过 LauncherApps 来实现的，它和 LauncherAppsImpl 通过 aidl 进行通讯。

## LauncherApps

LauncherApps 提供了一些接口：

 - getProfiles
 - getActivityList
 - resolveActivity
 - startMainActivity
 - startPackageInstallerSessionDetailsActivity
 - startAppDetailsActivity
 - getShortcutConfigActivityList
 - getShortcutConfigActivityIntent
 - isPackageEnabled
 - getSuspendedPackageLauncherExtras
 - shouldHideFromSuggestions
 - getApplicationInfo
 - getAppUsageLimit
 - isActivityEnabled
 - hasShortcutHostPermission
 - getShortcuts
 - getShortcutInfo
 - pinShortcuts
 - getShortcutIconResId
 - getShortcutIconFd
 - getShortcutIconDrawable
 - getShortcutBadgedIconDrawable
 - startShortcut
 - registerCallback:注册 Callback
 - unregisterCallback
 - registerPackageInstallerSessionCallback
 - unregisterPackageInstallerSessionCallback
 - getAllPackageInstallerSessions
 - getPinItemRequest

在 LauncherApps 中有个 CallBack 类，如果注册了这个回调，那么当包有变化的时候将会通知客户端：

```
    public static abstract class Callback {
        abstract public void onPackageRemoved(String packageName, UserHandle user);
        abstract public void onPackageAdded(String packageName, UserHandle user);
        abstract public void onPackageChanged(String packageName, UserHandle user);
        abstract public void onPackagesAvailable(String[] packageNames, UserHandle user,
                boolean replacing);
        abstract public void onPackagesUnavailable(String[] packageNames, UserHandle user,
                boolean replacing);
        public void onPackagesSuspended(String[] packageNames, UserHandle user) {
        }
        public void onShortcutsChanged(@NonNull String packageName,
                @NonNull List<ShortcutInfo> shortcuts, @NonNull UserHandle user) {
        }
    }
```

在 Launcher 中 LauncherAppsCompatVL.WrappedCallback 继承了 LauncherApps.Callback，来监听安装应用的各种变更，从而来对应用的快捷方式做出变更。
在 LauncherAppState 中调用 `LauncherAppsCompat.getInstance(mContext).addOnAppsChangedCallback(mModel)` 通过 `mLauncherApps.registerCallback(wrappedCallback)` 来注册回调函数。
`mLauncherApps` 的实例化时通过 `(LauncherApps) context.getSystemService(Context.LAUNCHER_APPS_SERVICE)` 实现。

## LauncherAppsCompat

Launcher 中实现了 LauncherAppsCompatVL、LauncherAppsCompatVO 和 LauncherAppsCompatVQ 来应对 LauncherApps 版本的升级带来的变化。

## LauncherAppsService

LauncherAppsService 中的 LauncherAppsImpl 是负责和客户端进行通讯的。

```
    @Override
    public void onStart() {
        publishBinderService(Context.LAUNCHER_APPS_SERVICE, mLauncherAppsImpl);
    }
```

我们主要介绍一下 LauncherAppsImpl 的成员变量：

```
private final MyPackageMonitor mPackageMonitor
```

MyPackageMonitor 分别继承和实现了 PackageMonitor 和 ShortcutChangeListener，用来监听应用以及快捷方式的变化。

```
        private class MyPackageMonitor extends PackageMonitor implements ShortcutChangeListener {
            @Override
            public void onPackageAdded(String packageName, int uid) {}

            @Override
            public void onPackageRemoved(String packageName, int uid) {}

            @Override
            public void onPackageModified(String packageName) {}

            @Override
            public void onPackagesAvailable(String[] packages) {}

            @Override
            public void onPackagesUnavailable(String[] packages) {}

            @Override
            public void onPackagesSuspended(String[] packages, Bundle launcherExtras) {}

            @Override
            public void onPackagesUnsuspended(String[] packages) {}

            @Override
            public void onShortcutChanged(@NonNull String packageName,
                    @UserIdInt int userId) {}

            private void onShortcutChangedInner(@NonNull String packageName,
                    @UserIdInt int userId) {}
            }
        }

        class PackageCallbackList<T extends IInterface> extends RemoteCallbackList<T> {
            @Override
            public void onCallbackDied(T callback, Object cookie) {
                checkCallbackCount();
            }
        }
    }
```

PackageMonitor 继承自 BroadcastReceiver，内部添加了一些广播的监听器：

```
    static {
        sPackageFilt.addAction(Intent.ACTION_PACKAGE_ADDED);
        sPackageFilt.addAction(Intent.ACTION_PACKAGE_REMOVED);
        sPackageFilt.addAction(Intent.ACTION_PACKAGE_CHANGED);
        sPackageFilt.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
        sPackageFilt.addAction(Intent.ACTION_PACKAGE_RESTARTED);
        sPackageFilt.addAction(Intent.ACTION_PACKAGE_DATA_CLEARED);
        sPackageFilt.addDataScheme("package");
        sNonDataFilt.addAction(Intent.ACTION_UID_REMOVED);
        sNonDataFilt.addAction(Intent.ACTION_USER_STOPPED);
        sNonDataFilt.addAction(Intent.ACTION_PACKAGES_SUSPENDED);
        sNonDataFilt.addAction(Intent.ACTION_PACKAGES_UNSUSPENDED);
        sExternalFilt.addAction(Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE);
        sExternalFilt.addAction(Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE);
    }
```

又通过向 ShortcutService 注册回调来监听快捷方式的变化。

```
mShortcutServiceInternal.addListener(mPackageMonitor);
```