---
title: Android 深入理解 dumpsys
categories: Android源码分析
comments: true
tags: [Android源码分析, dumpsys]
description: 在Android开发过程中 dumpsys 命令是一个我们会经常使用的非常实用的命令，我们非常有必要来探究一下它是如何来带给我们一个如此便捷的查看Android各个服务信息的方式的。本文基于 Android6.0 来分析 dumpsys 源码以及其工作原理。
date: 2017-6-13 10:00:00
---

在我的博客 [Android实用技巧之adb命令：dumpsys命令的使用](http://www.heqiangfly.com/2014/10/15/android-development-skills-dumpsys/) 一文中详细介绍了 dumpsys 的基本用法，那么本文将介绍一下它是如何实现的以及它的工作原理。
dumpsys 相关源码位置：

```
frameworks/native/cmds/dumpsys/dumpsys.cpp
```

## 源码分析

看到 dumpsys 源码我们发现，它的实现较为简单，全部的代码都在 dumpsys.cpp 中，编译得到 dumpsys 二进制文件。实现基本的思路就是向 servicemanager 获取相关的系统服务，然后调用相应系统服务的 `dump()` 方法打印相关数据。

```cpp
int main(int argc, char* const argv[])
{
    signal(SIGPIPE, SIG_IGN);
    // 获取service_manager对象
    sp<IServiceManager> sm = defaultServiceManager();
    // 清除输出缓冲区的内容
    fflush(stdout);
    if (sm == NULL) {
	......
        return 20;
    }

    Vector<String16> services;
    Vector<String16> args;
    bool showListOnly = false;
    // 解析dumpsys命令的参数，参数 l 表示只输出服务列表
    if ((argc == 2) && (strcmp(argv[1], "-l") == 0)) {
        showListOnly = true;
    }

    // 获取需要输出的服务列表
    if ((argc == 1) || showListOnly) {
        // 不带参数表示要输出所有的服务
        services = sm->listServices();
        services.sort(sort_func);
        args.add(String16("-a"));
    } else {
        // 根据参数添加指定服务到services列表
        services.add(String16(argv[1]));
        for (int i=2; i<argc; i++) {
            args.add(String16(argv[i]));
        }
    }

    const size_t N = services.size();

    if (N > 1) {
        ......
        // 打印出当前的服务列表
        for (size_t i=0; i<N; i++) {
            // 根据服务名称获取服务对象
            sp<IBinder> service = sm->checkService(services[i]);
            if (service != NULL) {
                aout << "  " << services[i] << endl;
            }
        }
    }
    // 如果带有 l 参数，不再往下执行打印详细信息
    if (showListOnly) {
        return 0;
    }
    // 调用各个service的dump方法来完成服务信息的输出
    for (size_t i=0; i<N; i++) {
        // 根据服务名称获取服务对象
        sp<IBinder> service = sm->checkService(services[i]);
        if (service != NULL) {
            if (N > 1) {
                aout << "------------------------------------------------------------"
                        "-------------------" << endl;
                aout << "DUMP OF SERVICE " << services[i] << ":" << endl;
            }
            int err = service->dump(STDOUT_FILENO, args);
            ......
        }
        ......
    }

    return 0;
}
```

## 原理

我们都知道，Android的系统服务都是 `Binder` 的子类，他们都是由 servicemanager 来管理的，他们分别运行在不同的进程中。Android系统服务运行在 system_server 进程，servicemanager 运行在 servicemanager 进程中。他们之间的协作通过进程间的通信来完成。
`Binder.dump()`方法就是打印系统服务信息的方法，Android 系统服务重新实现了父类的 `dump()` 方法来实现系统服务信息的输出，输出的信息完全有系统服务自己来控制。

