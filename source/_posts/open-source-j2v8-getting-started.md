---
title: J2V8 -- 开始使用J2V8
categories: V8
comments: true
tags: [V8, J2V8]
description: 介绍 J2V8 的基本用法，如何在 Android 中使用 J2V8，如何通过 Java 来调用 JavaScript 的对象和方法。
date: 2017-08-07 10:00:00
---

本文译自[Getting Started With J2V8](https://eclipsesource.com/blogs/tutorials/getting-started-with-j2v8/)，并加入了自己的一些理解。

## 概述

J2V8 是对 Google 的目前非常流行的 JavaScript 引擎 V8 的 Java 封装，J2V8 的开发使 Android 高效执行 JavaScript 带来了可能。[Tabris.js](https://tabrisjs.com/) 就是基于 J2V8 开发的一款移动端 App。J2V8 可以运行在 Windows，Linux 以及 Mac OS 上面。这篇入门教程主要来介绍如何使用J2V8来在Android上面运行JavaScript脚本。
[Github源码地址](https://github.com/eclipsesource/J2V8)
J2V8 非常注重性能和内存的消耗，为了达到这些目的在设计的时候做了很大的努力。
如果一段 Javascript 代码的执行结果是一个32位整数，那么它可以直接作为一个原始类型被访问，而不必转化为一个包装类的实例。这对于64位的浮点数（doubles）和布尔类型的数据来说同样是适用的。
J2V8 使用了一套“懒加载”技术，也就是说，一个 JavaScript 只有在它被需要使用的时候才会通过 JNI 复制到 Java 中，如果 Javascript 返回了一个大型的数组，这个数组的内容直到数组中的元素的被需要的时候才会被加载到 Java 中。
J2V8 仅仅是对 V8 的 Java 封装并暴露出来一系列的接口供 Java 调用，V8 引擎是用 C++ 写的，为了使用 V8，就需要使用JNI来进行调用。C++ 中需要开发者进行内存管理，有申请就需要有对应的释放。V8 的垃圾回收器会帮助我们做一些工作，但是一些 Native 的一些句柄对象如果不再使用的时候仍然需要我们调用 `release()` 去释放它们。
如果有内存泄漏，当程序运行结束时会有一些报告，比如会有一些打印和异常抛出。

## J2V8的使用

J2V8目前支持的平台

 - j2v8
 - j2v8_android
 - j2v8_android_armv7l
 - j2v8_android_x86
 - j2v8_linux_x86_64
 - j2v8_macosx_x86_64
 - j2v8_win32_x86
 - j2v8_win32_x86_64

仅介绍在 Android 环境下的使用。

### 添加依赖

在build.gradle中添加依赖：
```
dependencies {
    compile 'com.eclipsesource.j2v8:j2v8:4.5.0@aar'
}
```

### Hello World

运行下面一段 Hello World 代码，这段脚本将两个字符串连接起来并且返回了结果字符串的长度：

```javascript
 var hello = 'hello, ';
 var world = 'world!';
hello.concat(world).length;
```

要使用J2V8，首先你必须创建一个运行时环境，J2V8为此提供了一个静态工厂方法。在创建一个运行时环境时，同时也会加载J2V8的本地库。

```java
        V8 runtime = V8.createV8Runtime();
        int result = runtime.executeIntegerScript(""
                + "var hello = 'hello, ';\n"
                + "var world = 'world!';\n"
                + "hello.concat(world).length;\n");
        Log.e("Test","JS result = "+result);
        runtime.release();
```

会打印：

```
JS result = 13
```

为了执行脚本，它提供了多个基于不同返回值的执行方法。在这个例子里，我们使用了 `executeIntegerScript()` 这个方法，因为脚本执行的结果是一个int类型的整数，并且不需要任何的类型转换和包装。当应用结束时，运行时环境必须被释放。
稍微改动一下：

```java
        V8 runtime = V8.createV8Runtime();
        String result = runtime.executeStringScript(""
                + "var hello = 'hello, ';\n"
                + "var world = 'world!';\n"
                + "hello.concat(world);\n");
        Log.e("Test","JS result = "+result);
        runtime.release();
```

会打印：

```
JS result = hello, world!
```

这里我们使用 `executeStringScript()` 方法返回的是一个 `String` 的对象。

## 获取Javascript对象

使用J2V8你可以从Java中获取javascript对象的句柄，下面用一段代码来演示：

```java
    private void testJ2V8(){
        V8 runtime = V8.createV8Runtime();
        runtime.executeVoidScript(""
                + "var person = {};\n"
                + "var hockeyTeam = {name : 'WolfPack'};\n"
                + "person.first = 'Ian';\n"
                + "person['last'] = 'Bull';\n"
                + "person.hockeyTeam = hockeyTeam;\n");

        V8Object person = runtime.getObject("person");
        V8Object hockeyTeam = person.getObject("hockeyTeam");
        Log.e("Test"," JS result name = "+hockeyTeam.getString("name"));
        person.release();
        hockeyTeam.release();

        runtime.release();
```

结果：

```
JS result name = WolfPack
```

因为 `V8Object` 是底层 Javascript 对象的引用，那么我们也可以对这个对象进行操作，比如现在为Javascript增加新的属性，比如 `hockeyTeam.add("captain", person); ` 在进行了这一步操作之后，新添加的属性 `captain` 可以在Javascript中立刻被访问到。以下代码可以验证这一点：

```java
        hockeyTeam.add("captain", person);
        Log.e("Test"," JS result  "+runtime.executeBooleanScript("person === hockeyTeam.captain"));
```

结果：

```
JS result  true
```

## V8Array

`V8Array` 继承自 `V8Object`，因此提供了相同的存取器方法（accessor / mutator methods，相当于setter / getter方法）。除此之外，`V8Array` 的元素也可以通过索引来进行访问。`V8Object` 和 `V8Array` 都遵循了流式编程模型 ，这使得创建新的JavaScript对象变得非常简单。

```java
        V8Object player1 = new V8Object(runtime).add("name", "John");
        V8Object player2 = new V8Object(runtime).add("name", "Chris");
        V8Array players = new V8Array(runtime).push(player1).push(player2);
        hockeyTeam.add("players", players);
        player1.release();
        player2.release();
        players.release();
```

## 调用JavaScript函数

除了执行JavaScript脚本，也可以使用J2V8来调用JavaScript函数。既可以返回一个结果，也可以没有返回值。请看以下javascript函数：

```javascript
var hockeyTeam = {
     name      : 'WolfPack',
     players   : [],
     addPlayer : function(player) {
                   this.players.push(player);
                   return this.players.size;
     }
}
```

为了在Java中调用上述对象中的函数，我们仅仅只需要一个 `hockeyTeam` 的句柄。通过这个对象句柄，我们可以向执行脚本一样执行函数。不同与脚本的是，可以传递给函数一个 `V8Array` 作为它的参数。

```java
        V8 runtime = V8.createV8Runtime();
        runtime.executeVoidScript(""
                + "var hockeyTeam = {\n"
                + "name      : 'WolfPack',\n"
                + "players   : [],\n"
                + "addPlayer : function(player) {\n"
                + "              this.players.push(player);\n"
                + "              return this.players.length;\n"
                + "}\n"
                + "}\n");

        V8Object hockeyTeam = runtime.getObject("hockeyTeam");
        V8Object player1 = new V8Object(runtime).add("name", "John");
        V8Array parameters = new V8Array(runtime).push(player1);
        int size = hockeyTeam.executeIntegerFunction("addPlayer", parameters);
        Log.e("Test", "JS result size = "+size);
        parameters.release();
        player1.release();
        hockeyTeam.release();
        runtime.release();
```

结果：

```
JS result size = 1
```

参数数组的元素被映射为 JavaSscrip t函数的参数。参数数组元素的数量和在函数中声明的参数的数量不必相符，`undefined` 会被作为默认的值。
