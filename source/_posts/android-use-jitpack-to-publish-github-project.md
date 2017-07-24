---
title: Android：使用JitPack发布Github开源库 
categories: Android
comments: true
tags: [Android, JitPack]
description: 介绍使用JitPack发布Github开源库的流程
date: 2017-7-21 10:00:00
---

JitPack 是一个发布流程非常简单的自定义的 Maven 仓库，可以用来发布自己的 JVM 或者 Android 开源库。
JitPack 的官方文档在这里 [Publish an Android library](https://jitpack.io/docs/ANDROID/)。
参考我的Github上面一个[开源项目](https://github.com/heqiangflytosky/FastScrollWebView)。
下面来介绍一下使用JitPack发布一个开源项目的步骤。

## GitHub准备

### 代码准备

首先将需要发布的library工程准备好。
打开根目录的build.gradle，在 dependencies 节点添加：

```
buildscript {
    ...
    dependencies {
        ...
        classpath 'com.github.dcendents:android-maven-gradle-plugin:1.3'

    }
}
```

如图：

![效果图](/images/android-use-jitpack-to-publish-github-project/jitpack-root-gradle.png)

然后打开 library/build.gradle，添加：

```
...
apply plugin: 'com.github.dcendents.android-maven'
group='com.github.<YourUsername>'
...

```

如图：
![效果图](/images/android-use-jitpack-to-publish-github-project/jiapack-lib-gradle.png)

检查一下工程的 gradle/wrapper/ 目录下面是否有下面两个文件，并检查一下确保这两个文件没有添加到.gitignore文件：

```
gradle-wrapper.jar
gradle-wrapper.properties
```

如果没有的话用`gradle wrappe`和`./gradlew install`生成一下。
然后就可以把工程push到Github上面。

### Github配置

push到Github上以后，要添加一个Release版本，如图点击项目中的 release 按钮：

![效果图](/images/android-use-jitpack-to-publish-github-project/jitpack-github-add-release.png)

如果还没有发布过版本，点击Create a new release 按钮，如果以前发布过，点击 Draft a new release，然后填写版本信息：

![效果图](/images/android-use-jitpack-to-publish-github-project/jiapack-github-add-version.png)

填写完成后点击Publish release。

## 发布到jitpack

复制项目地址，打开 https://jitpack.io/ ，把项目地址粘贴到输入框，然后点击 look up 然后就可以看到你创建的开源库了：

![效果图](/images/android-use-jitpack-to-publish-github-project/gitpack-look-up.png)

点击 get it，在页面下方就可以看到使用方法了。

![效果图](/images/android-use-jitpack-to-publish-github-project/jitpack-get-it.png)







## 遇到的问题
### Error:Unable to load class 'org.gradle.internal.logging.LoggingManagerInternal

![效果图](/images/android-use-jitpack-to-publish-github-project/jitpack-error-1.png)

gradle 版本和 android-maven-gradle-plugin 版本不协调，我原来用的gradle版本是1.3.0，android-maven-gradle-plugin版本是1.5，把android-maven-gradle-plugin版本改为1.3就可以了。

```
        classpath 'com.android.tools.build:gradle:1.3.0'
        classpath 'com.github.dcendents:android-maven-gradle-plugin:1.3'
```

