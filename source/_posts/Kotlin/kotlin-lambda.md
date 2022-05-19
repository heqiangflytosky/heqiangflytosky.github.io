---
title: Kotlin -- Lambda 表达式
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin 中 Lambda 表达式、内联函数的用法
date: 2019-12-16 10:00:00
---


## Lambda 表达式

Lambda表达式就是一个匿名函数，它是函数式编程的基础，函数式编程的思想是将计算机运算视为函数的计算，并且计算的函数可以接收函数作为输入参数或者当作返回值来使用。使用函数式编程可以减少代码的重复，提高程序的开发效率。
普通函数按照参数和返回值类型可以分为4种，即无参数无返回值、无参数有返回值、有参数无返回值、有参数有返回值。而Lambda表达式只有两种类型，即无参数有返回值、有参数有返回值。

### 无参数有返回值

在定义无参数有返回值的Lambda表达式时，只需要将函数体写在“{}”中，函数体可以是表达式或语句块，具体语法格式如下：

```
{函数体}
```

无参数有返回值的Lambda表达式调用的语法格式如下：

```
{函数体}()
```

从上述代码可以看出，在调用Lambda表达式时，只是在Lambda表达式后添加了“()”，“()”就代表了调用该表达式。当Lambda表达式被调用之后便会执行表达式中的函数体。

```
                {
                    Log.e("Test","test")
                }()
```

### 有参数有返回值

具体语法格式如下：

```
{参数名1:参数类型1,参数名2:参数类型3 ... -> 函数体}
```

参数之间使用英文“,”分隔，且参数类型可以省略。

有参数有返回值的Lambda表达式调用的语法格式如下：

```
{参数名1:参数类型1,参数名2:参数类型3 ... -> 函数体}(参数1,参数2...)
```

```
                var s = {
                    a:Int,b:Int -> Log.e("Test","1")
                    a+b
                }(6,6)

                Log.e("Test","test $s")
```

上面是在声明Lambda表达式之后直接就调用此函数，还可以将Lambda表达式赋值给一个变量，通过变量来直接调用。

```
                {
                    Log.e("Test","test")
                    2
                }()

                var s = {
                    a:Int,b:Int -> Log.e("Test","1")
                    a+b
                }

                Log.e("Test","test ${s(6,8)}")
```

### Lambda表达式返回值

在每次调用Lambda表达式时，不管方法体里面的语句执行多少条，返回值的类型和返回值都是由方法体中最后一条语句决定的，因此在实际返回值后不要编写任何语句，不要将不是返回值的语句放置在方法体的最后一条语句的位置。

## 高阶函数

前面介绍的 Lambda 表达式都是定义在方法内部，我们其实还可以在其他地方定义 Lambda 表达式。
Lambda 表达式还可以作为函数的实际参数或者返回值存在，而这种声明，在 Kotlin 中叫做高阶函数。

### Lambda 表达式作为参数使用

下面我们以标准库中的高阶函数 repeat 方法为例子来说明 Lambda 表达式作为参数使用方法。
repeat 方法得实现如下：

```
@kotlin.internal.InlineOnly
public inline fun repeat(times: Int, action: (Int) -> Unit) {
    contract { callsInPlace(action) }
    for (index in 0 until times) {
        action(index)
    }
}
```

它有两个参数，第一个参数 times 表示重复执行的次数，第二个参数 action 是个 Lambda 表达式。
针对 `action: (Int) -> Unit` 参数，需要知晓下面几点：

 - 该表达式参数为 Int 类型，返回值为 Unit类型。
 - 形式参数名，可以任意修改。

用法：

```
                repeat(5,{value ->
                    Log.e("Test","test $value")
                })
```

#### Lambda 表达式作为参数的优化

当Lambda表达式作为函数参数使用时，还有3种优化形式，这3种优化形式分别是省略小括号、将参数移动到小括号外面、使用it关键字、下划线`_`代替未使用的参数。

##### 省略小括号

如果函数只有一个参数，且这个参数类型是一个函数类型，则在调用函数时可以去掉函数名称后面的小括号。
先定义一个高阶函数：

```
    fun testLambda(action: (Int) -> Unit) {
        action
    }
```

用法：

```
// 省略前
testLambda( { value -> Log.e("Test","test $value")})
// 省略后
testLambda{ value -> Log.e("Test","test $value")}
```

##### 将参数移动到小括号外面

如果一个函数有多个参数，但是最后一个参数类型是函数类型，那么在调用函数时，可以将最后一个参数从括号中移出，并且去掉参数之间的符号“,”。

```
// 参数移出前
repeat(5,{value -> Log.e("Test","test $value")})
// 参数移出后
repeat(5){value -> Log.e("Test","test $value")}
```

##### 使用it关键字

无论函数包含多少个参数，如果其中有参数是函数类型，并且函数类型满足只接收一个参数的要求，可以用it关键字代替函数的形参以及箭头

```
// 使用it前
repeat(5){value -> Log.e("Test","test $value")}
// 使用it后
repeat(5){Log.e("Test","test $it")}
```

##### 下划线`_`代替未使用的参数

从 Kotlin1.1 起，如果 Lambda 表达式的参数未使用，则可以用下划线代替其名称

```
testLambda { _ ->  Log.e("Test","test")}
```

#### 双冒号操作符

Kotlin 中 双冒号操作符是代表函数引用，表示把一个方法当做一个参数，传递到另一个方法中进行使用，或者赋值给一个变量。加上双冒号，这个函数就编程了一个对象。
Kotlin 如果一个函数左边添加了双冒号，那么它就不代表函数本身了，而是表示一个对象，或者说一个指向对象的引用。但是这个对象不是函数本身，而是一个和这个函数具有相同功能的对象。

```
    {
        ......
        testLambda(6,::para)
        var b = ::para
        b(8)
    }

    fun testLambda(type: Int,method: (Int) -> Int) {
        method(type)
        Log.e("Test","testLambda $type")
    }

    fun para(p:Int):Int{
        Log.e("Test","para $p")
        return p
    }
```

### Lambda 表达式作为返回值

函数不仅可以作为参数使用，还可以作为返回值使用。
声明 Lambda 表达式作为返回值的高阶函数：

```
    fun testLambda(type: Int) : (Int) -> Int{
        if (type == 1) {
            return { Log.e("Test","test1 $it")
            it }
        } else {
            return{Log.e("Test","test2 $it")
                0}
        }
    }
```

调用方法：

```
Log.e("Test","${testLambda(1)(100)}")
```


## 内联函数

在Kotlin语法中，Lambda表达式都会被编译成一个匿名类。这样的话每次调用Lambda表达式时，就会创建出新的对象，造成额外的内存开销，导致程序效率降低，为了解决这个问题，Kotlin中提供了一个修饰符“inline”，被inline修饰的Lambda（函数）称为内联函数，使用内联函数可以降低程序的内存消耗。
内联函数要合理使用，不要内联一个复杂功能的函数，尤其在循环中。