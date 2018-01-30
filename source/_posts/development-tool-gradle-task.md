---
title: Gradle 使用指南 -- Gradle Task
categories: Gradle
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

## 任务的执行条件

### 使用判断条件

可以使用 `onlyIf()` 方法来为一个任务加入判断条件。就和 Java 里的 `if` 语句一样，任务只有在条件判断为真时才会执行。可以通过一个闭包来实现判断条件。

```
task A << {
    println 'Hello from A'
}
A.onlyIf{!project.hasProperty('skipA')}
```

执行 "./gradlew A -PskipA"，输出：

```
:app:A SKIPPED
```

可以看到 A 任务被跳过。

### 使用 StopExecutionException

如果想要跳过一个任务的逻辑并不能被判断条件通过表达式表达出来，那么可以使用 `StopExecutionException`。如果这个异常是被一个任务要执行的动作抛出的，这个动作之后的执行以及所有紧跟它的动作都会被跳过。构建将会继续执行下一个任务。

```
task hello {
    doFirst {
        println 'task hello doFirst'
        throw new StopExecutionException()
    }
    doLast {
        println 'task hello doLast'
    }
}
```

如果你直接使用 Gradle 提供的任务，这项功能还是十分有用的。它允许你为内建的任务加入条件来控制执行。

### 激活和注销任务

每一个任务都有一个已经激活的标记(enabled flag)，这个标记一般默认为真。 将它设置为假，那它的任何动作都不会被执行。

```
task A << {
    println 'Hello from A'
}
A.enabled = false
```

执行 "./gradlew A"，输出：

```
:app:A SKIPPED
```

## 声明任务的输入和输出

我们在执行 Gradle 任务的时候，你可能会注意到 Gradle 会跳过一些任务，这些任务后面会标注 up-to-date。代表这个任务已经运行过了或者说是最新的状态，不再需要产生一次相同的输出。
Gradle 通过比较两次 build 之间输入和输出有没有变化来确定这个任务是否是最新的，如果从上一个执行之后这个任务的输入和输出没有发生改变这个任务就标记为 up-to-date，跳过这个任务。
因此，要想跳过 up-to-date 的任务，我们必须为任务指定输入和输出。
任务的输入属性是 [TaskInputs](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/TaskInputs.html) 类型. 任务的输出属性是 [TaskOutputs](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/TaskOutputs.html) 类型.
下面的例子中把上面的 `Copy` 示例中的输入和输出文件作为 `hello` task 的输入和输出文件：

```
task hello {
    inputs.file ("src/main/AndroidManifest.xml")
    outputs.file ("build/test/AndroidManifestCopy.xml")
    doFirst {
        println 'task hello doFirst'
    }
}
```

第一次执行 `./gradlew hello`输出：

```
:app:hello
task hello doFirst
```

可以看到任务正常的执行。
然后进行第二次执行，输出：

```
:app:hello UP-TO-DATE
```

跳过这个任务。可以看到，Gradle 能够检测出任务是否是 up-to-date 状态.
如果我们修改一下 `src/main/AndroidManifest.xml` 文件，输入上面命令就会再次执行该任务。


### UP-TO-DATE 原理

当一个任务是首次执行时，Gradle 会取一个输入的快照 (snapshot)。该快照包含组输入文件和每个文件的内容的散列。然后当 Gradle 执行任务时，如果任务成功完成，Gradle 会获得一个输出的快照。该快照包含输出文件和每个文件的内容的散列。Gradle 会保留这两个快照用来在该任务的下一次执行时进行判断。
之后，每次在任务执行之前，Gradle 都会为输入和输出取一个新的快照，如果这个快照和之前的快照一样，Gradle 就会假定这个任务已经是最新的 (up-to-date) 并且跳过任务，反之亦然。
**需要注意的是**，如果一个任务有指定的输出目录，自从该任务上次执行以来被加入到该目录的任务文件都会被忽略，并且不会引起任务过时 (out of date)。这是因为不相关任务也许会共用同一个输出目录。如果这并不是你所想要的情况，可以考虑使用 `TaskOutputs.upToDateWhen()`。


## 参考文档

http://wiki.jikexueyuan.com/project/GradleUserGuide-Wiki/
https://docs.gradle.org/current/userguide/userguide.html
http://www.jianshu.com/p/cd1a78dc8346
