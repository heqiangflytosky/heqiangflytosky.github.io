---
title: Android实用技巧之adb命令：getprop，setprop，watchprops命令的使用 
categories: Android实用技巧
comments: true
tags: [Android, getprop, setprop, watchprops]
description: 介绍dumpsys的使用
date: 2014-10-18 10:00:00
---

## getprop

### getprop简介

`getprop`命令的作用就是从系统的各种配置文件中读取一些设备的信息。这些文件在我们的手机设备中是可以找到的：
```
init.rc
default.prop
/system/build.prop
```

### 查询所有的配置

输入命令：
```
adb shell getprop
```
就会列出所有的配置信息：
```
[aricent_ims_op_status]: [1]
[aricent_ims_type]: [1]
[config.disable_bluetooth]: [false]
[dalvik.vm.dex2oat-Xms]: [64m]
[dalvik.vm.dex2oat-Xmx]: [512m]
[dalvik.vm.dex2oat-filter]: [speed]
[dalvik.vm.dexopt-flags]: [m=y]
[dalvik.vm.heapgrowthlimit]: [256m]
[dalvik.vm.heapsize]: [512m]
[dalvik.vm.image-dex2oat-Xms]: [64m]
[dalvik.vm.image-dex2oat-Xmx]: [64m]
[dalvik.vm.image-dex2oat-filter]: [speed]
[dalvik.vm.isa.arm.features]: [div]
[dalvik.vm.isa.arm64.features]: [default]
[dalvik.vm.stack-trace-file]: [/data/anr/traces.txt]
[debug.force_rtl]: [0]
[debug.hwc.force_gpu]: [0]
[debug.hwc.winupdate]: [1]
[debug.hwui.render_dirty_regions]: [true]
[dev.bootcomplete]: [1]
[dhcp.wlan0.dns1]: [172.17.16.99]
[dhcp.wlan0.dns2]: [172.17.16.98]
[dhcp.wlan0.dns3]: []
[dhcp.wlan0.dns4]: []
[dhcp.wlan0.domain]: [meizu.com]
[dhcp.wlan0.gateway]: [172.17.100.1]
[dhcp.wlan0.ipaddress]: [172.17.100.29]
[dhcp.wlan0.leasetime]: [28800]
[dhcp.wlan0.mask]: [255.255.252.0]
[dhcp.wlan0.mtu]: []
[dhcp.wlan0.pid]: [7115]
[dhcp.wlan0.reason]: [BOUND]
[dhcp.wlan0.result]: [ok]
[dhcp.wlan0.server]: [1.1.1.2]
[dhcp.wlan0.vendorInfo]: []
[drm.service.enabled]: [true]
[exynos.modempath]: [/system/vendor/firmware/modem.bin]
[exynos.telephony.feature]: [true]
[gsm.current.phone-type]: [1,1]
[gsm.defaultpdpcontext.active]: [true]
[gsm.network.type]: [Unknown,LTE]

……

[sys.usb.bicr]: [no]
[sys.usb.charging]: [no]
[sys.usb.config]: [mtp,adb]
[sys.usb.state]: [mtp,adb]
[sys.usb.vid]: [2A45]
[vold.post_fs_data_done]: [1]
[wifi.interface]: [wlan0]
[wlan.driver.status]: [ok]
```
这些配置中以`ro`开头的是只读属性。

### 查看单个配置信息

可以在`adb shell getprop`后面加属性名称来输出单个配置信息：
命令格式：`getprop [key]`
比如：
```
$ adb shell getprop dalvik.vm.heapgrowthlimit
256m
```
表示进程默认虚拟机最大堆内存。
如果你对某个属性名称不是那么确定的话就用下面的命令来过滤：
```
$ adb shell getprop | grep dalvik
[dalvik.vm.dex2oat-Xms]: [64m]
[dalvik.vm.dex2oat-Xmx]: [512m]
[dalvik.vm.dex2oat-filter]: [speed]
[dalvik.vm.dexopt-flags]: [m=y]
[dalvik.vm.heapgrowthlimit]: [256m]
[dalvik.vm.heapsize]: [512m]
[dalvik.vm.image-dex2oat-Xms]: [64m]
[dalvik.vm.image-dex2oat-Xmx]: [64m]
[dalvik.vm.image-dex2oat-filter]: [speed]
[dalvik.vm.isa.arm.features]: [div]
[dalvik.vm.isa.arm64.features]: [default]
[dalvik.vm.stack-trace-file]: [/data/anr/traces.txt]
[ro.dalvik.vm.native.bridge]: [0]
```

### 查询其他信息

`usage: getprop [-TZ] [NAME [DEFAULT]]`

```
-T	Show property types instead of values
-Z	Show property contexts instead of values

-T 参数可以查询属性值的类型
-Z 参数可以查看属性值需要的一些权限之类的
```

## setprop

`setprop`可以对手机一些配置进行设置，当然这些配置必须是可写的。
命令格式：`setprop [key] [value]`
如果你想修改进程默认分配的可使用堆内存大小：
```
adb shell setprop dalvik.vm.heapgrowthlimit 512m
```

## watchprops

`watchprops`命令用来监听系统属性的变化，如果期间系统的属性发生变化则把变化的值显示出来。
```
$ adb shell watchprops
1491476973 dalvik.vm.heapgrowthlimit = '512m'
1491476323 init.svc.debuggerd = 'running'
1491476323 init.svc.debuggerd64 = 'running'
1491476323 init.svc.debuggerd = 'restarting'
1491476323 init.svc.debuggerd64 = 'restarting'
1491476980 gsm.operator.alpha = ''
1491476980 gsm.operator.numeric = ''
1491476980 gsm.operator.iso-country = ''
1491476980 gsm.operator.isroaming = 'false,false'
```

## 一些参数说明

 - dalvik.vm.heapgrowthlimit：默认给进程分配的可使用堆内存
 - dalvik.vm.heapsize：设置了`android:largeHeap`以后可使用的内存大小
 - ro.product.brand：手机品牌
 - ro.product.device：设备名称
 - ro.product.model：设备内部代号
 - ro.product.name：设备名称
 - ro.product.manufacturer：设备制造商
 - ro.serialno：设备序列号
 - ro.sf.lcd_density：设备屏幕密度
 - ro.config.ringtone：默认来电铃声
 - ro.config.notification_sound：默认通知铃声
 - ro.config.alarm_alert：默认闹钟铃声
 - dalvik.vm.stack-trace-file：trace文件放置目录
 - ro.product.cpu.abilist：系统支持的ABI列表

     
