---
title: Markdown 绘制 UML 图 -- PlantUML + Gravizo
categories: 开发工具
comments: true
tags: [开发工具, Markdown, PlantUML, Gravizo]
description: 介绍使用 PlantUML 绘制 UML 图，以及渲染引擎组合 PlantUML + Gravizo 的使用
date: 2017-7-8 10:00:00
---

我们在工作中会经常使用UML图，实现UML的工具有很多，首先是绘图软件，但是所有的绘图软件有这样一个问题：这些软件绘制成的图片无法进行版本控制。也就是说如果后面你想修改软件的话，如果在软件里面的原图没有保存的话，就要重新再画了。这对于我们习惯于版本控制的码农来说显然是无法忍受的。
那么下面介绍一种可以在Markdown中使用的绘制UML工具 —— PlantUML，以及渲染引擎 Gravizo。

## PlantUML简介

可以登陆[PlantUML官网](http://plantuml.com/)看一下，里面有支持的UML类型以及使用方法。

![支持的类型](/images/development-tool-markdown-plant-uml/plantuml-uml-supported.png)

## 使用方法

打开[在线作图网址](http://www.plantuml.com/plantuml)，在上面的代码框里面输入代码。
完成后点击 Submit 按钮提交查看预览图，同时在预览图下面的 URL 地址框里面会有生成的 UML 图的 png 地址图，当然你也可以选择生成 SVG 或者 ASCII Art。把这个图片地址复制到 Markdown 就可以使用了。
比如：

```
@startuml
Title "BundleLauncher类图"
interface BundleExtractor
abstract class BundleLauncher
abstract class SoBundleLauncher
abstract class AssetBundleLauncher

BundleLauncher <|-- ActivityLauncher
BundleLauncher <|-- SoBundleLauncher
SoBundleLauncher <|-- ApkBundleLauncher
BundleExtractor <|.. SoBundleLauncher
SoBundleLauncher <|-- AssetBundleLauncher
AssetBundleLauncher <|-- WebBundleLauncher

class ActivityLauncher {
+ public preloadBundle(Bundle bundle)
}

class SoBundleLauncher {
+ public preloadBundle(Bundle bundle)
}

class ApkBundleLauncher {
+ public loadBundle(Bundle bundle)
}

class AssetBundleLauncher {
+ public loadBundle(Bundle bundle)
}
@enduml
```

![UML类图](http://www.plantuml.com/plantuml/png/2yaioKbLK78gpKl9IVL9BCrBpaWjUhvnzzFP-vIuClDAKelI4fDJ5I3ohXKbHOd99Vb5N8b9nM2cGd9EOd6n0gfsTDdWVFpoZiN5gILeIhXG-GesDRgw2ex99PbbcIMLS5NO567OXYu0DQiW6qqTcX-1olJqY3ODYnUmY44KXwSceViM6X1e_bEevj9MA2XDoibCLYWeIit9Jqo1QDI0K0f9O4gJgnPc0eRhI3O18roGZI16FnPV4sS20000)

如果想修改怎么办？
打开[在线作图网址](http://www.plantuml.com/plantuml/uml/SyfFKj2rKt3CoKnELR1Io4ZDoSa70000)，把图片的 url 复制到下面的 url 框里面，点击 Submit ，在上面代码框里面会出现该图片对应的代码，然后修改即可。修改完成后再次生成图片即可。
是不是相当的方便呢？

## 基本通用语法教程

这里只介绍一些通用的语法命令，其他具体的语法请参考[官方文档](http://translate.plantuml.com/zh/PlantUML_Language_Reference_Guide_ZH.pdf)

 1. 添加标题

可以用 `Title` 后面加标题

```
Title "继承关系图"

Father <|-- Son
```

![PlantUML](http://www.plantuml.com/plantuml/png/2yaioKbLK7g-U_cpplrFMpS_txpxwUnzIbnSReab6Qb52ZOrkheAmVbv0000)

 2. 注释

所有以单引号开头的行 ' 都是注释。你也可以使用多行注释,多行注释以 /' 开头 '/ 结尾。

### 类图

#### 方法和属性的访问权限

| 标识 | 属性 |
| :-------------: |:-------------:|
| - | private |
| # | protected |
| + | public |
| ~ | package private |

```
class Dummy {
- private field1
# protected field2
+ public field3
~ package method1()
- private method3()
# protected method4()
+ public method2()
}
```

![UML类图](http://www.plantuml.com/plantuml/png/Iyv9B2vMS2dDpQrKgERILIWeoYnBB4bLICjCpKanv5862kINf2QNfAP0X8ouj1KAIfDoCfCXV6EkEeM2nEJinFHKXTpKaepy54CDJIHp86B6G35aeo2Y9a1Hk6aG8IEWK2q0)

#### 关系

 - 继承

```
Father <|-- Son
```

![UML类图](http://www.plantuml.com/plantuml/png/SqiioKWjKh2fqTLL2CxF0m00)

 - 实现

```
abstract class AbstractList
interface List
List <|.. AbstractList
```

![UML类图](http://www.plantuml.com/plantuml/png/IqmgBYbAJ2vHICv9B2vMS8HoVJABIxWoyqfIYz8IarCLm5mGeM1JewU7eWe0)

 - 组合

关联关系的一种特例，他体现的是一种contains-a的关系，这种关系比聚合更强，也称为强聚合；他同样体现整体与部分间的关系，但此时整体与部分是不可分的，整体的生命周期结束也就意味着部分的生命周期结束。

```
Human *-- Brain
```

![UML类图](http://www.plantuml.com/plantuml/png/yoZDJSnJqDBLLN0gIipC0m00)

 - 聚合

关联关系的一种特例，他体现的是整体与部分、拥有的关系，即has-a的关系，此时整体与部分之间是可分离的，他们可以具有各自的生命周期。

```
Company o-- Human
```

![UML类图](http://www.plantuml.com/plantuml/png/SyxFBKZCgrJ8rzLLy2ZDJSm30000)

 - 关联

类与类之间的联接。强依赖关系，表现在代码层面，为被关联类B以类属性的形式出现在关联类A中。

```
class Water
class Human
Human --> Water
```

![UML类图](http://www.plantuml.com/plantuml/png/Iyv9B2vM24yiIItYIWQpFKfp4_EumAI2hguTH0u0)

 - 依赖

类与类之间的联接。一个类A使用到了另一个类B，而这种使用关系是具有偶然性的、临时性的、非常弱的，表现在代码层面，类B作为参数被类A在某个方法中使用，例如人和烟草的关系。

```
Human ..> Cigarette
```

![UML类图](http://www.plantuml.com/plantuml/png/yoZDJSnJqDEpKt3EJ4yiIYqfIGK0)

#### 域

在作图时，如果遇到了名称相同包名不同的类这时就要用域来进行区分。可以用 `namespace`，也可以直接用包名：

```
class BaseClass

namespace net.dummy #DDDDDD {
    .BaseClass <|-- Person
    Meeting o-- Person
    
    .BaseClass <|- Meeting
}

namespace net.foo {
  net.dummy.Person  <|- Person
  .BaseClass <|-- Person

  net.dummy.Meeting o-- Person
}

BaseClass <|-- net.unused.Person

class net.unused.Person {
  + public void test();
}
```

![UML类图](http://www.plantuml.com/plantuml/png/TKvB2i8m4Dtd50_SAD9SG5VgLl0ACHabq2J5IGgYlRjjf87M-bR3lA-k5JCEYkauN49uvOWRfGcUeZJ9kITMfmoy17h8eiR-NLMuq8E3pzIPA5f_HvY-5soZL7Jpobi8kQZKosyIigsa_banCIxCwUjcna6UV68oSipGcVqXygmjcdIjhKORh44aZklDJdGV)

### 时序图

语法参考[官方网站](http://plantuml.com/sequence-diagram)

## PlantUML + Gravizo

上面用 PlantUML 来绘制 UML 的方法，我们需要去在线编写代码生成 UML 图片，然后再把图片地址集成到 Markdown 里面，步骤还是稍微繁琐，现在介绍 Gravizo 的使用可以解决这个问题。
[Gravizo](http://www.gravizo.com/) 是一个绘图引擎，只需要用 Url 包含 PlantUML 代码放到一个 `img` 标签中，就可以在线实时的绘制出我们需要的 UML 图。
比如下面的代码：

```
<img src='https://g.gravizo.com/svg?
abstract class AbstractList;
interface List;
List <|.. AbstractList;
'/>
```

把上面的代码放到 Markdown 里面，就可以实时的把 UML 图绘制出来了。

<img src='https://g.gravizo.com/svg?
abstract class AbstractList;
interface List;
List <|.. AbstractList;
'/>

当然这个也需要集成的 Markdown 里面支持 Gravizo 才能显示出来，Hexo 博客是可以显示出来的，CSDN 博客就显示不出来。


## 参考文章

http://www.jianshu.com/p/1256e2643923
http://www.jianshu.com/p/e92a52770832
https://yq.aliyun.com/articles/25405
