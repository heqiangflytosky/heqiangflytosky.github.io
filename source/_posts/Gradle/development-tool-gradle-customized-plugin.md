
---
title: Gradle 使用指南 -- Plugin 的使用和创建
categories: Gradle
comments: true
tags: [Android, Gradle, Android Studio]
description: 介绍如何在 Android Studio 中自定义 Gradle 插件
date: 2016-4-15 10:00:00
---

[官方文档：Using Gradle Plugins](https://docs.gradle.org/current/userguide/plugins.html)
[官方文档：Writing Custom Plugins](https://docs.gradle.org/current/userguide/custom_plugins.html)

## 插件的类型

插件可以分为下两种：

 - 脚本插件：脚本插件是额外的构建脚本，进一步配置构建，常常用于声明一些操作构建的方法，它们通常在构建内部使用。
 - 二进制插件：二进制插件是实现了Plugin的类，并且通过编程方式来操作构建，二进制插件可以在构建脚本中，也可以在buildSrc中，也可以以jar的方式导入。

## 插件的使用

先来介绍一下插件的使用，可以分为两步骤：

 - 解析插件：找到给定的包含插件的 jar 的正确版本，并把它添加到脚本的 classpath 中。一旦插件被解析，它的 API 就能在构建脚本中使用，脚本可以用文件路径或者 url 进行解析。脚本插件是可以自动解析的，它可以是相对于项目目录的本地路径，也可以是个http的远程路径。比如：`apply from: 'other.gradle'`。二进制插件是需要应用插件id或者jar包路径来自动解析。比如 `classpath 'com.android.hq.testplugin:testplugin:1.0.0'`,`classpath files("../../TestGradlePlugin/app/build/libs/app-1.0.0.jar")`
 - 应用插件：应用插件其实就是在需要应用插件的项目上执行 `Plugin.apply()` 方法。比如：`apply plugin: 'com.android.application'`。

### 引用插件的两种方式：

第一种方式：

```
buildscript {
  repositories {
    maven {
      url "https://plugins.gradle.org/m2/"
    }
  }
  dependencies {
    classpath "org.hidetake:gradle-ssh-plugin:2.9.0"
  }
}

apply plugin: "org.hidetake.ssh"

```

上面这个是 2.0及以前版本的用法。

第二种方式：

```
plugins {
  id 'org.hidetake.ssh' version '2.9.0'
}
```

这是 gradle 2.1 及以上最新版本的语法。
plugins{} 块提供了下面的语法约束：

```
plugins {
    id «plugin id»                                            // (1)
    id «plugin id» version «plugin version» [apply «false»]   // (2)
}
```

«plugin version»和«plugin id»必须是常量，文字或字符串，apply则必须是bool类型（可以用于快速禁用插件）。
plugins {}目前仅仅可以使用在构建脚本中，而不能使用在脚本插件，settings.gradle和初始化脚本。

### 插件的配置

插件引入到工程之后，我们就可以执行在插件中定义的各种 Task 了，我们可以通过 `./gradlew tasks --all` 来查看可以执行的所有 Task。另外，我们还可以在 build.gradle 中对插件进行配置，其实这一部分我们是经常使用到的，比如：

```
android {
    compileSdkVersion 26
    defaultConfig {
        applicationId "com.example.hq.myapplication"
        minSdkVersion 21
        targetSdkVersion 26
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

上面代码就是对 android 插件进行的各种配置。

### 把插件应用到子工程中

在多项目构建中，你可能想吧插件应用到一部分或者所有的子项目，而不是根项目，plugins {}默认的行为是快速解析和应用插件，但是你可以通过 apply false来告诉Gradle不要应用插件，然后通过apply plugin: «plugin id»应用到子项目中。

```
plugins {
    id 'org.gradle.sample.hello' version '1.0.0' apply false
    id 'org.gradle.sample.goodbye' version '1.0.0' apply false
}

subprojects {
    if (name.startsWith('hello')) {
        apply plugin: 'org.gradle.sample.hello'
    }
}
```

## 自定义插件库

默认情况下，plugins {}DSL将会从Gradle插件官网下载插件，但很多构建者想从自己的私有Maven仓库下载Gradle插件，因为可能需要私有的实现细节，而且可以更好的控制插件的可用性。

为了指定自定义的插件资源库，需要在settings.gradle中使用pluginManagement {}下嵌套的repositories {}块

```
pluginManagement {
    repositories {
        maven {
            // 本地
            url '../maven-repo'
            // 远程
            // uri 'http://127.0.0.1:8081/maven-repo/'
        }
        gradlePluginPortal()
        ivy {
            url '../ivy-repo'
        }
    }
}
```

这会告诉Gradle首先在 `../maven-repo` 资源库中查找，如果没有找到然后才会寻找官方的插件资源库，如果你不想寻找官方资源库，可以去掉gradlePluginPortal()。

## 插件的创建

创建插件一般我们是通过实现 Plugin 接口来实现：

```
public interface Plugin<T> {
    void apply(T var1);
}
```

它有个apply方法，这个方法是在插件的配置阶段执行的方法，也即是说当我们引入插件时，这个方法就执行。
比如当我们通过 `apply plugin: 'com.android.application'` 引入 android 插件时，对应的 apply 方法就会执行了。

```
class CustomExtension {
    String msg
}

class CustomPlugin implements Plugin<Project>{

    @Override
    void apply(Project target) {
        // 创建一个扩展属性 myExtension，使用 CustomExtension 进行管理外部属性配置
        target.extensions.create("myExtension", CustomExtension)

        // 实现一个名称为testPluginTask1的task，设置分组为 myPlugin，并设置描述信息
        target.task('testPluginTask1', group: "myPlugin", description: "This is my test plugin") << {
            println "## This is my first gradle plugin in testPlugin task. msg = "+target.myExtension.msg
        }
        // 实现一个名称为testPluginTask2的task，设置分组为 myPlugin，并设置描述信息
        target.task('testPluginTask2', group: "myPlugin", description: "This is my test plugin") << {
            println "## This is my first gradle plugin in testPlugin task. msg = "+target.myExtension.msg
        }
        println "** This is my first gradle plugin. msg = "+target.myExtension.msg
    }
}

apply plugin: CustomPlugin

myExtension {
    msg "testMSG"
}
```

测试 Task，运行 `./gradlew testPluginTask1`：

```
> Configure project :app
** This is my first gradle plugin. msg = null

> Task :app:testPluginTask1
## This is my first gradle plugin in testPlugin task. msg = testMSG
```

上面的配置阶段 msg 输出为 null，是因为调用 `apply plugin: CustomPlugin` 时还没有给 msg 赋值。
下面我们测试一下 apply 方法的执行时机：

```println "before apply CustomPlugin"
apply plugin: CustomPlugin
println "after apply CustomPlugin"
```

运行 `./gradlew testPluginTask1`：

```
> Configure project :app
before apply CustomPlugin
** This is my first gradle plugin. msg = null
after apply CustomPlugin

> Task :app:testPluginTask1
## This is my first gradle plugin in testPlugin task. msg = testMSG
```

可以看到 apply 方法确实是在 `apply plugin: CustomPlugin` 时调用的。

## 创建插件的方式

Gradle 的插件可以有三种形式来提供：

 - 直接在build.gradle中编写 Plugin，这种方式这种方法写的Plugin无法被其他 build.gradle 文件引用。
 - 单独的一个Module，这个Module的名称必须为buildSrc，同一个工程中所有的构建文件够可以引用这个插件，但是不能被其他工程引用。
 - 在一个项目中自定义插件，然后上传到远端maven库等，其他工程通过添加依赖，引用这个插件。

第一种方式在上面的插件的创建这一节中已经介绍了，本文接下来再对后面两种方式来进行简单介绍。


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
        // 实现一个名称为testPlugin的task，设置分组为 myPlugin，并设置描述信息
        project.task('testPlugin', group: "myPlugin", description: "This is my test plugin") << {
            println "## This is my first gradle plugin in testPlugin task"
        }
        println "** This is my first gradle plugin"
    }
}
```

这里在简单实现了一个名称为testPlugin的 task，apply 方法会在配置阶段执行。
当执行该插件的 testPlugin taks 时，会打印 `## This is my first gradle plugin in testPlugin task`，而`** This is my first gradle plugin`则在配置阶段打印出来。因为在执行所有的task时都会进行执行配置阶段，那么这个打印在执行任何一个 task 时都会打印。

### 使用

使用方式比较简单，在 app 的 build.gradle 中添加

```
apply plugin: com.android.hq.testplugin.TestPlugin
```

即可。


### 测试
1. 在 Gradle 的 task 视窗里面: app/myplugin 下多了一个 testPlugin 的 task。

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

MyPlugin tasks
--------------
testPlugin - This is my test plugin
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

## 独立的插件项目（发布的本地或者远程仓库）

这种类型的插件可以在本地独立引用，也可以上传到远端 maven 库等，其他工程通过添加依赖，引用这个插件。
实质是二进制插件被发布成外部的 jar 文件，然后通过将其增加到构建脚本的 classpath 中来进一步应用插件。
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
        // 实现一个名称为testPlugin的task，设置分组为 myPlugin，并设置描述信息
        project.task('testPlugin', group: "myPlugin", description: "This is my test plugin") << {
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

![效果图](/images/development-tool-customized-gradle-plugin/gradle-plugin-upload-archives.png)

可以看到有 uploadArchives 这个 Task,双击 uploadArchives 就会执行打包上传。执行完成后，去我们的 Maven 本地仓库查看一下：

```
hq@EF-hq:/mnt/TestRepos/com/android/hq/testplugin/testplugin/1.0.0$ ls
testplugin-1.0.0.jar  testplugin-1.0.0.jar.md5  testplugin-1.0.0.jar.sha1  testplugin-1.0.0.pom  testplugin-1.0.0.pom.md5  testplugin-1.0.0.pom.sha1
```

其中，/com/android/hq/testplugin/ 这几层目录是由我们的 group 指定，testplugin 是模块的名称，1.0.0是版本号（version指定）。

### 使用

首先我们需要在使用的 Module 的build.gradle文件中里面指定Maven地址、自定义插件的名称以及依赖包名。
当然也可以直接引用 jar 包的方式，这种方式下面介绍。
代码如下：

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


## 独立的插件项目2（生成jar包直接引用）

上面介绍的独立插件项目的做法有点繁琐，要建各种文件，还要上传本地仓库或者远程仓库，下面再介绍一个方法，直接生成插件 jar 包后，然后通过将其增加到需要使用的工程中的构建脚本的 classpath 中来直接引用插件。

### 创建Plugin 

 - 创建一个新的 Project。
 - 删除app模块（这个模块名可以随意定）的 src/main 下面的所有文件，创建 groovy 目录。
 - 在 groovy 目录下面创建 com/android/hq/testplugin 目录
 - 在 com/android/hq/testplugin 目录下面创建 TestPlugin.groovy 文件。

### 实现

实现 TestPlugin 类，这一步和前面的一样：

```
package com.android.hq.testplugin
import org.gradle.api.Plugin
import org.gradle.api.Project;
public class TestPlugin implements Plugin<Project>{
    @Override
    void apply(Project project) {
        // 实现一个名称为testPlugin的task，设置分组为 myPlugin，并设置描述信息
        project.task('testPlugin', group: "myPlugin", description: "This is my test plugin") << {
            println "** Test This is my first gradle plugin in testPlugin task **"
        }
        println "** Test This is my first gradle plugin **"
    }
}
```

### 打包

打包前先实现 app 模块的 build.gradle 文件：

```
// 引用插件
plugins {
    // 用来生成plugin的插件，gradlePlugin 方法
    id "java-gradle-plugin"
    // 用来发布plugin的插件，publishing 方法
    id 'maven-publish'
    id "groovy"
}

//group和version在后面使用自定义插件的时候会用到
group='com.android.hq.testplugin'
version='1.0.0'
gradlePlugin {
    plugins {
        hello {
            id = 'com.android.hq.testplugin.test'
            implementationClass = 'com.android.hq.testplugin.TestPlugin'
        }
    }
}

publishing {
    repositories {
        maven {
            url '../maven-repo'
        }
    }
}
```

然后运行命令发布，也可以只编译生成 jar 包。
在当前工程下运行 `./gradlew publish`，会在 `app/build/libs/` 目录下面生成 app-1.0.0.jar 文件。还会在对应的 maven-repo 目录下面生成资源库。

```
# ls maven-repo/com/android/hq/testplugin/app/
1.0.0                   maven-metadata.xml      maven-metadata.xml.md5  maven-metadata.xml.sha1
# ls maven-repo/com/android/hq/testplugin/app/1.0.0/
app-1.0.0.jar      app-1.0.0.jar.md5  app-1.0.0.jar.sha1 app-1.0.0.pom      app-1.0.0.pom.md5  app-1.0.0.pom.sha1
```
如果只运行 `./gradlew build`，只会在 `app/build/libs/` 目录下面生成 app-1.0.0.jar 文件。这个文件就是插件包。

### 使用

这里只介绍引用jar包的方法，当然也可以通过上面的指定本地或者远程仓库地址然后通过应用插件id的方式引用。
在其他工程的中使用该插件，需要在使用的模块的 build.gradle 文件中添加下面的代码：

```
// 引用插件
apply plugin: 'com.android.hq.testplugin.test'

// 添加插件的依赖路径
buildscript {
    dependencies {
        // 这里直接使用了上面插件工程中生成 jar 包的路径，
        // 当然可以把jar包copy到你喜欢的路径然后在此指定就可以了
        classpath files("../../TestGradlePlugin/app/build/libs/app-1.0.0.jar")
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
```

### 测试

在测试工程下面运行 `./gradlew tasks`：

```
MyPlugin tasks
--------------
testPlugin - This is my test plugin
```

已经多了 testPlugin 这个 task。
执行该 task，`./gradlew testPlugin`：

```
> Configure project :app
The Task.leftShift(Closure) method has been deprecated and is scheduled to be removed in Gradle 5.0. Please use Task.doLast(Action) instead.
        at build_f32z80bmob9ip5197i06gw0om.run(/home/heqiang/android-studio-workspace/TestPlugin/app/build.gradle:3)
** Test This is my first gradle plugin **

> Task :app:testPlugin
** Test This is my first gradle plugin in testPlugin task **
```

输出了我们的测试代码。


## 说明

上面介绍的两种独立的插件项目的使用，其中打包和使用两个步骤并不是严格的一一对照关系，它们是可以交叉使用的，任何一种打包出来的jar包都可以使用介绍的两种引用方法。


## 参考

https://docs.gradle.org/current/userguide/plugins.html
https://docs.gradle.org/current/userguide/custom_plugins.html
http://www.jianshu.com/p/d53399cd507b
https://blog.csdn.net/lastsweetop/article/details/79643576

