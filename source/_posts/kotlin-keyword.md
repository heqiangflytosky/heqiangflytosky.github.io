---
title: Kotlin -- 关键字
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin 的关键字
date: 2019-12-6 10:00:00
---

## 概述

本文介绍一下 Kotlin 相对于 Java 来说一些特殊的关键字。

先介绍一个技能，通过 Android Studio 的菜单 Tools -> Kotlin -> Show Kotlin Bytecode 可查看当前文件的字节码，如果想查看对应的 java 代码，可以点击 Kotlin Bytecode 窗口上面的 Decompile 按钮，就会出现 Kotlin 代码对应的 Java 代码。这对于我们分析和加深理解 Kotlin 很有帮助。

## 关键字

### open

Kotlin 中的类、方法和变量默认是final的，不能被继承和重写，可以使用 open 关键字修饰后，类可以被继承，方法和变量可以被重写。

### override

重写类的方法和变量。

### fun

kotlin 中用来定义函数。

### data

data 关键字用来修饰一个只包含数据的类，在 Kotlin 中，不需要自己动手去写一个 JavaBean，可以直接使用 data 类，编译器会自动的从主构造函数中根据所有声明的属性提取以下函数：

 -  equals() / hashCode() 
 -  toString() 格式如 "User(name=John, age=42)" 
 -  componentN() functions 对应于属性，按声明顺序排列
 -  copy() 函数 

data class 默认没有无参构造函数，而且默认为final类型，不可以被继承。这会在某些情况下给我们的使用造成一些不便，不过，我们可以使用官方给出的插件来解决这些问题（noarg、allopen）。
[官方使用文档](https://www.kotlincn.net/docs/reference/compiler-plugins.html)
下面来简单介绍使用方法：
1.添加依赖：

```
buildscript {
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-allopen:$kotlin_version"
        classpath "org.jetbrains.kotlin:kotlin-noarg:$kotlin_version"
    }
}
```

2.应用插件

```
apply plugin: 'kotlin-allopen'
apply plugin: 'kotlin-noarg'
```

3.声明注解

```
annotation class OpenDataClass
```

4.添加配置

在build.gradle 中添加：

```
allOpen{
    annotation("com.android.hq.gank.gankkotlin.utils.OpenDataClass")
}

noArg{
    annotation("com.android.hq.gank.gankkotlin.utils.OpenDataClass")
}
```

5.为 data class 添加注解

```
@OpenDataClass
data class GankItemBean (
    ...
)
```

经过上面的操作，编译器就会帮我去掉 final 关键字，并且生成一个无参的构造方法。

### object

object 关键字来声明一个对象或者一个对象表达式。
Kotlin 中我们可以方便的通过对象声明来获得一个单例。
```
object AppUtils {
    const val INTENT_ITEM_INFO = "intent_item_info"
    fun getCacheDir(): String {
        ....
    }
}
```

使用： 

```
AppUtils.getCacheDir()
```

通过对象表达式实现一个匿名内部类的对象用于方法的参数中：

```
        view.setOnSizeChangeListener(object : OnSizeWillChangeListener {
            override fun onSizeWillChanged(w: Int, h: Int) {
                
            }
        })
```


参考：
https://www.runoob.com/kotlin/kotlin-object-declarations.html
https://blog.csdn.net/xlh1191860939/article/details/79460601

### companion

伴生对象，类内部的对象声明可以用 companion 关键字标记，这样它就与外部类关联在一起，我们就可以直接通过外部类访问到对象的内部元素。

```
class MyClass {
    companion object Factory {
        fun create(): MyClass = MyClass()
    }
}

val instance = MyClass.create()   // 访问到对象的内部元素
// 或者
val instance = MyClass.Factory.create() 
```

我们可以省略掉该对象的对象名：

```
class MyClass {
    companion object {
       fun create(): MyClass = MyClass()
    }
}

val x = MyClass.create()
// 或者
val x = MyClass.Companion.create()
```

**注意**：一个类里面只能声明一个内部关联对象，即关键字 companion 只能使用一次。

### var 与 val

var 关键字进行可变变量的定义，对应的是 Java 语言中的普通变量：

```
var <标识符> : <类型> = <初始化值>
```

val关键字进行不可变变量定义,只能赋值一次的变量，类似Java中final修饰的变量：

```
val <标识符> : <类型> = <初始化值>
```

常量与变量都可以没有初始化值,但是在引用前必须初始化
编译器支持自动类型判断,即声明时可以不指定类型,由编译器判断。

我们看一下下面代码的字节码：

```
class GankApi {
    var GANK_BASE_URL : String = "http://gank.io/api/"
    val GANK_DAY_HISTORY : String  = "day/history"
}
```

对应字节码：

```
public final class com/android/hq/gank/gankkotlin/data/GankApi {
  // access flags 0x2
  private Ljava/lang/String; GANK_BASE_URL = "http://gank.io/api/"
  ......
  public final getGANK_BASE_URL()Ljava/lang/String;
  ......
  public final setGANK_BASE_URL(Ljava/lang/String;)V
  ......
  private final Ljava/lang/String; GANK_DAY_HISTORY = "day/history"
  ......
  public final getGANK_DAY_HISTORY()Ljava/lang/String;
  ......
}
```

可见，var 相当于 Java 的 private 变量属性，但带有 public 的 set 和 get 方法。
val 相当于 Java 的 private 常量属性，带有 public 的 get 方法。

### const

const 必须修饰 val，且只能在top-level级别和object中声明。
top-level 也就是将静态常量的定义放到类的外面，不依赖类的而存在。
比如：

```
const val GANK_SEARCH_URL : String = "http://gank.io/api/search/query"
class GankApi {
    val GANK_DAY_HISTORY : String  = "day/history"
}
```

在 Kotlin 中可以直接这样使用：`com.android.hq.gank.gankkotlin.data.GANK_SEARCH_URL`，在 Java 中可以采取 文件名+Kt为类名调用：`GankApiKt.GANK_SEARCH_URL`。

或者在 object 中定义：

```
object GankApi {
    val GANK_DAY_HISTORY : String  = "day/history"
    const val GANK_SEARCH_URL : String = "http://gank.io/api/search/query"
}
```

对应字节码：

```
public final class com/android/hq/gank/gankkotlin/data/GankApi {
  private final static Ljava/lang/String; GANK_DAY_HISTORY = "day/history"
  ......
  public final getGANK_DAY_HISTORY()Ljava/lang/String;
  ......
  public final static Ljava/lang/String; GANK_SEARCH_URL = "http://gank.io/api/search/query"
  ......
  public final static Lcom/android/hq/gank/gankkotlin/data/GankApi; INSTANCE
  ......
}
```

可见，在 object 中，val 对应 Java 的 private final static 属性，带有 public 的 get 方法，而 const val 对应 Java 的 public final static 属性。
对于 val 和 var 的访问，我们都是通过其对应的 get 或者 set 方法来访问，因此，当定义常量时，出于效率考虑，我们应该使用 const val 方式，避免频繁函数调用。

### sealed

使用 sealed class 声明密封类。
密封类可以解决 when 语句在编程中的问题。

### init

在类中声明一个初始化模块，初始化模块中的代码，在类被初始化时自动执行，它先于从构造函数执行，在主构造函数之后执行。
初始化模块可以直接使用主构造函数中的参数，用来弥补主构造函数没有代码的不足。

### when

when 类似其他语言的 switch 操作符，语法如下：

```
when (表达式/语句) {
    目标值1 -> 执行语句1
    目标值2 -> 执行语句2
    else -> {
        执行语句3
    }
}
```

在 when 中，else 同 switch 的 default。如果其他分支都不满足条件将会求值 else 分支。
如果很多分支需要用相同的方式处理，则可以把多个分支条件放在一起，用逗号分隔：

```
when (x) {
    0, 1 -> print("x == 0 or x == 1")
    else -> print("otherwise")
}
```

when 中使用 in 运算符来判断集合内是否包含某实例：

```
fun main(args: Array<String>) {
    val items = setOf("apple", "banana", "kiwi")
    when {
        "orange" in items -> println("juicy")
        "apple" in items -> println("apple is fine too")
    }
}
```

### field

在 Kotlin 中，用 field 关键字表示幕后字段，幕后字段主要用于自定义getter和setter中，并且只能在getter 和setter中访问。
那么我们为什么要使用幕后字段呢？它有什么作用呢？
我们知道，kotlin中，会为类的非私有的属性生成getter或者setter方法，当我们调用属性或者为属性赋值时，实际会调用这些方法，而这些getter或者setter方法我们也是可以自定义的。
我们先来看一下下面的自定义方法是否正确：

```
class TestClass{
    var para1:String = ""
    set(value) {
        Log.e("Test","para1=$value")
        para1 = value
    }
}
```

```
        var test = TestClass()
        test.para1 = "test"
```

运行上面的代码后会发生 crash：`Caused by: java.lang.StackOverflowError: stack size 8MB`，为什么会这样呢？我们看一下它反编译后的java代码：

```
public final class TestClass {
   @NotNull
   private String para1 = "";

   @NotNull
   public final String getPara1() {
      return this.para1;
   }

   public final void setPara1(@NotNull String value) {
      Intrinsics.checkParameterIsNotNull(value, "value");
      Log.e("Test", "para1=" + value);
      this.setPara1(value);
   }
}
```

可以看到，在我们自定义的 `setPara1` 方法内部嵌套调用了方法本身，因此导致溢出报错。
把上面的方法改一下：

```
class TestClass{
    var para1:String = ""
    set(value) {
        Log.e("Test","para1=$value")
        field = value
    }
}
```

这下就正常了。可以看出，幕后字段field指的就是当前的这个属性，它不是一个关键字，只是在setter和getter的这个两个特殊作用域中有着特殊的含义，就像一个类中的this一样。幕后字段可以避免访问器的自递归而导致程序崩溃的 StackOverflowError异常。
如果我们重写了这个变量的 getter 方法和 setter 方法，并且在 getter 方法和 setter 方法中都没有出现过 filed 这个关键字，则编译器会报错，提示 `Initializer is not allowed here because this property has no backing field`，除非显式写出 filed 关键字。


### inline

用来修饰function，那么这个function就被称作inline function（内联函数）。
具体参考[Kotlin -- Lambda 表达式](http://www.heqiangfly.com/2019/12/16/kotlin-lambda/) 中关于内联函数部分的说明。

### lazy 、lateinit 

 - lateinit：如果不想在定义变量的时候初始化，一种做法是 `var para4:String ?=null`，表示可为空。另外一种方法是使用lateinit `lateinit var para4:String`，表示在后面合适的时机进行初始化，如果在调用的时候还没有进行初始化，会抛出未初始化的异常。
  - 只能修饰 var 变量，不能修饰 val 常量。
  - 不能对可空类型使用，也就是说不能同时使用 lateinit 和 `para4:String ?=null`。
  - 不能对java基本类型使用，例如Double，Int，Long等
 - lazy 也是用于延迟初始化，第一次使用时再实例化。在类型后面加 `by lazy{}`即可，`{}` 中的最后一行要返回初始化的结果。使用方法如下：
  - lazy 只能对常量 val 使用，不能修饰变量 var
  - lazy 的加载时机为第一次调用常量的时候，而且只会加载一次。


```
    val para5:String by lazy {
        "Hello"
    }
```


### suspend



### is

`!is`
判断类型用，类似于 Java 中的 instanceof()。
另外 is 可以对符合条件的类型进行智能转换，不必像 Java 那也进行显式的类型转换。

```
        var a = 5
        if (a is Int) {
            Log.e("Test", "${a*a}")
        }
```

输出 25。

### as

类型转换

```
o as HistoryFavItem
```

```
(list?.get(position) as GankImageItem).imageUrl
```

### inner

Kotlin 中使用 inner 关键字来修饰内部类。具体介绍参考 [Kotlin -- 类和对象](http://www.heqiangfly.com/2019/12/10/kotlin-class-object/) 中关于内部类部分的介绍。

### vararg

可变参数


### `$`

字符串模板（取值）

### 双冒号`::`

Kotlin 中 双冒号操作符是代表函数引用，表示把一个方法当做一个参数，传递到另一个方法中进行使用，或者赋值给一个变量。加上双冒号，这个函数就编程了一个对象。
Kotlin 如果一个函数左边添加了双冒号，那么它就不代表函数本身了，而是表示一个对象，或者说一个指向对象的引用。但是这个对象不是函数本身，而是一个和这个函数具有相同功能的对象。


## 数据类型


### UInt


## 符号

### `?.`

安全调用符，`foo?.bar` 的意思就是如果 `foo != null`，就调用 `foo.bar`，如果 `foo == null`就返回 `null`。

### `?:`

`foo?:bar` 的意思就是如果 `foo != null`，就返回 `foo`，如果 `foo == null`就返回 `bar`。

### `as?`

`foo as? Type` 的意思就是如果 `foo is Type` 就相当于执行 `foo as Type`，如果 `foo !is Type` 就返回 `null`，并不抛出异常。

### `!!`

`foo !!` 的意思就是如果 `foo != null`，就执行 `foo`，如果 `foo == null`，就抛出 NullPointerException 异常。
相当于非空断言。

### 位运算符

 - or：按位或
 - and：按位与
 - shl：有符号左移
 - shr：有符号右移
 - ushr：无符号右移
 - xor：按位异或
 - inv：按位取反

### 区间运算符

 - in：在某个范围中
 - downTo：递减，循环时可用，每次减1
 - step：步长，循环时可用，设置每次循环的增加或减少的量


## 注解

### JvmField

### JvmStatic