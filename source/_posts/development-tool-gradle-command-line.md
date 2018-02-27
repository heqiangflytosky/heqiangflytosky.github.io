---
title: Gradle 使用指南 -- 玩转命令行
categories: Gradle
comments: true
tags: [Android, Gradle]
description: 介绍 Gradle 常用命令的使用
date: 2016-3-26 10:00:00
---

## 概述

Gradle 的命令行是用户了解可用选项、检查项目配置和控制执行的有力工具。它可以分为三类：探索和帮助相关 task、构建设置相关 task 和配置输入。
Gradle 命令的用法：

```
gradle [options...] [tasks...]
```

## 获取构建信息类任务

很多探索类的任务都提供了构建信息，它们是你了解项目配置最好的地方。

| 名字 | 描述 | 详细 |
|:-------------|:-------------|:---------|
| help | gradle的帮助命令，显示gradle的基本用法。如果你运行 gradle 而没有指定任何 task，那么 help task 就会被自动执行，比如运行 `gradle` |  |
| tasks | 显示项目中所有可以运行的 task，包括它们的描述信息。应用于项目的插件提供了一些额外的 task。显示可用 task 的附加信息，可以使用 --all 选项。 | 下面详细介绍 |
| dependencies | 列出项目依赖，包括传递性依赖。 | 下面详细介绍 |
| dependencyInsight | 解释依赖树中的一个依赖如何被选择，为什么被选择。 | 下面详细介绍 |
| projects | 显示多项目构建中的所有子项目，单项目构建没有子项目。 |  |
| properties | 列出项目中所有可以属性。有些属性是 gradle 的 project 对象提供的，有的是用户自定义属性，可能来自于属性文件、属性命令行选项，或者直接在构建脚本中定义的。 |  |
| buildEnvironment | 列出所有 build 脚本的依赖关系 |  |

<!--
<table>
  <tr>
    <th width=20%, bgcolor=yellow>名字</th>
    <th width=60%, bgcolor=yellow>描述</th>
    <th width="20%", bgcolor=yellow>详细</th>
  </tr>
  <tr>
    <td bgcolor=#eeeeee> help </td>
    <td> gradle的帮助命令，显示gradle的基本用法。如果你运行 gradle 而没有指定任何 task，那么 help task 就会被自动执行，比如运行 `gradle`  </td>
    <td> </td>
  </tr>
  <tr>
    <td bgcolor=#00FF00> tasks </td>
    <td> 显示项目中所有可以运行的 task，包括它们的描述信息。应用于项目的插件提供了一些额外的 task。显示可用 task 的附加信息，可以使用 --all 选项。 </td>
    <td> 下面详细介绍 </td>
  </tr>
  <tr>
    <td bgcolor=rgb(0,10,0)> dependencies </td>
    <td> 列出项目依赖，包括传递性依赖。 </td>
    <td>  下面详细介绍 </td>
  </tr>
  <tr>
    <td bgcolor=rgb(0,10,0)> dependencyInsight </td>
    <td> 解释依赖树中的一个依赖如何被选择，为什么被选择。 </td>
    <td>  下面详细介绍 </td>
  </tr>
  <tr>
    <td bgcolor=rgb(0,10,0)> projects </td>
    <td> 显示多项目构建中的所有子项目，单项目构建没有子项目。 </td>
    <td>   </td>
  </tr>
  <tr>
    <td bgcolor=rgb(0,10,0)> properties </td>
    <td> 列出项目中所有可以属性。有些属性是 gradle 的 project 对象提供的，有的是用户自定义属性，可能来自于属性文件、属性命令行选项，或者直接在构建脚本中定义的。 </td>
    <td>   </td>
  </tr>
  <tr>
    <td bgcolor=rgb(0,10,0)> buildEnvironment </td>
    <td> 列出所有 build 脚本的依赖关 </td>
    <td>   </td>
  </tr>
</table>
-->

下面对与一些常用的 task 进行详细的介绍。

### 获取任务列表

运行命令：

```
gradle tasks
```

就可以列出项目中所有可以运行的 task：

```
Android tasks
-------------
androidDependencies - Displays the Android dependencies of the project.
signingReport - Displays the signing info for each variant.
sourceSets - Prints out all the source sets defined in this project.

Build tasks
-----------
assemble - Assembles all variants of all applications and secondary packages.
assembleAndroidTest - Assembles all the Test applications.
assembleDdd - Assembles all Ddd builds.
assembleDebug - Assembles all Debug builds.
assembleRelease - Assembles all Release builds.
build - Assembles and tests this project.
......
jar - Assembles a jar archive containing the main classes.
mockableAndroidJar - Creates a version of android.jar that's suitable for unit tests.
testClasses - Assembles test classes.

Build Setup tasks
-----------------
init - Initializes a new Gradle build. [incubating]
wrapper - Generates Gradle wrapper files. [incubating]

Documentation tasks
-------------------
groovydoc - Generates Groovydoc API documentation for the main source code.
javadoc - Generates Javadoc API documentation for the main source code.

Help tasks
----------
buildEnvironment - Displays all buildscript dependencies declared in root project 'TestSomeThing'.
components - Displays the components produced by root project 'TestSomeThing'. [incubating]
......
tasks - Displays the tasks runnable from root project 'TestSomeThing' (some of the displayed tasks may belong to subprojects).

Install tasks
-------------
installDebug - Installs the Debug build.
installDebugAndroidTest - Installs the android (on device) tests for the Debug build.
......
uninstallRelease - Uninstalls the Release build.

Verification tasks
------------------
check - Runs all checks.
connectedAndroidTest - Installs and runs instrumentation tests for all flavors on connected devices.
connectedCheck - Runs all device checks on currently connected devices.
......
testDebugUnitTest - Run unit tests for the debug build.
testReleaseUnitTest - Run unit tests for the release build.

Other tasks
-----------
assembleDefault
......
transformResourcesWithMergeJavaResForReleaseUnitTest
```

上面是运行 `gradle tasks` 的执行结果，可以发现，这里的 task 列表也有个分类：Android tasks、Build tasks 和 Help tasks 等，对与感兴趣的 task 你可以一一去尝试。
显示可用 task 的附加信息，可以使用 --all 选项。

### 获取依赖关系

#### dependencies

使用 `gradle dependencies` 可以所有项目依赖。
`gradle <project>:dependencies --configuration <configuration>` 可以列出指定项目指定 Configuration 的依赖关系，project 为 settings.gradle 里面配置的各个 project ，如果没有配置就不加。
比如：`gradle app:dependencies --configuration compile` 就只列出了 app 项目的 compile 任务的依赖关系，这个命令也是我们经常使用的。

```
+--- com.android.support:support-annotations:22.0.0
+--- com.squareup.retrofit2:retrofit:2.3.0
|    \--- com.squareup.okhttp3:okhttp:3.8.0
|         \--- com.squareup.okio:okio:1.13.0
+--- com.squareup.retrofit2:converter-gson:2.3.0
|    +--- com.squareup.retrofit2:retrofit:2.3.0 (*)
|    \--- com.google.code.gson:gson:2.7
+--- com.squareup.retrofit2:adapter-rxjava2:2.3.0
|    +--- com.squareup.retrofit2:retrofit:2.3.0 (*)
|    \--- io.reactivex.rxjava2:rxjava:2.0.0 -> 2.1.5
|         \--- org.reactivestreams:reactive-streams:1.0.1
+--- org.jooq:joor:0.9.5
+--- com.github.heqiangflytosky:FastScrollWebView:v1.0.0
+--- com.eclipsesource.j2v8:j2v8:4.5.0
+--- org.apache.bcel:bcel:6.0
+--- org.javassist:javassist:3.20.0-GA
+--- io.reactivex.rxjava2:rxjava:2.1.5 (*)
+--- io.reactivex.rxjava2:rxandroid:2.0.1
|    \--- io.reactivex.rxjava2:rxjava:2.0.1 -> 2.1.5 (*)
\--- com.squareup.okhttp3:okhttp:3.5.0 -> 3.8.0 (*)
```

上面为一个依赖关系树，仔细观察你会发现有些传递依赖标注了 * 号，表示这个依赖被忽略了，这是因为其他顶级依赖中也依赖了这个传递的依赖，Gradle 会自动分析下载最合适的依赖。

#### dependencyInsight

`dependencyInsight` 的用法和 `dependencies` 类似，检查一个特定的依赖需要使用 `--dependency` 参数，用 `--configuration` 来指定一个特定的 Configuration。
`--dependency` 可以不写全称，gradle 会过滤包含参数字段的所有依赖库。
比如：
```
./gradlew -q app:dependencyInsight --dependency rxjava --configuration compile
```

## 构建设置类任务

每个 Gradle 项目都至少需要一个 build.gradle 文件来定义构建逻辑，这个文件可以通过手动创建或者是通过构建设置插件的 task 方便的生成。

| 名字 | 描述 | 详细 |
|:-------------:|:-------------:|:-------------:|
| init | 初始化一个新的 build.gradle 文件 |  |
| wrapper | 在 Gradle 项目目录下生成 Gradle Wrapper 文件，使用与 Gradle 运行时相同的版本 |  |

## 配置输入

### 常用选项

| 名字 | 描述 | 详细 |
|:-------------:|:-------------:|:-------------:|
| -? -h --help | 打印出所有可用的命令行选项，包含描述信息。 |  |
| -a, --no-rebuild | 避免重新构建多项目构建中的所有子项目（也叫部分构建）。通过部分构建，可以节约检查子项目模型的开销，降低构建执行时间 |  |
| -b, --build-file | Gradle 构建脚本默认的命名约定是 build.gradle。使用这个选项执行其他名字的构建脚本（比如：gradle -b test.gradle build） |  |
| -c, --settings-file | Gradle 设置文件默认的命名约定是 settings.gradle，使用这个选项执行非标准设置文件名的构建（比如：gradle -c mySettings.gradle build）|  |
| --configure-on-demand | 这个选项的目的是优化初始化多项目构建的配置时间。这种模式尝试只配置跟正在请求的 task 相关的项目。这个选项可以通过在 gradle.properties 文件中设置 org.gradle.on-demand 属性来激活。 |  |
| --continue | 在一个 task 执行失败后 Gradle 会继续执行。在多项目构建中这个选项及其有用。它让你在构建时发现所有可能的问题，并一起修复他们，而不是一一修复。 |  |
| -g, --gradle-user-home | Gradle 的默认 home 目录位于 home 目录下的 .gradle 目录中。如果你想要指向不同的目录，则使用这个选项。 |  |
| --gui | 运行一个基于 Swing 的图形化用户界面。 |  |
| -I, --init-script | 设置一个初始化脚本用于构建。这个脚本会在所有构建 task 执行前被执行。 |  |
| -m, --dry-run | 打印 task 的执行顺序，而不必真的执行它们。如果你想要快速地确定 task 的执行顺序，这个选项会很方便。 |  |
| -p, --project-dir | Gradle 默认会在当前目录下执行构建。通过这个选项，可以指定不同的目录来执行构建 |  |
| --parallel | 这个选项可以通过在 gradle.properties 文件中设置 org.gradle.parallel=true |  |
| --parallel-threads | 当并行构建多项目的时候，这个选项可以被用来重写线程数（比如：--parallel-threads=5）。 |  |
| --profile | 除了每次构建时输出总的构建时间，你可以将构建时间拆分的更小。profile 选项在 build/reports/profile 目录下生成了详细的 HTML 报告，其中列出了所有 task 的执行时间和在配置阶段所有的时间。 |  |
| --rerun-tasks | 重新运行 task 执行图中所有确定的 task。这个选项会忽略前面 task 执行的任何 UP-TO-DATE 状态。 |  |
| -u, --no-search-upward | 告诉 Gradle 不要在父目录中寻找设置文件。在有深层次嵌套的项目结构中，这个选项被用来避免在父目录中搜索，从而节约时间。 |  |
| -v, --version | 打印出 Gradle 运行时的版本信息。 |  |
| -x, --exclude-task | 指定某个 task 在构建的时候不执行。一个比较典型的例子就是如果你想执行一个完整的构建，但是不执行所有的单元测试（比如：gradle -x test build） |  |

### 属性选项

属性提供了一种在命令行配置构建的方式。

| 名字 | 描述 | 详细 |
|:-------------:|:-------------:|:-------------:|
| -D, --system-prop | Gradle 以 JVM 进程的形式运行。与所有的 java 进程一样，你可以指定系统属性如：-Dmyprop=myvalue |  |
| -P, --project-prop | 项目属性是在构建脚本中能够被使用的变量。你可以使用这个选项从命令行直接将一个参数传递到构建脚本中。比如： -Pmyprop=myvalue|  |

-P 参数的使用：

```
./gradlew assembleDebug -Pcustom=true
```

就可以在build.gradle中使用下面代码来判断：

```
if (project.hasProperty('custom')){
    // do something
}
```

### 日志选项

Gradle 允许访问构建产生的所有日志信息。根据情况，你可以提供日志选项来过滤相关的重要消息。

| 名字 | 描述 | 详细 |
|:-------------:|:-------------:|:-------------:|
| -i,--info | 在默认设置下 Gradle 构建并不会输出大量的日志信息。通过这个选项将日志级别设置成 INFO 来获取更多的日志信息。 |  |
| -d,--debug | 以 DEBUG 日志级别运行 Gradle 构建，将会产生大量的日志信息，包括堆栈跟踪信息，这在查错的时候特别有用。 |  |
| -q,--quiet | 减少构建输出只剩下错误信息。 |  |
| -s,--stacktrace | 如果构建出错了，你会希望知道哪里出错了。如果有异常抛出，这个 -s 选项会打印一个简单的堆栈跟踪信息，这个调试失败构建的时候非常有用。 |  |
| -S,--full-stacktrace | 打印完整的异常堆栈跟踪信息。 |  |

### 缓存选项

Gradle 利用多级别的缓存机制来提高构建性能。利用一些选项可以改变默认的缓存行为。

| 名字 | 描述 | 详细 |
|:-------------:|:-------------:|:-------------:|
| --offline | 通常你的构建定义了一些依赖，如果这些依赖在本地没有存储，并且进行构建的时候没有网络连接就会引起构建失败。使用这个选项，可以让构建采用离线模式运行，并且只检查本地存储的依赖。 |  |
| --project-cache-dir | 默认的依赖缓存目录位于用户的 .gradle 目录下。这个选项可以被用来指定一个不同的目录。 |  |
| --recompile-scripts | Gradle 默认编译所有的脚本并存储在本地缓存中以提高构建性能。使用这个选项来清空这些缓存。  |  |
| --refresh-dependencies | 手动刷新缓存中的依赖。这个标志强制检查依赖的版本。 |  |

### 后台守护进程选项

守护进程以后台进程的形式运行 Gradle。一旦开始，gradle 命令会重用已经获得的后台进程执行后续构建，从而避免每次启动时的开销。

| 名字 | 描述 | 详细 |
|:-------------:|:-------------:|:-------------:|
| --daemon | 使用后台守护进程模式执行构建可以提高构建性能。如果后台进程已经存在，则会重用它，如果不存在，则会启动一个新的后台进程。后台守护进程可以通过在 gradle.properties 文件中设置 org.gradle.deamon=true 来激活。 |  |
| --foreground | 在终端中运行 Gradle 后台守护进程，用于调试和监控目的。 |  |
| --no-daemon | 不使用已有的 Gradle 后台守护进程执行构建。 |  |
| --stop | 终止一个已经存在的 Gradle 后台守护进程。 |  |


## 参考

《实战 Gradle》

<!--  
http://blog.csdn.net/jjwwmlp456/article/details/41512289

http://wiki.jikexueyuan.com/project/gradle-2-user-guide/using-the-gradle-command-line.html
http://wiki.jikexueyuan.com/project/gradle-2-user-guide/using-the-gradle-graphical-user-interface.html
http://www.yiibai.com/gradle/gradle_running_a_build.html
http://blog.csdn.net/jjwwmlp456/article/details/41512289
-->
