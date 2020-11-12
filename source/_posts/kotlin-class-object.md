---
title: Kotlin -- 类和对象
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin 的类和对象
date: 2019-12-10 10:00:00
---

Kotlin 中使用关键字 class 声明类，这一点和 Java 一致。

## 构造方法

 Kotlin的构造函数分为主构造方法和次级构造方法，下面我们分别介绍一下。

### 主构造方法

主构造方法有多种写法，主构造器中不能包含任何代码，初始化代码可以放在初始化代码段中，初始化代码段使用 init 关键字作为前缀。或者在定义属性时直接将主构造器中的形参赋值给它。
比如：

```
class MyClass (p1 :String){
    var para1:String ? = null
    init {
        para1 = p1
    }
}
```

或者：

```
class MyClass (p1 :String) {
    var para1: String = p1
}
```

下面介绍一下构造方法的几种写法。

#### 省略constructor

如果构造器有注解，或者有可见度修饰符，这时constructor关键字是必须的，其他情况下可以省略。就像上面的写法一样。

#### constructor

class 类名 constructor(形参1, 形参2, 形参3){}

```
class MyClass constructor(p1 :String) {
    private var para1: String = p1
}
```

#### 直接对属性进行赋值

```
class MyClass constructor(private var para1 :String) {
    fun print() {
        Log.e("Test","print para1$para1")
    }
}
```

#### 不显式提供主构造函数

当我们定义一个类时，我们如果没有为其显式提供主构造函数，Kotlin 编译器会默认为其生成一个无参主构造，这点和Java是一样的。比如

```
class MyClass {
    private var para1 = "p1"
    fun print() {
        Log.e("Test","print para1$para1")
    }
}
```

### 次构造函数

类可以有次构造函数，次构造函数定义在类体中，次构造函数可以有多个，次构造函数需要加前缀 constructor。


```
class MyClass {
    private var para1:String ?=null
    private var para2:String ?=null
    constructor(p1:String){
        para1 = p1
    }
    constructor(p1:String,p2:String){
        para1 = p1
        para2 = p2
    }
}
```

如果类有主构造函数，每个次构造函数都要直接或间接通过另一个次构造函数代理主构造函数。在同一个类中代理另一个构造函数使用 this 关键字： 
次级构造器: this(参数列表)

```
class MyClass (private var para1:String){
    private var para2:String ?=null
    constructor(p1:String,p2:String):this(p1){
        para2 = p2
    }
}
```

## 属性

类的属性可以用关键字 var 声明为可变的，或者关键字 val 声明为不可变。

### getter 和 setter

我们可以为属性创建 getter 和 setter 方法：

```
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```

```
class MyClass{
    var para3:String ="p3"
        get() {
            Log.e("Test","get para")
            return field
        }
        set(value){
            Log.e("Test","set para")
            field = value
        }
    val para4:String ?=null
        get() {
            Log.e("Test","get para")
            return field
        }
}
```

关键字 field 代表的是当前域。 
val不允许设置setter函数，因为它是只读的。
getter和setter是可选的，如果你没有在代码中创建它们，它是会默认自动生成。
比如下面代码：

```
class MyClass{
    private var para1:String ?=null
    private val para2:String ?=null
    var para3:String ?=null
    val para4:String ?=null
}
```

对应字节码为：

```
public final class com/hq/android/testkotlin/MyClass {
  private Ljava/lang/String; para1
  private final Ljava/lang/String; para2
  
  private Ljava/lang/String; para3
  public final getPara3()Ljava/lang/String;
  ...
  public final setPara3(Ljava/lang/String;)V
  ...
  private final Ljava/lang/String; para4
  ...
  public final getPara4()Ljava/lang/String;
  ...
}
```

可见，编译器自动为public属性的 var para3 生成了getter 和 setter方法，为public属性的 val para4 生成了getter 方法。


## 嵌套类

我们可以把类嵌套在其他类中，看以下实例：

```
class Outer {                  // 外部类
    private val bar: Int = 1
    class Nested {             // 嵌套类
        fun foo() = 2
    }
}

fun main(args: Array<String>) {
    val demo = Outer.Nested().foo() // 调用格式：外部类.嵌套类.嵌套类方法/属性
    println(demo)    // == 2
}
```

## 内部类

内部类使用 inner 关键字来表示。
内部类会带有一个对外部类的对象的引用，所以内部类可以访问外部类成员属性和成员函数。

```
class Outer {
    private val bar: Int = 1
    var v = "成员属性"
    /**嵌套内部类**/
    inner class Inner {
        fun foo() = bar  // 访问外部类成员
        fun innerTest() {
            var o = this@Outer //获取外部类的成员变量
            println("内部类可以引用外部类的成员，例如：" + o.v)
        }
    }
}

fun main(args: Array<String>) {
    val demo = Outer().Inner().foo()
    println(demo) //   1
    val demo2 = Outer().Inner().innerTest()   
    println(demo2)   // 内部类可以引用外部类的成员，例如：成员属性
}
```

为了消除歧义，要访问来自外部作用域的 this，我们使用this@label，其中 @label 是一个 代指 this 来源的标签。

## 匿名内部类

使用对象表达式来创建匿名内部类：

```
class Test {
    var v = "成员属性"

    fun setInterFace(test: TestInterFace) {
        test.test()
    }
}

/**
 * 定义接口
 */
interface TestInterFace {
    fun test()
}

fun main(args: Array<String>) {
    var test = Test()

    /**
     * 采用对象表达式来创建接口对象，即匿名内部类的实例。
     */
    test.setInterFace(object : TestInterFace {
        override fun test() {
            println("对象表达式创建匿名内部类的实例")
        }
    })
}
```

## 类的修饰符

类的修饰符包括 classModifier 和 _accessModifier_ :

 - classModifier: 类属性修饰符，标示类本身特性。
 ```
abstract    // 抽象类  
final       // 类不可继承，默认属性
enum        // 枚举类
open        // 类可继承，类默认是final的
annotation  // 注解类
 ```
 - accessModifier: 访问权限修饰符
 ```
private    // 仅在同一个文件中可见
protected  // 同一个文件中或子类可见
public     // 所有调用的地方都可见
internal   // 同一个模块中可见
 ```

## 委托

在委托模式中，有两个对象参与处理同一个请求，接受请求的对象将请求委托给另一个对象来处理。
Kotlin 直接支持委托模式，更加优雅，简洁。当很多访问器的代码存在相同或者类似的逻辑时，可以将访问器的代码封装成一个特定的类，属性声明时使用委托将访问器的功能委托给该特定的类，借此可以省略重复的访问器代码的编写。
Kotlin 通过关键字 by 实现委托。

### 类委托

https://www.runoob.com/kotlin/kotlin-delegated.html

### 属性委托



