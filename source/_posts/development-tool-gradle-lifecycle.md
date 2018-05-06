---
title: Gradle 使用指南 -- Gradle 生命周期
categories: Gradle
comments: true
tags: [Android, Gradle]
description: 介绍 Gradle 生命周期
date: 2016-3-18 10:00:00
---


## 生命周期概述

无论什么时候执行 Gradle 构建，都会运行三个不同的生命周期阶段：

 1. 初始化阶段
 2. 配置阶段
 3. 执行阶段

在**初始化阶段**，Gradle 为项目创建了 Project 实例。在给定的构建脚本中只定义了一个项目。在多项目构建中，这个构建阶段变得更加重要。根据你正在执行的项目，Gradle 找出哪些项目需要参与到构建中。实质为执行 settings.gradle 脚本。注意，在这个阶段当前已有的构建脚本代码都不会被执行。
在**配置阶段**，Gradle 构造了一个模型来表示任务，并参与到构建中来。增量式构建特性决定来模型中的 task 是否需要运行。配置阶段完成后，整个 build 的 project 以及内部的 Task 关系就确定了。这个阶段非常适合于为项目或指定 task 设置所需的配置。实质为解析每个被加入构建项目的 build.gradle 脚本。注意，项目的每一次构建的任何配置代码都可以被执行--即使你只执行 gradle tasks。
在**执行阶段**，所有的 task 都应该以正确的顺序被执行。执行顺序时由它们的依赖决定的。如果任务被认为没有被修改过，将被跳过。
Gradle 的增量式的构建特性紧紧地与生命周期相结合。
作为一个开发人员，不能仅限于编写在不同构建阶段执行的 task 动作或者配置逻辑。有时候当一个特定的生命周期事件发生时你可能想要执行代码。一个声明周期事件可能发生在某个构建阶段之前、期间或者之后。在执行阶段之后发生的生命周期事件是构建的完成。
我们有两种方式可以编写回调声明周期事件：在闭包中，或者是通过 Gradle API 所提供的监听器接口实现。Gradle 不会引导你采用哪种方式去监听生命周期事件，着完全取决于你的选择。
下面提供一个有用的声明周期钩子（Hook）的用法。

![图片](http://wiki.jikexueyuan.com/project/gradleIn-action/images/dag27.png)

许多生命周期回调方法被定义在 `Gradle` 和 `Projet` 接口中。
不要害怕使用生命周期钩子，它们不是 Gradle API 的秘密后门，相反，它们是特意提供给开发者使用的。


## 生命周期监听方法

如果我们想在 Gradle 特定的阶段去 Hook 指定的任务，那么就需要对如何监听生命周期回调做一些了解。
`Gradle` 和 `Projet` 对象提供了一些方法来供我们设置一些生命周期的回调方法。
生命周期监听的设置有两种方法：

 1. 实现一个特定的监听接口；
 2. 提供一个用于在收到通知时执行的闭包。

上面两个对象对这两种方法的都是支持的。
`Projet` 提供的一些生命周期回调方法：

 - afterEvaluate(closure)，afterEvaluate(action)
 - beforeEvaluate(closure)，beforeEvaluate(action)

`Gradle` 提供的一些生命周期回调方法：

 - afterProject(closure)，afterProject(action)
 - beforeProject(closure)，beforeProject(action)
 - buildFinished(closure)，buildFinished(action)
 - projectsEvaluated(closure)，projectsEvaluated(action)
 - projectsLoaded(closure)，projectsLoaded(action)
 - settingsEvaluated(closure)，settingsEvaluated(action)
 - addBuildListener(buildListener)
 - addListener(listener)
 - addProjectEvaluationListener(listener)


可以看到，每个方法都有两个不同参数的方法，一个接收闭包作为回调，另外一个接受 `Action` 作为回调，下面的介绍时只介绍闭包为参数的方法。
**请注意**：一些声明周期事件只有在适当的位置上声明才会发生。

下面开始介绍 project 的几个方法：

### beforeEvaluate

`beforeEvaluate()`是在 project 开始配置前调用，当前的 project 作为参数传递给闭包。
这个方法很容易误用，你要是直接当前子模块的 build.gradle 中使用是肯定不会调用到的，因为Project都没配置好所以也就没它什么事情，这个代码块的添加只能放在父工程的 build.gradle 中,如此才可以调用的到。

```
this.project.subprojects { sub ->
    sub.beforeEvaluate { project
        println "#### Evaluate before of "+project.path
    }
}
```

如果你想用 `Action` 作为参数的方法：

```
this.project.subprojects { sub ->
    sub.beforeEvaluate(new Action<Project>() {
        @Override
        void execute(Project project) {
            println "#### Evaluate before of "+project.path
        }
    })
}
```

### afterEvaluate

`afterEvaluate` 是一般比较常见的一个配置参数的回调方式，只要 project 配置成功均会调用，不论是在父模块还是子模块。参数类型以及写法与afterEvaluate相同：

```
project.afterEvaluate { pro ->
    println("#### Evaluate after of " + pro.path)
}
```


再来看一下 Gradle 对象的几个回调，可以通过 project 获取当前的 gradle 对象，gradle 设置的回调监控的是所有的 project 实现。

### afterProject

设置一个 project 配置完毕后立即执行的闭包或者回调方法。
afterProject 在配置参数失败后会传入两个参数，前者是当前 project，后者显示失败信息。

```
this.getGradle().afterProject { project,projectState ->
    if(projectState.failure){
        println "Evaluation afterProject of "+project+" FAILED"
    } else {
        println "Evaluation afterProject of "+project+" succeeded"
    }
}
```

### beforeProject

设置一个 project 配置前执行的闭包或者回调方法。
当前 project 作为参数传递给闭包。
子模块的该方法声明在 root project 中回调才会执行，root project 的该方法声明在 settings.gradle 中才会执行。

```
gradle.beforeProject { p ->
    println("Evaluation beforeProject"+p)
}
```

### buildFinished

构建结束时的回调，此时所有的任务都已经执行，一个构建结果的对象  BuildResult 作为参数传递给闭包。

```
gradle.buildFinished { r ->
    println("buildFinished "+r.failure)
}
```

### projectsEvaluated

所有的 project 都配置完成后的回调，此时，所有的project都已经配置完毕，准备开始生成 task 图。gradle 对象会作为参数传递给闭包。

```
gradle.projectsEvaluated {gradle ->
    println("projectsEvaluated")
}
```

### projectsLoaded

当 setting 中的所有project 都创建好时执行闭包回调。gradle 对象会作为参数传递给闭包。
这个方法也比较特殊，只有声明在适当的位置上才会发生，如果将这个声明周期挂接闭包声明在 build.gradle 文件中，那么将不会发生这个事件，因为项目创建发生在初始化阶段。
放在 settings.gradle 中是可以执行的。

```
gradle.projectsLoaded {gradle ->
    println("@@@@@@@ projectsLoaded")
}
```

### settingsEvaluated

当 settings.gradle 加载并配置完毕后执行闭包回调，setting对象已经配置好并且准备开始加载构建 project。
这个回调在 build.gradle 中声明也是不起作用的，在 settings.gradle 中声明是可以的。

```
gradle.settingsEvaluated {
    println("@@@@@@@ settingsEvaluated")
}
```


前面我们说过，设置监听回调还有另外一种方法，通过设置接口监听添加回调来实现。作用的对象均是所有的 project 实现。

### addProjectEvaluationListener

```
gradle.addProjectEvaluationListener(new ProjectEvaluationListener() {
    @Override
    void beforeEvaluate(Project project) {
        println " add project evaluation lister beforeEvaluate,project path is: "+project
    }

    @Override
    void afterEvaluate(Project project, ProjectState state) {
        println " add project evaluation lister afterProject,project path is:"+project
    }
})
```

### addListener

添加一个实现来 listener 接口的对象到 build。

### addBuildListener

添加一个 `BuildListener` 对象到 Build 。

```
gradle.addBuildListener(new BuildListener() {
    @Override
    void buildStarted(Gradle gradle) {
        println("### buildStarted")
    }

    @Override
    void settingsEvaluated(Settings settings) {
        println("### settingsEvaluated")
    }

    @Override
    void projectsLoaded(Gradle gradle) {
        println("### projectsLoaded")
    }

    @Override
    void projectsEvaluated(Gradle gradle) {
        println("### projectsEvaluated")
    }

    @Override
    void buildFinished(BuildResult result) {
        println("### buildFinished")
    }
})
```

## Task 执行图（TaskExecutionGraph）

在配置时，Gradle 决定了在执行阶段要运行的 task 的顺序，他们的依赖关系的内部结构被建模为一个有向无环图，我们可以称之为 taks 执行图，它可以用 `TaskExecutionGraph` 来表示。可以通过 `gradle.taskGraph` 来获取。
在 `TaskExecutionGraph` 中也可以设置一些 Task 生命周期的回调：

 - addTaskExecutionGraphListener(TaskExecutionGraphListener listener)
 - addTaskExecutionListener(TaskExecutionListener listener)
 - afterTask(Action<Task> action)，afterTask(Closure closure)
 - beforeTask(Action<Task> action)，beforeTask(Closure closure)
 - whenReady(Action<TaskExecutionGraph> action)，whenReady(Closure closure)

下面来进行详细介绍。

### addTaskExecutionGraphListener

添加 task 执行图的监听器，当执行图配置好会执行通知。

```
gradle.taskGraph.addTaskExecutionGraphListener(new TaskExecutionGraphListener() {
    @Override
    void graphPopulated(TaskExecutionGraph graph) {
        println("@@@ gradle.taskGraph.graphPopulated ")
    }
})
```

### addTaskExecutionListener

添加 task 执行监听器，当 task 执行前或者执行完毕会执行回调发出通知。

```
gradle.taskGraph.addTaskExecutionListener(new TaskExecutionListener() {
    @Override
    void beforeExecute(Task task) {
        println("@@@ gradle.taskGraph.beforeTask "+task)
    }

    @Override
    void afterExecute(Task task, TaskState state) {
        println("@@@ gradle.taskGraph.afterTask "+task)
    }
})
```

### afterTask

设置一个 task 执行完毕的闭包或者回调方法。
该 task 作为参数传递给闭包。

```
gradle.taskGraph.afterTask { task ->
    println("### gradle.taskGraph.afterTask "+task)
}
```

### beforeTask

设置一个 task 执行前的闭包或者回调方法。
该 task 作为参数传递给闭包。

```
gradle.taskGraph.beforeTask { task ->
    println("### gradle.taskGraph.beforeTask "+task)
}
```

### whenReady

设置一个 task 执行图准备好后的闭包或者回调方法。
该 taskGrahp 作为参数传递给闭包。

```
gradle.taskGraph.whenReady { taskGrahp ->
    println("@@@ gradle.taskGraph.whenReady ")
}
```

## 生命周期顺序

我们通过在生命周期回调中添加打印的方法来看一下他们的执行顺序。
为了看一下配置 task 的时机，我们在 app 模块中创建来一个 taks：

```
task hello {
    doFirst {
        println '*** task hello doFirst'
    }
    doLast {
        println '*** task hello doLast'
    }
    println '*** config task hello'
}
```

为了保证生命周期的各个回调方法都被执行，我们在 settings.gradle 中添加各个回调方法。

```
gradle.addBuildListener(new BuildListener() {
    @Override
    void buildStarted(Gradle gradle) {
        println("### gradle.buildStarted")
    }

    @Override
    void settingsEvaluated(Settings settings) {
        println("### gradle.settingsEvaluated")
    }

    @Override
    void projectsLoaded(Gradle gradle) {
        println("### gradle.projectsLoaded")
    }

    @Override
    void projectsEvaluated(Gradle gradle) {
        println("### gradle.projectsEvaluated")
    }

    @Override
    void buildFinished(BuildResult result) {
        println("### gradle.buildFinished")
    }
})

gradle.afterProject { project,projectState ->
    if(projectState.failure){
        println "### gradld.afterProject "+project+" FAILED"
    } else {
        println "### gradle.afterProject "+project+" succeeded"
    }
}

gradle.beforeProject { p ->
    println("### gradle.beforeProject "+p)
}

gradle.allprojects(new Action<Project>() {
    @Override
    void execute(Project project) {
        project.beforeEvaluate { project
            println "### project.beforeEvaluate "+project
        }
        project.afterEvaluate { pro ->
            println("### project.afterEvaluate " + pro)
        }
    }
})



gradle.taskGraph.addTaskExecutionListener(new TaskExecutionListener() {
    @Override
    void beforeExecute(Task task) {
        if (task.name.equals("hello")){
            println("@@@ gradle.taskGraph.beforeTask "+task)
        }
    }

    @Override
    void afterExecute(Task task, TaskState state) {
        if (task.name.equals("hello")){
            println("@@@ gradle.taskGraph.afterTask "+task)
        }
    }
})

gradle.taskGraph.addTaskExecutionGraphListener(new TaskExecutionGraphListener() {
    @Override
    void graphPopulated(TaskExecutionGraph graph) {
        println("@@@ gradle.taskGraph.graphPopulated ")
    }
})

gradle.taskGraph.whenReady { taskGrahp ->
    println("@@@ gradle.taskGraph.whenReady ")
}
```

执行 task hello：

```
./gradlew hello
### gradle.settingsEvaluated
### gradle.projectsLoaded

> Configure project : 
### gradle.beforeProject root project 'TestSomething'
### project.beforeEvaluate root project 'TestSomething'
### gradle.afterProject root project 'TestSomething' succeeded
### project.afterEvaluate root project 'TestSomething'

> Configure project :app 
### gradle.beforeProject project ':app'
### project.beforeEvaluate project ':app'
*** config task hello
### gradle.afterProject project ':app' succeeded
### project.afterEvaluate project ':app'

> Configure project :common 
### gradle.beforeProject project ':common'
### project.beforeEvaluate project ':common'
### gradle.afterProject project ':common' succeeded
### project.afterEvaluate project ':common'

### gradle.projectsEvaluated
@@@ gradle.taskGraph.graphPopulated 
@@@ gradle.taskGraph.whenReady 

> Task :app:hello 
@@@ gradle.taskGraph.beforeTask task ':app:hello'
*** task hello doFirst
*** task hello doLast
@@@ gradle.taskGraph.afterTask task ':app:hello'

BUILD SUCCESSFUL in 1s
1 actionable task: 1 executed
### gradle.buildFinished
```

因此，生命周期回调的执行顺序是：
gradle.settingsEvaluated->
gradle.projectsLoaded->
gradle.beforeProject->
project.beforeEvaluate->
gradle.afterProject->
project.afterEvaluate->
gradle.projectsEvaluated->
gradle.taskGraph.graphPopulated->
gradle.taskGraph.whenReady->
gradle.buildFinished

<!--  
https://blog.csdn.net/chf1142152101/article/details/72830565
http://baijiahao.baidu.com/s?id=1591357844180592405&wfr=spider&for=pc
https://segmentfault.com/q/1010000004503896/a-1020000004504034
http://wiki.jikexueyuan.com/project/deep-android-gradle/four-four.html
http://wiki.jikexueyuan.com/project/gradleIn-action/build-lifetime.html
http://blog.csdn.net/zxc123e/article/details/72846762

http://www.sohu.com/a/132990618_46

https://www.oreilly.com/library/view/gradle-beyond-the/9781449373801/ch03.html

-->
