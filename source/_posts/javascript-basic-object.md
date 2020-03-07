---
title: JavaScritp 基础 -- 类和对象
categories: JavaScript
comments: true
tags: [JavaScript]
description: 介绍 JavaScritp 的类和对象
date: 2018-5-6 10:00:00
---
## 概述

Javascript 是一种基于对象的语言，你遇到的所有东西几乎都是对象。但是，它又不是一种真正的面向对象编程（OOP）语言，因为它的语法中没有 class（类），没有 extend 关键字，这也就导致传统的面向对象编程方法无法直接使用。

## 类

虽然 Javascript 中没有 class 关键字，但是可以用一些变通的方法，模拟出"类"。

### 定义类

我们可以是用 function 来定义一个类，相当于一个构造函数，内部用 this 来指代当前实例对象。

```
      function Person(name, age, interests) {
        this.name = name
        this.age = age
        this.interests = interests
        this.run = function () {
          console.log(this.name + " run")
        }
      }
```

现在，我们可以通过 new 来创建对象了，然后通过对象访问它的属性和方法：

```
      var person = new Person("John", 10, ["music", 'swimming']);
      person.run()
      console.log(person.interests)
```

### 定义公有属性和私有属性

下面我们定义下面的类：

```
      function Person(n, age) {
        var name = n
        this.age = age
        this.run = function () {
          console.log(this.name + " run")
        }
      }
```

然后创建对象并访问它的属性：

```
      var person = new Person("John", 10);
      console.log(person.name)
      console.log(person.age)
```

你会发现第一个会打印 undefined，第二个 age 属性正常访问。
这是因为，用 var 定义的属性是私有的。我们需要使用this关键字来定义公有的属性。下面 name 在 run 方法中是正常访问的。

```
      function Person(n, age) {
        var name = n
        this.age = age
        this.run = function () {
          console.log(name + " run")
        }
      }
```

```
      var person = new Person("John", 10);
      person.run()
```

### 定义公有方法和私有方法

同样，用this指定的方法就是公有方法，用 var 定义的方法是私有方法。

```
      function Person(n, age) {
        var name = n
        this.age = age
        this.run = function () {
          console.log(name + " run")
        }
        var darw = function () {
          console.log(name + " draw")
        }
      }
```

### 构造函数

同样我们仍然可以模拟构造函数：

```
      function Person(n, age) {
        var name = n
        this.age = age
        this.run = function () {
          console.log(name + " run")
        }
        var init = function () {
          console.log(name + " init")
        }
        init()
      }
```

在 Person 的最后，调用了init函数，在创建了一个 Person 对象时，init 总会被自动调用，可以模拟我们的构造函数了。

### 动态添加类的属性和方法

```
      function Person(n, age) {
        var name = n
        this.age = age
        this.run = function () {
          console.log(name + " run")
        }
      }

      Person.id = 10
      Person.draw = function () {
        console.log(Person.id +" draw")
      }
      Person.draw()
```

上面的代码扩展了类的属性id和draw方法，然后调用他们。它们相当于静态属性和方法。
是属于类的，不是属于对象的。如果你通过对象来调用他们，你们会发现会报错。
那么如何为扩展的属性和方法让对象也拥有呢？下面我们介绍过 prototype 就会知道了。


## prototype

prototype 称为原型对象，所有的 JavaScript 对象都会从一个 prototype 中继承属性和方法。
前面我们介绍，为类动态添加的属性和方法只能通过类来访问，对象是无法访问的。除非我们在定义类的时候在构造器中去添加属性和方法。
使用 prototype 属性就可以给对象的构造函数添加新的属性：

```
      function Person(n, age) {
        var name = n
        this.age = age
        this.run = function () {
          console.log(name + " run")
        }
      }

      Person.prototype.id = 10
      Person.prototype.draw = function () {
        console.log(Person.id +" draw")
      }

      var person = new Person("John", 10);
      console.log(person.id)
      person.draw()
```

这个时候你就不能再通过 Person 类来访问 id 和 draw  方法了。

## 对象

在 JavaScript 中，几乎所有的事物都是对象。对象是一个包含相关数据和方法的集合，通常由一些变量和函数组成，我们称之为对象里面的属性和方法。

如果我们要把"属性"（property）和"方法"（method），封装成一个对象，甚至要从原型对象生成一个实例对象，我们应该怎么做呢？

### 使用构造器生成对象

这种方法前面已经介绍过：

```
      function Person(name, age, interests) {
        this.name = name
        this.age = age
        this.interests = interests
        this.run = function () {
          console.log(this.name + " run")
        }
      }
      var person = new Person("John", 10, ["music", 'swimming']);
      person.run()
      console.log(person.interests)
```

### 使用 Object 创建对象

JavaScript 中的所有对象都是继承自 Object 对象的。我们可以创建一个 Object 对象，然后为它添加响应的属性和方法。

```
      var person = new Object()
      person.name = 'John'
      person.run = function() {
        console.log(this.name + " run")
      }
```

```
      console.log(person.name)
      person.run()
```

### 使用 JSON 创建对象

下面的方法：

```
var person = {};
```

就生成了一个对象，但这是一个空对象，所以我们做不了更多的事情。
下面我们为对象添加一些属性和方法。一个对象由许多的成员组成，每一个成员都拥有一个名字（像上面的name、age），和一个值（如John、10）。每一个名字/值（name/value）对被逗号分隔开，并且名字和值之间由冒号（:）分隔，就像 JSON 数据格式一样。

```
      var person = {
        name: 'John',
        age: 10,
        interests: ['music', 'swimming'],
        run () {
          console.log(this.name + ' run')
        }
      }
```

那么现在我们就可以访问这些属性和方法了。

```
      console.log(person.age)
      console.log(person.interests[0])
      person.run()
```

我们同样也采用为空对象灵活地添加属性或者方法的形式创建对象。

```
      var person = {};
      person.id = 100
      person.sing = function () {
        console.log(this.id + ' sing')
      }
```