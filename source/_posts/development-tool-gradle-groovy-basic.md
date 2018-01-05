---
title: Groovy 教程系列（一）-- Groovy 入门
categories: Gradle
comments: true
tags: [Groovy]
description: 介绍 Groovy 语言的一些基本语法规则
date: 2016-3-16 10:00:00
---

## 概述

Groovy 是一种基于Java平台的面向对象语言。Groovy 的语法和 Java 非常的相似，可以使用现有的 Java 库来进行 Groovy 开发。可以将它想像成 Java 语言的一种更加简单、表达能力更强的变体。
用 Groovy 编写的任何内容都可以编译成标准的 Java 类文件并在 Java 代码中重用。类似地，用标准 Java 代码编写的内容也可以在 Groovy 中重用。所以，可以轻易地使用 Groovy 为 Java 代码编写单元测试。而且，如果用 Groovy 编写一个方便的小工具，那么也可以在 Java 程序中使用这个小工具。
我们为什么要学习 Groovy 语言呢？
Groovy 是一种更有生产力的语言。它具有松散的语法和一些特殊功能，能够加快编码速度。
在 Android 开发中，我们会经常接触 Gradle，而前面博客中我们说过，Gradle 是一个使用 Groovy 语言实现的用于构建项目的框架，如果不懂 Groovy，你就不能说精通了 Gradle。
前面介绍的几篇关于 Gradle 的博客中都有涉及到 Groovy，比如 Gradle 的配置和插件开发，那么现在再读 Grovvy 的介绍的文章，会有一种豁然开朗的感觉。
Groovy 语言的一些特点：

 - Groovy 的松散的 Java 语法允许省略分号和 return 关键字。
 - 变量的类型和方法的返回值也是可以省略的。
 - 方法调用时，括号也是可以省略的。
 - 除非另行指定，Groovy 的所有内容都为 public。
 - Groovy 允许定义简单脚本，同时无需定义正规的 class 对象。
 - Groovy 在普通的常用 Java 对象上增加了一些独特的方法和快捷方式，使得它们更容易使用。
 - Groovy 语法还允许省略变量类型。

[官方网站](http://groovy-lang.org/)
[Groovy API 文档](http://www.groovy-lang.org/api.html)：遇到不懂的类或者方法，这个是好帮手。

## 运行环境

我们可以安装 Groovy SDK 来设置运行环境，如果你不想麻烦，也可以在Android工程中运行 Groovy。

### 在 build.gradle 中编写代码

这一点在前面的博客 [Gradle 使用指南 -- Gradle Task ](http://www.heqiangfly.com/2016/03/13/development-tool-gradle-task/) 其实已经有所运用，即在里面创建一个 Task，然后在 Task 中编写 Groovy 代码即可。

```java
task hello {
    doFirst {
        println 'task hello doFirst'
        Student st = new Student()
        st.setStudentName("Joe")

        println(st.getStudentName());
    }
}

class Student {
    String StudentName;
}
```

然后运行 `./gradlew hello` 命令即可。

### 以插件的方式

参考博客 [Gradle 使用指南 -- 创建Plugin](http://www.heqiangfly.com/2016/03/15/development-tool-gradle-customized-plugin/)

### 在 build.gradle 中直接调用 Groovy 方法

这一种方式是前面两种方式的结合，但是不用创建 gradle 插件。
在 buildSrc 中创建 TestGroovy.groovy 文件：

```
package com.android.hq.myfirstplugin

public class TestGroovy {

    public static void testGroovy() {
        Student st = new Student()
        st.setStudentName("James");

        println(st.getStudentName())
    }

    public static class Student {
        String StudentName;
    }
}
```

在 Task 中调用方法：

```
task hello {
    doFirst {
        println 'task hello doFirst'
        TestGroovy.testGroovy()
    }
}
```
然后运行 `./gradlew hello` 命令即可；

## Groovy 入门

由于 Groovy 和 Java 极其的类似，因此，基本的语法规范就参考[W3C school Groovy教程](https://www.w3cschool.cn/groovy/)即可，下面只来介绍一下 Groovy 的一些新特性。

### 变量类型定义和方法声明

在 Java 中，变量是必须指定类型的，但是在 Groovy 中，所有的变量类型都可以用 `def` 去指定，Groovy 会根据对象的值来判断它的类型。

```
        def helloStr = "Hello World";
        def a = 1, b = 2
        println helloStr
        println a + b
```

函数的的返回值的类型当然也可以用 `def` 来声明：

```
    def getStr() {
        return "Hello World"
    }
```

在声明函数时，参数变量的类型是可以省略的：

```
    def add(arg1, arg2) {
        return arg1+arg2
    }
```

前面我们说过，方法返回值的关键字 `return` 是可以省略的：

```
    def add(arg1, arg2) {
        arg1+arg2
    }
```

方法调用时括号是可以省略的，见 `println a + b` 的调用。
在 Groovy 中，类型是弱化的，所有的类型都可以动态推断，但是 Groovy 仍然是强类型的语言，类型不匹配仍然会报错；

上述两个类完全一致，只有有属性就有Getter/Setter；同理，只要有Getter/Setter，那么它就有隐含属性。
### 字符串

在Groovy中有两种风格的字符串：String（java.lang.String）和GString（groovy.lang.GString）。GString允许有占位符而且允许在运行时对占位符进行解析和计算。
这里我们只介绍对占位符的一种使用：嵌入表达式

#### 嵌入表达式

```
        def worldStr = "World"
        def helloStr = "Hello ${worldStr}";
        println helloStr
        println "value: ${3+3}"
```

除了${}占位符的{}其实是可以省去的。
占位符里面可以包含一个闭包表达式。

### 对象

#### Getter/Setter 方法
在Groovy中，对象的 Getter/Setter 方法和属性是默认关联的，比如：

```
    public static class Student {
        String StudentName;
    }

    public static class Student {
        String StudentName;

        String getStudentName() {
            return StudentName
        }

        void setStudentName(String studentName) {
            StudentName = studentName
        }
    }
```

#### with 方法

当对同一个对象进行操作时，可以使用 `with`：

```
        Student st = new Student()
        st.with {
            id = 10;
            name = "James";
        }

        println st.getName()
        ......
    public static class Student {
        def name
        def id
    }
```

#### join 方法

用指定的字符连接集合中的元素：

```
        def list = [2017,1,6]
        println list.join("-")
```

其他实用方法请参考 Groovy API 文档中的 `DefaultGroovyMethods`。

### 闭包

闭包是一个短的匿名代码块。它通常跨越几行代码。一个方法甚至可以将代码块作为参数。它们是匿名的。
下面是一个简单闭包的例子：

```
        def clos = {
            def worldStr = "World"
            def helloStr = "Hello $worldStr";
            println helloStr

            def person = [name: 'Joe', age: 36]
            println "$person.name is $person.age years old"
        }
        clos.call()
```

代码行 {...}被称为闭包。此标识符引用的代码块可以使用call语句执行。

闭包也可以包含形式参数，以使它们更有用，就像Groovy中的方法一样。

```
        def clos = { param ->
            def helloStr = "Hello $param";
            println helloStr
        }
        clos.call("World")
```

如果闭包不指定参数，那么它会有一个隐含的参数 `it`：

```
        def clos = {
            def helloStr = "Hello $it";
            println helloStr
        }
        clos.call("World")
```

闭包可以有返回值：

```
        def clos = {
            def helloStr = "Hello $it";
            return helloStr
        }
        println clos.call("World")
```

闭包还可以作方法的参数。

### 本地集合

### 内置的正则表达式



## 学习资料

[W3C school Groovy教程](https://www.w3cschool.cn/groovy/)
[Gradle从入门到实战 - Groovy基础](http://blog.csdn.net/singwhatiwanna/article/details/76084580)

<!-- 
https://www.ibm.com/developerworks/cn/education/java/j-groovy/j-groovy.html
https://www.jianshu.com/p/f704af0a3da5
https://www.cnblogs.com/nowgood/p/Gradle-xue-xi-bi-ji-zhiGroovy.html
https://www.jianshu.com/p/ba55dc163dfd
http://blog.csdn.net/dora_310/article/details/52895835
http://blog.csdn.net/rosten/article/details/20370849
-->

