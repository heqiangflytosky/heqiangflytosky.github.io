---
title: Android PackageManager相关源码分析之installd进程
categories: Android源码分析
comments: true
tags: [Android源码分析, PackageManager]
description: 基于Android6.0来分析Android的installd进程
date: 2016-5-18 10:00:00
---

## 概述

通过前面的介绍我们知道，PMS 里面有个成员变量`mInstaller`，PMS的很多工作比如dex优化，删除目录等工作都是由它来完成的。`Installer`类也比较简单，它也有个`mInstaller`的成员变量，它是`InstallerConnection`的一个对象，它通过 socket 和 Deamon 进程 installd 进行通信。`Installer`类所承担的工作实际是由 installd 进程来完成的。
installd进程在init.rc中定义如下：

```
service installd /system/bin/installd
    class main
    socket installd stream 600 system system
```

我们可能有这样的疑问，为什么要创建这样一个进程出来呢？它的工作由PMS来完成不是更好吗？这是因为PMS所在的进程 SystemServer 属于 system 用户组，没有 root 权限。像一些操作文件系统的工作比如移动和删除文件都需要 root 权限才是可以的。
相关代码位置：

```
frameworks/base/core/java/com/android/internal/os/InstallerConnection.java
frameworks/base/services/core/java/com/android/server/pm/Installer.java
frameworks/native/cmds/installd/installd.cpp
```

## 初始化

```cpp
int main(const int argc __unused, char *argv[]) {
    char buf[BUFFER_MAX];
    struct sockaddr addr;
    socklen_t alen;
    int lsocket, s;
    int selinux_enabled = (is_selinux_enabled() > 0);

    setenv("ANDROID_LOG_TAGS", "*:v", 1);
    android::base::InitLogging(argv);

    ALOGI("installd firing up\n");

    union selinux_callback cb;
    cb.func_log = log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);
    // 初始化一些全局变量
    if (initialize_globals() < 0) {
        ALOGE("Could not initialize globals; exiting.\n");
        exit(1);
    }
    // 初始化系统目录
    if (initialize_directories() < 0) {
        ALOGE("Could not create directories; exiting.\n");
        exit(1);
    }

    if (selinux_enabled && selinux_status_open(true) < 0) {
        ALOGE("Could not open selinux status; exiting.\n");
        exit(1);
    }
    // 从环境变量 ANDROID_SOCKET_installd 中获取用于监听的本地 socket
    lsocket = android_get_control_socket(SOCKET_PATH);
    if (lsocket < 0) {
        ALOGE("Failed to get socket from environment: %s\n", strerror(errno));
        exit(1);
    }
    if (listen(lsocket, 5)) {
        ALOGE("Listen on socket failed: %s\n", strerror(errno));
        exit(1);
    }
    fcntl(lsocket, F_SETFD, FD_CLOEXEC);

    for (;;) {
        alen = sizeof(addr);
        // 接收命令
        s = accept(lsocket, &addr, &alen);
        if (s < 0) {
            ALOGE("Accept failed: %s\n", strerror(errno));
            continue;
        }
        fcntl(s, F_SETFD, FD_CLOEXEC);
        // 处理请求的命令
        ALOGI("new connection\n");
        for (;;) {
            unsigned short count;
            // 读取命令的长度
            if (readx(s, &count, sizeof(count))) {
                ALOGE("failed to read size\n");
                break;
            }
            // 如果长度< 1或者是>= BUFFER_MAX，按出错处理
            if ((count < 1) || (count >= BUFFER_MAX)) {
                ALOGE("invalid size %d\n", count);
                break;
            }
            // 读取命令
            if (readx(s, buf, count)) {
                ALOGE("failed to read command\n");
                break;
            }
            buf[count] = 0;
            if (selinux_enabled && selinux_status_updated() > 0) {
                selinux_android_seapp_context_reload();
            }
            // 执行命令
            if (execute(s, buf)) break;
        }
        ALOGI("closing connection\n");
        close(s);
    }

    return 0;
}
```

## 支持的命令

```cpp
struct cmdinfo cmds[] = {
    { "ping",                 0, do_ping },
    { "install",              5, do_install },
    { "dexopt",               10, do_dexopt },
    { "markbootcomplete",     1, do_mark_boot_complete },
    { "movedex",              3, do_move_dex },
    { "rmdex",                2, do_rm_dex },
    { "remove",               3, do_remove },
    { "rename",               2, do_rename },
    { "fixuid",               4, do_fixuid },
    { "freecache",            2, do_free_cache },
    { "rmcache",              3, do_rm_cache },
    { "rmcodecache",          3, do_rm_code_cache },
    { "getsize",              8, do_get_size },
    { "rmuserdata",           3, do_rm_user_data },
    { "cpcompleteapp",        6, do_cp_complete_app },
    { "movefiles",            0, do_movefiles },
    { "linklib",              4, do_linklib },
    { "mkuserdata",           5, do_mk_user_data },
    { "mkuserconfig",         1, do_mk_user_config },
    { "rmuser",               2, do_rm_user },
    { "idmap",                3, do_idmap },
    { "restorecondata",       4, do_restorecon_data },
    { "createoatdir",         2, do_create_oat_dir },
    { "rmpackagedir",         1, do_rm_package_dir },
    { "linkfile",             3, do_link_file }
};
```

中间的数字表示需要的参数个数，后面表示各个命令对应的执行函数。

```cpp
struct cmdinfo {
    const char *name;
    unsigned numargs;
    int (*func)(char **arg, char reply[REPLY_MAX]);
};
```

下面简单介绍一下这些命令的作用：

 * ping：用于测试的空操作
 * install：安装应用
 * dexopt：dex优化操作，6.0之前的生成dex，6.0包括之后的生成dex和oat
 * markbootcomplete： 通知启动完成
 * movedex：移动apk文件
 * rmdex：删除/data/dalvik-cache下面相关文件
 * remove：卸载应用
 * rename：更改应用数据目录的名称
 * fixuid：更改应用数据目录的uid
 * freecache：清除/cache目录下的文件
 * rmcache：删除/cache目录下面某个应用的目录
 * rmcodecache：清除代码缓存文件
 * getsize：计算一个应用占用空间的大小，包括apk、数据目录、cache目录等
 * rmuserdata：删除一个user的所有安装的应用
 * cpcompleteapp：复制整个app
 * movefiles：执行/system/etc/updatecmds目录下的移动目录的脚本
 * linklib：为动态库建立符号链接
 * mkuserdata：为一个用户创建目录
 * mkuserconfig：为一个用户建立配置文件
 * rmuser：删除一个用户的所有文件
 * idmap：对两个apk进程执行idmap操作
 * restorecondata：恢复目录的SELinux安全上下文
 * createoatdir：创建OAT的目录
 * rmpackagedir：删除包的目录
 * linkfile：对文件建立符号链接


