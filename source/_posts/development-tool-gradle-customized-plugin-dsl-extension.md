---
title: Gradle 使用指南 -- Plugin DSL 扩展
categories: Gradle
comments: true
tags: [Android, Gradle, Android Studio]
description: 介绍如何在 Android Studio 中自定义 Gradle 插件
date: 2016-4-16 10:00:00
---

## 概述

前面的博客[Gradle 使用指南 -- 创建Plugin ](http://www.heqiangfly.com/2016/03/15/development-tool-gradle-customized-plugin/) 介绍了如何去创建一个插件，那么这篇文章将介绍一些深入的知识：如何对自定义插件进行 DSL 扩展。
在博客[Gradle 使用指南 -- Android DSL 扩展 ](http://www.heqiangfly.com/2016/03/08/development-tool-gradle-android-dsl-extension/) Android 插件对 Gradle 进行的 DSL 扩展，那么我们自定义插件也是完全可以做到的。

## DSL 扩展基本实现

我们在进行 Gradle 配置时，很多的配置都是在 build.gradle 文件中进行的，插件可以在构建过程中获取这些输入。我们自定义的插件也是可以做到这一点的。这就要借助 `ExtensionContainer` 来实现。
怎么来得到一个 `ExtensionContainer` 对象呢？我们来看一下 `Project` 的 `getExtensions()` 方法：

```
    ExtensionContainer getExtensions();
```

它的返回者是一个 `ExtensionContainer` 对象。
通过 `ExtensionContainer` 我们可以向目标对象添加DSL扩展，通过 `ExtensionContainer` 的 `create()` 方法来创建新的 DSL 域，并与一个对应的委托类关联起来（即新建一个 DSL 域，并委托给一个具体类），通过它来跟踪传递给插件的所有设置和属性。

首先实现一个扩展类：

```
class MyExtension {
    String message
    Boolean isDebug
    
    String getMessage() {
        return message
    }

    void setMessage(String message) {
        println "set message = "+message
        this.message = message
    }
}
```

然后在插件中的 `apply` 方法中通过 `project.extensions.create()` 创建DSL扩展：

```
public class MyFirstPlugin implements Plugin<Project>{
    @Override
    void apply(Project project) {
        // 创建一个扩展属性 myExtension，使用 MyExtension 进行管理外部属性配置
        project.extensions.create('myExtension', MyExtension.class)
        // 实现一个名称为myPlugin的task
        project.task('myPlugin') << {
            println project.myExtension.message
            println project.myExtension.isDebug
        }
    }
}
```

在 build.gradle 中通过引入插件后就可以配置了

```
apply plugin: com.android.hq.myfirstplugin.MyFirstPlugin

myExtension {
    message "Hello Plugin"
    isDebug true
}
```

运行 myPlugin task 后可以通过打印看到配置中的输入。

### 传递参数

我们先来看一下 `create` 方法：

```
<T> T create(String var1, Class<T> var2, Object... var3);
```

第一个参数是在 build.gradle 中可以配置的代码块的方法名称；
第二个参数是关联的扩展实体类的名称
后面的参数表示传递给实体类构造函数的参数

比如想把 `apply` 方法的 `project` 参数传递进来，就需要这样引用：

```
project.extensions.create('myExtension', MyExtension.class, project)
```

那么对应 `MyExtension` 类要加一个带 `Project` 类的构造函数：

```
class MyExtension {
    String message
    Boolean isDebug
    public MyExtension(Project project) {
    }
    String getMessage() {
        return message
    }

    void setMessage(String message) {
        println "set message = "+message
        this.message = message
    }
}
```

### 子  Script blocks 配置

在使用配置 gradle 中我们一定使用过这样的配置：

```
android {
    compileSdkVersion 23
    buildToolsVersion "23.0.1"
    defaultConfig {
        applicationId "com.example.heqiang.testsomething"
        minSdkVersion 23
        targetSdkVersion 23
    }
}
```

在 `android` 代码块中还可以进行 `defaultConfig` 代码块的配置，那么在自定义 Plugin 中这个配置方法也是可以实现的，也是要借助 `ExtensionContainer` 的 `create()` 方法来实现新的 DSL 域的创建。
比如在上面的例子中我们想在 `myExtension` 域中创建一个 `defaultConfig` 配置方法，那么可以在 `MyExtension` 中通过下面的代码来实现：
实现一个 `DefaultConfig` 类：
```
public class DefaultConfig {
    String applicationId
    int minSdkVersion
    int targetSdkVersion
    public DefaultConfig(String name) {
        println "name = " + name
    }
}
```

在 `MyExtension` 中添加映射关系：

```
class MyExtension {
    String message
    Boolean isDebug

    public MyExtension() {
        this.extensions.create("defaultConfig", DefaultConfig, "defaultConfig")
    }
    
    String getMessage() {
        return message
    }

    void setMessage(String message) {
        println "set message = "+message
        this.message = message
    }
}
```

```
public class MyFirstPlugin implements Plugin<Project>{
    @Override
    void apply(Project project) {
        project.extensions.create('myExtension', MyExtension.class)
        // 实现一个名称为myPlugin的task
        project.task('myPlugin') << {
            println project.myExtension.message
            println project.myExtension.isDebug
            println project.myExtension.defaultConfig.applicationId
            println project.myExtension.defaultConfig.minSdkVersion
            println project.myExtension.defaultConfig.targetSdkVersion
        }
    }
}
```

配置代码：

```
myExtension {
    message "Hello Plugin"
    isDebug true

    defaultConfig {
        applicationId "com.android.hq.test"
        minSdkVersion 23
        targetSdkVersion 23
        
        println "print in defaultConfig"
    }
}
```

## 容器扩展

在进行 Android 配置时，我们一定用过 `buildTypes` 的配置，类似：

```
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
        debug {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
```

这种类型可以用于在代码块中创建新的指定类型的对象。
先来看一下源码：
```
    public void buildTypes(Action<? super NamedDomainObjectContainer<BuildType>> action) {
        this.checkWritability();
        action.execute(this.buildTypes);
    }
```

它传入的是一个 `BuildType` 类型列表的闭包代码。
`NamedDomainObjectContainer` 是一个容器，追根溯源它是继承自 `Collection<T>`。
我们这里叫它命名对象容器，可以用于在 builds cript 中创建对象,创建的对象必须要有 name 属性作为容器内元素的标识。
怎么来得到这样的容器对象呢？我们来看一下 `Project` 的 `container` 方法：

```
    <T> NamedDomainObjectContainer<T> container(Class<T> var1);
    <T> NamedDomainObjectContainer<T> container(Class<T> var1, NamedDomainObjectFactory<T> var2);
    <T> NamedDomainObjectContainer<T> container(Class<T> var1, Closure var2);
```

下面通过实例来介绍一下。

### 实例介绍

首先创建一个 `BuildConfig` 类，上面说了，这个类必须有个 `name` 属性：

```
public class BuildConfig {
    final String name
    public boolean debugMode
    String applicationId

    BuildConfig(String name) {
        this.name = name
        println "BuildConfig name = "+name
    }
}
```

然后创建一个容器并将容器添加为 extension：

```
    public MyExtension(Project project) {
        this.extensions.create("defaultConfig", DefaultConfig, "defaultConfig")

        // 创建一个容器
        NamedDomainObjectContainer<BuildConfig> buildConfigs = project.container(BuildConfig)
        // 将容器添加为 extension
        this.extensions.add("buildConfigs", buildConfigs)
    }
```

配置：

```
myExtension {
    message "Hello Plugin"
    isDebug true

    defaultConfig {
        applicationId "com.android.hq.test"
        minSdkVersion 23
        targetSdkVersion 23

        println "print in defaultConfig"
    }

    buildConfigs {
        debug {
            debugMode = true
            applicationId = "com.android.hq.test"
        }
        release {
            debugMode = false
            applicationId = "com.android.hq.test"
        }
        demo {
            debugMode = true
            applicationId = "com.android.hq.test"
        }

    }
}
```

在这里配置赋值时必须要用 `=` 号，否则会报错，没搞清楚原因。

<!--  
http://www.yiibai.com/gradle/gradle_plugins.html
https://www.jianshu.com/p/3c59eded8155
http://wiki.jikexueyuan.com/project/deep-android-gradle/four-four.html
-->

