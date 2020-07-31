---
title: Android -- 多CPU架构支持所需要了解的知识
categories: Android
comments: true
tags: [CPU架构]
description: 多CPU架构支持所需要了解的知识
date: 2017-2-24 10:00:00
---

## 前言

Android系统目前支持以下七种不同的CPU架构：ARMv5，ARMv7 (从2010年起)，x86 (从2011年起)，MIPS (从2012年起)，ARMv8，MIPS64和x86_64 (从2014年起)，每一种都关联着一个相应的ABI。ABI是指应用基于哪种指令集来进行编译。
如果项目中使用到了NDK，它将会生成.so文件，Android应用支持的ABI取决于APK中位于lib/ABI目录中的.so文件，其中ABI可能是上面说过的七种ABI中的一种。 Android apk 打包时，只有指定的.so文件会被安装到 apk 中，Android包管理器安装APK时，如果在对应的lib／ABI目录中存在.so文件的话，会自动选择APK包中为对应系统ABI预编译好的.so文件。当一个应用安装在设备上，只有该设备支持的CPU架构对应的.so文件会被安装。在x86设备上，libs/x86目录中如果存在.so文件的 话，会被安装，如果不存在，则会选择armeabi-v7a中的.so文件，如果也不存在，则选择armeabi目录中的.so文件（因为x86设备也支 持armeabi-v7a和armeabi）。

## 添加动态库

第一种方法：

在 src/main中 新建 `jniLibs` 文件夹，再创建对应 ABI 架构的文件夹，比如：armeabi-v7a 或者 arm64-v8a 等，然后把 so 放到对应的 ABI 架构的文件夹中。

第二种方法：

在app 中新建和 src 同级的 libs 文件夹，再创建对应 ABI 架构的文件夹，比如：armeabi-v7a 或者 arm64-v8a 等，然后把 so 放到对应的 ABI 架构的文件夹中。
然后在 buuild.gradle 脚本中添加：

```
android {
    ......
    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }
}
```

意思是会把libs文件夹当成 jniLibs 文件夹，可以直接用so库了。

## ABI 架构选择

### 问题

如果我们平时开发过程中没有很好注意这些，那么就会带来一些兼容性的问题。
比如：你的App支持`armeabi-v7a`和`x86`架构，然后使用 Android Studio 新增了一个函数库依赖，这个函数库包含.so文件并支持更多的CPU架构，那么在某些进行上可能会发生Crash，会出现`"UnsatisfiedLinkError"`，`"dlopen: failed"`等等问题：

```
AndroidRuntime: java.lang.UnsatisfiedLinkError: dalvik.system.PathClassLoader[DexPathList[***********] couldn't find "lib*******.so"
```

或者

```
AndroidRuntime: java.lang.UnsatisfiedLinkError: dlopen failed: "*********.so" is 32-bit instead of 64-bit
```

上面报错表示，在 64 位的运行环境中加载 32 位的 so。
这是因为，你添加的一些库可能里面做了更多的架构适配，因为只要出现了这个目录，系统就只会在这个目录里找.so文件而不会遍历其他的目录，所以就出现了找不到so文件的情况（因为其他目录没有这个so文件）。
举个例子来详细说明一下：我们的APP只支持armeabi-v7a和x86架构，然后我们的APP使用了一个第三方的Library，而这个Library提供了AMR64等更多类型CPU架构的支持，构建APK的时候，这些ARM64的SO库依然会被打包进APK里面，也就是说我们自己的SO库没有对应的ARM64的SO库，而第三方的Library却有。这时候，某些ARM64的设备安装该APK的时候，发现我们的APK里带有ARM64的SO库，会误以为我们的APP已经做好了AMR64的适配工作，所以只会选择安装APK里面ARM64类型的SO库，这样会导致我们自己项目的SO库没有被正确安装（虽然armeabi-v7a和x86类型的SO库确实存在APK包里面）。

### 解决办法

给我们自己的SO库也提供AMR64支持，或者不打包第三方Library项目的ARM64的SO库。
使用第二种方案时，可以把APK里面不需要支持的ABI文件夹给删除，然后重新打包，而在Android Studio下，则可以通过以下的构建方式指定需要类型的SO库。
这个时候我们就需要使用：

```
    defaultConfig {
        ......
        ndk {
            abiFilters "***", "***"
        }
    }
```

来解决这些问题。
如果你用的`Android Studio`编译器，那么 `abiFilters` 后面加的应该是 `jniLibs/<ABI名称>` 里面所支持的机型。（当然这个目录也可以通过在 build.gradle 文件中的设置 `jniLibs.srcDir` 属性自己指定）。

## APP多架构支持

我们平时用的so的存放位置：

 - Android Studio工程放在 `jniLibs/<ABI名称>` 目录中（当然也可以通过在build.gradle文件中的设置 `jniLibs.srcDir` 属性自己指定放置 so 目录）
 - Eclipse工程放在 `libs/<ABI名称>` 目录中（这也是ndk-build命令默认生成.so文件的目录）
 - AAR压缩包中位于 `jni/<ABI名称>` 目录中（.so文件会自动包含到引用AAR压缩包的APK中）
 - 最终这些so会放在编译好的APK文件中的 `lib/<ABI名称>` 目录中

所有的 `x86/x86_64/armeabi-v7a/arm64-v8a` 设备都支持 `armeabi` 架构的 .so 文件，x86 设备能够很好的运行 ARM 类型函数库，但并不保证100%不发生crash，特别是对旧设备。64位设备（arm64-v8a, x86_64, mips64）能够运行32位（armeabi-v7a）的函数库，但是以32位模式运行，在64位平台上运行32位版本的ART和Android组件，将丢失专为64位优化过的性能（ART，webview，media等等）。
基于这些问题，我们应该尽可能为每种CPU类型都提供对应的SO库。
因此，我们可以通过在app中只放置32位so，来减小apk的打包体积。
但是在我们需要体现64位CPU给我们带来的优异性能的前提下，以减少APK包大小为由来减少架构的支持显然不是一个好的理由。因为也可以选择在应用市场上传指定ABI版本的APK，生成不同ABI版本的APK可以在build.gradle中如下配置：

```
android {
   ... 
   splits {
	   abi {
		   enable true
		   reset()
		   include 'x86', 'x86_64', 'armeabi-v7a', 'arm64-v8a' //select ABIs to build APKs for
		   universalApk true //generate an additional APK that contains all the ABIs
	   }
   }
   // map for the version code
   project.ext.versionCodes = ['armeabi': 1, 'armeabi-v7a': 2, 'arm64-v8a': 3, 'mips': 5, 'mips64': 6, 'x86': 8, 'x86_64': 9]
   android.applicationVariants.all { variant ->
	   // assign different version code for each output
	   variant.outputs.each { output ->
		   output.versionCodeOverride =
				   project.ext.versionCodes.get(output.getFilter(com.android.build.OutputFile.ABI), 0) * 1000000 + android.defaultConfig.versionCode
	   }
   }
}
```

## 多架构在 APK 打包时的选择

前面我们说到 `abiFilters` 可以决定把哪种 .so 打包到 apk 中去，它也只是决定了打包时的多架构选择，具体 app 运行时选择什么架构，还是由 app 安装时来决定的。

## 多架构的安装和加载机制

apk在安装的过程中，系统就会对apk进行解析根据里面so文件类型，确定这个apk安装是在32 还是 64位的虚拟机上，如果是32位虚拟机那么就不能使用64位so，如果是64位虚拟机也不能使用32位so。而64位设备可以提供32和64位两种虚拟机，根据apk选择开启哪一种，因此说64位设备兼容32的so库。
另外，安装过程中虚拟机类型的选择和 AndroidManifest 里面的 `android:multiArch` 有关系，系统在解包动态库的时候，如果 multiArch设置为true，那么 abilist32 和 abilist64这两个数组都被选中；否则，在abilist和 abilist32间选择一个合适的数组。选择好abilist后，数组中的<primary-abi>/*.so 会被拷贝出来，如果不存在，则选择 secondary-abi下的so文件。
一旦安装完成，就确定了该 App 在当前机器上的运行模式，系统是不允许 32位 so 和 64 位 so 混合使用的。因为虚拟机不允许这样做。
具体机制，分下面四种情况：

 1. 假设apk的lib目录放置了32和64位两种so，那么安装时根据当前设备的cpu架构从上到下筛选(X86 > arm64 > arm32)，一旦发现lib里面有和设备匹配的so文件，那么直接选定这种架构为标准。比如当前设备是64位并且发现lib有一个64位的so，那么apk会拷贝lib下所有64位的so文件到 `data/data/<packageName>/lib/` 目录（查看此目录需要ROOT）
 后面调用System.loadLibrary其实就是加载这个目录下的so文件，此时如果有某个64位so文件在我们项目的lib中没有提供，就会直接报错程序崩溃。因此这里如果放置部分功能32位so，部分功能放置放置64位so，即使用多进程来加载模型，也会报错崩溃。
 2. apk的lib目录只放置32位so，参照上面原理，运行在32位设备是OK的。绝大多数64位设备也是OK的，不过x86的设备肯定会崩溃。假设现在运行在64位设备，然后在代码中动态加载64位so文件，会报错：`so is 64-bit instead of 32-bit`
 3. apk的lib目录只放64位的so，那这个apk只能运行在64位的设备了，同理如果在代码中动态加载32位的so，会报错：`so is 32-bit instead of 64-bit`
 4. apk的lib不放任何so文件，全部动态加载。安装在32位设备就只能加载32位so，安装在64位的设备系统会默认你的apk运行在64位虚拟机，此时动态加载32位so也是不行的。


## NDK生成支持多种CPU架构的SO

在android NDK项目的根目录下面有一个libs文件夹，这个文件夹下面有下面七个文件夹中的一个或者多个：`arm64-v8a`，`armeabi`，`armeabi-v7a`，`mips`，`mips64`，`x86`，`x86_64`。
默认情况下，NDK的编译系统根据 `armeabi` ABI生成机器代码。可以使用APP_ABI 来选择一个不同的ABI，我们可以通过下面的配置来制定支持的ABI：

```
TARGET_CPU_API := all
APP_ABI := all
```

或者是

```
TARGET_CPU_API :=  armeabi armeabi-v7a x86 x86_64 arm64-v8a mips mips64
APP_ABI :=  armeabi armeabi-v7a x86 x86_64 arm64-v8a mips mips64
```

## 一些命令

### 查看设备支持的 ABI 类型

```
adb shell getprop |grep ro.product.cpu.abilist
```

### 查看某个应用的 ABI 类型

```
adb shell dumpsys package <packageName>
```

查看 Packages 中的 primaryCpuAbi 和 secondaryCpuAbi

## 推荐文章

[Android中app进程ABI确定过程](https://zhuanlan.zhihu.com/p/29801087)
[APK安装流程详解4——安装中关于so库的那些事](https://www.jianshu.com/p/c9fc6743a383)