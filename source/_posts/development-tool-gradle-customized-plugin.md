
---
title: Gradle 使用指南 -- 创建Plugin
categories: Gradle
comments: true
tags: [Android, Gradle, Android Studio]
description: 介绍如何在 Android Studio 中自定义 Gradle 插件
date: 2016-4-15 10:00:00
---

## 概述

Gradle 的插件可以有三种形式来提供：

 - 直接在build.gradle中编写Plugin，这种方式这种方法写的Plugin无法被其他 build.gradle 文件引用。
 - 单独的一个Module，这个Module的名称必须为buildSrc，同一个工程中所有的构建文件够可以引用这个插件，但是不能被其他工程引用。
 - 在一个项目中自定义插件，然后上传到远端maven库等，其他工程通过添加依赖，引用这个插件。

本文只对后面两种方式来进行简单介绍。

## 在当前项目中创建插件

最终的目录结构为：

![效果图](/images/development-tool-customized-gradle-plugin/gradle-plugin-module.png)

### 创建Plugin

 1. 创建一个 Module （Phone&Tablet Module 或 Android Librarty 都可以），Module的名称必须为 buildSrc。
 2. 将Module里面的内容删除，只保留build.gradle文件和src/main目录。	
 3. 我们开发的 gradle 插件相当于一个 groovy 项目。所以需要在 main 目录下新建 groovy 目录。
 4. 然后创建一个 Java 文件一样的方式创建一个 groovy 文件，比如报名为 com.android.hq.testplugin 的 TestPlugin.groovy 文件。

### 修改build.gradle

因为我们要用到 groovy，以及要用到gradle和groovy的sdk，因此将 buildSrc 下面的 build.gradle 修改为：

```
apply plugin: 'groovy'

dependencies {
    //gradle sdk
    compile gradleApi()
    //groovy sdk
    compile localGroovy()
}

repositories {
    jcenter()
}
```

### 实现

实现 `TestPlugin` 类，在脚本中通过实现gradle的Plugin接口，实现apply方法即可：

```java
package com.android.hq.testplugin

import org.gradle.api.Plugin
import org.gradle.api.Project;

public class TestPlugin implements Plugin<Project>{
    @Override
    void apply(Project project) {
        // 实现一个名称为testPlugin的task
        project.task('testPlugin') << {
            println "## This is my first gradle plugin in testPlugin task"
        }

        println "** This is my first gradle plugin"
    }
}
```

这里在简单实现了一个名称为testPlugin的task，当执行该task时，会打印 `## This is my first gradle plugin in testPlugin task`，而`** This is my first gradle plugin`在执行所有的task时都会打印。

### 使用

使用方式比较简单，在 app 的 build.gradle 中添加

```
apply plugin: com.android.hq.testplugin.TestPlugin
```

即可。


### 测试
1. 在 Gradle 的 task 视窗里面: app/other 下多了一个 testPlugin 的 task。

2. 输入 `./gradlew tasks`，我们可以看到 testPlugin 已经在task列表中。

```
:tasks

------------------------------------------------------------
All tasks runnable from root project
------------------------------------------------------------

Android tasks
-------------
androidDependencies - Displays the Android dependencies of the project.
signingReport - Displays the signing info for each variant.
sourceSets - Prints out all the source sets defined in this project.

Build tasks
-----------
assemble - Assembles all variants of all applications and secondary packages.
assembleAndroidTest - Assembles all the Test applications.
assembleDebug - Assembles all Debug builds.
assembleRelease - Assembles all Release builds.
build - Assembles and tests this project.

......

Other tasks
-----------
jarDebugClasses
jarReleaseClasses
lintVitalRelease - Runs lint on just the fatal issues in the Release build.
testPlugin
```

执行 testPlugin task，输出：

```
$ ./gradlew testPlugin

:buildSrc:compileJava UP-TO-DATE
:buildSrc:compileGroovy UP-TO-DATE
:buildSrc:processResources UP-TO-DATE
:buildSrc:classes UP-TO-DATE
:buildSrc:jar UP-TO-DATE
:buildSrc:assemble UP-TO-DATE
:buildSrc:compileTestJava UP-TO-DATE
:buildSrc:compileTestGroovy UP-TO-DATE
:buildSrc:processTestResources UP-TO-DATE
:buildSrc:testClasses UP-TO-DATE
:buildSrc:test UP-TO-DATE
:buildSrc:check UP-TO-DATE
:buildSrc:build UP-TO-DATE
** This is my first gradle plugin
** build versionName=2.1.0
:app:testPlugin
## This is my first gradle plugin in testPlugin task

BUILD SUCCESSFUL

Total time: 5.358 secs
```
可以看到，`** This is my first gradle plugin`和`## This is my first gradle plugin in testPlugin task`都会打印出来。
执行 assembleDebug task，输出：

```
$ ./gradlew assembleDebug

:buildSrc:compileJava UP-TO-DATE
:buildSrc:compileGroovy UP-TO-DATE
:buildSrc:processResources UP-TO-DATE
:buildSrc:classes UP-TO-DATE
:buildSrc:jar UP-TO-DATE
:buildSrc:assemble UP-TO-DATE
:buildSrc:compileTestJava UP-TO-DATE
:buildSrc:compileTestGroovy UP-TO-DATE
:buildSrc:processTestResources UP-TO-DATE
:buildSrc:testClasses UP-TO-DATE
:buildSrc:test UP-TO-DATE
:buildSrc:check UP-TO-DATE
:buildSrc:build UP-TO-DATE
** This is my first gradle plugin
** build versionName=2.1.0
:app:preBuild UP-TO-DATE
:app:preDebugBuild UP-TO-DATE

......

:app:assembleDebug UP-TO-DATE

BUILD SUCCESSFUL

Total time: 4.925 secs


```

可以看到，只有 `** This is my first gradle plugin` 输出。

## 独立的插件项目

这种类型的插件可以上传到远端 maven 库等，其他工程通过添加依赖，引用这个插件。
创建步骤和前面的在当前项目中创建插件的步骤有些是类似的。

### 创建Plugin

这里的步骤除了3,4,5和前的一样，其他的都是不一样的。 

 1. 创建一个新的 Project。
 2. 同样创建一个 Module （Phone&Tablet Module 或 Android Librarty 都可以），Module的名称随意。
 3. 将Module里面的内容删除，只保留build.gradle文件和src/main目录。
 4. 我们开发的 gradle 插件相当于一个groovy项目。所以需要在main目录下新建groovy目录。
 5. 然后创建一个 Java 文件一样的方式创建一个 groovy文件，比如报名为 com.android.hq.testplugin 的 TestPlugin.groovy 文件。
 6. 现在，我们已经定义好了自己的 gradle 插件类，接下来就是告诉 gradle，哪一个是我们自定义的插件类，因此，需要在 main 目录下新建 resources 目录，然后在 resources 目录里面再新建 META-INF 目录，再在 META-INF 里面新建 gradle-plugins 目录。最后在 gradle-plugins 目录里面新建 properties 文件，注意这个文件的命名，你可以随意取名，但是后面使用这个插件的时候，会用到这个名字。比如，你取名为com.android.hq.testplugin.properties，而在其他 build.gradle 文件中使用自定义的插件时候则需写成：
```
apply plugin: 'com.android.hq.testplugin'
```
 7. 然后在com.android.hq.testplugin.properties文件里面指明你自定义的类：
```
implementation-class=com.android.hq.testplugin.TestPlugin
```
 8. 修改build.gradle
因为我们要用到 groovy 以及后面打包要用到 maven ,所以在我们自定义的 Module 下的 build.gradle 需要添加如下代码：

```
apply plugin: 'groovy'
apply plugin: 'maven'

dependencies {
    //gradle sdk
    compile gradleApi()
    //groovy sdk
    compile localGroovy()
}

repositories {
    mavenCentral()
}
```

### 实现

实现 `TestPlugin` 类，这一步和前面的一样：

```java
package com.android.hq.testplugin

import org.gradle.api.Plugin
import org.gradle.api.Project;

public class TestPlugin implements Plugin<Project>{
    @Override
    void apply(Project project) {
        // 实现一个名称为testPlugin的task
        project.task('testPlugin') << {
            println "## This is my first gradle plugin in testPlugin task"
        }

        println "** This is my first gradle plugin"
    }
}
```

### 打包

前面我们已经自定义好了插件，接下来就是要打包到Maven库里面去了，你可以选择打包到本地，或者是远程服务器中。在我们自定义Module目录下的build.gradle添加如下代码：

```
//group和version在后面使用自定义插件的时候会用到
group='com.android.hq.testplugin'
version='1.0.0'

uploadArchives {
    repositories {
        mavenDeployer {
            //提交到远程服务器：
            //repository(url: "http://www.xxx.com/repos") {
            //    authentication(userName: "admin", password: "admin")
            //}
            //本地的Maven地址设置为/mnt/TestRepos/
            repository(url: uri('/mnt/TestRepos/'))
        }
    }

}
```

现在我们定义好了打包地址以及打包相关配置，还需要我们让这个打包 task 执行。点击 AndroidStudio 右侧的 gradle 工具，如下图所示：

![效果图](/images/development-tool-customized-gradle-plugin/gradle-plugin-independent.png)

可以看到有 uploadArchives 这个 Task,双击 uploadArchives 就会执行打包上传。执行完成后，去我们的 Maven 本地仓库查看一下：

```
hq@EF-hq:/mnt/TestRepos/com/android/hq/testplugin/testplugin/1.0.0$ ls
testplugin-1.0.0.jar  testplugin-1.0.0.jar.md5  testplugin-1.0.0.jar.sha1  testplugin-1.0.0.pom  testplugin-1.0.0.pom.md5  testplugin-1.0.0.pom.sha1
```

其中，/com/android/hq/testplugin/ 这几层目录是由我们的 group 指定，testplugin 是模块的名称，1.0.0是版本号（version指定）。

### 使用

首先我们需要在使用的 Module 的build.gradle文件中里面指定Maven地址、自定义插件的名称以及依赖包名。代码如下：

```
//com.android.hq.testplugin.TestPlugin为resources/META-INF/gradle-plugins 下的properties文件名称
apply plugin: 'com.android.hq.testplugin'

buildscript {
    repositories {
        maven {
            //本地Maven仓库地址
            url uri('/mnt/TestRepos/')
        }
    }
    dependencies {
        //格式为-->group:module:version
        classpath 'com.android.hq.testplugin:testplugin:1.0.0'
    }
}
```

然后就可以测试使用情况了。

### 测试

和前面的方法一样，不再详述。

## 参考

http://www.jianshu.com/p/d53399cd507b


