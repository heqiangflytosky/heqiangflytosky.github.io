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
[官方文档：Authoring Tasks](https://docs.gradle.org/current/userguide/more_about_tasks.html#header)
[官方文档：Writing Custom Gradle Tasks](https://guides.gradle.org/writing-gradle-tasks/)

Gradle 中的每一个 Project 都是由一个或者多个 Task 来构成的，它是 gradle 构建脚本的最小运行单元，一个 Task 代表一些更加细化的构建，可能是编译一些 classes、创建一个 Jar、生成 javadoc、或者生成某个目录的压缩文件。
Task 有一些重要的功能：任务动作(task action)和任务依赖(task dependency)。task action定义了任务执行时最小的工作单元，比如 doFirst和doLast。task dependency定义了task之间的依赖关系，例如在某个task运行之前要运行另外一个task，尤其是需要另一个task的输出作为输入的时候。


## Task 相关命令

 - `./gradlew tasks`：列出当前工程的所有Task
 - `./gradlew [-q] <task name>`：单独执行某个task，-q 代表 quite 模式，它不会生成 Gradle 的日志信息 (log messages)，所以用户只能看到 tasks 的输出。

## 创建任务

默认情况下，我们创建的每一个 Task 都是 org.gradle.api.DefaultTask 类型，这是一个通用的 Task 类型。另外 Gradle 还提供了具有一些特定功能的 Task，比如 Copy 和 Delete 等，我们需要时直接继承即可。另外，我们还可以创建自己的 Task 类型，并且可以自行定义我们创建的 Task 类型。
Task 的定义和构造方式也是多种多样的，Gradle 提供了多种方法来定义 Task。另外 Task 既可以在 build.gradle 文件中直接创建，也可以由不同的 Plugin 引入。 

### Task 构造方法

可以通过下面几个方法来构造 Task：

 - task myTask：用 task 关键字构造
 - task()：用 project 的 task 方法构造
 - tasks.create：用 TaskContainer 的 create 方法构造
 - myTask extends DefaultTask 

### Task 示例

#### 代码示例


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

task hello3(group: "myTest", description:"This is test task paras", dependsOn: ["A", "B"]){
    println 'task hello3'
}

task("hello4", group: "myTest", description:"This is test task para name"){
    println 'task hello4'
}

tasks.create("hello5") {
    group = 'Welcome'
    description = 'Produces a greeting'

    doLast {
        println 'Hello, World'
    }
}
```

执行 `./gradlew  tasks --all`，上面的 Task 就会在列表中显示出来。

```
MyTest tasks
------------
hello3 - This is test task paras
hello4 - This is test task para name

Other tasks
-----------
...
hello
hello1
hello2

Welcome tasks
-------------
hello5 - Produces a greeting

```

我们执行 `./gradlew -q hello`，会有下面的输出：

```
config task hello
task hello doFirst
task hello doLast
```

hello1 task的声明方式 << 只是简写的 `doLast`，或者说当这个任务不需要任何在配置状态下运行的内容时，这两种声明方式是一样的。doLast还有一个等价的操作leftShift，leftShift还可以缩写为<<。<< 操作符在Gradle 5.0已经废弃了。
实际上大部分时候 task 都应该是在执行状态下才真正执行的，配置状态大部分时候用于声明执行时需要用到的变量等为执行服务的前置动作。
hello2: Task创建的时候可以通过 `type: SomeType` 指定Type，Type其实就是告诉Gradle，这个新建的Task对象会从哪个基类Task派生。比如，Gradle本身提供了一些通用的Task，最常见的有CopyC、Delete、Sync、Tar等任务。Copy是Gradle中的一个类。当我们：task myTask(type:Copy)的时候，创建的Task就是一个Copy Task。下面会详细介绍。Gradle 本身提供了很多已经定义好的这些 task 来供我们使用，我们使用时直接继承就行了，这个后面会有介绍。当然我们也可以自定义一些这样的 Task 来供其他 Task 来继承。下面也会详细介绍。

#### Task 的一些构造方法

Task 提供了下面几个构造方法：

 - task myTask
 - task myTask { configure closure }
 - task myTask(type: SomeType)
 - task myTask(type: SomeType) { configure closure }

Project 提供了下面几个 task 方法来创建 task：

 - task(name)
 - task(name, configureClosure)
 - task(name, configureAction)
 - task(args, name)
 - task(args, name, configureClosure)

TaskContainer 提供了下面几个方法来创建 task，TaskContainer 可以通过 Project 的 tasks 属性或者 getTasks() 方法获取：

 - create​(String name)
 - create​(String name, Closure configureClosure)
 - create​(String name, Class<T> type)
 - create​(String name, Class<T> type, Object... constructorArgs)
 - create​(String name, Class<T> type, Action<? super T> configuration)
 - create​(Map<String,​?> options)
 - create​(Map<String,​?> options, Closure configureClosure)

#### TaskContainer create 方法定义 Task

来介绍一下 create 方法的用法：
先来看一段代码：

```
tasks.create("hello") { 
    doLast { 
        println 'Hello, World!'
    }
}
```

上面代码做了两件事情：

 - 创建了一个叫 hello 的 task
 - 在 doLast action 中打印 Hello, World! 到终端

运行 `./gradlew tasks --all` 命令后，会在 Other tasks 的组中发现 hello task。
运行 `./gradlew hello` 输出：

```
> Task :hello
Hello, World!
```

下面我们为 task 添加组信息和task描述信息：

```
tasks.create("hello") {
    group = 'Welcome'
    description = 'Produces a greeting'

    doLast {
        println 'Hello, World'
    }
}
```

为该 task 生成组和描述信息：

```
Welcome tasks
-------------
hello - Produces a greeting
```

#### 实现 TaskAction，定义公共 Task

上面介绍的集中定义 task 的方法，实际真正执行的时候什么都没有做，或者是在 doFirst 和 doLast action 时做了一些事情。
我们还可以通过继承 Task 的方式通过 @TaskAction 操作符也可以指定一个 Task 执行时要做的事情，它区别于doFirst 和 doLast。

```
// 这个类可以放到当前 build.gradle 文件中，也可以放到单独的 gradle 文件中，也可以放到插件的 java 或者 groovy 文件中。
class TestTask extends DefaultTask {
    String source

    @TaskAction
    void testAction() {
        println getName()+" ### testAction!"
    }

    void testMothod(String para) {
        println "Source is "+source+ " para = "+para
    }
}

task testTask (type: TestTask) {
    source "MyApplication"
    testMothod "test"
    doLast {
        println 'GoodBye'
    }
}
```

代码中定义了一个名为 TestTask 的 Task，它继承自 DefaultTask， 通过注解 TaskAction 标记了默认的 Action。@TaskAction表示该Task要执行的动作，即在调用该Task时，要执行的方法。
另外演示了属性和方法的应用。
我们可以定制一些这样的 Task 放到 plugin 中，作为公共的 Task 供开发者调用和继承。在项目中存在大量自定义的 Task 类型时，我们是推荐使用这种方法的。

下面来介绍一下在 plugin 的 java 文件中定义 task：
创建自定义 task：

```
package com.android.hq.testplugin;

import org.gradle.api.DefaultTask;
import org.gradle.api.tasks.TaskAction;


public class TestJavaTask extends DefaultTask {

    @TaskAction
    public void testAction(){
        System.out.println("### This is TestJavaTask");
    }
}
```

定义 task：

```
// 引用插件
apply plugin: 'com.android.hq.testplugin.test'


// 添加插件的依赖路径
buildscript {
    dependencies {
        // 这里直接使用了上面插件工程中生成 jar 包的路径
        classpath files("../../TestGradlePlugin/app/build/libs/app-1.0.0.jar")
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

task testCustom(type: com.android.hq.testplugin.TestJavaTask) {
    doFirst {
        println "Hello"
    }

    doLast {
        println "GoodBye"
    }
}
```

执行 './gradlew testCustom' 后生成信息：

```
> Task :app:testCustom
Hello
### This is TestJavaTask
GoodBye
```

#### 配置 Task 属性

下面我们再来介绍如何进行灵活地进行定制 task 的一些属性，比如下面例子中的要输出的信息。
这里我们通过实现一个 Task 类来实现创建 task。
在 build.gradle 中添加代码：

```
class Greeting extends DefaultTask {
    String message
    String recipient

    @TaskAction
    void sayGreeting() {
        println "${message}, ${recipient}!"
    }
}

tasks.create("hello", Greeting) {
    group = 'Welcome'
    description = 'Produces a world greeting'
    message = 'Hello'
    recipient = 'World'
}

tasks.create("gutenTag", Greeting) {
    group = 'Welcome'
    description = 'Produces a German greeting'
    message = 'Guten Tag'
    recipient = 'Welt'
}
```

运行 `./gradlew tasks` 命令：

```
Welcome tasks
-------------
gutenTag - Produces a German greeting
hello - Produces a world greeting
```

下面几点说明：

 - 虽然 Gradle API 的其他 Task 类我们也可以使用，但继承 DefaultTask 是常见的扩展 Task 的方法。
 - 上面例子中添加了自定义的 message 和 recipient 实现任务类型的可配置，分别实现了不同输出的 hello 和 gutenTag 两个任务。
 - 通过注解 TaskAction 标记了默认的 Action。@TaskAction表示该Task要执行的动作，即在调用该Task时，要执行的 方法。
 - 通过引用 Greeting 实现了不同的任务类。

### Task 的状态

这里会有个奇怪的现象，我们在执行 `./gradlew -q hello` 时有上面三条输出是容易理解的，但是在执行 `./gradlew tasks` 或者是执行其他 Task 比如 `./gradlew -q hello1` 时，`config task hello` 这个打印也会输出，为什么呢？
这是因为 Task 有两种状态，分别是：

 - 配置状态（Configuration State）
 - 执行状态（Execution State）

这其实对应了 Gradle 三个生命周期中的配置阶段和执行阶段，task 的配置块永远在 task 动作执行之前被执行。
Gradle 会在进入执行之前，配置所有 Task，而 `println 'config task hello'` 这段代码就是在配置时进行执行。所以哪怕没有显式调用 `gradlew hello`，只是调用列出所有 task 的命令，hello task 仍然需要进入到配置状态，也就仍然执行了一遍。
很多时候我们并不需要配置代码，我们想要我们的代码在执行 task 的时候才执行，这个时候可以通过 doFirst、doLast或者实现 TaskAction 来完成。

### Task 参数

我们用命令查看 Task 信息时一般是这样的：

```
Build tasks
-----------
assemble - Assembles all variants of all applications and secondary packages.
assembleAndroidTest - Assembles all the Test applications.
assembleDdd - Assembles all Ddd builds.
assembleDebug - Assembles all Debug builds.
assembleRelease - Assembles all Release builds.
```

一般是由 group、name 和 description组成，其实在上面的示例中大家应该看到 hello3、hello4和其他任务的不同了。他们两个处在同一个 group 中并且 Task 名称后面有一些描述信息。
Task的一般的属性有下面几种，可以在创建 Task 的时候在闭包中声明：

| Property | Property |
|:-------------:|:-------------:|
| name  |  task的名字 |
| type | task的“父类” |
| overwrite | 是否替换已经存在的task |
| dependsOn | task依赖的task的集合 |
| group | task属于哪个组 |
| description | task的描述 |


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

### 在Task中调用其他Task

```
task A << {
    C.execute()
}

task C << {
    println 'Hello from C'
}

// 将自定义任务挂接到assemble任务的最后再执行
afterEvaluate { Project project ->
    def t = project.tasks.findByName("assemble")
    t.doLast {
        def testTask = project.tasks.findByName("C")
        testTask.execute()
    }
}
```

上面代码通过 `execute()` 方法来调用 Task C。

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

除了前面我们介绍的 task 定义方式以外，Gradle 本身还提供了一些已有的 task 供我们使用，比如 Copy、Delete、Sync 等。因此我们定义 task 的时候是可以继承已有的 task，比如我们可以继承自系统的 Copy Task 来完成文件的拷贝操作。

### Copy

[Copy的API文档](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.Copy.html)

```
task hello2 (type: Copy){
    from 'src/main/AndroidManifest.xml'  // 调用 from 方法
    into 'build/test'  // 调用 into 方法
    // 调用 rename 方法
    rename {String fileName ->
        fileName = "AndroidManifestCopy.xml"
    }
}
```
Copy Task的其他属性和方法参考源码或者文档。

其他 Type 具体详见文档，此处不详细解释。

### Exec

[Exec Task](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.Exec.html)用来执行命令行。

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
