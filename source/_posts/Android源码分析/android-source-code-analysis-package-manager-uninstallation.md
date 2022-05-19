---
title: Android PackageManager相关源码分析之卸载应用
categories: Android源码分析
comments: true
tags: [Android源码分析, PackageManager]
description: 基于Android6.0来分析Android的应用卸载流程
date: 2016-5-16 10:00:00
---

## 卸载应用

我们可以通过下面的代码执行一个应用的卸载工作：

```java
    getPackageManager().deletePackage(mAppInfo.packageName, observer,  mAllUsers ? PackageManager.DELETE_ALL_USERS : 0);
```

它最后会调用到 `ApplicantPackageManger` 的`deletePackage()`方法，通过Binder调用，会调用 PMS 中的`deletePackageAsUser()`方法。
那么接下来我们就从 PMS 的 `deletePackageAsUser()` 方法来开始分析应用的卸载流程。
Android的应用卸载有下面几个阶段的工作：
 - 删除和该包相关的 acitivity、service、provider 等信息
 - 删除 code、library和resource 等信息
 - 调用 installd 删除 /data/data/ 以及 /data/dalvik-cache 下面的文件(经过dexopt过的文件)
 - 更新 `Settings` 中的 package 信息

代码调用流程如下：

```java
├── PMS.deletePackageAsUser()
    └── PMS.deletePackage()
         ├── deletePackageX()
         │    ├── deletePackageLI()
         │    │    ├── Installer.clearUserData()
         │    │    ├── removeKeystoreDataIfNeeded()
         │    │    ├── schedulePackageCleaning()
         │    │    ├── clearPackagePreferredActivitiesLPw()
         │    │    ├── scheduleWritePackageRestrictionsLocked()
         │    │    ├── resetUserChangesToRuntimePermissionsAndFlagsLPw()
         │    │    ├── removePackageDataLI()
         │    │    ├── 1 deleteSystemPackageLI()
         │    │    │      ├── deleteInstalledPackageLI
         │    │    │      ├── Settings.enableSystemPackageLPw()
         │    │    │      ├── NativeLibraryHelper.removeNativeBinariesLI()
         │    │    │      ├── scanPackageLI()
         │    │    │      ├── updatePermissionsLPw()
         │    │    │      └── Settings.writeLPr()
         │    │    ├── 2 killApplication()
         │    │    └── 2 deleteInstalledPackageLI()
         │    │           ├── removePackageDataLI()
         │    │           │    ├── removePackageLI()
         │    │           │    │    ├── PackageSetting.remove()
         │    │           │    │    └── cleanPackageDataStructuresLILPw()
         │    │           │    ├── removeDataDirsLI()
         │    │           │    ├── schedulePackageCleaning()
         │    │           │    │    │ post Message START_CLEANING_PACKAGE
         │    │           │    │    └── startCleaningPackages()
         │    │           │    ├── clearIntentFilterVerificationsLPw()
         │    │           │    ├── clearDefaultBrowserIfNeeded()
         │    │           │    ├── Settings.removePackageLPw()
         │    │           │    ├── updatePermissionsLPw()
         │    │           │    ├── clearPackagePreferredActivitiesLPw()
         │    │           │    ├── PackageSetting.setInstalled()
         │    │           │    ├── Settings.writeLPr()
         │    │           │    └── removeKeystoreDataIfNeeded()
         │    │           └── createInstallArgsForExisting()
         │    ├── PackageRemovedInfo.sendBroadcast()
         │    └── FileInstallArgs.doPostDeleteLI()
         │         └── FileInstallArgs.cleanUpResourcesLI()
         │              ├── PackageParser.parsePackageLite()
         │              ├── FileInstallArgs.cleanUp()
         │              └── removeDexFiles()
         └── IPackageDeleteObserver.onPackageDeleted()
```

### deletePackage()

```java
    @Override
    public void deletePackage(final String packageName,
            final IPackageDeleteObserver2 observer, final int userId, final int flags) {
        // 执行 DELETE_PACKAGES 权限检查
        mContext.enforceCallingOrSelfPermission(
                android.Manifest.permission.DELETE_PACKAGES, null);
        Preconditions.checkNotNull(packageName);
        Preconditions.checkNotNull(observer);
        final int uid = Binder.getCallingUid();
        if (UserHandle.getUserId(uid) != userId) {
            mContext.enforceCallingPermission(
                    android.Manifest.permission.INTERACT_ACROSS_USERS_FULL,
                    "deletePackage for user " + userId);
        }
        // 如果该用户不允许卸载应用，则执行回调函数并返回
        if (isUserRestricted(userId, UserManager.DISALLOW_UNINSTALL_APPS)) {
            try {
                observer.onPackageDeleted(packageName,
                        PackageManager.DELETE_FAILED_USER_RESTRICTED, null);
            } catch (RemoteException re) {}
            return;
        }
        // 查看是否禁止有某用户卸载该应用，可以通过PMS.setBlockUninstallForUser()来修改该标记
        boolean uninstallBlocked = false;
        if ((flags & PackageManager.DELETE_ALL_USERS) != 0) {
            int[] users = sUserManager.getUserIds();
            for (int i = 0; i < users.length; ++i) {
                if (getBlockUninstallForUser(packageName, users[i])) {
                    uninstallBlocked = true;
                    break;
                }
            }
        } else {
            uninstallBlocked = getBlockUninstallForUser(packageName, userId);
        }
        if (uninstallBlocked) {
            try {
                observer.onPackageDeleted(packageName, PackageManager.DELETE_FAILED_OWNER_BLOCKED,
                        null);
            } catch (RemoteException re) {}
            return;
        }

        // 由于卸载应用可能耗时较长，因此通过异步来执行卸载工作
        mHandler.post(new Runnable() {
            public void run() {
                mHandler.removeCallbacks(this);
                // 调用deletePackageX()来执行卸载工作
                final int returnCode = deletePackageX(packageName, userId, flags);
                if (observer != null) {
                    try {
                        // 卸载完成，调用回调函数
                        observer.onPackageDeleted(packageName, returnCode, null);
                    } catch (RemoteException e) {} //end catch
                } //end if
            } //end run
        });
    }
```

### deletePackageX()

实际的卸载工作是由 `deletePackageX()` 方法来完成的。

```java
    private int deletePackageX(String packageName, int userId, int flags) {
        final PackageRemovedInfo info = new PackageRemovedInfo();
        final boolean res;

        final UserHandle removeForUser = (flags & PackageManager.DELETE_ALL_USERS) != 0
                ? UserHandle.ALL : new UserHandle(userId);
        // 检查userId代表的用户是否有权限删除该应用
        if (isPackageDeviceAdmin(packageName, removeForUser.getIdentifier())) {
            return PackageManager.DELETE_FAILED_DEVICE_POLICY_MANAGER;
        }

        boolean removedForAllUsers = false;
        boolean systemUpdate = false;

        int[] allUsers;
        boolean[] perUserInstalled;
        synchronized (mPackages) {
            // 获取Settings保存的该应用的信息
            PackageSetting ps = mSettings.mPackages.get(packageName);
            // 检查一下是否系统中的所有用户都安装了这个应用
            allUsers = sUserManager.getUserIds();
            perUserInstalled = new boolean[allUsers.length];
            for (int i = 0; i < allUsers.length; i++) {
                perUserInstalled[i] = ps != null ? ps.getInstalled(allUsers[i]) : false;
            }
        }

        synchronized (mInstallLock) {
            // 调用deletePackageLI()来卸载应用
            res = deletePackageLI(packageName, removeForUser,
                    true, allUsers, perUserInstalled,
                    flags | REMOVE_CHATTY, info, true);
            systemUpdate = info.isRemovedPackageSystemUpdate;
            if (res && !systemUpdate && mPackages.get(packageName) == null) {
                removedForAllUsers = true;
            }
        }

        if (res) {
            info.sendBroadcast(true, systemUpdate, removedForAllUsers);

            // 如果被卸载的是系统应用，那么它的低版本的应用将被重新使用
            // 因此下面通过广播来通知这种变化
            if (systemUpdate) {
                Bundle extras = new Bundle(1);
                extras.putInt(Intent.EXTRA_UID, info.removedAppId >= 0
                        ? info.removedAppId : info.uid);
                extras.putBoolean(Intent.EXTRA_REPLACING, true);

                sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName,
                        extras, null, null, null);
                sendPackageBroadcast(Intent.ACTION_PACKAGE_REPLACED, packageName,
                        extras, null, null, null);
                sendPackageBroadcast(Intent.ACTION_MY_PACKAGE_REPLACED, null,
                        null, packageName, null, null);
            }
        }
        // 卸载完成后执行一次GC操作
        Runtime.getRuntime().gc();
        // Delete the resources here after sending the broadcast to let
        // other processes clean up before deleting resources.
        // 调用FileInstallArgs.doPostDeleteLI()方法来执行一些资源文件的清理工作，后面会详细介绍该方法
        if (info.args != null) {
            synchronized (mInstallLock) {
                info.args.doPostDeleteLI(true);
            }
        }

        return res ? PackageManager.DELETE_SUCCEEDED : PackageManager.DELETE_FAILED_INTERNAL_ERROR;
    }
```

### deletePackageLI()

```java
    private boolean deletePackageLI(String packageName, UserHandle user,
            boolean deleteCodeAndResources, int[] allUserHandles, boolean[] perUserInstalled,
            int flags, PackageRemovedInfo outInfo,
            boolean writeSettings) {
        if (packageName == null) {
            return false;
        }
        PackageSetting ps;
        boolean dataOnly = false;
        int removeUser = -1;
        int appId = -1;
        synchronized (mPackages) {
            ps = mSettings.mPackages.get(packageName);
            // 判断该应用是否存在
            if (ps == null) {
                return false;
            }
            if ((!isSystemApp(ps) || (flags&PackageManager.DELETE_SYSTEM_APP) != 0) && user != null
                    && user.getIdentifier() != UserHandle.USER_ALL) {
                // 如果是只卸载某个用户下的应用
                final int userId = user.getIdentifier();
                ps.setUserState(userId,
                        COMPONENT_ENABLED_STATE_DEFAULT,
                        false, //installed
                        true,  //stopped
                        true,  //notLaunched
                        false, //hidden
                        null, null, null,
                        false, // blockUninstall
                        ps.readUserState(userId).domainVerificationStatus, 0);
                if (!isSystemApp(ps)) {
                    // 非系统应用的卸载
                    if (ps.isAnyInstalled(sUserManager.getUserIds())) {
                        // 如果有其他用户还在使用这个应用，删除应用在这个用户下面的数据
                        removeUser = user.getIdentifier();
                        appId = ps.appId;
                        scheduleWritePackageRestrictionsLocked(removeUser);
                    } else {
                        // 如果没有其他用户在使用了，设置一下PackageUserState.installed，这样可以完全删除这个应用
                        ps.setInstalled(true, user.getIdentifier());
                    }
                } else {
                    // 如果卸载的是系统应用，可能别的用户还在使用它，因此这里只需要清除需要卸载的应用目录下的数据即可
                    removeUser = user.getIdentifier();
                    appId = ps.appId;
                    scheduleWritePackageRestrictionsLocked(removeUser);
                }
            }
        }

        if (removeUser >= 0) {
            // 如果是单个用户应用卸载，继续执行这里
            if (outInfo != null) {
                outInfo.removedPackage = packageName;
                outInfo.removedAppId = appId;
                outInfo.removedUsers = new int[] {removeUser};
            }
            // 调用installd执行rmuserdata，清除应用的该用户的数据
            mInstaller.clearUserData(ps.volumeUuid, packageName, removeUser);
            removeKeystoreDataIfNeeded(removeUser, appId);
            schedulePackageCleaning(packageName, removeUser, false);
            synchronized (mPackages) {
                if (clearPackagePreferredActivitiesLPw(packageName, removeUser)) {
                    scheduleWritePackageRestrictionsLocked(removeUser);
                }
                resetUserChangesToRuntimePermissionsAndFlagsLPw(ps, removeUser);
            }
            return true;
        }

        if (dataOnly) {
            // 先删除应用的数据
            removePackageDataLI(ps, null, null, outInfo, flags, writeSettings);
            return true;
        }

        boolean ret = false;
        if (isSystemApp(ps)) {
            // 删除系统应用，这里会把当前版本的资源删除然后恢复旧版本
            ret = deleteSystemPackageLI(ps, allUserHandles, perUserInstalled,
                    flags, outInfo, writeSettings);
        } else {
            // 如果不是系统应用，停止该进程，然后删除文件
            killApplication(packageName, ps.appId, "uninstall pkg");
            ret = deleteInstalledPackageLI(ps, deleteCodeAndResources, flags,
                    allUserHandles, perUserInstalled,
                    outInfo, writeSettings);
        }

        return ret;
    }
```
`deletePackageLI()`的主要工作就是根据不同用户的安装情况来删除应用的APK文件和数据。

### deleteSystemPackageLI()

顾名思义，`deleteSystemPackageLI()`就是用来删除系统应用的方法。

```java
    private boolean deleteSystemPackageLI(PackageSetting newPs,
            int[] allUserHandles, boolean[] perUserInstalled,
            int flags, PackageRemovedInfo outInfo, boolean writeSettings) {
        final boolean applyUserRestrictions
                = (allUserHandles != null) && (perUserInstalled != null);
        PackageSetting disabledPs = null;
        // 首先从Settings里面看该应用是否是已经升级过的应用
        synchronized (mPackages) {
            disabledPs = mSettings.getDisabledSystemPkgLPr(newPs.name);
        }
        if (disabledPs == null) {
            // 如果是没有升级过的应用，就直接返回，因为系统应用是无法完全卸载的，只能卸载更新
            return false;
        }
        ...
        // 开始卸载更新
        outInfo.isRemovedPackageSystemUpdate = true;
        if (disabledPs.versionCode < newPs.versionCode) {
            // 当前应用版本小于被升级应用，不需要保留数据
            flags &= ~PackageManager.DELETE_KEEP_DATA;
        } else {
            // 保留数据
            flags |= PackageManager.DELETE_KEEP_DATA;
        }
        // 调用deleteInstalledPackageLI删除应用数据，下面有详细介绍
        boolean ret = deleteInstalledPackageLI(newPs, true, flags,
                allUserHandles, perUserInstalled, outInfo, writeSettings);
        if (!ret) {
            return false;
        }
        // writer
        synchronized (mPackages) {
            // 继续使用旧的系统应用
            mSettings.enableSystemPackageLPw(newPs.name);
            // 删除更新包的native依赖库
            NativeLibraryHelper.removeNativeBinariesLI(newPs.legacyNativeLibraryPathString);
        }

        // 开始安装旧的系统应用包
        int parseFlags = PackageParser.PARSE_MUST_BE_APK | PackageParser.PARSE_IS_SYSTEM;
        if (locationIsPrivileged(disabledPs.codePath)) {
            parseFlags |= PackageParser.PARSE_IS_PRIVILEGED;
        }

        // 扫描
        final PackageParser.Package newPkg;
        try {
            newPkg = scanPackageLI(disabledPs.codePath, parseFlags, SCAN_NO_PATHS, 0, null);
        } catch (PackageManagerException e) {
            return false;
        }

        synchronized (mPackages) {
            PackageSetting ps = mSettings.mPackages.get(newPkg.packageName);
            ps.getPermissionsState().copyFrom(newPs.getPermissionsState());
            updatePermissionsLPw(newPkg.packageName, newPkg,
                    UPDATE_PERMISSIONS_ALL | UPDATE_PERMISSIONS_REPLACE_PKG);

            if (applyUserRestrictions) {
                for (int i = 0; i < allUserHandles.length; i++) {
                    ps.setInstalled(perUserInstalled[i], allUserHandles[i]);
                    mSettings.writeRuntimePermissionsForUserLPr(allUserHandles[i], false);
                }
                mSettings.writeAllUsersPackageRestrictionsLPr();
            }
            if (writeSettings) {
                mSettings.writeLPr();
            }
        }
        return true;
    }
```

### deleteInstalledPackageLI()

卸载非系统应用，包括删除和该包相关的数据结构以及数据目录，以及创建一个`FileInstallArgs`来删除 code 和 resources 文件。

```java
    private boolean deleteInstalledPackageLI(PackageSetting ps,
            boolean deleteCodeAndResources, int flags,
            int[] allUserHandles, boolean[] perUserInstalled,
            PackageRemovedInfo outInfo, boolean writeSettings) {
        if (outInfo != null) {
            outInfo.uid = ps.appId;
        }
        // 删除和该包相关的数据结构以及数据目录，下面有详细介绍
        removePackageDataLI(ps, allUserHandles, perUserInstalled, outInfo, flags, writeSettings);
        // 删除 code 和 resources，在deletePackageX()的info.args.doPostDeleteLI(true)执行
        if (deleteCodeAndResources && (outInfo != null)) {
            outInfo.args = createInstallArgsForExisting(packageFlagsToInstallFlags(ps),
                    ps.codePathString, ps.resourcePathString, getAppDexInstructionSets(ps));
        }
        return true;
    }
```

### removePackageDataLI()
删除和该包相关的数据，包括activity、provider、service等数据，如果没有设置 `PackageManager.DELETE_KEEP_DATA`，还会删除/data/data下面的目录。
```java
    private void removePackageDataLI(PackageSetting ps,
            int[] allUserHandles, boolean[] perUserInstalled,
            PackageRemovedInfo outInfo, int flags, boolean writeSettings) {
        String packageName = ps.name;
        // 删除和该包相关的数据，包括activity、provider、service等数据
        removePackageLI(ps, (flags&REMOVE_CHATTY) != 0);
        final PackageSetting deletedPs;
        synchronized (mPackages) {
            deletedPs = mSettings.mPackages.get(packageName);
            if (outInfo != null) {
                outInfo.removedPackage = packageName;
                outInfo.removedUsers = deletedPs != null
                        ? deletedPs.queryInstalledUsers(sUserManager.getUserIds(), true)
                        : null;
            }
        }
        if ((flags&PackageManager.DELETE_KEEP_DATA) == 0) {
            // 调用installd去删除/data/data下面的目录
            removeDataDirsLI(ps.volumeUuid, packageName);
            // 调用startCleaningPackages()，调用DefaultContainerService与该package相关的文件去删除
            schedulePackageCleaning(packageName, UserHandle.USER_ALL, true);
        }
        // writer
        synchronized (mPackages) {
            if (deletedPs != null) {
                if ((flags&PackageManager.DELETE_KEEP_DATA) == 0) {
                    clearIntentFilterVerificationsLPw(deletedPs.name, UserHandle.USER_ALL);
                    clearDefaultBrowserIfNeeded(packageName);
                    if (outInfo != null) {
                        mSettings.mKeySetManagerService.removeAppKeySetDataLPw(packageName);
                        // 从Settings中删除该包的信息
                        outInfo.removedAppId = mSettings.removePackageLPw(packageName);
                    }
                    // 检查mPermissionTrees和mPermissions两个数组中的权限是否是被删除的Package提供，如果有就删除。
                    updatePermissionsLPw(deletedPs.name, null, 0);
                    if (deletedPs.sharedUser != null) {
                        // Remove permissions associated with package. Since runtime
                        // permissions are per user we have to kill the removed package
                        // or packages running under the shared user of the removed
                        // package if revoking the permissions requested only by the removed
                        // package is successful and this causes a change in gids.
                        for (int userId : UserManagerService.getInstance().getUserIds()) {
                            final int userIdToKill = mSettings.updateSharedUserPermsLPw(deletedPs,
                                    userId);
                            if (userIdToKill == UserHandle.USER_ALL
                                    || userIdToKill >= UserHandle.USER_OWNER) {
                                // If gids changed for this user, kill all affected packages.
                                mHandler.post(new Runnable() {
                                    @Override
                                    public void run() {
                                        killApplication(deletedPs.name, deletedPs.appId,
                                                KILL_APP_REASON_GIDS_CHANGED);
                                    }
                                });
                                break;
                            }
                        }
                    }
                    // 从Settings.mPreferredActivities中删除相关的PreferredActivity
                    clearPackagePreferredActivitiesLPw(deletedPs.name, UserHandle.USER_ALL);
                }
                // 设置PackageUserState.installed状态，因为还有可能此次执行的是系统应用降级到初始版本
                if (allUserHandles != null && perUserInstalled != null) {
                    for (int i = 0; i < allUserHandles.length; i++) {
                        ps.setInstalled(perUserInstalled[i], allUserHandles[i]);
                    }
                }
            }
            if (writeSettings) {
                // 将改动的信息写到package.xml中
                mSettings.writeLPr();
            }
        }
        if (outInfo != null) {
            // A user ID was deleted here. Go through all users and remove it
            // from KeyStore.
            removeKeystoreDataIfNeeded(UserHandle.USER_ALL, outInfo.removedAppId);
        }
    }
```

### FileInstallArgs.cleanUpResourcesLI()
删除 code、resource、library 文件以及 /data/dalvik-cache 下面相关文件，/data/dalvik-cache 放的是经过 installd dexop t过的文件，可以看一下[这篇文章](http://blog.csdn.net/cnzx219/article/details/48714121)。
```java
        void cleanUpResourcesLI() {
            List<String> allCodePaths = Collections.EMPTY_LIST;
            if (codeFile != null && codeFile.exists()) {
                try {
                    final PackageLite pkg = PackageParser.parsePackageLite(codeFile, 0);
                    allCodePaths = pkg.getAllCodePaths();
                } catch (PackageParserException e) {}
            }
            // 删除code、resource、library文件
            cleanUp();
            //删除apk文件
            removeDexFiles(allCodePaths, instructionSets);
        }
```
```java
        private boolean cleanUp() {
            if (codeFile == null || !codeFile.exists()) {
                return false;
            }

            if (codeFile.isDirectory()) {
                mInstaller.rmPackageDir(codeFile.getAbsolutePath());
            } else {
                codeFile.delete();
            }

            if (resourceFile != null && !FileUtils.contains(codeFile, resourceFile)) {
                resourceFile.delete();
            }

            return true;
        }
```
```java
    private void removeDexFiles(List<String> allCodePaths, String[] instructionSets) {
        if (!allCodePaths.isEmpty()) {
            if (instructionSets == null) {
                throw new IllegalStateException("instructionSet == null");
            }
            String[] dexCodeInstructionSets = getDexCodeInstructionSets(instructionSets);
            for (String codePath : allCodePaths) {
                for (String dexCodeInstructionSet : dexCodeInstructionSets) {
                    // 调用 installd 删除 /data/dalvik-cache 下面相关文件
                    int retCode = mInstaller.rmdex(codePath, dexCodeInstructionSet);
                    if (retCode < 0) {...}
                }
            }
        }
    }
```

## adb uninstall

可以通过下面的命令来执行一个应用的卸载工作：
```
  adb uninstall [-k] <package> - remove this app package from the device
                                 ('-k' means keep the data and cache directories)
```


