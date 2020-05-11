---
title: JavaScript 基础 -- export 和 import
categories: JavaScript
comments: true
tags: [JavaScript]
description: 介绍import和export的使用
date: 2018-5-10 10:00:00
---


## 概述

和 Java 函数的定义类似，JavaScript 函数是被设计为执行特定任务的代码块，它会在某代码调用它时被执行。函数是一段可以反复调用的代码块。函数还能接受输入的参数，不同的参数会返回不同的值。它体现的是一种封装的思想。
函数是 JavaScript 中一种复杂的数据结构，它既有对象的复杂度，又有函数独特的特性，尤其是函数中的 this 的指向。
函数是对象，它有自己的属性和方法。

## 函数定义

JavaScript 使用关键字 function 定义函数。函数定义可以用下面三种方式来实现：

### 使用 function 声明

```
function add(a,b){
	return a+b;
}
```

通过 `add(4,3)` 调用。

### 函数表达式

```
var fun = function (a, b) {
    return a + b
};
```

相当于把函数表达式存储在变量中，那么变量久可作为一个函数使用了。
通过 `fun(4,3)` 调用。
再看下面一种定义形式：

```
var fun = function add(a, b) {
    return a + b
};
```

上面的定义方法仍然只能通过 `fun(4,3)` 调用，而不能通过 `add(4,3)` 调用，但是在函数体内，可以通过 `add(4,3)` 来递归调用。

### Function构造函数

函数同样可以通过内置的 JavaScript 函数构造器（Function()）定义。

```
var myFunction = new Function("a", "b", "return a * b");
var x = myFunction(4, 3);
```

其实 new 关键字也是可以省略的，这种声明函数的方式非常不直观，较少使用。


### 箭头函数

ES6 提供了 lambda 表达式定义函数的方法。

```
var myFunction = (a,b) => {return a*b};
var x = myFunction(4, 3);
```

当箭头函数有一个参数的时候，参数可以不加括号，没有参数的时候就必须要加。

```
var myFunction = a => {return a};
var x = myFunction(4);
```

```
var myFunction = () => {return "Test"};
var x = myFunction();
```

如何函数体就一行，且不用return显式返回，函数体可以不适用花括号。如果函数体不止一行，应该用花括号括起来，这时就要显示地返回。

```
var myFunction = () => console.log ("Test");
var x = myFunction();
```

## 立即执行函数

立即执行函数也叫自调用函数，是一个在定义的时候就立即执行的JavaScript函数。
如果表达式后面紧跟 () ，则会自动调用。通过添加括号，来说明它是一个函数表达式。

```
(function () {
    console.log("Hello! 我是自己调用的")
})();
```

也可以加参数：

```
var str = "Test";
(function (str) {
    console.log("Hello! 我是自己调用的 "+str)
})(str);
```

## 函数属性

前面说过，函数是对象，有自己的属性和方法。

### name

name属性返回函数的名字。

```
// 函数声明的方式定义函数
function f1() {}
f1.name // "f1"

// 函数表达式的方式：1、等号右边是个匿名函数
var f2 = function () {};
f2.name // "f2"

// 函数表达式的方式：2、等号右边是个具名函数（function关键字后面又名称）
var f3 = function myName() {};
f3.name // "myName"
```

### length

函数的length属性返回函数在定义时，预期要传入的参数个数，也就是定义时()中的变量个数，也就是形参的个数

```
function f1() {}
f1.length //0

function f2(ac_a,ac_b,ac_c){}
f2.length //3
```

### arguments对象

由于JavaScript允许函数有不定数目的参数，所以需要一种机制，可以再函数体内部读取到所有的实际传入的参数（实参），这样arguments对象就诞生了。
arguments是一个运行时的对象，只有在函数运行的时候才会产生。它的数据方式访问类似数组的形式：arguments[0]，第一个实参，依次类推；它也有一个length属性，记录实参个数

```
function arg(arg1,arg2){
  console.log('arguments.length:'+arguments.length)
  console.log(arguments[0])
  console.log(arguments[1])
  console.log(arguments[2])
}

arg(1) 
// arguments.length:1
// 1
// undefined
// undefined
arg(2,3,5)
// arguments.length:3
// 2
// 3
// 5
```

在正常模式下，arguments是可以被修改的，但是在严格模式下，arguments是只读的。
arguments这种像数组的对象，我们通常称为伪数组。
arguments对象还有个callee属性，返回它所对应的原函数。

```
var f = function () {
  console.log(arguments.callee === f);
}

f() // true
```

## 作用域

### 函数提升

提升（Hoisting）是 JavaScript 默认将当前作用域提升到前面去的的行为。
提升（Hoisting）应用在变量的声明与函数的声明。
因此，函数可以在声明之前调用：

```
myFunction(5);

function myFunction(y) {
    return y * y;
}
```

使用表达式定义函数时无法提升。
因为函数提升的存在，因此，当一个函数被多次重复声明的时候，后面的声明会覆盖前面的声明。

### 变量提升

在全局作用域存在变量提升和函数提升，在函数作用域同样存在。var关键字声明的变量，不论在什么位置，变量声明都会被提升到函数体的头部。

```
function foo(x) {
  if (x > 100) {
    var tmp = x - 100;
  }
}

// 等同于
function foo(x) {
  var tmp;
  if (x > 100) {
    tmp = x - 100;
  };
}
```

通常在实际工作中，我们推荐在代码的一开始集中定义变量，就是因为这个原因，让代码一目了然。

## 函数参数

https://www.runoob.com/js/js-function-parameters.html