---
title: Gradle 使用指南 -- Extra Properties 
categories: Gradle
comments: true
tags: [Android, Gradle]
description: 介绍 Gradle 扩展属性的使用
date: 2016-3-15 10:00:00
---

## 概述

Gradle 提供了一种名为 extra property 的方法。这就使我们扩展一些自定义属性成为可能。
可以查看[Project 官方文档](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html)中对 Extra Properties 的介绍。
[ExtraPropertiesExtension 官方文档](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtraPropertiesExtension.html)
所有的扩展属性都必须通过 `ext` 命名空间来定义，定义完后就可以直接通过对象来引用。ext 属性支持 Project、Gradle 对象。扩展属性是可读写的。扩展属性，使得自定义属性的跨脚本传递成为可能。

## 扩展属性的使用

### 在 Gradle 对象中使用扩展属性

比如我们在 settings.gradle 中定义扩展属性：

```
gradle.ext.testGradleExt=10
```

那么就可以在 build.gradle 中引用：

```
    println gradle.testGradleExt
```

### 在 Project 对象中使用扩展属性

比如我们在 root project 中的build.gradle 中定义扩展属性：

```
ext {
    testProjectExt1 = 20
    testProjectExt2 = "testProjectExt2"
}
```

可以在 sub project 中的 build.gradle 中引用：

```
    println rootProject.testProjectExt2
```

当然你的定义方式还可以是这样：

```
ext.testProjectExt3 = "testProjectExt3"
```

## 扩展属性的多种定义方式

除了上面的定义方式，你还可以这样来定义扩展属性：

### 1

在 sub project 中定义 root project 的属性，

```
rootProject.ext {
    testProjectExt4 = "testProjectExt4"
}
```

然后在 root project 中引用：

```
task test << {
    println this.testProjectExt4
}
```

至于这里为什么要放在 task 的执行阶段来使用这个扩展属性，相信大家了解了 Gradle  的生命周期的同学都会知道的，如果放在 配置阶段去执行，会报错的。

### 2

还可以这样定义：

```
ext.set("testProjectExt5", 5)
```

### 3

还可以这样引用：

```
println rootProject.ext.get("testProjectExt5")
```

还可以这样引用

```
    println rootProject.ext["testProjectExt5"]
```

### 4

可以这样去更新一个扩展属性的值：

```
rootProject.testProjectExt5 = 55
```

也可以这样：

```
rootProject.ext["testProjectExt5"] = 66
```

## 扩展属性的一些属性和方法

可以参考[ExtraPropertiesExtension 官方文档](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtraPropertiesExtension.html)

### properties

`properties` 属性返回一个 `Map<String, Object>` 对象，存储了当前对象定义的所有扩展属性。

```
    println rootProject.properties.each { key, value ->
        println key
    }
```

```
    println rootProject.properties.containsKey("testProjectExt5")
```

### has

返回当前对象是否包含给定的属性：

```
    println rootProject.ext.has("testProjectExt5")
    println rootProject.hasProperty("testProjectExt5")
```

### get，set

设置和取值的两个方法：
get

```
 project.ext { foo = "bar" }

 println project.ext.get("foo") == "bar"
 println project.ext.foo == "bar"
 println project.ext["foo"] == "bar"

 println project.foo == "bar"
 println project["foo"] == "bar"
```

set

```
 project.ext.set("foo", "bar")
 project.ext.foo = "bar"
 project.ext["foo"] = "bar"

 // Once the property has been created via the extension, it can be changed by the owner.
 project.foo = "bar"
 project["foo"] = "bar"
```

## 命令行自定义扩展属性

在 [Gradle 使用指南 -- 基础配置](http://www.heqiangfly.com/2016/03/03/development-tool-gradle-command-config/) 一文中我们也介绍了如何在命令行自定义扩展属性。

```
./gradlew assembleDebug -Pcustom=true
```

在脚本中使用：

```
if (project.hasProperty('custom')){

}
```

<!--  
https://blog.csdn.net/zxc123e/article/details/72846762
https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtraPropertiesExtension.html
-->
