
---
title: Java 基础 -- 范型
categories: Java
comments: true
tags: [Java范型]
description: 本文介绍 Java 范型的特点和使用方法
date: 2014-2-14 10:00:00
---

## 概述

所谓范型，就是允许在定义类、接口、方法时使用类型形参，这个类型形参将在声明变量、创建对象、调用方法时动态地指定（即传入实际的类型参数，也可称为类型实参）。
范型可以减少强制类型的转换，可以规范集合的元素类型，还可以提高代码的安全性和可读性，正是因为有这些优点，自从 Java 引入范型后，项目的编码规则上便多了一条：优先使用范型。


## 范型的使用

### 类型通配符

类型通配符的形式有 `<?>`、 `<? extends Class>` 和 `<? super Class>`

#### 基本使用

为了表示各种范型 List 的父类，我们需要使用类型通配符，类型的通配符就是一个问号 ?，将它作为类型实参传给 List集合：`List<?>`，就标示未知类型元素的 List，它的元素类型可以匹配任何类型。
比如：

```
    public void testGeneric() {
        List<String> name = new ArrayList<String>();
        List<Integer> age = new ArrayList<Integer>();
        List<Shape> shape = new ArrayList<Shape>();

        name.add("icon");
        age.add(18);
        shape.add(new Shape());

        getData(name);
        getData(age);
        getData(shape);
    }

    public static void getData(List<?> data) {
        Log.e("Test","data :" + data.get(0));
    }

    public static class Shape {
        @Override
        public String toString() {
            return "Shape";
        }
    }
```

编译没问题，运行结果：

```
E/Test: data :icon
E/Test: data :18
E/Test: data :Shape
```

因为 `getData()` 方法的参数是 `List` 类型的，所以 name，age，shape 都可以作为这个方法的实参，这就是通配符的作用。

> 上面程序中使用的 `List<?>`，其实这种写法可以适用于任何支持范型声明的接口和类，比如 `Set<?>`、`Map<?,?>`等。

#### 设定类型通配符的上限

假设有下面的使用场景，我们不想使 `List<?>` 使任何范型 List 的父类，只想表示它是某一类范型List的父类，这时候我们就要限定通配符的上限了。
`<? extends Class>` 表示该通配符所代表的类型是 Class 类型本身或者它的子类。
把上面的 `getData` 方法修改一下：

```
    public void testGeneric() {
        List<String> name = new ArrayList<String>();
        List<Integer> age = new ArrayList<Integer>();
        List<Shape> shape = new ArrayList<Shape>();
        List<Circle> circle = new ArrayList<Circle>();

        name.add("icon");
        age.add(18);
        shape.add(new Shape());

        getData(name);  // 编译报错
        getData(age);   // 编译报错
        getData(shape); // 编译通过
        getData(circle);// 编译通过
    }

    public static void getData(List<? extends Shape> data) {
        Log.e("Test","data :" + data.get(0));
    }

    public static class Shape {
        @Override
        public String toString() {
            return "Shape";
        }
    }

    public static class Circle extends Shape {
        @Override
        public String toString() {
            return "Circle";
        }
    }
```

前面两个用法就会报错：

```
Error:(74, 17) 错误: 不兼容的类型: List<String>无法转换为List<? extends Shape>
Error:(75, 17) 错误: 不兼容的类型: List<Integer>无法转换为List<? extends Shape>
```

#### 设定类型通配符的下限

`<? super Class>` 表示该通配符所代表的类型是 Class 类本身或者它的父类。
把上面的 `getData` 方法再修改一下：

```
    public static void getData(List<? super Shape> data) {
        Log.e("Test","data :" + data.get(0));
    }
```

```
        getData(name);  // 编译报错
        getData(age);   // 编译报错
        getData(shape); // 编译通过
        getData(circle);// 编译报错
```

### 泛型方法

范型方法就是在声明方法时定义一个或多个类型形参，该方法在调用时可以接收不同类型的参数。根据传递给泛型方法的参数类型，编译器适当地处理每一个方法调用。
范型方法的类型作用域是整个方法。
范型方法的用法格式如下：

```
修饰符 <T, S> 返回值类型 方法名 (形参列表) {

}

```

下面是定义泛型方法的规则：

 - 所有泛型方法声明都有一个类型参数声明部分（由尖括号分隔），该类型参数声明部分在方法返回类型之前（在上面例子中的`<T, S>`）。
 - 每一个类型参数声明部分包含一个或多个类型参数，参数间用逗号隔开。一个泛型参数，也被称为一个类型变量，是用于指定一个泛型类型名称的标识符。
 - 类型参数可以用来声明方法参数。
 - 类型参数能被用来声明返回值类型。

先来看一个范型参数来声明方法参数的例子：

```
    public <T> String getData(T t) {
        return String.valueOf(t);
    }
```

再来看一个范型参数来声明返回值类型的例子：

```
    private Map<String, Object> mDatas = new ArrayMap<>();

    public <T> T getData(String name) {
        return (T) mDatas.get(name);
    }
```

### 泛型类

泛型类的声明和非泛型类的声明类似，除了在类名后面添加了类型参数声明部分。
和泛型方法一样，泛型类的类型参数声明部分也包含一个或多个类型参数，参数间用逗号隔开。
一个泛型参数，也被称为一个类型变量，是用于指定一个泛型类型名称的标识符。因为他们接受一个或多个参数，这些类被称为参数化的类或参数化的类型。

```
    public class Box<T> {
        private T t;
        public void add(T t) {
            this.t = t;
        }
        public T get() {
            return t;
        }
    }
```

### 限定类型参数上限

Java 范型不仅允许在使用通配符形参时设定上限，而且也可以在定义类型参数时设定上限，用于表示传给该类型形参的实际类型要么是该类型上限，要么使该类型的子类。
这种做法可以用在范型方法和范型类中。
用法：`<U, T extends Class1>`

```
    public <T extends Shape & Serializable> String getData(T t) {
        return String.valueOf(t);
    }

    public static class Shape {
        @Override
        public String toString() {
            return "Shape";
        }
    }
```

形如 `<U, T extends Class1 & Interface1>` 表示 T 是继承了 Class1 的类以及实现了 Interface1，后面的接口可以有多个，因为 Java 是单继承，因此父类只能有1个。类要写在接口的前面。

## 范型的特点

### 范型是类型擦除的

Java 的范型在编译器有效，在运行期被删除，也就是说所有的反省参数类型在编译期后都会被清除掉。
下面看一段代码：

```
    public void listMethod(List<String> strings) {

    }

    public void listMethod(List<Integer> strings) {

    }
```

这段代码是否能编译呢？
事实上，这段代码时无法编译的，编译时报错信息如下：

```
`listMethod(List<String>)` clashes with `listMethod(List<Integer>)`; both methods have same erasure 
```

此错误信息是说 `listMethod(List<String>)` 方法在编译时擦除类型后的方法是 `listMethod(List<E>)`，它与另一个方法相冲突。这就是 Java 范型擦除引起的问题：在编译后所有的范型类型都会做相应的转化：
转化规则如下：

 - `List<String>`、`List<Integer>`、`List<T>`擦除后的类型为`List`
 - `List<String>[]`擦除后的类型为`List[]`
 - `List<? extends E>、List<? super E>` 擦除后的类型为 `List<E>`
 - `List<T extends Serializable & Cloneable>` 擦除后为 `List<Serializable>`

范型的擦除还表现在当把一个具有范型信息的对象赋给另一个没有范型信息的变量时，所有再尖括号之间的类型信息都将被扔掉。
比如：一个 `List<String>` 类型被转换为 `List`，则该 `List` 对集合元素的类型检查变成了类型变量的上限（即 Object）。
下面用一个例子来示范这种擦除：

```
    public void testGenericErasure() {
        // 传入 Integer 作为类型形参的值
        Box<Integer> box = new Box<>(8);
        // box 的 get 方法返回 Integer 对象
        Integer size = box.get();
        // 把 box 对象赋值给b变量，此时会丢失<>里的类型信息
        Box b = box;
        //这个代码会引起编译错误
        Integer size1 = b.get();
        // b只知道size的类型是Number，但具体是Number的哪个子类就不清楚了。
        //下面的用法是正确的
        Number size2 = b.get();
    }
    
    public class Box<T extends Number> {

        private T size;
        public Box(T size) {
            this.size = size;
        }

        public void add(T size) {
            this.size = size;
        }

        public T get() {
            return size;
        }
    }
```

类型转换：

```
    public void testGenericConvert() {
        List<Integer> li = new ArrayList<>();
        li.add(8);
        // 类型擦除
        List list = li;
        // 类型转换
        List<String> ls = list;
        // 下面的代码会引起运行时异常
        // Caused by: java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
        Log.e("Test",ls.get(0));
    }
```

明白了这些，对下面的代码就容易理解了：

```
        List<String> stringList = new ArrayList<>();
        List<Integer> integerList = new ArrayList<>();

        Log.e("Test",""+stringList.getClass().equals(integerList.getClass()));
```

返回结果为 true。 `List<String>` 和 `List<Integer>` 擦除后的类型都是 List，没有任何区别。
之所以设计成可擦除的，有下面两个原因：

 - 避免JVM大换血。由于范型是Java5以后才支持的，如果JVM也把范型类型延续到运行期，那么JVM就需要进行大量的重构工作了。也就是说，Java 中的泛型机制其实就是一颗语法糖，并不涉及JVM的改动。
 - 版本兼容问题。在编译器擦除可以更好地支持原生类型，在Java5或者Java6平台上，即使声明一个List这样的原生类型也是支

### 不能创建一个范型类型实例

如果 T 是一个类型变量，那么下面的语句是非法的：

```
T obj = new T();
```

T 由它的限界代替，这可能是 Object，或者是抽象类，因此对 `new T()` 的调用没有意义。

### 不能初始化范型数组

数组元素的类型不能包含类型变量或者类型形参，除非是无上限的类型通配符。但是可以声明元素类型包含类型变量或类型形参的数据。
也就是说，下面的代码是OK的：

```
List<String>[] stringList ;
```

```
    public class Box<T extends Number> {

        private List<T> list = new ArrayList<T>();
    }
```

下面的代码，等号后面的代码是非法的：

```
List<String>[] stringList = new ArrayList<String>[10];
```

### 基本类型不能做类型参数

因此，`List<int>` 是非法的，我们必须使用包装类。

### static 的语境不能引用类型变量

在一个范型类中，static 方法和 static 域均不可以引用类的类型变量，因为类型擦除后类型变量就不存在了。而且，static 域在该类的诸范型实例之间是共享的。因此，static 的语境不能引用类型变量。




