---
title: Android UserManager相关源码分析
categories: Android源码分析
comments: true
tags: [Android源码分析, UserManager]
description: 基于Android6.0来分析Android的用户管理系统
date: 2017-3-6 10:00:00
---

## 概述
Android从4.2开始支持多用户模式，不同的用户运行在独立的用户空间，它们的的锁屏设置，PIN码和壁纸等系统设置是各不相同的，而且不同用户安装的应用和应用数据都是不一样的，但是系统中和硬件相关的设置则是共用的，比如网络设置等。通常第一个在系统中注册的用户将成为系统管理员，可以管理手机上的其他用户。但由于专利的原因，目前手机上并没有开启多用户模式，这种模式只能用在平板上面。
手机上经常见到的一个功能就是访客模式，它是多用户模式的一种应用。访客模式下用户的所有数据（通讯录，短信，应用等）会被隐藏，访客只能查看和使用手机的基本功能，另外你还可以设置访客是否有接听电话、发送短信等权限。
与其它系统服务的实现类似，用户管理也采用了经由`Binder`调用的远程服务机制。`UserManagerService`（以下简称 UMS ）继承自`IUserManager.Stub`为接口的底层实现，作为Binder的服务端。`UserManager`为暴露给用户的接口，它的成员变量`mService`是`IUserManager.Stub.Proxy`的实例，作为Binder的客户端，`UserManager`的大部分工作都是通过`mService`与服务端进行通信调用UMS的对应方法来完成的。
本文的代码是基于Android6.0来进行介绍。
`UserManager`相关代码位置：

```
frameworks/base/core/java/android/os/UserManager.java
frameworks/base/services/core/java/com/android/server/pm/UserManagerService.java
frameworks/base/core/java/android/content/pm/UserInfo.java
frameworks/base/core/java/android/os/UserHandle.java
frameworks/base/core/java/com/android/server/pm/PackageManagerService.java
frameworks/base/core/java/com/android/server/wm/WindowManagerService.java
frameworks/base/core/java/com/android/server/am/ActivityManagerService.java
frameworks/base/core/java/android/os/Process.java
```

 - PackageManagerService：在多用户环境中，所有安装的应用还是位于 /data/app/ 目录中，应用的数据还是保存在 /data/data 下面，这些数据只对 Id 为 0 的用户即管理员用户有效，但是这些应用的数据在 /data/user/<用户id>/ 目录下面都会有单独的一份。

## UserManager

`UserManager`可以称为 UMS 的代理对象，它通过`IUserManager mService`来与 UMS 进行进程间的通信。
`UserManager`是暴露出来的应用程序接口。对于普通应用程序，提供用户数查询，用户状态判断和用户序列号查询等基本功能。普通应用没有用户操作权限。
对于系统应用，`UserManager`提供了创建/删除/擦除用户、用户信息获取、用户句柄获取等用户操作的接口。
这些操作均由远程调用 UMS 服务的对应方法实现。

### 几个常用方法介绍

#### 判断是否支持多用户模式：

```java
    public static boolean supportsMultipleUsers() {
        return getMaxSupportedUsers() > 1
                && SystemProperties.getBoolean("fw.show_multiuserui",
                Resources.getSystem().getBoolean(R.bool.config_enableMultiUserUI));
    }

    public static int getMaxSupportedUsers() {
        // Don't allow multiple users on certain builds
        // 不允许用户在特定的版本上使用，因为多用户手机专利早已被Symbian注册，设计专利原因
        if (android.os.Build.ID.startsWith("JVP")) return 1;
        // Svelte devices don't get multi-user.
        // 是否是低内存设备
        if (ActivityManager.isLowRamDeviceStatic()) return 1;
        // 读取设备的配置文件
        return SystemProperties.getInt("fw.max_users",
                Resources.getSystem().getInteger(R.integer.config_multiuserMaximumUsers));
    }
```

首先会读取系统配置`fw.show_multiuserui`和`fw.max_users`，如果系统没有这个配置项则从配置文件中读取默认值，配置文件在`frameworks/base/core/res/res/values/config.xml`中：

```xml
    <!--  Maximum number of supported users -->
    <integer name="config_multiuserMaximumUsers">1</integer>
    <!-- Whether UI for multi user should be shown -->
    <bool name="config_enableMultiUserUI">false</bool>
```

也可以看到，手机上默认是不支持多用户模式的。

#### 查询是否是访客模式

查询当前进程的用户是否是访客。

```java
        UserManager userManager = (UserManager) context.getSystemService(Context.USER_SERVICE);
        return userManager.isGuestUser();
```

#### 查询用户权限

查询当前进程的用户是否拥有某个权限，

```java
        UserManager userManager = (UserManager) context.getSystemService(Context.USER_SERVICE);
        boolean canSendSms = userManager.hasUserRestriction(UserManager.DISALLOW_SMS);
```

还有个方法是查询指定用户的权限：

```java
public boolean hasUserRestriction(String restrictionKey, UserHandle userHandle)
```

### 如何创建Guest用户

比如要创建一个访客用户，就可以调用`UserManager.createGuest(Context context, String name)`方法来完成：

```java
    public UserInfo createGuest(Context context, String name) {
        UserInfo guest = createUser(name, UserInfo.FLAG_GUEST);
        if (guest != null) {
            Settings.Secure.putStringForUser(context.getContentResolver(),
                    Settings.Secure.SKIP_FIRST_USE_HINTS, "1", guest.id);
            // 对Guset用户的权限进行约束
            try {
                Bundle guestRestrictions = mService.getDefaultGuestRestrictions();
                // 禁止发送信息
                guestRestrictions.putBoolean(DISALLOW_SMS, true);
                // 禁止安装来源不明的应用
                guestRestrictions.putBoolean(DISALLOW_INSTALL_UNKNOWN_SOURCES, true);
                mService.setUserRestrictions(guestRestrictions, guest.id);
            } catch (RemoteException re) {
                Log.w(TAG, "Could not update guest restrictions");
            }
        }
        return guest;
    }
```

从代码我们可以看到`createGuest`最终也会调用到`UserManagerService.createUser`方法，然后通过`setUserRestrictions`方法进一步对Guest用户的权限进行限制。因此，Guset用户和普通用户的区别也就在于权限的不同。
可以通过下面的方式创建一个访客用户：

```java
    private void initGuestUser() {
        boolean needCreate = true;
        List<UserInfo> users = mUserManager.getUsers(true);
        for (UserInfo user : users) {
            if (user.isGuest()) {
                mGuestUserHandle = new UserHandle(user.id);
                mDefaultGuestRestrictions = mUserManager.getUserRestrictions(mGuestUserHandle);
                if (!mDefaultGuestRestrictions.getBoolean(UserManager.DISALLOW_CONFIG_CREDENTIALS)) {
                    mDefaultGuestRestrictions.putBoolean(UserManager.DISALLOW_CONFIG_CREDENTIALS, true);
                    mUserManager.setUserRestrictions(mDefaultGuestRestrictions, mGuestUserHandle);
                }
                needCreate = false;
                break;
            }
        }

        if (needCreate) {
            createGuest(false);
        }

    }

    private boolean createGuest(boolean fromRecreate) {
        UserInfo newGuest = mUserManager.createGuest(getActivity(), "Guest");
        if (newGuest != null) {
            Log.d(TAG, "new Guest:" + newGuest.id);
            mGuestUserHandle = new UserHandle(newGuest.id);
        } else {
            Log.d(TAG, "create guest failed!");
            return false;
        }

        if (fromRecreate) {
            mUserManager.setUserRestrictions(mDefaultGuestRestrictions, mGuestUserHandle);
            return true;
        }

        Bundle guestRestrictions = ……

        mUserManager.setUserRestrictions(guestRestrictions, mGuestUserHandle);
        mDefaultGuestRestrictions = guestRestrictions;

        return true;
    }
```

### 如何切换用户

一般通过 `ActivityManagerNative.getDefault().switchUser(int userId)`进行调用，这个在后面会有详细介绍。

```java
    private void enterGuestMode() {
        if (lossMode())
            return;
        int id = -1;
        List<UserInfo> users = mUserManager.getUsers(true);
        for (UserInfo user : users) {
            if (user.isGuest()) {
                id = user.id;
            }
        }
        try {
            ActivityManagerNative.getDefault().switchUser(id);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
```

### 用户权限

不同的用户`UserManager`通过一些限制赋予了不同的权限，下面列举一些限制权限：

 - DISALLOW_MODIFY_ACCOUNTS
 - DISALLOW_CONFIG_WIFI
 - DISALLOW_INSTALL_APPS
 - DISALLOW_UNINSTALL_APPS
 - DISALLOW_SHARE_LOCATION
 - DISALLOW_INSTALL_UNKNOWN_SOURCES
 - DISALLOW_CONFIG_BLUETOOTH
 - DISALLOW_USB_FILE_TRANSFER
 - DISALLOW_CONFIG_CREDENTIALS
 - DISALLOW_REMOVE_USER
 - DISALLOW_DEBUGGING_FEATURES
 - DISALLOW_CONFIG_VPN
 - DISALLOW_CONFIG_TETHERING
 - DISALLOW_NETWORK_RESET
 - DISALLOW_FACTORY_RESET
 - DISALLOW_ADD_USER
 - DISALLOW_CONFIG_CELL_BROADCASTS
 - DISALLOW_CONFIG_MOBILE_NETWORKS
 - DISALLOW_APPS_CONTROL
 - DISALLOW_MOUNT_PHYSICAL_MEDIA
 - DISALLOW_UNMUTE_MICROPHONE
 - DISALLOW_ADJUST_VOLUME
 - DISALLOW_OUTGOING_CALLS
 - DISALLOW_SMS
 - DISALLOW_FUN
 - DISALLOW_CREATE_WINDOWS
 - DISALLOW_CROSS_PROFILE_COPY_PASTE
 - DISALLOW_OUTGOING_BEAM
 - DISALLOW_WALLPAPER
 - DISALLOW_SAFE_BOOT
 - DISALLOW_RECORD_AUDIO

## 相关adb命令

### 获取用户信息

用`adb shell dumpsys user`可以查看当前的用户情况：

```
Users:
  UserInfo{0:机主:13} serialNo=0
    Created: <unknown>
    Last logged in: +1h26m30s184ms ago
  UserInfo{15:Guest:14} serialNo=15
    Created: +2d3h37m2s326ms ago
    Last logged in: +1h27m49s171ms ago
```

上面的打印是在`UserManagerService.dump()`中打印的。
UserInfo{0:机主:13} 所代表的含义可以参考代码`UserInfo.toString()`：

```java
    @Override
    public String toString() {
        return "UserInfo{" + id + ":" + name + ":" + Integer.toHexString(flags) + "}";
    }
```

 - 0:id
 - 机主:name
 - 13:flags

## UserManagerService

UMS 是用来管理用户的系统服务，是创建、删除以及查询用户的执行者。
下面先看一下 UMS 的初始化过程。

### 初始化

UMS 的创建是在`PackageManagerService`的构造函数中进行的。

```java
            sUserManager = new UserManagerService(context, this,
                    mInstallLock, mPackages);
```

接下来再看一下 UMS 的构造函数：
UMS 构造函数有两个参数：分别是`/data`目录和`/data/user/`目录。

```java
    private UserManagerService(Context context, PackageManagerService pm,
            Object installLock, Object packagesLock,
            File dataDir, File baseUserPath) {
        mContext = context;
        mPm = pm;
        mInstallLock = installLock;
        mPackagesLock = packagesLock;
        mHandler = new MainHandler();
        synchronized (mInstallLock) {
            synchronized (mPackagesLock) {
                // mUsersDir是/data/system/users目录
                mUsersDir = new File(dataDir, USER_INFO_DIR);
                mUsersDir.mkdirs();
                // 创建第一个用户的目录/data/system/users/0
                File userZeroDir = new File(mUsersDir, "0");
                userZeroDir.mkdirs();
                // 设置目录的权限
                FileUtils.setPermissions(mUsersDir.toString(),
                        FileUtils.S_IRWXU|FileUtils.S_IRWXG
                        |FileUtils.S_IROTH|FileUtils.S_IXOTH,
                        -1, -1);
                // 创建代表/data/system/users/userlist.xml文件的对象
                mUserListFile = new File(mUsersDir, USER_LIST_FILENAME);
                // 添加一些对访客用户的默认限制，DISALLOW_OUTGOING_CALLS和DISALLOW_SMS，不允许打电话和发信息
                initDefaultGuestRestrictions();
                // 读取userlist.xml文件，将用户的信息保存在mUsers列表中
                // 如果该文件不存在则创建该文件
                readUserListLocked();
                sInstance = this;
            }
        }
    }
```

然后再看一下`systemReady()`函数，这里面也有一些初始化的工作，它的调用是在`PackageManagerService.systemReady()`中进行的。

```java
    void systemReady() {
        synchronized (mInstallLock) {
            synchronized (mPackagesLock) {
                // 寻找那些在userlist.xml文件中partially或者partial为true的用户
                ArrayList<UserInfo> partials = new ArrayList<UserInfo>();
                for (int i = 0; i < mUsers.size(); i++) {
                    UserInfo ui = mUsers.valueAt(i);
                    if ((ui.partial || ui.guestToRemove) && i != 0) {
                        partials.add(ui);
                    }
                }
                // 将这些未创建完成的用户从系统中移除掉
                for (int i = 0; i < partials.size(); i++) {
                    UserInfo ui = partials.get(i);
                    removeUserStateLocked(ui.id);
                }
            }
        }
        // 更新一下OWNER用户的登陆时间
        onUserForeground(UserHandle.USER_OWNER);
        // 调用AppOpsManager（权限管理器）设置每个用户的权限
        mAppOpsService = IAppOpsService.Stub.asInterface(
                ServiceManager.getService(Context.APP_OPS_SERVICE));
        for (int i = 0; i < mUserIds.length; ++i) {
            try {
                mAppOpsService.setUserRestrictions(mUserRestrictions.get(mUserIds[i]), mUserIds[i]);
            } catch (RemoteException e) {
                Log.w(LOG_TAG, "Unable to notify AppOpsService of UserRestrictions");
            }
        }
    }
```

初始化过程中，会创建`/data/system/users/`目录，以及在`/data/system/users/`下面创建`0/`目录，然后调用`readUserListLocked()`分析`/data/system/users/userlist.xml`文件，这个文件保存了系统中所有用户的Id信息。
`/data/system/users`目录下面的文件有：

```
0
0.xml
10
10.xml
userlist.xml
```

0/目录下面的文件：

```
accounts.db
accounts.db-journal
appwidgets.xml
package-restrictions.xml
registered_services
runtime-permissions.xml
settings_global.xml
settings_secure.xml
settings_system.xml
wallpaper_info.xml
```

里面存放了针对该用户的各种设置数据。

userlist.xml示例：

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<users nextSerialNumber="11" version="5">
    <guestRestrictions>
        <restrictions no_outgoing_calls="true" no_sms="true" />
    </guestRestrictions>
    <user id="0" />
    <user id="10" />
</users>
```

`nextSerialNumber`指的是创建下一个用户时它的`serialNumber`，`version`指的是当前多用户的版本，使 userlist.xml 文件像数据库一样是可以升级的。
`guestRestrictions`标签指的是为访客用户设置的权限，可以通过`UserManager.setDefaultGuestRestrictions()`来设置。
这里面有两个用户的信息，分别为 id 为 0 和 10 的用户。
得到 Id 信息后还要读取保存了用户注册信息的 xml 文件，这个文件也位于`/data/system/users`目录下，文件名用用户 Id 数字表示。
0.xml 示例：

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<user id="0" serialNumber="0" flags="19" created="0" lastLoggedIn="1493886997408">
    <name>机主</name>
    <restrictions />
</user>
```

10.xml：

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<user id="10" serialNumber="10" flags="4" created="1493886837832" lastLoggedIn="0">
    <name>Guest</name>
    <restrictions no_install_apps="false" no_install_unknown_sources="true" no_config_credentials="true" no_outgoing_calls="false" no_sms="false" />
</user>
```

标签的属性值对应了`UserInfo`里面的成员变量，读取用户的xml文件后，会根据文件的内容来创建和初始化一个`UserInfo`来保存，并把该对象加入到`mUsers`列表中去。
`restrictions`表示针对该用户做的权限限制，可以通过`UserManager.setUserRestriction()`或`UserManager.setUserRestrictions()`进行设置。
如果有未创建完成的用户，即`partial=true`的用户，则把它们从用户列表中移除出去。
因此，UMS 的初始化工作主要就是分析`userlist.xml`文件、创建用户列表`mUsers`以及设置用户权限。

## UserInfo

```java
public class UserInfo implements Parcelable {

    // 用户类型
    public static final int FLAG_MASK_USER_TYPE = 0x000000FF;

    // 机主用户的标志，通常是第一个ID为0的用户
    public static final int FLAG_PRIMARY = 0x00000001;

    // 管理员标志，有此标志的才能创建和删除其他用户，通常是机主用户
    public static final int FLAG_ADMIN   = 0x00000002;

    // 访客用户标志
    public static final int FLAG_GUEST   = 0x00000004;

    // 标记权限受限的用户
    public static final int FLAG_RESTRICTED = 0x00000008;

    // 标记改用户是否进行了初始化
    public static final int FLAG_INITIALIZED = 0x00000010;

    /**
     * Indicates that this user is a profile of another user, for example holding a users
     * corporate data.
     */
    // 标志该UserInfo是否是另外一个用户的一份profile，Android中允许一个用户拥有另一份profile
    public static final int FLAG_MANAGED_PROFILE = 0x00000020;

    // 标记该用户已经被禁止
    public static final int FLAG_DISABLED = 0x00000040;

    // 无效值定义
    public static final int NO_PROFILE_GROUP_ID = -1;

    public int id;               //用户id
    public int serialNumber;     //用户序列号，唯一的，不会重复
    public String name;          //用户名称
    public String iconPath;      //用户头像的路径
    public int flags;            //用户的标记信息
    public long creationTime;    //用户创建时间
    public long lastLoggedInTime;//用户上次登陆时间
    public int profileGroupId;   //用户的profile group id

    // 用来标记没有创建完成的用户
    public boolean partial;
    public boolean guestToRemove;
}
```

需要注意的是，用户的Id用来表示用户，如果用户被删除了它的Id会分配给下一个新建的用户，用来保持Id的连续性；
但是serialNumber是一个不会重复的数字，是不会被重复利用的，用来惟一标识一个用户。

## UserHandle

`UserHandle`就代表了设备中的一个用户，提供了一系列针对用户操作的API。
现在先来分析几个用户相关的概念：

 - userId：用户id
 - appId：是指跟用户无关的应用程序id，取值范围 `0 <= appId < 100000`
 - uid：是指跟用户紧密相关的应用程序id

他们之间有一下的换算关系：

```java
uid = userId * 100000  + appId
```

在`UserHandle`中有常量`PER_USER_RANGE`，它的值为`100000`，也就是说每个 user 最大可以有 100000 个 appid 。

下面介绍几个`UserHandle`的API：

 - isSameUser(int uid1, int uid2)：比较两个uid的userId是否相同，即它们是否属于同一个用户
 - isSameApp(int uid1, int uid2)：比较两个uid的appId是否相同
 - getUserId(int uid)：根据uid获取userId
 - getAppId(int uid)：根据uid获取appId

userId的几种类型：

| userId | 值 | 含义 |
| --------- |:-------:| ------------- |
| USER_OWNER | 0 | 机主用户 |
| USER_ALL | -1 | 所有用户 |
| USER_CURRENT | -2 | 当前用户 |
| USER_CURRENT_OR_SELF | -3 | 当前用户或者调用者所在用户 |
| USER_NULL | -1000 | 未定义用户 |

```
    public static final UserHandle OWNER = new UserHandle(USER_OWNER);
    public static final UserHandle CURRENT = new UserHandle(USER_CURRENT);
```

`CURRENT`就代表了当前用户的`UserHandle`。

### UID

终端输入命令`adb shell ps`，可以得到设备的进程信息，下面仅列举部分具有代表性的进程信息：

```
USER      PID   PPID  VSIZE  RSS   WCHAN              PC  NAME
root      1     0     12844  1768  SyS_epoll_ 0000000000 S /init
root      2     0     0      0       kthreadd 0000000000 S kthreadd
root      3     2     0      0     smpboot_th 0000000000 S ksoftirqd/0
root      5     2     0      0     worker_thr 0000000000 S kworker/0:0H
root      7     2     0      0     rcu_gp_kth 0000000000 S rcu_preempt
root      8     2     0      0     rcu_gp_kth 0000000000 S rcu_sched
root      9     2     0      0     rcu_gp_kth 0000000000 S rcu_bh
root      10    2     0      0     smpboot_th 0000000000 S migration/0

root      202   1     14456  1324  SyS_epoll_ 0000000000 S /system/bin/lmkd
system    203   1     14148  1492  binder_thr 0000000000 S /system/bin/servicemanager
system    204   1     159072 10000 SyS_epoll_ 0000000000 S /system/bin/surfaceflinger
system    207   1     13696  1248  poll_sched 0000000000 S /system/bin/6620_launcher
root      209   1     28380  2532  hrtimer_na 0000000000 S /system/bin/netd
root      210   1     12460  1764  poll_sched 0000000000 S /system/bin/debuggerd
root      211   1     15752  1724  poll_sched 0000000000 S /system/bin/debuggerd64
drm       212   1     53596  12036 binder_thr 0000000000 S /system/bin/drmserver
media     213   1     299956 25568 binder_thr 0000000000 S /system/bin/mediaserver
root      214   1     14004  1304  unix_strea 0000000000 S /system/bin/installd
keystore  215   1     18784  2752  binder_thr 0000000000 S /system/bin/keystore
shell     216   1     13164  1336  hrtimer_na 0000000000 S /system/bin/mobile_log_d
shell     217   1     16148  1636  futex_wait 0000000000 S /system/bin/netdiag
root      222   2     0      0     rescuer_th 0000000000 S pvr_workqueue
root      223   2     0      0     rescuer_th 0000000000 S pvr_workqueue
root      224   2     0      0     rescuer_th 0000000000 S pvr_workqueue
gps       225   1     13564  1476  SyS_epoll_ 0000000000 S /system/xbin/mnld

system    559   229   2306532 147320 SyS_epoll_ 0000000000 S system_server
media_rw  686   185   16980  2064  inotify_re 0000000000 S /system/bin/sdcard
u0_a24    716   229   2245132 180428 SyS_epoll_ 0000000000 S com.android.systemui

wifi      1136  1     19712  4228  poll_sched 0000000000 S /system/bin/wpa_supplicant
root      1223  2     0      0     worker_thr 0000000000 S kworker/2:2

shell     9548  9482  13796  1260  __skb_recv 7fb24a54f4 S logcat
u0_a6     9635  229   1605504 71224 SyS_epoll_ 0000000000 S android.process.acore
root      10836 2     0      0     worker_thr 0000000000 S kworker/0:0
root      11438 2     0      0     worker_thr 0000000000 S kworker/0:2
root      11530 2     0      0     worker_thr 0000000000 S kworker/0:1
```

可以看到 UID 有root，system，shell等都属于系统 uid，它们定义在在`Process.java`文件，如下：

| UID | 值 | 含义 |
| --------- |:-------:| ------------- |
| ROOT_UID | 0 | root UID |
| SYSTEM_UID | 1000 |  |
| PHONE_UID | 1001 |  |
| SHELL_UID | 2000 |  |
| LOG_UID | 1007 |  |
| WIFI_UID | 1010 |  |
| MEDIA_UID | 1013 |  |
| DRM_UID | 1019 |  |
| VPN_UID | 1016 |  |
| NFC_UID | 1017 |  |
| BLUETOOTH_UID | 1002 |  |
| MEDIA_RW_GID | 1023 |  |
| PACKAGE_INFO_GID | 1032 |  |
| SHARED_RELRO_UID | 1037 |  |
| SHARED_USER_GID | 9997 |  |

在`Process.java`中还有下面常量：
 - FIRST_APPLICATION_UID = 10000
 - LAST_APPLICATION_UID = 19999
 - FIRST_SHARED_APPLICATION_GID = 50000
 - LAST_SHARED_APPLICATION_GID = 59999
 - FIRST_ISOLATED_UID = 99000
 - LAST_ISOLATED_UID = 99999

还有像`u0_a24`这种uid代表的uid为10024，代表user 0 的一个应用程序uid，这个转换方法在`UserHandle.formatUid()`：

```java
    public static void formatUid(StringBuilder sb, int uid) {
        if (uid < Process.FIRST_APPLICATION_UID) {
            sb.append(uid);
        } else {
            sb.append('u');
            sb.append(getUserId(uid));
            final int appId = getAppId(uid);
            if (appId >= Process.FIRST_ISOLATED_UID && appId <= Process.LAST_ISOLATED_UID) {
                sb.append('i');
                sb.append(appId - Process.FIRST_ISOLATED_UID);
            } else if (appId >= Process.FIRST_APPLICATION_UID) {
                sb.append('a');
                sb.append(appId - Process.FIRST_APPLICATION_UID);
            } else {
                sb.append('s');
                sb.append(appId);
            }
        }
    }
```

如果我们切换到访客模式，还会发现一个UID为`u10_a24`的`com.android.systemui`进程，因为该应用的appid为10024，不同的是它们是在不同的用户下面创建的进程。
但是在访客模式下面，root，system，shell等这些系统进程是和主用户是共用的，不会再创建新的进程。

## UserState

先了解一下用户的几个运行状态：

```java
public final class UserState {
    // 用户正在启动
    public final static int STATE_BOOTING = 0;
    // 用户正在运行
    public final static int STATE_RUNNING = 1;
    // 用户正在关闭
    public final static int STATE_STOPPING = 2;
    // 用户已经被关闭
    public final static int STATE_SHUTDOWN = 3;
}
```

UMS 有下列变量来存储用户的相关信息：

 - `mCurrentUserId`表示当前正在允许的用户的Id；
 - `mTargetUserId`记录当前正在切换到该用户；
 - `mStartedUsers`保存了当前已经运行过的用户的列表，这个列表中的用户会有上面的四种状态
 - `mUserLru`用最近最少使用算法保存的用户列表，最近的前台用户保存在列表的最后位置
 - `mStartedUserArray`表示mStartedUsers中当前正在运行的用户列表的index，即mStartedUsers中除去正在关闭和已经被关闭状态外的用户

## 创建用户

### UserManagerService

UMS 中创建用户是在`createUser()`方法中实现的：

```java
    @Override
    public UserInfo createUser(String name, int flags) {
        checkManageUsersPermission("Only the system can create users");
        return createUserInternal(name, flags, UserHandle.USER_NULL);
    }
```

首先检查调用者的权限，只有 UID 是 system 或者具有`android.Manifest.permission.MANAGE_USERS`的应用才有权限，否则抛出异常。然后就调用`createUserInternal`来执行真正的创建工作。

```java
    private UserInfo createUserInternal(String name, int flags, int parentId) {
        // 检查一下该用户是否被限制创建用户
        if (getUserRestrictions(UserHandle.getCallingUserId()).getBoolean(
                UserManager.DISALLOW_ADD_USER, false)) {
            return null;
        }
        // 检查是否是低内存设备
        if (ActivityManager.isLowRamDeviceStatic()) {
            return null;
        }
        final boolean isGuest = (flags & UserInfo.FLAG_GUEST) != 0;
        final boolean isManagedProfile = (flags & UserInfo.FLAG_MANAGED_PROFILE) != 0;
        final long ident = Binder.clearCallingIdentity();
        UserInfo userInfo = null;
        final int userId;
        try {
            synchronized (mInstallLock) {
                synchronized (mPackagesLock) {
                    UserInfo parent = null;
                    if (parentId != UserHandle.USER_NULL) {
                        parent = getUserInfoLocked(parentId);
                        if (parent == null) return null;
                    }
                    if (isManagedProfile && !canAddMoreManagedProfiles()) {
                        return null;
                    }
                    // 如果添加的不是Guest用户，也不是用户的profile，而且已经到达用户的上限，则不允许再添加
                    if (!isGuest && !isManagedProfile && isUserLimitReachedLocked()) {
                        return null;
                    }
                    // 如果添加的是Guest用户，但是系统中已经存在一个，则不允许再添加
                    if (isGuest && findCurrentGuestUserLocked() != null) {
                        return null;
                    }
                    // 得到新用户的Id
                    userId = getNextAvailableIdLocked();
                    // 为该新用户创建UserInfo
                    userInfo = new UserInfo(userId, name, null, flags);
                    // 设置序列号
                    userInfo.serialNumber = mNextSerialNumber++;
                    // 设置创建时间
                    long now = System.currentTimeMillis();
                    userInfo.creationTime = (now > EPOCH_PLUS_30_YEARS) ? now : 0;
                    // 设置partial，表示正在创建
                    userInfo.partial = true;
                    Environment.getUserSystemDirectory(userInfo.id).mkdirs();
                    // 放入mUsers列表
                    mUsers.put(userId, userInfo);
                    // 把用户信息写入userlist.xml
                    writeUserListLocked();
                    if (parent != null) {
                        if (parent.profileGroupId == UserInfo.NO_PROFILE_GROUP_ID) {
                            parent.profileGroupId = parent.id;
                            scheduleWriteUserLocked(parent);
                        }
                        userInfo.profileGroupId = parent.profileGroupId;
                    }
                    // 创建用户目录
                    final StorageManager storage = mContext.getSystemService(StorageManager.class);
                    for (VolumeInfo vol : storage.getWritablePrivateVolumes()) {
                        final String volumeUuid = vol.getFsUuid();
                        try {
                            final File userDir = Environment.getDataUserDirectory(volumeUuid,
                                    userId);
                            // 创建userDir，如果volumeUuid为空，创建/data/user/<userId>/，不为空，创建/mnt/expand/<volumeUuid>/user/<userId>/
                            prepareUserDirectory(userDir);
                            enforceSerialNumber(userDir, userInfo.serialNumber);
                        } catch (IOException e) {
                            Log.wtf(LOG_TAG, "Failed to create user directory on " + volumeUuid, e);
                        }
                    }
                    // 调用PackageManagerService的createNewUserLILPw方法，此方法很重要，后面会单独分析
                    // 这个函数会在新建用户的userDir目录下面为所有应用创建数据
                    // 此方法将新用户可以使用的App在/data/user/<用户Id>文件夹下创建数据目录，目录名称为包名
                    mPm.createNewUserLILPw(userId);
                    // 创建成功，将partial改为false
                    userInfo.partial = false;
                    // 会调用writeUserLocked()创建xxx.xml，类似0.xml，xxx是用户Id
                    scheduleWriteUserLocked(userInfo);
                    // 更新mUserIds数组
                    updateUserIdsLocked();
                    Bundle restrictions = new Bundle();
                    // 添加为该用户设置的限制条件
                    mUserRestrictions.append(userId, restrictions);
                }
            }
            // 发送用户创建完成的广播，广播附带用户的id
            if (userInfo != null) {
                Intent addedIntent = new Intent(Intent.ACTION_USER_ADDED);
                addedIntent.putExtra(Intent.EXTRA_USER_HANDLE, userInfo.id);
                mContext.sendBroadcastAsUser(addedIntent, UserHandle.ALL,
                        android.Manifest.permission.MANAGE_USERS);
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
        return userInfo;
    }
```

由上面的代码可知，`createUser`主要做了一下工作：
1. 检查调用进程所属用户是否被限制添加用户、当前设备是否是低内存设备、当前用户数是否已达上限，如是则停止创建新用户
2. 为新用户设置用户信息，比如Id，序列号创建时间等
3. 将用户信息写入userlist.xml，注意，此时的userInfo.partial = true，表示正在创建
4. 创建用户目录/data/user/<userId>/或者/mnt/expand/<volumeUuid>/user/<userId>/
5. 调用PackageManagerService的createNewUserLILPw方法，这个函数会在新建用户的目录下面为所有应用创建数据目录
6. 创建用户完成，将userInfo.partial设置为false，创建用户的信息文件，比如0.xml
7. 发送用户创建完成的广播

### PackageManagerService

#### 创建用户数据createNewUserLILPw()

```java
    void createNewUserLILPw(int userHandle) {
        if (mInstaller != null) {
            //通过mInstaller调用守护进程installd执行mkuserconfig，创建用户配置文件。
            mInstaller.createUserConfig(userHandle);
            // 调用mSettings.createNewUserLILP为新用户中的应用创建应用数据目录
            mSettings.createNewUserLILPw(this, mInstaller, userHandle);
            // 设置默认的Browser
            applyFactoryDefaultBrowserLPw(userHandle);
            primeDomainVerificationsLPw(userHandle);
        }
    }
```

下面来看一下`Settings.createNewUserLILP()`方法的代码：

```java
    void createNewUserLILPw(PackageManagerService service, Installer installer, int userHandle) {
        // 为每一个应用创建数据目录
        for (PackageSetting ps : mPackages.values()) {
            if (ps.pkg == null || ps.pkg.applicationInfo == null) {
                continue;
            }
            // Only system apps are initially installed.
            //为系统应用修改PackageUserState的installed标志
            ps.setInstalled((ps.pkgFlags&ApplicationInfo.FLAG_SYSTEM) != 0, userHandle);
            // 通过mInstaller调用守护进程installd执行mkuserdata，为用户创建用户目录。
            installer.createUserData(ps.volumeUuid, ps.name,
                    UserHandle.getUid(userHandle, ps.appId), userHandle,
                    ps.pkg.applicationInfo.seinfo);
        }
        // TODO
        applyDefaultPreferredAppsLPw(service, userHandle);
        writePackageRestrictionsLPr(userHandle);
        writePackageListLPr(userHandle);
    }
```

对每一个用户，系统都会以 `PackageuserState` 类来保护其安装的每一个包状态。它以`SparseArray`的形式由 `PackageSetting` 类来保护，`PackageSetting` 存储了每一个安装包的数据。也就是说对于每个安装包，里面都有个每个用户对应的列表 `userState` 来存储该安装包对于不同用户的不同状态，比如针对该用户是否隐藏以及是否标识为已安装状态。具体详细介绍参考我的介绍 `PackageManagerService` 的博客。

## 切换用户

### ActivityManagerService

#### startUser()

下面介绍一下切换用户的流程，比如我们从机主用户切换到访客用户是就是走的这个流程。
用户切换是通过调用`ActivityManager`的`public boolean switchUser(int userId)`方法进行。一般通过 `ActivityManagerNative.getDefault().switchUser(int userId)`进行调用。
最终会调用`ActivityManagerService.switchUser`方法：

```java
    @Override
    public boolean switchUser(final int userId) {
        enforceShellRestriction(UserManager.DISALLOW_DEBUGGING_FEATURES, userId);
        String userName;
        synchronized (this) {
            UserInfo userInfo = getUserManagerLocked().getUserInfo(userId);
            if (userInfo == null) {
                Slog.w(TAG, "No user info for user #" + userId);
                return false;
            }
            // 该user只是一个user的profile，无法切换
            if (userInfo.isManagedProfile()) {
                Slog.w(TAG, "Cannot switch to User #" + userId + ": not a full user");
                return false;
            }
            userName = userInfo.name;
            // 把该用户Id记录到mTargetUserId变量
            mTargetUserId = userId;
        }
        // 发送切换用户的消息
        mUiHandler.removeMessages(START_USER_SWITCH_MSG);
        mUiHandler.sendMessage(mUiHandler.obtainMessage(START_USER_SWITCH_MSG, userId, 0, userName));
        return true;
    }
```

`Handler`收到`START_USER_SWITCH_MSG`消息后，会调用`showUserSwitchDialog()`来弹出一个确认的对话框。

```java
    private void showUserSwitchDialog(int userId, String userName) {
        // The dialog will show and then initiate the user switch by calling startUserInForeground
        Dialog d = new UserSwitchingDialog(this, mContext, userId, userName,
                true /* above system */);
        d.show();
    }

```

点击确定后最终会调用到`startUser()`来执行切换用户的动作。

```java
    private boolean startUser(final int userId, final boolean foreground) {
        //切换用话需要INTERACT_ACROSS_USERS_FULL权限
        if (checkCallingPermission(INTERACT_ACROSS_USERS_FULL)
                != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException(msg);
        }

        final long ident = Binder.clearCallingIdentity();
        try {
            synchronized (this) {
                // 如果当前用户已经是需要切换的用户，退出当前流程
                final int oldUserId = mCurrentUserId;
                if (oldUserId == userId) {
                    return true;
                }

                mStackSupervisor.setLockTaskModeLocked(null, ActivityManager.LOCK_TASK_MODE_NONE,
                        "startUser", false);

                // 如果没有需要启动的用户的信息，则直接退出
                final UserInfo userInfo = getUserManagerLocked().getUserInfo(userId);
                if (userInfo == null) {
                    return false;
                }
                // 如果是前台启动且是一份profile的话，则直接退出
                if (foreground && userInfo.isManagedProfile()) {
                    return false;
                }

                // 如果是前台启动，则需要将屏幕冻结
                if (foreground) {
                    mWindowManager.startFreezingScreen(R.anim.screen_user_exit,
                            R.anim.screen_user_enter);
                }

                boolean needStart = false;

                // 如果当前已经启动过的用户列表中没有该用户，则需要先启动该用户
                if (mStartedUsers.get(userId) == null) {
                    mStartedUsers.put(userId, new UserState(new UserHandle(userId), false));
                    updateStartedUserArrayLocked();
                    needStart = true;
                }

                // 调整该用户在mUserLru列表中的位置，当前用户放到最后位置
                final Integer userIdInt = Integer.valueOf(userId);
                mUserLru.remove(userIdInt);
                mUserLru.add(userIdInt);

                if (foreground) {
                    // 如果是前台切换，直接切换到需要启动的用户
                    mCurrentUserId = userId;
                    // 重置mTargetUserId，因为userId已经赋值给mCurrentUserId了
                    mTargetUserId = UserHandle.USER_NULL; // reset, mCurrentUserId has caught up
                    // 更新与该用户相关的profile列表
                    updateCurrentProfileIdsLocked();
                    // 在WindowManagerService中设置要启动的用户为当前用户
                    mWindowManager.setCurrentUser(userId, mCurrentProfileIds);
                    // Once the internal notion of the active user has switched, we lock the device
                    // with the option to show the user switcher on the keyguard.
                    // 执行锁屏操作
                    mWindowManager.lockNow(null);
                    }
                } else {
                    // 如果是后台启动
                    final Integer currentUserIdInt = Integer.valueOf(mCurrentUserId);
                    // 更新与该用户相关的profile列表
                    updateCurrentProfileIdsLocked();
                    mWindowManager.setCurrentProfileIds(mCurrentProfileIds);
                    // 重新调整mUserLru
                    mUserLru.remove(currentUserIdInt);
                    mUserLru.add(currentUserIdInt);
                }

                final UserState uss = mStartedUsers.get(userId);

                if (uss.mState == UserState.STATE_STOPPING) {
                    // 如果该用户是正在停止，这个时候还没有发送ACTION_SHUTDOWN广播，则切换为正在运行
                    uss.mState = UserState.STATE_RUNNING;
                    // 更新mStartedUserArray列表
                    updateStartedUserArrayLocked();
                    needStart = true;
                } else if (uss.mState == UserState.STATE_SHUTDOWN) {
                    // 如果该用户是正在停止，这个时候已经发送ACTION_SHUTDOWN广播，则切换为正在启动状态
                    uss.mState = UserState.STATE_BOOTING;
                    updateStartedUserArrayLocked();
                    needStart = true;
                }

                if (uss.mState == UserState.STATE_BOOTING) {
                    // 如果用户的状态是正在启动，则发送一个SYSTEM_USER_START_MSG消息，该消息的处理下面会介绍
                    mHandler.sendMessage(mHandler.obtainMessage(SYSTEM_USER_START_MSG, userId, 0));
                }

                if (foreground) {
                    // 发送SYSTEM_USER_CURRENT_MSG消息
                    mHandler.sendMessage(mHandler.obtainMessage(SYSTEM_USER_CURRENT_MSG, userId,
                            oldUserId));
                    // 分别发送REPORT_USER_SWITCH_MSG和USER_SWITCH_TIMEOUT_MSG，防止切换时间过长，后面会介绍
                    mHandler.removeMessages(REPORT_USER_SWITCH_MSG);
                    mHandler.removeMessages(USER_SWITCH_TIMEOUT_MSG);
                    mHandler.sendMessage(mHandler.obtainMessage(REPORT_USER_SWITCH_MSG,
                            oldUserId, userId, uss));
                    mHandler.sendMessageDelayed(mHandler.obtainMessage(USER_SWITCH_TIMEOUT_MSG,
                            oldUserId, userId, uss), USER_SWITCH_TIMEOUT);
                }

                if (needStart) {
                    // 发送一个 USER_STARTED 广播
                    Intent intent = new Intent(Intent.ACTION_USER_STARTED);
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                            | Intent.FLAG_RECEIVER_FOREGROUND);
                    intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                    broadcastIntentLocked(null, null, intent,
                            null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                            null, false, false, MY_PID, Process.SYSTEM_UID, userId);
                }

                if ((userInfo.flags&UserInfo.FLAG_INITIALIZED) == 0) {
                    // 当该用户还没有初始化，如果是普通用户则会发送ACTION_USER_INITIALIZE广播，如果是机主用户，则直接标记为已初始化
                    if (userId != UserHandle.USER_OWNER) {
                        Intent intent = new Intent(Intent.ACTION_USER_INITIALIZE);
                        intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND);
                        broadcastIntentLocked(null, null, intent, null,
                                new IIntentReceiver.Stub() {
                                    public void performReceive(Intent intent, int resultCode,
                                            String data, Bundle extras, boolean ordered,
                                            boolean sticky, int sendingUser) {
                                        onUserInitialized(uss, foreground, oldUserId, userId);
                                    }
                                }, 0, null, null, null, AppOpsManager.OP_NONE,
                                null, true, false, MY_PID, Process.SYSTEM_UID, userId);
                        // 标记为正在初始化，该变量会在completeSwitchAndInitialize()重置
                        uss.initializing = true;
                    } else {
                        getUserManagerLocked().makeInitialized(userInfo.id);
                    }
                }

                if (foreground) {
                    // 如果用户已经初始化过了，则设置为前台用户，后面会介绍这一部分
                    if (!uss.initializing) {
                        moveUserToForeground(uss, oldUserId, userId);
                    }
                } else {
                    // 如果是后台启动用户，先加入到mStartingBackgroundUsers列表中去
                    mStackSupervisor.startBackgroundUserLocked(userId, uss);
                }

                if (needStart) {
                    Intent intent = new Intent(Intent.ACTION_USER_STARTING);
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                    intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                    broadcastIntentLocked(null, null, intent,
                            null, new IIntentReceiver.Stub() {
                                @Override
                                public void performReceive(Intent intent, int resultCode,
                                        String data, Bundle extras, boolean ordered, boolean sticky,
                                        int sendingUser) throws RemoteException {
                                }
                            }, 0, null, null,
                            new String[] {INTERACT_ACROSS_USERS}, AppOpsManager.OP_NONE,
                            null, true, false, MY_PID, Process.SYSTEM_UID, UserHandle.USER_ALL);
                }
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }

        return true;
    }
```

下面来分析一下`moveUserToForeground()`函数

```java
    void moveUserToForeground(UserState uss, int oldUserId, int newUserId) {
        //从mStackSupervisor获取newUserId用户在切换之前的stack状态，以便将原来在前台的应用推到前台
        boolean homeInFront = mStackSupervisor.switchUserLocked(newUserId, uss);
        if (homeInFront) {
            // 如果原来是从桌面切换的用户，则启动桌面
            startHomeActivityLocked(newUserId, "moveUserToFroreground");
        } else {
            // 如果是其他应用，则将此应用推到前台
            mStackSupervisor.resumeTopActivitiesLocked();
        }
        EventLogTags.writeAmSwitchUser(newUserId);
        getUserManagerLocked().onUserForeground(newUserId);
        // 发送ACTION_USER_BACKGROUND广播，通知和oldUserId相关的用户(包括profile)进入后台的消息
        // 发送ACTION_USER_FOREGROUND广播，通知和newUserId相关的用户(包括profile)进入前台的消息
        // 发送ACTION_USER_SWITCHED广播，通知用户切换
        sendUserSwitchBroadcastsLocked(oldUserId, newUserId);
    }
```

#### 消息处理

下面介绍一下`startUser()`中发出的几条消息的处理。
SYSTEM_USER_CURRENT_MSG：

```java
            case SYSTEM_USER_CURRENT_MSG: {
                mBatteryStatsService.noteEvent(
                        BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_FINISH,
                        Integer.toString(msg.arg2), msg.arg2);
                mBatteryStatsService.noteEvent(
                        BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_START,
                        Integer.toString(msg.arg1), msg.arg1);
                mSystemServiceManager.switchUser(msg.arg1);
                break;
            }
```

这里主要是通知`BatteryStatsService`用户切换的消息以及让`SystemServiceManager`调用各个`SystemService`的`onSwitchUser(userHandle)`方法。

REPORT_USER_SWITCH_MSG、USER_SWITCH_TIMEOUT_MSG、CONTINUE_USER_SWITCH_MSG和REPORT_USER_SWITCH_COMPLETE_MSG

```java
            case REPORT_USER_SWITCH_MSG: {
                dispatchUserSwitch((UserState) msg.obj, msg.arg1, msg.arg2);
                break;
            }
            case USER_SWITCH_TIMEOUT_MSG: {
                timeoutUserSwitch((UserState) msg.obj, msg.arg1, msg.arg2);
                break;
            }
            case CONTINUE_USER_SWITCH_MSG: {
                continueUserSwitch((UserState) msg.obj, msg.arg1, msg.arg2);
                break;
            }
            case REPORT_USER_SWITCH_COMPLETE_MSG: {
                dispatchUserSwitchComplete(msg.arg1);
            } break;
```

`REPORT_USER_SWITCH_MSG`消息的处理方法是`dispatchUserSwitch()`：

```java
    void dispatchUserSwitch(final UserState uss, final int oldUserId,
            final int newUserId) {
        // 获取需要通知的观察者数量
        final int N = mUserSwitchObservers.beginBroadcast();
        if (N > 0) {
            // 创建一个回调函数供观察者对象调用
            final IRemoteCallback callback = new IRemoteCallback.Stub() {
                int mCount = 0;
                @Override
                public void sendResult(Bundle data) throws RemoteException {
                    synchronized (ActivityManagerService.this) {
                        if (mCurUserSwitchCallback == this) {
                            // 收到一条返回结果mCount加一
                            mCount++;
                            if (mCount == N) { //待所有结果都返回了，发送继续处理的消息，后面介绍该方法
                                sendContinueUserSwitchLocked(uss, oldUserId, newUserId);
                            }
                        }
                    }
                }
            };
            synchronized (this) {
                // 标记正在进行用户切换
                uss.switching = true;
                mCurUserSwitchCallback = callback;
            }
            // 调用观察者对象的onUserSwitching方法
            for (int i=0; i<N; i++) {
                try {
                    mUserSwitchObservers.getBroadcastItem(i).onUserSwitching(
                            newUserId, callback);
                } catch (RemoteException e) {
                }
            }
        } else {
            synchronized (this) {
                sendContinueUserSwitchLocked(uss, oldUserId, newUserId);
            }
        }
        // 通知完成后清除状态
        mUserSwitchObservers.finishBroadcast();
    }

    void sendContinueUserSwitchLocked(UserState uss, int oldUserId, int newUserId) {
        mCurUserSwitchCallback = null;
        mHandler.removeMessages(USER_SWITCH_TIMEOUT_MSG);
        mHandler.sendMessage(mHandler.obtainMessage(CONTINUE_USER_SWITCH_MSG,
                oldUserId, newUserId, uss));
    }
```

如果系统中有对切换用户感兴趣的模块，可以调用AMS的`registerUserSwitchObserver()`方法来注册观察对象，这个对象会保存在`mUserSwitchObservers`中，当有用户切换时AMS会通过调用`onUserSwitching`来通知这些模块，模块处理完成后会调用参数中传递的`callback`来通知AMS。最后都调用完成后会调用`sendContinueUserSwitchLocked`来继续进行切换用户的工作。
`sendContinueUserSwitchLocked()`方法会先清除前面延迟发送的`USER_SWITCH_TIMEOUT_MSG`消息，然后再发送一条`CONTINUE_USER_SWITCH_MSG`消息。
`CONTINUE_USER_SWITCH_MSG`消息的执行函数是`completeSwitchAndInitialize()`方法，

```java
    void timeoutUserSwitch(UserState uss, int oldUserId, int newUserId) {
        synchronized (this) {
            sendContinueUserSwitchLocked(uss, oldUserId, newUserId);
        }
    }

    void continueUserSwitch(UserState uss, int oldUserId, int newUserId) {
        completeSwitchAndInitialize(uss, newUserId, false, true);
    }

    void completeSwitchAndInitialize(UserState uss, int newUserId,
            boolean clearInitializing, boolean clearSwitching) {
        boolean unfrozen = false;
        synchronized (this) {
            if (clearInitializing) {
                // 标记初始化完成
                uss.initializing = false;
                getUserManagerLocked().makeInitialized(uss.mHandle.getIdentifier());
            }
            if (clearSwitching) {
                // 标记用户切换完成
                uss.switching = false;
            }
            if (!uss.switching && !uss.initializing) {
                // 结束冻屏
                mWindowManager.stopFreezingScreen();
                unfrozen = true;
            }
        }
        if (unfrozen) {
            // 发送切换完成的消息
            mHandler.removeMessages(REPORT_USER_SWITCH_COMPLETE_MSG);
            mHandler.sendMessage(mHandler.obtainMessage(REPORT_USER_SWITCH_COMPLETE_MSG,
                    newUserId, 0));
        }
        // 停止在后台的访客用户
        stopGuestUserIfBackground();
    }
```

`REPORT_USER_SWITCH_COMPLETE_MSG`消息的处理函数是`dispatchUserSwitchComplete`，此方法的主要工作就是调用观察者的`onUserSwitchComplete()`方法，进行用户切换的收尾工作。

```java
    void dispatchUserSwitchComplete(int userId) {
        final int observerCount = mUserSwitchObservers.beginBroadcast();
        for (int i = 0; i < observerCount; i++) {
            try {
                mUserSwitchObservers.getBroadcastItem(i).onUserSwitchComplete(userId);
            } catch (RemoteException e) {
            }
        }
        mUserSwitchObservers.finishBroadcast();
    }
```

再介绍一下`USER_SWITCH_TIMEOUT_MSG`消息，发送`REPORT_USER_SWITCH_MSG`消息的同时发送`USER_SWITCH_TIMEOUT_MSG`消息是为了防止用户切换时间过长，毕竟只有所有的观察者都处理完了才能继续进行切换用户的操作。发送完`USER_SWITCH_TIMEOUT_MSG`消息以后，如果后面没有进行清除该消息，那么时间一到就表示处理超时，就会调用`timeoutUserSwitch()`方法进行超时处理，`timeoutUserSwitch()`执行`sendContinueUserSwitchLocked()`来结束切换工作，不再等待各个观察者处理任务的结束。

```java
    void timeoutUserSwitch(UserState uss, int oldUserId, int newUserId) {
        synchronized (this) {
            sendContinueUserSwitchLocked(uss, oldUserId, newUserId);
        }
    }
```

上面`startUser()`方法介绍完了，但是现在用户的状态还是在`UserState.STATE_BOOTING`的状态，那么什么时候切换到`UserState.STATE_RUNNING`状态呢？

#### 更新用户状态，停止旧用户

在`Activity`进入Idle状态后会调用AMS的`activityIdle()`方法，此方法会调用`mStackSupervisor.activityIdleInternalLocked(token, false, config);`，在`activityIdleInternalLocked()`方法内有下面的处理：

```java
        if (!booting) {
            // Complete user switch
            if (startingUsers != null) {
                for (int i = 0; i < startingUsers.size(); i++) {
                    mService.finishUserSwitch(startingUsers.get(i));
                }
            }
            // Complete starting up of background users
            if (mStartingBackgroundUsers.size() > 0) {
                startingUsers = new ArrayList<UserState>(mStartingBackgroundUsers);
                mStartingBackgroundUsers.clear();
                for (int i = 0; i < startingUsers.size(); i++) {
                    mService.finishUserBoot(startingUsers.get(i));
                }
            }
        }
```

当前有正在切换的用户的话就会调用AMS的`finishUserSwitch()`和`finishUserBoot()`方法，来更新用户状态，发送广播以及处理需要停止的用户工作。
下面来看一下 AMS 的`finishUserSwitch()`和`finishUserBoot()`方法：

```java
    void finishUserSwitch(UserState uss) {
        synchronized (this) {
            finishUserBoot(uss);

            startProfilesLocked();

            int num = mUserLru.size();
            int i = 0;
            while (num > MAX_RUNNING_USERS && i < mUserLru.size()) {
                Integer oldUserId = mUserLru.get(i);
                UserState oldUss = mStartedUsers.get(oldUserId);
                if (oldUss == null) {
                    mUserLru.remove(i);
                    num--;
                    continue;
                }
                if (oldUss.mState == UserState.STATE_STOPPING
                        || oldUss.mState == UserState.STATE_SHUTDOWN) {
                    // 已经停止的用户不做处理
                    num--;
                    i++;
                    continue;
                }
                if (oldUserId == UserHandle.USER_OWNER || oldUserId == mCurrentUserId) {
                    // 机主用户和当前用户会继续运行，不做处理
                    i++;
                    continue;
                }
                // 将老用户停止，后面会介绍
                stopUserLocked(oldUserId, null);
                num--;
                i++;
            }
        }
    }

    void finishUserBoot(UserState uss) {
        synchronized (this) {
            if (uss.mState == UserState.STATE_BOOTING
                    && mStartedUsers.get(uss.mHandle.getIdentifier()) == uss) {
                uss.mState = UserState.STATE_RUNNING;
                final int userId = uss.mHandle.getIdentifier();
                Intent intent = new Intent(Intent.ACTION_BOOT_COMPLETED, null);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                intent.addFlags(Intent.FLAG_RECEIVER_NO_ABORT);
                broadcastIntentLocked(null, null, intent,
                        null, null, 0, null, null,
                        new String[] {android.Manifest.permission.RECEIVE_BOOT_COMPLETED},
                        AppOpsManager.OP_NONE, null, true, false, MY_PID, Process.SYSTEM_UID,
                        userId);
            }
        }
    }
```
通过`finishUserSwitch`->`finishUserBoot`来将用户的状态置为`UserState.STATE_RUNNING`，并发出广播`Intent.ACTION_BOOT_COMPLETED`广播。
至此，用户切换工作全部结束。

## 删除用户

### UserManagerService

`UserManagerService`中删除用户是在`removeUser()`方法中实现的：

```java
    public boolean removeUser(int userHandle) {
        // 检查该进程是否具有删除用户的权限
        checkManageUsersPermission("Only the system can remove users");
        // 检查一下该用户是否被限制删除用户
        if (getUserRestrictions(UserHandle.getCallingUserId()).getBoolean(
                UserManager.DISALLOW_REMOVE_USER, false)) {
            return false;
        }

        long ident = Binder.clearCallingIdentity();
        try {
            final UserInfo user;
            synchronized (mPackagesLock) {
                user = mUsers.get(userHandle);
                // 检查被删除的用户是不是管理员用户userHandle=0，检查用户列表中是否有该用户，以及该用户是否是正在被删除的用户
                if (userHandle == 0 || user == null || mRemovingUserIds.get(userHandle)) {
                    return false;
                }

                // 将该用户放入mRemovingUserIds列表中，防止重复删除
                // mRemovingUserIds中的数据会一直保存直到系统重启，防止Id被重复使用
                mRemovingUserIds.put(userHandle, true);

                try {
                    mAppOpsService.removeUser(userHandle);
                } catch (RemoteException e) {
                    Log.w(LOG_TAG, "Unable to notify AppOpsService of removing user", e);
                }
                // 将partial设置为true，这样如果后面的过程意外终止导致此次删除失败，系统重启后还是会继续删除工作的
                user.partial = true;
                // 设置FLAG_DISABLED，禁止该用户
                user.flags |= UserInfo.FLAG_DISABLED;
                // 将上面更新的用户文件信息写入到xml文件中去
                writeUserLocked(user);
            }

            // 如果该user是一个user的一份profile，则发出一个ACTION_MANAGED_PROFILE_REMOVED广播
            if (user.profileGroupId != UserInfo.NO_PROFILE_GROUP_ID
                    && user.isManagedProfile()) {
                sendProfileRemovedBroadcast(user.profileGroupId, user.id);
            }

            // 调用AMS停止当前的用户，这部分后面会详细介绍
            int res;
            try {
                res = ActivityManagerNative.getDefault().stopUser(userHandle,
                        // 设置回调函数，调用finishRemoveUser继续后面的删除工作
                        new IStopUserCallback.Stub() {
                            @Override
                            public void userStopped(int userId) {
                                finishRemoveUser(userId);
                            }
                            @Override
                            public void userStopAborted(int userId) {
                            }
                        });
            } catch (RemoteException e) {
                return false;
            }
            return res == ActivityManager.USER_OP_SUCCESS;
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```

`ActivityManagerNative.getDefault().stopUser`执行完后 UMS 会继续执行删除工作。

```java
    void finishRemoveUser(final int userHandle) {
        // Let other services shutdown any activity and clean up their state before completely
        // wiping the user's system directory and removing from the user list
        long ident = Binder.clearCallingIdentity();
        try {
            Intent addedIntent = new Intent(Intent.ACTION_USER_REMOVED);
            addedIntent.putExtra(Intent.EXTRA_USER_HANDLE, userHandle);
            mContext.sendOrderedBroadcastAsUser(addedIntent, UserHandle.ALL,
                    android.Manifest.permission.MANAGE_USERS,

                    new BroadcastReceiver() {
                        @Override
                        public void onReceive(Context context, Intent intent) {
                            new Thread() {
                                public void run() {
                                    synchronized (mInstallLock) {
                                        synchronized (mPackagesLock) {
                                            removeUserStateLocked(userHandle);
                                        }
                                    }
                                }
                            }.start();
                        }
                    },
                    null, Activity.RESULT_OK, null, null);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```

根据代码可以看到`finishRemoveUser`方法只是发送了一个有序广播ACTION_USER_REMOVED，同时注册了一个广播接收器，这个广播接收器是最后一个接收到该广播的接收器，这样做的目的是让关心该广播的其他接收器处理完之后， UMS 才会进行删除用户的收尾工作，即调用`removeUserStateLocked`来删除用户的相关文件。

```java
    private void removeUserStateLocked(final int userHandle) {
        // 调用mPm.cleanUpUserLILPw来删除用户目录/data/user/<用户id>/下面的应用数据，后面会详细介绍
        mPm.cleanUpUserLILPw(this, userHandle);

        // 从mUsers列表中移除该用户信息
        mUsers.remove(userHandle);
        AtomicFile userFile = new AtomicFile(new File(mUsersDir, userHandle + XML_SUFFIX));
        // 删除用户信息文件 <用户id>.xml
        userFile.delete();
        // 更新userlist.xml文件，将删除调用的用户从中移除
        writeUserListLocked();
        // 更新mUserIds列表
        updateUserIdsLocked();
        // 删除用户目录以及该用户的所有文件
        removeDirectoryRecursive(Environment.getUserSystemDirectory(userHandle));
    }
```

至此，删除用户的工作已经全部完成。

### ActivityManagerService

UMS的`removeUser()`会调用AMS的`stopUser()`来处理停止用户的一些工作，在AMS内部也会调用`stopUser()`。该方法在进行了权限检查之后，主要的工作都是由`stopUserLocked()`来完成的。

```java
    private int stopUserLocked(final int userId, final IStopUserCallback callback) {
        // 当前用户是不能被停止的，如果是当前用户则直接返回
        if (mCurrentUserId == userId && mTargetUserId == UserHandle.USER_NULL) {
            return ActivityManager.USER_OP_IS_CURRENT;
        }

        final UserState uss = mStartedUsers.get(userId);
        if (uss == null) {
            // 如果当前用户还没有被启动，那么直接结束当前流程，调用回调函数，返回操作成功标志
            if (callback != null) {
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            callback.userStopped(userId);
                        } catch (RemoteException e) {
                        }
                    }
                });
            }
            return ActivityManager.USER_OP_SUCCESS;
        }

        if (callback != null) {
            uss.mStopCallbacks.add(callback);
        }

        if (uss.mState != UserState.STATE_STOPPING
                && uss.mState != UserState.STATE_SHUTDOWN) {
            // 将状态切换到正在停止
            uss.mState = UserState.STATE_STOPPING;
            // 更新mStartedUserArray列表
            updateStartedUserArrayLocked();

            long ident = Binder.clearCallingIdentity();
            try {
                final Intent stoppingIntent = new Intent(Intent.ACTION_USER_STOPPING);
                stoppingIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                stoppingIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                stoppingIntent.putExtra(Intent.EXTRA_SHUTDOWN_USERSPACE_ONLY, true);
                final Intent shutdownIntent = new Intent(Intent.ACTION_SHUTDOWN);
                // 注册一个Intent.ACTION_SHUTDOWN广播接收器
                final IIntentReceiver shutdownReceiver = new IIntentReceiver.Stub() {
                    @Override
                    public void performReceive(Intent intent, int resultCode, String data,
                            Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                        //收到Intent.ACTION_SHUTDOWN广播，下面介绍此方法
                        finishUserStop(uss);
                    }
                };
                // 注册一个Intent.ACTION_USER_STOPPING广播接收器
                final IIntentReceiver stoppingReceiver = new IIntentReceiver.Stub() {
                    @Override
                    public void performReceive(Intent intent, int resultCode, String data,
                            Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                        synchronized (ActivityManagerService.this) {
                            if (uss.mState != UserState.STATE_STOPPING) {
                                return;
                            }
                            // 收到Intent.ACTION_USER_STOPPING广播后更新用户状态
                            uss.mState = UserState.STATE_SHUTDOWN;
                        }
                        mBatteryStatsService.noteEvent(
                                BatteryStats.HistoryItem.EVENT_USER_RUNNING_FINISH,
                                Integer.toString(userId), userId);
                        // 回调各个SystemServer的onStopUser方法
                        mSystemServiceManager.stopUser(userId);
                        //发送Intent.ACTION_SHUTDOWN广播，此为有序广播，确保shutdownReceiver最后一个调用
                        broadcastIntentLocked(null, null, shutdownIntent,
                                null, shutdownReceiver, 0, null, null, null, AppOpsManager.OP_NONE,
                                null, true, false, MY_PID, Process.SYSTEM_UID, userId);
                    }
                };
                // 发送Intent.ACTION_USER_STOPPING广播，此为有序广播，确保stoppingReceiver最后一个调用
                broadcastIntentLocked(null, null, stoppingIntent,
                        null, stoppingReceiver, 0, null, null,
                        new String[] {INTERACT_ACROSS_USERS}, AppOpsManager.OP_NONE,
                        null, true, false, MY_PID, Process.SYSTEM_UID, UserHandle.USER_ALL);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }

        return ActivityManager.USER_OP_SUCCESS;
    }
```

`stopUserLocked()`方法首先检查当前的用户是否需要执行停止的工作，如果不需要直接调用参数的回调函数结束停止工作，如果需要，则先后发送`Intent.ACTION_USER_STOPPING`和`Intent.ACTION_SHUTDOWN`广播，方法中也注册了两个广播的接收器，在`Intent.ACTION_SHUTDOWN`广播接收器中执行`finishUserStop(uss)`方法。

```java
    void finishUserStop(UserState uss) {
        final int userId = uss.mHandle.getIdentifier();
        boolean stopped;
        ArrayList<IStopUserCallback> callbacks;
        synchronized (this) {
            callbacks = new ArrayList<IStopUserCallback>(uss.mStopCallbacks);
            if (mStartedUsers.get(userId) != uss) {
                stopped = false;
            } else if (uss.mState != UserState.STATE_SHUTDOWN) {
                stopped = false;
            } else {
                stopped = true;
                // User can no longer run.
                mStartedUsers.remove(userId);
                mUserLru.remove(Integer.valueOf(userId));
                updateStartedUserArrayLocked();

                // 杀掉和该用户相关的所有进程
                forceStopUserLocked(userId, "finish user");
            }

            // 清除用户相关的Recent Task列表
            mRecentTasks.removeTasksForUserLocked(userId);
        }

        //调用在stopUserLocked中添加的回调函数
        for (int i=0; i<callbacks.size(); i++) {
            try {
                if (stopped) callbacks.get(i).userStopped(userId);
                else callbacks.get(i).userStopAborted(userId);
            } catch (RemoteException e) {
            }
        }

        if (stopped) {
            // 回调各个SystemServer的onCleanupUser方法
            mSystemServiceManager.cleanupUser(userId);
            synchronized (this) {
                mStackSupervisor.removeUserLocked(userId);
            }
        }
    }
```

`finishUserStop()`方法从`mStartedUsers`和`mUserLru`列表中删除该用户，更新`mStartedUserArray`列表，清理和该用户有关的进程，发送`Intent.ACTION_USER_STOPPED`广播来通知该用户已经停止，接下来清除用户相关的Recent Task列表以及从`mStackSupervisor`中删除用户的信息。

```java
    private void forceStopUserLocked(int userId, String reason) {
        // 杀掉该用户相关的所有进程，具体流程会在ActivityManager相关文章中介绍
        forceStopPackageLocked(null, -1, false, false, true, false, false, userId, reason);
        // 发出Intent.ACTION_USER_STOPPED广播
        Intent intent = new Intent(Intent.ACTION_USER_STOPPED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                | Intent.FLAG_RECEIVER_FOREGROUND);
        intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
        broadcastIntentLocked(null, null, intent,
                null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                null, false, false, MY_PID, Process.SYSTEM_UID, UserHandle.USER_ALL);
    }

```

至此，AMS相关的删除用户的相关工作全部完成。

### PackageManagerService

删除用户数据cleanUpUserLILPw()

```java
    void cleanUpUserLILPw(UserManagerService userManager, int userHandle) {
        mDirtyUsers.remove(userHandle);
        mSettings.removeUserLPw(userHandle);
        mPendingBroadcasts.remove(userHandle);
        if (mInstaller != null) {
            // Technically, we shouldn't be doing this with the package lock
            // held.  However, this is very rare, and there is already so much
            // other disk I/O going on, that we'll let it slide for now.
            final StorageManager storage = mContext.getSystemService(StorageManager.class);
            for (VolumeInfo vol : storage.getWritablePrivateVolumes()) {
                final String volumeUuid = vol.getFsUuid();
                mInstaller.removeUserDataDirs(volumeUuid, userHandle);
            }
        }
        mUserNeedsBadging.delete(userHandle);
        removeUnusedPackagesLILPw(userManager, userHandle);
    }

```

删除用户的工作比较简单，删除用户的数据。同时调用`mSettings.removeUserLPw(userHandle)`来删除和 PMS 中和用户相关的信息。

```java
    void removeUserLPw(int userId) {
        // 删除每个应用中的该用户的信息
        Set<Entry<String, PackageSetting>> entries = mPackages.entrySet();
        for (Entry<String, PackageSetting> entry : entries) {
            entry.getValue().removeUser(userId);
        }
        mPreferredActivities.remove(userId);
        File file = getUserPackagesStateFile(userId);
        file.delete();
        file = getUserPackagesStateBackupFile(userId);
        file.delete();
        removeCrossProfileIntentFiltersLPw(userId);

        mRuntimePermissionsPersistence.onUserRemoved(userId);
        // 更新/data/system/packages.list文件
        writePackageListLPr();
    }

```

