---
title: Gradle 使用指南 -- Gradle Task
categories: 开发工具
comments: true
tags: [Android, Gradle]
description: 介绍 Gradle 中 Task 的使用
date: 2016-3-13 10:00:00
---

## 概述

[Gradle 官方文档](https://docs.gradle.org/current/userguide/userguide.html)
[Gradle User Guide 中文版](http://wiki.jikexueyuan.com/project/GradleUserGuide-Wiki/)
[Task的API文档](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html)

Gradle 中的每一个 Project 都是由一个或者多个 Task 来构成的，它是 gradle 构建脚本的最小运行单元，一个 Task 代表一些更加细化的构建，可能是编译一些 classes、创建一个 Jar、生成 javadoc、或者生成某个目录的压缩文件。

## Task 相关命令

 - `./gradlew tasks`：列出当前工程的所有Task
 - `./gradlew [-q] <task name>`：单独执行某个task，-q 代表 quite 模式，它不会生成 Gradle 的日志信息 (log messages)，所以用户只能看到 tasks 的输出。

## 创建任务

```
task hello {
    doFirst {
        println 'task hello doFirst'
    }
    doLast {
        println 'task hello doLast'
    }
    println 'config task hello'
}

task hello1 << {
    println 'task hello1'
}

task hello2 (type: Copy){
    from 'src/main/AndroidManifest.xml'
    into 'build/test'
}
```

我们执行 `./gradlew -q hello`，会有下面的输出：

```
config task hello
task hello doFirst
task hello doLast
```

这里会有个奇怪的现象，我们在执行 `./gradlew -q hello` 时有上面三条输出是容易理解的，但是在执行 `./gradlew tasks` 或者是执行其他 Task 比如 `./gradlew -q hello1` 时，`config task hello` 这个打印也会输出，为什么呢？
这是因为 Task 有两种状态，分别是：

 - 配置状态（Configuration State）
 - 执行状态（Execution State）

Gradle 会在进入执行之前，配置所有 Task，而 `println 'config task hello'` 这段代码就是在配置时进行执行。所以哪怕没有显式调用 `gradlew hello`，只是调用列出所有 task 的命令，hello task 仍然需要进入到配置状态，也就仍然执行了一遍。
hello1 task的声明方式 << 只是简写的 `doLast`，或者说当这个任务不需要任何在配置状态下运行的内容时，这两种声明方式是一样的。
实际上大部分时候 task 都应该是在执行状态下才真正执行的，配置状态大部分时候用于声明执行时需要用到的变量等为执行服务的前置动作。
hello2: Task创建的时候可以通过 `type: SomeType` 指定Type，Type其实就是告诉Gradle，这个新建的Task对象会从哪个基类Task派生。比如，Gradle本身提供了一些通用的Task，最常见的有CopyC、Delete、Sync、Tar等任务。Copy是Gradle中的一个类。当我们：task myTask(type:Copy)的时候，创建的Task就是一个Copy Task。下面会详细介绍。

### 动态任务

```
4.times { counter ->
    task "task$counter" << {
        println "I'm task number $counter"
    }
}

```
这里动态的创建了 task0，task1，task2，task3。
执行 `./gradlew -q task1`，输出：

```
I'm task number 1
```

## 调用任务

### 短标记法

可以使用短标记 $ 可以访问一个存在的任务：

```
task hello << {
    println 'Hello world!'
}
hello.doLast {
    println "Greetings from the $hello.name task."
}
```

执行 `gradle -q hello` 输出：

```
Hello world!
Greetings from the hello task.
```

## 任务依赖

### 创建依赖

在Gradle中，Task之间是可以存在依赖关系的。这种关系可以通过 `dependsOn` 来实现。

```
task A << {
    println 'Hello from A'
}
task B << {
    println 'Hello from B'
}
B.dependsOn A
```
或者
```java
task A << {
    println 'Hello from A'
}
task B (dependsOn: A) << {
    println 'Hello from B'
}
```

执行 `./gradlew -q B` 的同时会先去执行任务 A：

```
Hello from A
Hello from B
```

下面的代码同样能创建有依赖关系的任务：

```
task A << {
    println 'Hello from A'
}
task B {
    dependsOn A
    doLast {
        println 'Hello from B'
    }
}
```

### 插入依赖

也可以在已经存在的 task 依赖中插入我们的 task 。
比如前面的A和B已经存在了依赖关系，我们想在中间插入任务B1，可以通过下面的代码实现：

```
B1.dependsOn A
B.dependsOn B1
```

输出结果：

```
Hello from A
Hello from B1
Hello from B
```

## 任务排序

### mustRunAfter 与 shouldRunAfter

在某些情况下，我们希望能控制任务的的执行顺序，这种控制并不是向上面那样去显示地加入依赖关系。我们可以通过 `mustRunAfter` 和 `shouldRunAfter` 来实现。
使用 `mustRunAfter` 意味着 taskB 必须总是在 taskA 之后运行，`shouldRunAfter` 和 `mustRunAfter` 很像，只是没有这么严格。

```
task A << {
    println 'Hello from A'
}
task B << {
    println 'Hello from B'
}
A.mustRunAfter B
```

执行 `./gradlew -q A B`，结果：

```
Hello from B
Hello from A
```

如果换成 `shouldRunAfter`，结果也是一样的。
虽然我们将两个任务进行了排序，但是他们仍然是可以单独执行的，任务排序不影响任务执行。排序规则只有当两个任务同时执行时才会被应用。
比如执行 `./gradlew -q A` 会输出：

```
Hello from A
```

另外，`shouldRunAfter` 不影响任务之间的执行依赖。但如果 `mustRunAfter` 和任务依赖之间发生了冲突，那么执行时将会报错。

### finalizedBy

假如出现了这样一种使用场景，执行完任务 A 之后必须要执行一下任务 B，那么上面的方法是无法解决这个问题的，这时 `finalizedBy` 就派上用场了。

```
task A << {
    println 'Hello from A'
}
task B << {
    println 'Hello from B'
}
A.finalizedBy B
```

执行 `./gradlew -q A`，结果：

```
Hello from A
Hello from B
```

## Task Type

### Copy

[Copy的API文档](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.Copy.html)

```
task hello2 (type: Copy){
    from 'src/main/AndroidManifest.xml'
    into 'build/test'
    rename {String fileName ->
        fileName = "AndroidManifestCopy.xml"
    }
}
```
其他 Type 具体详见文档，此处不详细解释。

## 参考文档

http://wiki.jikexueyuan.com/project/GradleUserGuide-Wiki/
https://docs.gradle.org/current/userguide/userguide.html
http://www.jianshu.com/p/cd1a78dc8346
