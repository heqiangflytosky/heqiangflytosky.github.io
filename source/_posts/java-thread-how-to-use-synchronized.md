---
title: Java 线程 -- synchronized、wait、notify和notifyAll的使用
categories: Android 线程
comments: true
tags: [Android 线程]
description: 本文介绍 synchronized 的使用
date: 2015-9-4 10:00:00
---

## 概述

在并发编程中，多线程同时并发访问的资源叫做临界资源，当多个线程同时访问对象并要求操作相同资源时，分割了原子操作就有可能出现数据的不一致或数据不完整的情况，为避免这种情况的发生，我们会采取同步机制，以确保在某一时刻，方法内只允许有一个线程。它保证了线程对变量访问的**可见性**和**互斥性**。
采用 synchronized 修饰符实现的同步机制叫做互斥锁机制，它所获得的锁叫做互斥锁。每个对象都有一个 monitor (锁标记)，当线程拥有这个锁标记时才能访问这个资源，没有锁标记便进入锁池。任何一个对象系统都会为其创建一个互斥锁，这个锁是为了分配给线程的，防止打断原子操作。每个对象的锁只能分配给一个线程，因此叫做互斥锁。
synchronized 是可重入锁。

## 使用场景

|  分类  |  被锁对象  | 代码 |
|:-----:|:-----:|:-----:|
| 静态方法 | 类对象 | public static synchronized void method() {} |
| 实例方法 | 类的实例对象 | public synchronized void method() {} |
| 实例对象 | 类的实例对象 | synchronized (this) {} |
| class对象 | 类对象 | synchronized (Demo.class) {} |
| 任意实例对象Object | 实例对象Object | synchronized (mObject) {} |

前面两种是针对方法，后面三种是针对代码块。

## 实现原理

synchronized 关键字经过编译后，会在同步块的前后分别形成 monitorenter 和 monitorexit 这两个字节码指令。根据虚拟机规范的要求，在执行 monitorenter 指令时，首先要尝试获取对象的锁，如果获得了锁，把锁的计数器加 1，相应地，在执行 monitorexit 指令时会将锁计数器减 1，当计数器为 0 时，锁便被释放了。由于 synchronized 同步块对同一个线程是可重入的，因此一个线程可以多次获得同一个对象的互斥锁，同样，要释放相应次数的该互斥锁，才能最终释放掉该锁。

## 使用方法

### 代码测试

```
    public void testSynchronized() {
        Thread thread1 = new Thread(new MyRunnable());
        thread1.setName("Thread1");

        Thread thread2 = new Thread(new MyRunnable());
        thread2.setName("Thread2");

        Thread thread3 = new Thread(new MyRunnable());
        thread3.setName("Thread3");

        thread1.start();
        thread2.start();
        thread3.start();
    }

    class MyRunnable implements Runnable{

        @Override
        public void run() {
            synchronized (mObject) {
                try {
                    Log.e("Test", Thread.currentThread().getName() + " start");
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```

```
03-09 16:54:07.328 Thread1 start
03-09 16:54:08.329 Thread1 end
03-09 16:54:08.329 Thread2 start
03-09 16:54:09.329 Thread2 end
03-09 16:54:09.330 Thread3 start
03-09 16:54:10.330 Thread3 end

```

可以看到 Thread1 加锁代码块首先执行，首先获得类对象锁，后面 Thread2 和 Thread3 run 方法执行到加锁代码块时进入阻塞状态，Thread1 代码块执行完释放锁之后 Thread2 进入就绪状态并进入运行状态开始执行对应代码块，依次类推 Thead3。3个线程串行执行。


## wait、notify和notifyAll

任意一个 Java 对象，都拥有一组监视器方法（定义在 java.lang.Object 上），主要包括 wait、notify和notifyAll 及其重载的方法，这些方法和 synchronized 同步关键字配合使用，可以用来在多线程间通信，实现等待/通知机制。

### 方法介绍


 - wait()：释放当前对象锁，并进入阻塞队列
 - wait(long millis)：可以设置等待的超时时间
 - wait(long millis, int nanos)：后面一个参数可以附件设置一个纳秒级别的时间，增加超时时间的精度
 - notify()：唤醒当前对象阻塞队列里的任一线程（并不保证唤醒哪一个）
 - notifyAll()：唤醒当前对象阻塞队列里的所有线程

这几个方法必须要和 synchronized 一起使用，否则会报错。

```
java.lang.IllegalMonitorStateException: object not locked by thread before wait()
```

再来简单介绍一下这几个方法的特点：

 - wait、notify 和 notifyAll 方法是 Object 的方法，而且是 final 方法，无法被重写。
 - wait 会使当前的线程进入等待状态，而且它会释放当前的锁，然后让出CPU，进入等待状态。但前提是必须先获得锁。
 - wait、notify 和 notifyAll 一般是配合 synchronized 使用，用在 synchronized 方法或者代码块内，否则会抛出异常。
 - 当 notify/notifyAll() 被执行时候，才会唤醒一个或多个正处于等待状态的线程，然后继续往下执行，直到执行完 synchronized 代码块的代码或是中途遇到wait() ，再次释放锁。
 - notify/notifyAll() 的执行只是唤醒沉睡的线程，而不会立即释放锁，调用线程依旧持有锁。锁的释放要看代码块的具体执行情况。因此，线程虽被唤醒，但是仍无法获得锁。直到调用线程退出 synchronized 方法或者代码块，释放锁后，等待线程中的一个才有机会获得锁继续执行。所以在编程中，尽量在使用了notify/notifyAll() 后立即退出 synchronized 临界区，以便其他线程唤醒并获得锁以继续执行。
 - notify方法只唤醒一个等待（对象的）线程并使该线程开始执行。如果有多个线程等待一个对象，这个方法只会唤醒其中一个线程，选择哪个线程取决于操作系统对多线程管理的实现。notifyAll 会唤醒所有等待(对象的)线程。如果当前情况下有多个线程需要被唤醒，推荐使用notifyAll 方法。

### 代码测试 wait

创建两个 Runnable 对象，在 Thread2 代码执行中间加入 `wait()` 方法：

```
    public void testSynchronized() {
        Thread thread1 = new Thread(new MyRunnable1());
        thread1.setName("Thread1");

        Thread thread2 = new Thread(new MyRunnable2());
        thread2.setName("Thread2");

        thread2.start();
        thread1.start();
    }

    class MyRunnable1 implements Runnable{

        @Override
        public void run() {
            synchronized (mObject) {
                try {
                    Log.e("Test", Thread.currentThread().getName() + " start");
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    class MyRunnable2 implements Runnable{

        @Override
        public void run() {
            synchronized (mObject) {
                try {
                    Log.e("Test", Thread.currentThread().getName() + " start");
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " wait");
                    mObject.wait();
                    Log.e("Test", Thread.currentThread().getName() + " after wait");
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```


```
03-09 17:07:57.816 Thread2 start
03-09 17:07:58.816 Thread2 wait
03-09 17:07:58.817 Thread1 start
03-09 17:07:59.818 Thread1 end
```

Thread2 先执行获得锁，Thread1进入阻塞状态等待锁释放，执行到 `mObject.wait()` 方法 Thread2 进入等待状态并释放锁，此时 Thread1 获得锁自动进入就绪状态并开始执行，Thread1 加锁代码块执行完后由于没有发送 notify 唤醒等待线程， Thread2 仍然保持在等待状态，因此，Thread2 wait 方法后的代码没有执行。

### 代码测试 notify

改动一下 Thread1 代码，代码块中加入 `mObject.notify()`。
**注意：**`notify()` 方法必须放到 synchronized 代码块中。

```
    public void testSynchronized() {
        Thread thread1 = new Thread(new MyRunnable1());
        thread1.setName("Thread1");

        Thread thread2 = new Thread(new MyRunnable2());
        thread2.setName("Thread2");

        thread2.start();
        thread1.start();
    }

    class MyRunnable1 implements Runnable{

        @Override
        public void run() {
            synchronized (mObject) {
                try {
                    Log.e("Test", Thread.currentThread().getName() + " start");
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " notify");
                    mObject.notify();
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    class MyRunnable2 implements Runnable{

        @Override
        public void run() {
            synchronized (mObject) {
                try {
                    Log.e("Test", Thread.currentThread().getName() + " start");
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " wait");
                    mObject.wait();
                    Log.e("Test", Thread.currentThread().getName() + " after wait");
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```

```
03-09 17:17:42.239 Thread2 start
03-09 17:17:43.239 Thread2 wait
03-09 17:17:43.240 Thread1 start
03-09 17:17:44.240 Thread1 notify
03-09 17:17:45.241 Thread1 end
03-09 17:17:45.241 Thread2 after wait
03-09 17:17:46.242 Thread2 end

```

wait 前面的流程和上面是一样的，Thread2 先执行获得锁，Thread1进入阻塞状态等待锁释放，执行到 `mObject.wait()` 方法 Thread2 进入等待状态并释放锁，此时 Thread1 获得锁自动进入就绪状态并开始执行，中间执行 mObject.notify() 方法发出唤醒线程信号，此时 Thread2 被唤醒，注意还不能马上执行，因为 Thread1 还没有真正释放锁，Thread2 进入到阻塞状态，等待锁的释放。等 Thread1 执行完 notify 后面的代码，运行到 synchronized 代码块外面，此时才真正的释放锁，那么Thread2得到锁后进入就绪状态和运行状态，Thread2 wait 后面的代码才得以继续执行。

### 代码测试 notifyAll

下面来看一下 `notifyAll` 的使用情况：

```
    public void testSynchronized() {
        Thread thread1 = new Thread(new MyRunnable2());
        thread1.setName("Thread1");
        thread1.setPriority(10);

        Thread thread2 = new Thread(new MyRunnable2());
        thread2.setName("Thread2");
        thread2.setPriority(10);

        Thread thread3 = new Thread(new MyRunnable1());
        thread3.setName("Thread3");
        thread3.setPriority(1);

        thread1.start();
        thread2.start();
        thread3.start();
    }

    class MyRunnable1 implements Runnable{

        @Override
        public void run() {
            synchronized (mObject) {
                try {
                    Log.e("Test", Thread.currentThread().getName() + " start");
                    Thread.sleep(5000);
                    Log.e("Test", Thread.currentThread().getName() + " notifyAll");
                    mObject.notifyAll();
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    class MyRunnable2 implements Runnable{

        @Override
        public void run() {
            synchronized (mObject) {
                try {
                    Log.e("Test", Thread.currentThread().getName() + " start");
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " wait");
                    mObject.wait();
                    Log.e("Test", Thread.currentThread().getName() + " after wait");
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```

```
03-09 17:34:10.688 Thread2 start
03-09 17:34:11.688 Thread2 wait
03-09 17:34:11.689 Thread1 start
03-09 17:34:12.690 Thread1 wait
03-09 17:34:12.691 Thread3 start
03-09 17:34:17.691 Thread3 notifyAll
03-09 17:34:18.692 Thread3 end
03-09 17:34:18.692 Thread2 after wait
03-09 17:34:19.693 Thread2 end
03-09 17:34:19.693 Thread1 after wait
03-09 17:34:20.694 Thread1 end
```

上面代码 Thread2 首先获得锁开始执行，wait 释放锁后 Thread1 获得锁，开始执行，wait 释放锁后 Thread3 获得锁，开始执行到 notifyAll 后唤醒所有等待该锁的线程，等到 Thread3 执行完synchronized代码块后释放锁，Thread2 先获得锁，执行完释放锁后，由于 Thread1已经被唤醒，可以得到锁继续执行。
上面的执行情况是 Thread1 和 Thread2 都在等待状态的状态下被 Thread3 唤醒，这样三个线程都可以执行完成。
再看下面的执行情况，Thread1 先获得锁执行，wait 方法后进行等待状态，然后 Thread3 获得锁开始执行，notifyAll 后并释放锁后， Thread2 开始执行，wait 后进入等待状态释放锁，Thread1 开始执行，执行完毕后并没有唤醒 Thread1，虽然 Thread1 释放了锁，Thread2并没有继续执行。

```
03-09 17:32:49.031 Thread1 start
03-09 17:32:50.032 Thread1 wait
03-09 17:32:50.033 Thread3 start
03-09 17:32:55.034 Thread3 notifyAll
03-09 17:32:56.034 Thread3 end
03-09 17:32:56.035 Thread2 start
03-09 17:32:57.036 Thread2 wait
03-09 17:32:57.036 Thread1 after wait
03-09 17:32:58.037 Thread1 end
```
