---
title: Java 设计模式 -- 职责链模式
categories: Java
comments: true
tags: [Java]
description: 介绍职责链模式的实现
date: 2014-11-16 10:00:00
---


## 概述

职责链模式：使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这个对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。

使用场景：

 - 多个对象可以处理统一请求，但具体谁处理在运行时动态决定。
 - 在请求的处理者不明确的情况下，向多个对象的一个提交请求。
 - 需要动态指定一组对象处理请求。

比较经典的场景就是审批流程的实现。比如我们审批一批资金，在某个额度一下主管有权限审批，否则就要报送经理，超出经理的审批额度后又需要报送总监，一次类推。
接下来，我们就通过 Java 代码来实现职责链模式。

## 代码实现

实现职责链模式，我们先抽象出下面三个类：

 - Request：请求数据类，包含了请求的一些信息。
 - Handler：抽象处理者角色，声明一个处理请求的方法，并保持对下一个处理节点Handler对象的引用。
 - ConcreteHandler: 具体的处理者，对请求进行处理，如果不处理就讲请求转发给下一个节点上的处理对象。

```
public class Request {
    public int size;
    public boolean hasDone;
}
```

```
public abstract class Handler {
    private Handler next;
    public void setNext(Handler handler){
        this.next = handler;
    }

    public Handler getNext() {
        return next;
    }

    abstract public void handleRequest(Request request);
}
```

```
public class ConcreteHandlerA extends Handler {
    @Override
    public void handleRequest(Request request) {
        if (request.size > 0 && request.size < 10) {
            // do something
            request.hasDone = true;
            Log.e("Test","done by ConcreteHandlerA");
        } else  {
            Log.e("Test","read by ConcreteHandlerA");
            if (getNext() != null) {
                getNext().handleRequest(request);
            }
        }
    }
}

public class ConcreteHandlerB extends Handler{
    @Override
    public void handleRequest(Request request) {
        if (request.size >= 10 && request.size < 20) {
            // do something
            request.hasDone = true;
            Log.e("Test","done by ConcreteHandlerB");
        } else  {
            Log.e("Test","read by ConcreteHandlerB");
            if (getNext() != null) {
                getNext().handleRequest(request);
            }
        }
    }
}

public class ConcreteHandlerC extends Handler{
    @Override
    public void handleRequest(Request request) {
        if (request.size >= 20 && request.size < 30) {
            // do something
            request.hasDone = true;
            Log.e("Test","done by ConcreteHandlerC");
        } else  {
            Log.e("Test","read by ConcreteHandlerC");
            if (getNext() != null) {
                getNext().handleRequest(request);
            }
        }
    }
}
```

```
        Request request = new Request();
        request.size = 25;
        ConcreteHandlerA concreteHandlerA = new ConcreteHandlerA();
        ConcreteHandlerB concreteHandlerB = new ConcreteHandlerB();
        ConcreteHandlerC concreteHandlerC = new ConcreteHandlerC();

        concreteHandlerA.setNext(concreteHandlerB);
        concreteHandlerB.setNext(concreteHandlerC);

        concreteHandlerA.handleRequest(request);
```

上面实现的 Request 仅用一个数字来指定不同的处理者，处理规则也比较简单，在复杂的情形下，责任链中的请求处理规则是不尽相同的，这种时候需要对请求进行封装，同时对请求的处理规则也进行一个封装。
比如，抽象一个 Request 类，在 Handler 中定义抽象方法来实现处理规则。