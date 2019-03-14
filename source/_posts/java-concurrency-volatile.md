---
title: Java 并发编程 -- volatile 关键字
categories: Java 并发
comments: true
tags: [Java 并发]
description: 介绍 volatile 的用法
date: 2015-7-5 10:00:00
---

## 概述

Java 语言中的 volatile 关键字修饰的变量相当于声明该变量是易变的，可能被其他线程修改，因此，对该变量的读取时都会读取它的最新值，这就是说线程能够自动发现 volatile 变量的最新值。（对这个不太理解的同学可以先了解一下 Java 内存模型的知识）
volatile 可以被看作是Java虚拟机提供的最轻量级的同步机制。与 synchronized 块相比，volatile 变量所需的编码较少，并且运行时开销也较少，但是它所能实现的功能也仅是 synchronized 的一部分。
本文介绍了几种有效使用 volatile 变量的模式，并强调了几种不适合使用 volatile 变量的情形。

## volatile 型变量的特性

被volatile修饰的共享变量，具有了以下两点特性：

 - 保证了不同线程对该变量操作的内存可见性；
 - 禁止指令重排序；

这里的“可见性”是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。而普通变量不能做到这一点，普通变量的值在线程间传递需要通过主内存来完成。
volatile 变量只能保证可见性，不能保证原子性。
volatile 型变量的第二个特性是禁止指令重排序，普通变量仅仅会保证该方法的执行过程中所有依赖赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的顺序与程序代码的执行顺序一致。

## volatile 和 synchronized 区别

 - volatile 仅能修饰变量；synchronized 可以作用于变量、方法、对象和类。
 - volatile 仅能保证可见性；synchronized 可以保证可见性和原子性。
 - volatile不会造成线程的阻塞；synchronized可能会造成线程的阻塞。
 - volatile 变量所需的编码较少，并且运行时开销也较少。在某些情况下，如果读操作远远大于写操作，volatile 变量还可以提供优于锁的性能优势。 

## volatile 使用场景

volatile 变量可用于提供线程安全，在有限的一些情形下使用 volatile 变量替代锁。要使 volatile 变量提供理想的线程安全，必须同时满足下面两个条件：

 - 对变量的写操作不依赖于当前值，或者是能够确保只有单一的线程修改变量的值。
 - 该变量没有包含在具有其他变量的不变式中。

上面条件限制了：多个变量之间或者某个变量的当前值与修改后值之间没有约束。同时表明，可以被写入 volatile 变量的这些有效值独立于任何程序的状态，包括变量的当前状态。
因此，单独使用 volatile 还不能用作线程安全计数器、互斥锁或任何具有与多个变量相关的不变式的类（例如 “start <=end”）。 
虽然增量操作（x++）看上去类似一个单独操作，实际上它是一个由读取－修改－写入操作序列组成的组合操作，必须以原子方式执行，而 volatile 不能提供必须的原子特性。实现正确的操作需要使 x 的值在操作期间保持不变，而 volatile 变量无法实现这点。（然而，如果将值调整为只从单个线程写入，那么可以忽略第一个条件。） 

### 状态标记位

比如我们想使用一个布尔值标记某个状态，用于指示发生了一个重要的一次性事件，比如完成初始化或退出某个循环条件等。 
这种类型的状态标记的一个公共特性是：通常只有一种状态转换，标志从 false 转换为 true，然后程序停止。这种模式也是可以扩展到来回转换的状态标志的（从 false 到 true，再转换到 false）。


### 一次性安全发布

比如单例模式：

```
public class Manager {
    private static volatile Manager sInstance;

    public static Manager getInstance() {
        if(sInstance == null) {
            synchronized (Manager.class) {
                if(sInstance == null) {
                    sInstance = new Manager();
                }
            }
        }
        return sInstance;
    }
}
```

### 独立观察

这种情景是：定期 “发布” 观察结果供程序内部使用。例如，假设有一种环境传感器能够感觉环境温度，一个后台线程可能会每隔几秒读取一次该传感器，并更新到 volatile 变量。然后，其他线程可以读取这个变量，从而随时能够看到最新的温度值。
该模式是前面模式的扩展。将某个值发布以在程序内的其他地方使用，但是与一次性事件的发布不同，这是一系列独立事件。这个模式要求被发布的值是有效不可变的 —— 即值的状态在发布后不会更改。使用该值的代码需要清楚该值可能随时发生变化。

### “volatile bean” 模式

```
@ThreadSafe
public class Person {
    private volatile String firstName;
    private volatile String lastName;
    private volatile int age;
 
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public int getAge() { return age; }
 
    public void setFirstName(String firstName) { 
        this.firstName = firstName;
    }
 
    public void setLastName(String lastName) { 
        this.lastName = lastName;
    }
 
    public void setAge(int age) { 
        this.age = age;
    }
}
```

### 开销较低的读－写锁策略

 目前为止，您应该了解了 volatile 的功能还不足以实现计数器。因为 ++x 实际上是三种操作（读、添加、存储）的简单组合，如果多个线程凑巧试图同时对 volatile 计数器执行增量操作，那么它的更新值有可能会丢失。
然而，如果读操作远远超过写操作，您可以结合使用内部锁和 volatile 变量来减少公共代码路径的开销。清单 6 中显示的线程安全的计数器使用 synchronized 确保增量操作是原子的，并使用 volatile 保证当前结果的可见性。如果更新不频繁的话，该方法可实现更好的性能，因为读路径的开销仅仅涉及 volatile 读操作，这通常要优于一个无竞争的锁获取的开销。

```
@ThreadSafe
public class CheesyCounter {
    // Employs the cheap read-write lock trick
    // All mutative operations MUST be done with the 'this' lock held
    @GuardedBy("this") private volatile int value;
 
    public int getValue() { return value; }
 
    public synchronized int increment() {
        return value++;
    }
}
```

之所以将这种技术称之为 “开销较低的读－写锁” 是因为您使用了不同的同步机制进行读写操作。因为本例中的写操作违反了使用 volatile 的第一个条件，因此不能使用 volatile 安全地实现计数器 —— 您必须使用锁。然而，您可以在读操作中使用 volatile 确保当前值的可见性，因此可以使用锁进行所有变化的操作，使用 volatile 进行只读操作。其中，锁一次只允许一个线程访问值，volatile 允许多个线程执行读操作，因此当使用 volatile 保证读代码路径时，要比使用锁执行全部代码路径获得更高的共享度 —— 就像读－写操作一样。然而，要随时牢记这种模式的弱点：如果超越了该模式的最基本应用，结合这两个竞争的同步机制将变得非常困难。 

## 使用建议

 - 在多线程需要访问的变量上使用 volatile
 - 要访问的变量已在synchronized代码块中，或者为常量时，没必要使用volatile
 - volatile 屏蔽掉了JVM中必要的代码优化，所以在效率上比较低，因此一定在必要时才使用此关键字

## 注意事项

### 修饰数组

volatile 可以修饰数组，不过只是一个指向数组的引用，而不是整个数组。如果改变引用指向的数组，将会受到volatile的保护，但是如果多个线程同时改变数组的元素，volatile 修饰符就不能起到之前的保护作用了。

## 参考文献

[正确使用 Volatile 变量](https://www.ibm.com/developerworks/cn/java/j-jtp06197.html)
<!--  
https://www.toutiao.com/i6665566310002328077/?iid=65146341241&app=news_article&group_id=6665566310002328077&timestamp=1552268611
-->