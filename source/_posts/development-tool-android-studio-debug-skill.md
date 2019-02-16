---
title: Android Studio 使用技巧
categories: 开发工具
comments: true
keywords: Android Studio
tags: [Android Studio, 开发工具]
description: 介绍 Android Studio 的使用技巧：调试技巧，设置技巧，常用插件等
date: 2016-1-30 10:00:00
---

作为一名 Android 开发者，使用 Android Studio 作为 IDE 进行开发已经成为标配，本文就把 AS 的一些使用技巧来做一下归纳和总结。随着 AS  的版本升级，本文会同步更新。

## 调试技巧

在我们写代码或者是 Debug 过程中，各种调试手段必不可少，Android Studio 提供了强大的调试能力。

### 启动调试

启动调试方法我相信大家都会，一个是以Debug模式启动App，APP启动后，运行至第一处断点处会停下来，断点可以在运行之前设置，也可在运行后设置。一个是使一个已经启动的进程进入调试模式。这两种方式可以通过下面工具栏的的两个按钮启动。
![效果图](/images/development-tool-android-studio-debug-skill/as-start-debug.png)

### 基本调试方法

先来看一下 AS 的调试面板：
![效果图](/images/development-tool-android-studio-debug-skill/as-debug-panel.png)

 - 调试功能键：控制断点的执行；
 - 断点管理键：控制断点调试的运行；
 - 求值表达式：计算表达式的值；
 - 变量观察区：显示当前可以观察到的对象及其属性值；
 - 指定变量观察区：观察指定变量或者表达式的变化；
 - 当前线程堆栈区：（点击Restore ‘Threads’ View 按钮）查看当前断点所在的线程以及调用堆栈；

#### 调试功能键

 - Show Execution Point：点击该按钮，光标将定位到当前正在调试的位置。
 - Run to Cursor：程序执行到当前光标所在的位置。
 - Force Run to Cursor：程序执行到当前光标所在的位置，可以忽视已经存在的断点。注意和 Run to Cursor 的区别。如果我们在代码中添加了很多的断点，现在不想每个断点都调试，只想让程序一次到位运行到指定的位置，这个是非常有用的功能。
 - Get thread dump：获取线程Dump，点击该按钮将进入线程Dump界面。
  
#### 断点管理键

 - Restore Layout：重置调试窗口布局。
 - View Breakpoints：可以查看所有断点，管理或者配置断点的行为。
 - Mute Breakpoints：中途切换所有断点的状态。可以临时取消所有断点，不可用的时候断点是白色的。
 - Drop Frame：这个还不知道怎么用。
 - Show Values InLine：这个在 Settings 按钮列表中，可以在调试过程中代码右边显示变量值。
 - Show Method Return Values：这个在 Settings 按钮列表中，可以把调试过程中在对象变量区将带返回值方法的返回值显示出来。

#### 求值表达式

求值表达式的窗口如图所示：
![效果图](/images/development-tool-android-studio-debug-skill/as-evaluate-expression-window.png)
调用这个窗口有两种方式：

 1. 通过调试面板的**求值表达式**按钮；
 2. 选中需要调试的表达式，点击右键，选中“Evaluate Expression...”；

注意这里有个 Code Fragment Mode，这个是支持代码片段，可以输入多行表达式，返回最后结果。

#### 变量观察区

把一个变量或者表达式添加到变量观察区也有两个方法：

 1. 点击**变量观察区**的 + 号；
 2. 选中需要调试的表达式，点击右键，选中“Add to Watches”；

#### 修改变量值

在调试过程中，我们可以方便的修改某个变量的值，以方便我们的调试。
在**Variables**窗口中选中改变量，右键点击，然后选择“Set Value...”就可以修改变量值。

![效果图](/images/development-tool-android-studio-debug-skill/as-variables-set-value.png)

上图中变量 `a` 的值是2，现在我们想观察一下 `a` 为 20 的执行结果，我们不必在代码中修改然后编译运行程序这种方法来进行，用上图的方法和轻松的可以达到目的。

### 断点调试

基本的断点调试方法这里就不多介绍，先介绍一下观察变量值的方法。

#### 方法断点

在方法名所在行断点位置单击鼠标会生成一个方法断点，当执行到这个方法时使程序停止。
这种断点在当前的代码和运行代码不匹配是非常有用，可以执行到特定的方法。

![效果图](/images/development-tool-android-studio-debug-skill/as-function-break-point.png)

#### 条件断点

所谓的条件断点就是在特定条件发生的断点，也就是说，我们可将某个断点设置为只对某种事件感兴趣。
断点处右键单击，在弹框的Condition处填写过滤条件。此处我们只关心 `i==2` 的情况，因此填写`i==2`。那么代码在变量 `i` 为 2 时才会停下来。

![效果图](/images/development-tool-android-studio-debug-skill/as-condition-break-point.png)

#### 日志断点

该类型的断点不会使程序停下来，而是在输出我们要它输出的日志信息，然后继续执行。
步骤如下：
在断点出右键单击，在弹出的对话框中取消选中**Suspend**，在弹出的控制面板中，选中 **Log evaluated** 和 **Condition**，然后填写断点条件以及输出日志信息。 
如下图：

![效果图](/images/development-tool-android-studio-debug-skill/as-log-break-point.png)

在 `i` 为 2 时才会输出 `Test Log Break Point` 这条日志。

#### 异常断点

异常断点顾名思义就是在调试过程中，一旦发生异常（可以指定某类异常），则会立刻定位到异常抛出的地方。
通过这类断点，我们可以在调试异常中，可以进行运行异常时及时定位及调试。
步骤如下：
点击**View Breakpoints**，在窗口中点击 + 号，选择 Java Exception Breakpoints，

![效果图](/images/development-tool-android-studio-debug-skill/as-crash-break-point-1.png)

在输入框中输入我们关注的异常类型。

![效果图](/images/development-tool-android-studio-debug-skill/as-crash-break-point-2.png)

在调试过程一旦发生该异常，调试器就会定位到异常发生处。

![效果图](/images/development-tool-android-studio-debug-skill/as-crash-break-point-3.png)

#### 属性断点

我们可以为一个类的属性设置断点，那么这个断点就叫属性断点，当这个属性值被修改的时候，程序暂停在修改处。
属性断点的使用和添加普通的断点并无不同，只是断点图标稍有不同。

![效果图](/images/development-tool-android-studio-debug-skill/as-value-break-point.png)

<!--  
https://www.jianshu.com/p/f695b8f8839c
https://www.jianshu.com/p/f19ee61126ef
https://www.jianshu.com/p/011eb88f4e0d/
-->

## 设置技巧

http://blog.csdn.net/heqiangflytosky/article/details/51140346

## 常用插件

http://blog.csdn.net/heqiangflytosky/article/details/50764020

 - jclasslib bytecode viewer：查看 Java 类字节码的插件。安装成功后，选中类文件，在View->Show Bytecode With jclasslib
 - database navigator：查看数据库插件
