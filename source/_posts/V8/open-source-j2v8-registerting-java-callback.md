---
title: J2V8 -- 注册 Java 回调函数
categories: V8
comments: true
tags: [V8, J2V8]
description: 介绍 J2V8 如何注册 Java 回调函数
date: 2017-08-10 10:00:00
---

本文译自[Registering Java Callbacks with J2V8](https://eclipsesource.com/blogs/2015/06/06/registering-java-callbacks-with-j2v8/)，并加入了自己的一些理解。

使用 J2V8 时是可以使用 JavaScript 来调用 Java 的方法的，下面就介绍一些如何注册 Java 的回调函数来供 JavaScript 调用。

## 回调函数

在 JavaScript 中，函数也即是对象，可以被操作也可以被传递。在使用 J2V8 时，任何的 JavaScript 方法都可以映射到 Java 的方法之上，当这些方法被调用时，J2V8 就会调用 Java 的方法来代替，并把参数传递给 Java 的方法。

## 注册 Java 方法

Java 的方法可以通过两种方法注册为回调函数，可以通过实现 `JavaCallback` 接口来实现（如果没有返回值时，也可以通过 `JavaVoidCallback` 来实现），或者是反射地把已经实现的 Java 方法注册为回调函数。

### JavaCallback

先来看下面一段代码：

```java
        V8 v8 = V8.createV8Runtime();
        JavaVoidCallback callback = new JavaVoidCallback() {
            public void invoke(final V8Object receiver, final V8Array parameters) {
                if (parameters.length() > 0) {
                    Object arg1 = parameters.get(0);
                    Log.e("Test", "arg1 = " + arg1);
                    if (arg1 instanceof Releasable) {
                        ((Releasable) arg1).release();
                    }
                }
            }
        };
        v8.registerJavaMethod(callback, "print");
        v8.executeScript("print('hello, world');");

        v8.release();
```

执行结果：

```
arg1 = hello, world
```

上面的例子中创建了 `JavaVoidCallback` 的一个匿名内部类，把该类的一个实例注册为一个全局的作用域中的方法，并命名为 `print` 方法，那么我们在 JavaScript 中就可以向调用其他 JavaScript 的方法一样来调用 `print` 方法了。

### 通过反射来注册

先来看下面一段代码：

```java
    public static class Console {
        public void log(final String message) {
            Log.i("Test INFO","[INFO] " + message);
        }
        public void error(final String message) {
            Log.e("Test ERROR","[ERROR] " + message);
        }
    }
    public void start() {
        V8 v8 = V8.createV8Runtime();
        Console console = new Console();
        V8Object v8Console = new V8Object(v8);
        v8.add("console", v8Console);
        v8Console.registerJavaMethod(console, "log", "log", new Class<?>[] { String.class });
        v8Console.registerJavaMethod(console, "error", "error", new Class<?>[] { String.class });
        v8Console.release();
        v8.executeScript("console.log('hello, world');");
        v8.release();
    }
```

执行结果：

```
[INFO] hello, world
```

在上面的例子中，一个已经实现的类的方法被注册为回调函数。

## 参数

参数可以从 JavaScript  传递到 Java，如果是通过实现 `JavaCallback` 来注册回调函数，那么参数会以 `V8Array` 的形式来传递。`V8Array` 包含了一些 `V8Object` 对象或者是一些原始数据。这些 `V8Array` 并不需要我们去释放，因为它不是由开发者创建的。但是任何从这个 `V8Array` 中获取的作为参数的 `V8Object` 都需要手动的进行释放，因为它们是作为方法调用的结果返回给你的。
如果方法是通过反射的方式进行注册的，那么这个方法的所有的参数类型都是已知的。在这种情形下，传递给 Javascript 函数的参数必须与作为它的回调函数的 Java 方法的一致。

## 接收器

当 JavaScript 调用通过 Java 注册的回调函数时，这个被调用的 JavaScript 对象会作为第一个参数进行传递。
看下面一段 JavaScript 代码：

```javascript
var array1 = [{first:'Ian'}, {first:'Jordi'}, {first:'Holger'}];
for ( var i = 0; i < array1.length; i++ ) {
  print.call(array1[i], " says Hi.");
}
```

这中情况下 `print` 方法会被调用，并且 “says Hi.” 会被作为参数传递到 Java。但是当前的 JavaScript 会作为一个接收器被传递过来。

```java
    class PersonPrinter implements JavaVoidCallback {
        @Override
        public void invoke(final V8Object receiver, final V8Array parameters) {
            Log.e("Test",receiver.getString("first") + parameters.get(0));
        }
    }

    public void testReceiver() {
        V8 v8 = V8.createV8Runtime();
        v8.registerJavaMethod(new PersonPrinter(), "print");
        v8.executeVoidScript(""
                + "var array1 = [{first:'Ian'}, {first:'Jordi'}, {first:'Holger'}];\n"
                + "for ( var i = 0; i < array1.length; i++ ) {\n"
                + "  print.call(array1[i], \" says Hi.\");\n"
                + "}\n");
        v8.release();
    }
```

执行结果：

```
Ian says Hi.
Jordi says Hi.
Holger says Hi.
```

把调用者也打印出来了。

## V8Function 类

在 J2V8 3.0 版本中，引入了一种 `V8Function` 类。`V8Function` 类是 `V8Object` 的子类，只要一个函数通过是 `getObject()` 调用被返回的话，那么返回的就是一个 `V8Function` 对象。`V8Function` 类有一个 `call()` 方法，可以使用它来在 Java 中调用 JavaScript 方法。

