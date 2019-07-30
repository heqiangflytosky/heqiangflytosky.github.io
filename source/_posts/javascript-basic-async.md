---
title: JavaScript 基础 -- async/await 关键字
categories: JavaScript
comments: true
tags: [JavaScript]
description: 介绍 async 函数的使用
date: 2018-5-15 10:00:00
---

## 概述

async 关键字用来标识一个异步函数，它是 ES7 引入的语法，用来代替异步编程使用的 Callback 与 Promise，可以使代码更加整洁优雅。

## 基本用法

声明 async 函数很简单，直接在方法前面加上 async 就行了。
async 函数返回一个 Promise 对象，可以使用then方法添加回调函数。我们先来做个测试：

```
    test() {
      this.testAsync().then((value) => {
        console.log("value = "+value)
      },(error) => {
        console.log("error = "+error)
      })
    },
    async testAsync() {
      //throw new Error('test')
      console.log("testAsync before return")
      return "hello async"
    }
```

这里大家可能有个疑问，这里返回个 Promise 有什么用呢？下面再来介绍一下 await。
await 顾名思义就是等待的意思，等待后面的异步方法的执行结果。下面来通过例子来介绍。

```
    test() {
      this.testAsync()
      console.log("test end")
    },
    async testAsync() {
      await this.testPromise()
      console.log("testAsync end")
    },
    testPromise() {
      return new Promise((resolve,reject) => {
        setTimeout(function () {
          console.log("testPromise done")
          resolve('done')
        },1000)
      })
    }
```

在上面的代码中，testPromise() 返回了一个 Promise对象，异步执行一些耗时操作，1000ms 后打印 testPromise done。
testAsync() 方法中，`await this.testPromise()` 表示阻塞在这里等待testPromise的执行结果。注意，这里并不是说等待testPromise这个异步方法执行完毕返回，而是要等待异步的执行结果，也就是异步方法体执行完毕。
看输出：

```
16:04:43.528  test end
16:04:44.534  testPromise done
16:04:44.536  testAsync end
```

await 关键字必须要在 async 函数中使用，否则会报错。
上面我们介绍的是 await 后面跟一个返回 Promise对象的函数，那么可以加普通的函数吗？答案是可以的。

```
    test() {
      this.testAsync()
      console.log("test end")
    },
    async testAsync() {
      await this.testTimeout()
      console.log("testAsync end")
    },
    testTimeout() {
      setTimeout(function () {
          console.log("setTimeout done")
      },1000)
      console.log("testTimeout done")
    }
```

```
16:10:09.940 testTimeout done
16:10:09.941 test end
16:10:09.942 testAsync end
16:10:10.946 setTimeout done
```

这种情况下 await 仅仅等待函数返回，而不是异步执行的结果。

## 参考文献

[阮一峰：async 函数](http://es6.ruanyifeng.com/#docs/async)