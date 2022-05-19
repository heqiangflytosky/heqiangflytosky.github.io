---
title: Android：一个Multidex引发的VerifyError和Class Not Found问题 
categories: Android
comments: true
tags: [Android, Multidex]
description: 介绍一个在开发过程中遇到的Multidex引发的VerifyError和Class Not Found问题
date: 2016-2-16 10:00:00
---

一个困扰两天的问题终于解决了，下面记录一下该问题解决的历程，希望能对那些遇到类似问题的猿们有些帮助。
## 问题背景
由于项目要适配 android4.X，而应用需要引用的一个 jar 包的 4.X 版本就只能用 JDK1.6 来编译，而应用要用 JDK1.7 来编译，这个情况也为该问题的解决带来了干扰。
当把编译好的 jar 包放入应用中，且应用编译通过，运行时报各种问题：

```java
java.lang.VerifyError
java.lang.NoClassDefFoundError
```

等等……
网上说的各种解决办法都试过了也没用。后来看到一个帖子
http://www.cnblogs.com/successjerry/p/4402962.html
说 `java.lang.VerifyError` 的解决办法，说是 `proguard` 引起的。但是我编译的是 `debug` 版本，根本没有用 `proguard` 啊！但实在没办法了，我就想编个 `release` 版本看看我的 `proguard` 有没有问题。结果一试上面的两个报错都没有了，但是出现 了另外一个` class` 找不到的现象，是一个匿名内部类找不到。
其实这个时候应该可以想到的是可能是 `proguard` 去除了无用的方法，导致应用的方法数在 65535 之内，前面的出问题的几个类回到了主包，因此前面出现的两个问题就没有了。
但是但是只考虑是 `progurad` 混淆引起的，通过反编译 apk 也确实发现匿名内部类创建是变成了 `new 1(this)` 之类的，但是主要解决方向都在这里。
后来纠结了两天，当中也请教了好多人，还是没有解决。当时想着就先放一放，就把这个出错的地方注释掉了，应用可以跑起来了。
后来的一个线索的出现是，当时修改了一下 jar 包，往里面添加了几个方法，运行应用的时候又出现类找不到，方法找不到，这是就恍然大悟了，可能是方法超限的问题导致的。
后来看了一下应用的设置，在 build.gradle 里面确实配置了分包：

```
multiDexEnabled true
```

## 解决办法

后来在网上查资料，发现
http://blog.csdn.net/t12x3456/article/details/40837287
可能需要在 `Application` 类的 `attachBaseContext` 方法中加上：

```java
android.support.multidex.MultiDex.install(this);
```

一试，果然没有报错了。
在
http://weibo.com/p/1001603884849646685222
和
http://blog.csdn.net/qq2603825424/article/details/48311961
中有介绍：
使用了 `Multidex` 的 APK 运行在 Android5.0 之前的设备上时，还需要配合 support 库里面的 `MultiDex.install` 接口才行。有三种方法使用 `MultiDex.install` 接口：

 1. 如果没有自定义自己的 `Application`，那么在 AndroidManifest.xml 将 APK 的 `Application` 指定为 `MultiDexApplication`。
 2. 如果自定义了自己的 `Application`，那么将自己的 `Application` 继承于 `MultiDexApplication`。
 3. 如果不想继承于 `MultiDexApplication`，那么重写父类 `Applicatio` 的成员函数`attachBaseContext`，并且在该成员函数中调用 `MultiDex.install` 接口。

这里我们采用方法3。
最后，使用了 `Multidex` 的 APK 运行在 Android5.0 之后的设备上时，不需要 `MultiDex.install` 支持。这是因为 Android 5.0 使用的是 ART 虚拟机，ART 虚拟机解决了 Dalvik 虚拟机方法数限制在 65K 的问题。Android 5.0 在安装一个使用了 `MultiDex` 的APK时，会收集它的 Main Dex 和 Additional Dex，然后将它们翻译成Native Code，最终保存在一个OAT文件中。因此就不需要 `MultiDex.install` 了。
`MultiDex.install` 干了什么事情呢？它首先是收集 APK 里面的 Additional Dex，并且找到 APK 所使用的 `PathClassLoader`。接下来通过反射得到 `PathClassLoader` 的成员变量 `pathList`，这是一个类型为 `DexPathList` 的对象。`DexPathList` 里面又有一个成员变量`dexElements`，指向的是一个 `Element` 数组，该数组包含了系统主动为 APK 加载的 Dex。再接下来，又通过反射调用上述的 `DexPathList` 对象的成员函数` makeDexElements` 加载前面找到的 Additional Dex，并且将这些 Additional Dex 增加到它的成员变量`dexElements`描述的`Element`数组中。
Dalvik 虚拟机在查找一个`Class`的时候，会询问 APK 使用的`PathClassLoader`。`PathClassLoader`又询问它的成员变量 `pathList` 指向的`DexPathList`对象。`DexPathList`又询问保存在它的成员变量`dexElements`描述的一个`Element`数组中的 Dex。因此，就可以想象中，一旦 `MultiDex.install` 调用过后，APK 就可以正常使用打包在`AdditionalDex`中的 Class。
## 总结
其实在当发生 Debug 报错而 Release 版本有些报错就没有的时候，本来就可以往这方法考虑的，但是当时走错了方向，一直以为是混淆导致的问题，没有考虑到`minifyEnabled true`这个配置在 release中 也生效了。
希望对大家有所帮助。

