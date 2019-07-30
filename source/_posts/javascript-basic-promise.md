---
title: JavaScript 基础 -- Promise
categories: JavaScript
comments: true
tags: [JavaScript]
description: 介绍 Promise 的使用
date: 2018-5-12 10:00:00
---

## 概述

Promise 是从 ES6 开始提供的异步编程解决方案，在此之前需要使用回调函数和事件的方式来完成。Promise 比传统的回调函数和事件更合理和强大。

## 特点

 - Promise 对象一旦创建就会立即执行，不管有没有then函数来处理执行结果

## 基本用法

我们先来看一下 Promise 对象的创建方法：

```
      var promise = new Promise(function(resolve,reject){
        if(sucess) {
          resolve(value)
        } else {
          reject(error)
        }
      })
```

Promise构造函数接收一个函数作为参数，该函数有两个参数分别是resolve和reject，他们是两个函数，由Javascript引擎提供，不用自己部署。
resolve方法的作用是在异步操作成功时调用，并将异步操作的结果作为参数传递到下游进行处理。
reject方法的作用是在异步操作失败时调用，并将异步操作报出的错误作为参数传递到下游进行处理。
Promise实例生成以后，可以用then方法分别指定resolved状态和rejected状态的回调函数。

```
      promise.then(function(value){
        // do success
      },function(error){
        // do error
      })
```

then方法可以接受两个回调函数作为参数，第一个回调函数是异步操作执行成功时调用，第二个回调函数是异步操作执行失败时调用。其中，第二个函数是可选的，不一定要提供，这两个函数都接受Promise对象传出的值作为参数。
下面介绍一个简单的例子：

```
    testPromise() {
      console.log("##start")
      
      new Promise((resolve, reject) => {
        setTimeout(function() {
          console.log('do something')
          resolve({usetime:100,status:1})
          //reject('time out')
        },500)
      }).then((value) => {
          console.log('done, value='+JSON.stringify(value))
        },(error) => {
          console.log('reason='+error)
      })
      
      console.log("##end")
    }
```

resolve 还可以是一个 Promise 实例，在这种个情况下，两个 Promise 并行执行，后一个 Promise 的 resolve 方法将前一个 Promise 作为参数，也就是后一个的操作结果是前一个异步操作。

```
    testPromise() {
      console.log("##start")

      const promise1 =  new Promise((resolve, reject) => {
        console.log('the first stage')
        setTimeout(function() {
          console.log('the first stage done')
          resolve({usetime:100,status:1})
          //reject("time out")
        },1000)
      })
      const promise2 = new Promise((resolve,reject) => {
        console.log('the second stage')
        setTimeout(function() {
          console.log('the second stage done')
          resolve(promise1)
        },500)
      })

      promise2.then((value) => {
        console.log('done, value='+JSON.stringify(value))
      },(error) => {
        console.log('fail, error='+JSON.stringify(error))
      })

      console.log("##end")
    }
```

注意，这时 promise1 的状态就会传递给 promise2，也就是说，promise1的状态决定了 promise2 的状态。如果 promise1 的状态是未执行完状态，那么promise2的回调函数就会等待 promise1 的状态改变；如果 promise1 的状态已经是 resolved或者 rejected，那么 promise2 的回调函数将会立刻执行。

## then 方法

语法：`Promise.prototype.then(onFulfilled, onRejected);`

 - onFulfilled:当前实例变成fulfilled状态时，该参数作为回调函数被调用
 - onRejected:当前实例变成reject状态时，该参数作为回调函数被调用。

then 方法返回的是一个新的 Promise 对象，那么基于这样的特性，我们可以很方便的采用链式操作，也就是 then 方法后面再接着调用 then 方法，第一个回调函数完成以后，会将返回结果作为参数，传入第二个回调函数：

```
    testPromise() {
      console.log("##start")

      const promise1 =  new Promise((resolve, reject) => {
        console.log('the first stage')
        setTimeout(function() {
          console.log('the first stage done')
          resolve({usetime:100,status:1})
        },1000)
      })

      promise1.then((value) => {
        console.log('done 1, value='+JSON.stringify(value))
        return "stage 2"
      },(error) => {
        console.log('fail, error='+JSON.stringify(error))
      }).then((value) => {
        console.log('done 2, value='+JSON.stringify(value))
      })

      console.log("##end")
    }
```

有时我们有这样的使用场景：多个 Promise 异步操作配合使用，后一个需要等待前一个异步操作的执行结果。
下面可以用下面方式实现：

```
    testPromise() {
      console.log("##start")
      
      new Promise((resolve, reject) => {
        setTimeout(function() {
          console.log('the first stage')
          resolve({usetime:100,status:1})
        },500)
      }).then((value) => {
        console.log('the first stage done, value='+JSON.stringify(value))
        return new Promise((resolve,reject) => {
          setTimeout(function() {
            console.log('the second stage')
            resolve({usetime:200,status:1})
          },500)
        })
      }).then((value) =>{
        console.log('the second stage done, value='+JSON.stringify(value))
      })
      
      console.log("##end")
    }
```

这种方法的两个 Promise 是串行执行，后一个 Promise 的执行需要依赖前一个 Promise 的执行结果。后面一个 then 方法的value是第二个 Promise resolve 方法传递的结果。

如果想实现两个返回 Promise 方法的链式操作的话，要这样写：

```
  testPromise() {
    console.log("##11start")
    var root = this;
    this.promise1().then((value) => {
      console.log('the first stage done')
      return root.promise2();
    }).then((value) =>{
      console.log('the second stage done, value='+JSON.stringify(value))
    })
    console.log("##end")
  },
  
  promise1() {
    return new Promise((resolve, reject) => {
      setTimeout(function () {
        console.log('the first stage')
        resolve({ usetime: 100, status: 1 })
      }, 500)
    })
  },
  promise2() {
    return new Promise((resolve, reject) => {
      setTimeout(function () {
        console.log('the second stage')
        resolve({ usetime: 200, status: 1 })
      }, 500)
    })
  },
```

注意这里必须是把 promise2() 作为 then 回调方法的返回值。这里我踩过坑。刚开始接触 Promise 时用了下面的写法，把 promise2() 作为then方法的参数，这样是不对的。

```
  testPromise() {
    console.log("##11start")
    var root = this;
    this.promise1().then(root.promise2())
    console.log("##end")
  },
```

## catch 方法

catch 方法用于指定发生错误时的回调函数。它和 .then(null, rejection)或.then(undefined, rejection) 方法是等价的。

```
    testPromise() {
      console.log("##start")
      
      new Promise((resolve, reject) => {
        throw new Error('test error')
        //reject("time out")
      }).then((value) => {
          console.log('done, value='+JSON.stringify(value))
        }).catch((error) => {
        console.log('catch ='+error)
      })
      
      console.log("##end")
    }
```
reject 方法的作用，等同于抛出错误，也可以被 catch 方法捕获。
同样 throw 抛出的异常也可以被then方法指定的rejected状态的回调函数捕获，异常被拦截后，catch 方法就不会再收到异常。
另外，then方法指定的回调函数，如果运行中抛出错误，也会被catch方法捕获。
catch 方法和then方法指定的rejected状态的回调函数的区别就是
Promise 对象的错误具有“冒泡”性质，会一直向后传递，直到被捕获为止。

## 参考文献

[阮一峰：Promise 对象](http://es6.ruanyifeng.com/#docs/promise)