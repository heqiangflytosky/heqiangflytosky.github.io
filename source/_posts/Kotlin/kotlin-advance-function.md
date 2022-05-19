---
title: Kotlin -- 高阶函数
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin 中 高阶函数的用法
date: 2019-12-18 10:00:00
---


## Standard.kt 标准库中的高阶函数

在 Kotlin 源码的 Standard.kt 文件中提供了一些很好用的内置高阶函数，可以帮助我们写出更优雅的 Kotlin 代码，提高生产力。

### repeat

函数声明：

```
@kotlin.internal.InlineOnly
public inline fun repeat(times: Int, action: (Int) -> Unit) {
    contract { callsInPlace(action) }

    for (index in 0 until times) {
        action(index)
    }
}
```

用于重复执行 action 函数 times 次。它和循环语句非常相似，但是用起来会方便一些。

### run

定义了两种 run 方法：

```
public inline fun <R> run(block: () -> R): R {}
public inline fun <T, R> T.run(block: T.() -> R): R {}
```

第一种 run 方法接收一个 Lambda 表达式作为参数，返回值为最后一行的语句或者return表达式的值。

```
        var i = run {
            Log.e("Test"," test")
            6
        }
        Log.e("Test","i = $i ")
```

打印 i 为 6；
第二种方法调用指定的函数块 block，用 this 代表函数块中当前的引用对象，并且调用函数块中的方法时，this 可省略。该函数的返回值为最后一行的语句或者return表达式的值。

```
        var test:String = "aaa"
        var l = test.run {
            Log.e("Test"," $this")
            length
        }
        Log.e("Test","test = $l ")
```

打印结果为：

```
aaa
test = 3 
```

对于参数只是一个代码块的时候，可以直接用 `::fun`方法的形式传递到 () 中。
简单来讲，如果传递单代码块格式是 `block: ()` 这样的，我们可以这么干：

```
    fun testP():Int {
        Log.e("Test"," testP")
        return 8
    }
    
    
        var i = run(::testP)
        Log.e("Test","i = $i ")
```

结果为：

```
testP
i = 8 
```

对于 T.run() 方法，传递代码块参数为 `block: T.()`，我们也可以这样做，只是这个传递的方法要求必须含有一个参数传递。

```
    fun testP(p:String):Int {
        Log.e("Test"," testP $p")
        return 8
    }
    
        var test:String = "aaa"
        var l = test.run(::testP)
        Log.e("Test","test = $l ")
```

关于返回值，我们先做下面的例子：

```
        var test:String = "aaa"
        var l = test.run {
            Log.e("Test"," $this")
            return
            length
        }
        Log.e("Test","test = $l ")
        Log.e("Test","end")
```

结果是什么呢？只打印了run方法体内的 aaa，run 方法外部的两个语句都没有打印出来。
原因就是在 run 方法内使用了return语句，return语句在程序中使用时，会直接结束当前的方法以及方法外部的内容。
我们把上面的代码修改一下：

```
        var test:String = "aaa"
        var l = test.run {
            Log.e("Test"," $this")
            return@run length
            Log.e("Test"," after return")
        }
        Log.e("Test","test = $l ")
        Log.e("Test","end")
```

```
aaa
test = 3 
end
```

在上述代码中，通过使用 `return@run` 可以结束当前的 `run()` 函数，并且不会结束外部的方法。此处需要注意，结束 `run()` 方法使用 `return@run` 是固定格式，中间不需要空格分开，并且所有在Standard类中的方法都可以通过 `return@方法名` 这种格式结束当前方法。

### with

```
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```

with 的用法和 run 非常类似，只是 with 接收一个 T 和 一个代码块参数。

```
        var test:String = "aaa"
        with(test) {
            Log.e("Test"," $this")
            Log.e("Test"," $length")
        }
```

### let

```
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```

默认当前这个对象作为闭包的it参数，返回值是函数里面最后一行。

```
        var test:String = "aaa"
        var l = test.let{
            Log.e("Test"," $it.length")
            it.length
        }
        Log.e("Test","test = $l ")
```

结果：

```
3
test = 3 
```

### apply

```
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

执行 block 代码块，然后返回自身 T。

```
        var test:String = "aaa"
        var l = test.apply{
            Log.e("Test"," $length") // 省略this
            this.length              // 带 this
        }
        Log.e("Test","test = $l ")
```

结果：

```
3
test = aaa 
```

### also

```
public inline fun <T> T.also(block: (T) -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block(this)
    return this
}
```

执行 block 代码块，然后返回自身 T。和 apply 类似，不同的是在大括号中执行 `T` 自身方法的时候，必须要加上 `T. `。

```
        var test:String = "aaa"
        var l = test.also{
            Log.e("Test"," $test.length") // 用test
            it.length                     // 用it
        }
        Log.e("Test","test = $l ")
        
        var l1 = test.also{s->           // 自己指定参数名称
            Log.e("Test"," $test.length")
            s.length                      
        }
```



### takeIf

```
public inline fun <T> T.takeIf(predicate: (T) -> Boolean): T? {
    contract {
        callsInPlace(predicate, InvocationKind.EXACTLY_ONCE)
    }
    return if (predicate(this)) this else null
}
```

根据代码块的执行结果返回 null 或者 T 自身

```
        var test:String = "aaa"
        var l = test.takeIf {
            it.contains("a")
        }
        Log.e("Test","test = $l ")
```

结果：

```
test = aaa
```

### takeUnless

```
public inline fun <T> T.takeUnless(predicate: (T) -> Boolean): T? {
    contract {
        callsInPlace(predicate, InvocationKind.EXACTLY_ONCE)
    }
    return if (!predicate(this)) this else null
}
```

这个和 takeIf 差不多，不同的是代码块执行结果为 true 时返回 null，false 返回自身。


### TODO

可以用来标注某个方法需要重写，或者没有完成的事项等等，TODO 方法 会抛出异常，并可以指定异常原因！

```
@kotlin.internal.InlineOnly
public inline fun TODO(): Nothing = throw NotImplementedError()

@kotlin.internal.InlineOnly
public inline fun TODO(reason: String): Nothing = throw NotImplementedError("An operation is not implemented: $reason")
```


### 几点特性

 - ::fun ：在 run 方法中有介绍
 - return@方法名 ：在 run 方法中有介绍

