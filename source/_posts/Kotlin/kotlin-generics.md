---
title: Kotlin -- 泛型
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin 泛型
date: 2020-1-6 10:00:00
---

## 概述

与 Java 一样，Kotlin 也提供泛型，而且和 Java 泛型基本是类似的，因此，在 Java 泛型的基础上学习 Kotlin 泛型，会达到事半功倍的效果。

[我的博客：Java 基础 -- 范型 ](http://www.heqiangfly.com/2014/02/14/java-basic-knowledge-generic/)

## 泛型类和泛型方法

```
    class Box<T>(t : T) {
        var value = t
    }
```

```
    fun <T>test2(t:T) {
        Log.e("Test","$t")
    }
```

## 泛型约束

Kotlin 中使用 : 对泛型的类型上限进行约束。
最常见的约束是上界(upper bound)：

```
    fun <T : Fruit>test2(t:T) {
        Log.e("Test","$t")
    }

```

默认的上界是 Any?。
对于多个上界约束条件，可以用 where 子句：

```
    fun <T>test2(t:T) where T : Fruit,T : Interface {
        Log.e("Test","$t")
    }
```

## in 和 out

和 Java 泛型一样，Kolin 中的泛型本身也是不可变的。

 - 使用关键字 out 来支持协变，等同于 Java 中的上界通配符` ? extends`。
 - 使用关键字 in 来支持逆变，等同于 Java 中的下界通配符 `? super`。

out 表示，我这个变量或者参数只用来输出，不用来输入，你只能读我不能写我；in 就反过来，表示它只用来输入，不用来输出，你只能写我不能读我。

我们定义一个类：

```
    class Producer<T> (var t:T){
        fun produce(): T {
            return  t
        }
    }
```

下面的写法是错误的：

```
var producer : Producer<Fruit> = Producer<Apple>( Apple())
```

关于泛型不支持协变的问题，在Java泛型一文中已经介绍过。
现在轮到 out 出场了，下面的用法就是正确的了：

```
var producer : Producer<out Fruit> = Producer<Apple>( Apple())
```

我们在声明 Producer 的时候已经确定了泛型 T 只会作为输出来用，但是每次都需要在使用的时候加上 out Fruit 来支持协变，写起来很麻烦。
Kotlin 提供了另外一种写法：可以在声明类的时候，给泛型符号加上 out 关键字，表明泛型参数 T 只会用来输出，在使用的时候就不用额外加 out 了。



```
    class Producer<out T> (val t:T){
        fun produce(): T {
            return  t
        }
    }
    
    var producer : Producer<Fruit> = Producer<Apple>( Apple())
```

同样 in 的用法也类似：

```
    class Consumer<in T> {
        fun consume(t: T) {
            var s = t
        }
    }
    
    var consumer: Consumer<Apple> = Consumer<Fruit>()
```

## 星号投射

前面讲到了 Java 中单个 ? 号也能作为泛型通配符使用，相当于 ? extends Object。
它在 Kotlin 中有等效的写法：* 号，相当于 out Any。

```
var list: List<*>
```

和 Java 不同的地方是，如果你的类型定义里已经有了 out 或者 in，那这个限制在变量声明时也依然在，不会被 * 号去掉。
比如你的类型定义里是 out T : Number 的，那它加上 <*> 之后的效果就不是 out Any，而是 out Number。

## reified 关键字

由于 Java 中的泛型存在类型擦除的情况，任何在运行时需要知道泛型确切类型信息的操作都没法用了。
比如你不能检查一个对象是否为泛型类型 T 的实例：

```
<T> void printIfTypeMatch(Object item) {
    if (item instanceof T) { // IDE 会提示错误，illegal generic type for instanceof
        System.out.println(item);
    }
}
```

Kotlin 里同样也不行：

```
fun <T> printIfTypeMatch(item: Any) {
    if (item is T) { //IDE 会提示错误，Cannot check for instance of erased type: T
        println(item)
    }
}
```

这个问题，在 Java 中的解决方案通常是额外传递一个 Class<T> 类型的参数，然后通过 Class#isInstance 方法来检查：

```
<T> void check(Object item, Class<T> type) {
    if (type.isInstance(item)) {
        System.out.println(item);
    }
}
```

Kotlin 中同样可以这么解决，不过还有一个更方便的做法：使用关键字 reified 配合 inline 来解决：

```
inline fun <reified T> printIfTypeMatch(item: Any) {
    if (item is T) { // 这里就不会在提示错误了
        println(item)
    }
}

```