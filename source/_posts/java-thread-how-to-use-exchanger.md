---
title: Java 线程 -- Exchanger
categories: Android 线程
comments: true
tags: [Android 线程]
description: 本文介绍 Exchanger 的使用
date: 2015-9-10 10:00:00
---


## 概述

Exchanger 按字面意思理解是交换的意思。在Java并发编程中，引入了 `Exchanger` 类来实现不同线程之间数据的就交换。
它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。两个线程各自调用 `exchange` 方法进行交换，当线程 A 调用 `Exchange` 对象的 `exchange` 方法后，它会陷入阻塞状态，直到线程 B 也调用了 `exchange` 方法，然后以线程安全的方式交换数据，之后线程 A 和 B 继续运行。
`exchange` 方法有两个重载实现：

 - `V exchange(V x)`：交换数据，返回从其他线程交换回来的数据。
 - `V exchange(V x, long timeout, TimeUnit unit)`：可以设置超时时间，如果超过这个时间还没有与其他线程交换数据，则会抛出 `TimeoutException` 异常。

这个类在遇到类似生产者和消费者问题时，是非常有用的。只是 `Exchange` 类只能同步2个线程，所以你只能在你的生产者和消费者问题中只有一个生产者和一个消费者时使用这个类。
`Exchanger`可以用于遗传算法，遗传算法里需要选出两个人作为交配对象，这时候会交换两人的数据，并使用交叉规则得出2个交配结果。
它还可以用于校对工作，比如银行需要将银行流水录入电子银行流水，为了避免错误，采用AB两人进行录入，录入之后再加载两人的数据进行校对，看录入是否一致。

## 应用

```
    public static void testExchanger() {
        Exchanger<String> exchanger = new Exchanger<>();
        new Tomato(exchanger).start();
        new Potato(exchanger).start();
    }

    public static class Tomato extends Thread {
        private Exchanger mExchanger;
        public Tomato (Exchanger exchanger) {
            mExchanger = exchanger;
            setName("Tomato");
        }
        @Override
        public void run() {
            Log.e(getName() + "Test","enter ");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.e("Test",getName() + " after sleep ");
            try {
                Log.e("Test",getName() + " exchanger =  "+mExchanger.exchange("Tomato"));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static class Potato extends Thread {
        private Exchanger mExchanger;
        public Potato (Exchanger exchanger) {
            mExchanger = exchanger;
            setName("Potato");
        }
        @Override
        public void run() {
            Log.e(getName() + "Test","enter ");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.e("Test",getName() + " after sleep ");
            try {
                Log.e("Test",getName() + " exchanger =  "+mExchanger.exchange("Potato"));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
```

执行结果：

```
14:04:18.911 E/TomatoTest: enter 
14:04:18.911 E/PotatoTest: enter 
14:04:19.911 E/Test: Potato after sleep 
14:04:20.911 E/Test: Tomato after sleep 
14:04:20.911 E/Test: Tomato exchanger =  Potato
14:04:20.911 E/Test: Potato exchanger =  Tomato
```

## 实现原理

