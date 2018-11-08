---
title: Java 基础 -- Fork/Join 框架
categories:  Java
comments: true
tags: [Fork/Join]
description: Fork/Join 框架的使用
date: 2016-12-6 10:00:00
---

## 概述

Fork/Join 框架是 Java7 提供的一个用于并行执行任务的框架，是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。
下图是网上流传的 Fork Join 的运行流程图，直接拿过来用了：

![效果图](/images/java-basic-fork-join-base/fork-join-principle.png)

## 工作窃取算法

工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。
那么为什么要使用这个算法呢？
假如我们需要做一个比较大的任务，可以把这个任务分割为若干个互不依赖的子任务，为了减少线程间的竞争，把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应。比如A线程负责处理A队列里的任务。但是，有的线程会先把自己队列里的任务干完，而其他线程队列对应的队列里还有任务等待处理。干完活的线程与其等着，不如去帮其他线程干活，于是它就去其他线程的队列里窃取一个任务来执行。而在这时它们会访问同一个队列，为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。
工作窃取算法的优缺点：

 - 优点：充分利用线程进行并行计算，减少了线程间的竞争。
 - 缺点：在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且该算法会消耗了更多的系统资源，比如创建多个线程和多个双端队列。

## Fork/Join 框架工作流程

首先，是分割任务。需要有个fork类来把大任务分割成子任务，有可能子任务还是很大，那么还需要不停的分割，知道分割的任务足够小。
然后，执行任务的合并结果。分割的子任务分别放在双端队列里，然后几个启动线程分别从双端队列里获取任务执行。子任务执行完的结果统一放在一个队列里，启动一个线程从队列里拿数据，然后合并这些数据。

## Fork/Join 框架的使用

Fork/Join 的使用需要使用两个类：

 - ForkJoinTask：要使用 Fork/Join 框架首先要创建一个 ForkJoin 任务，它提供在任务中执行 fork() 和 join() 操作的机制。通常情况下，我们不需要直接继承ForkJoinTask类，只需要继承它的子类，Fork/Join 框架提供了下面两个子类：
  - RecursiveAction：用于没有返回结果的任务。
  - RecursizeTask：用于有返回结果的任务。
 - ForkJoinPool：ForkJoinTask 需要通过 ForkJoinPool 来执行。

任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列暂时没有其他任务时，它会随机从其他工作线程的队列的尾部获取一个任务。
下面我们通过一个简单的例子来了解一下 Fork/Join 框架的使用：计算 1+2+3+.....+10的结果。
使用 Fork/Join 框架首先要考虑的是如何分割任务，这个需要我们在代码里面实现。
因为这个是需要返回结果的任务，因此只能使用 `RecursiveTask` 来实现。代码如下：

```
    public static void testForkJoin() {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        // 生成一个计算任务
        CountTask countTask = new CountTask(1, 10);
        // 执行任务
        Future<Integer> result =  forkJoinPool.submit(countTask);
        int sum = 0;
        try {
            sum = result.get();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
        Log.e("Test","sum = "+sum);

    }

    public static class CountTask extends RecursiveTask<Integer> {
        // 设置一个阈值
        private static final int  THRESHOLD = 2;
        private int mStart;
        private int mEnd;

        public CountTask(int start, int end) {
            mStart = start;
            mEnd = end;
            Log.e("Test", "CountTask start = "+start+", end= = "+end);
        }

        @Override
        protected Integer compute() {
            int sum = 0;
            boolean canCompute = (mEnd - mStart) <= THRESHOLD;
            if (canCompute) {
                for(int i = mStart; i<=mEnd;i++){
                    sum += i;
                }
            } else {
                // 如果大于阈值，就要继续分割任务
                int middle = (mStart + mEnd) / 2;
                CountTask leftTask = new CountTask(mStart, middle);
                CountTask rightTask = new CountTask(middle + 1, mEnd);
                // 执行子任务
                leftTask.fork();
                rightTask.fork();
                //等待子任务执行完，并获取执行结果
                int leftResult = leftTask.join();
                int rightResult = rightTask.join();
                // 合并子任务
                sum = leftResult + rightResult;

            }
            return sum;
        }
    }
```

`ForkJoinTask` 与一般任务的主要区别在于它需要实现 `compute` 方法，在这个方法里面，首先要判断任务是否足够小，如果足够小就直接执行任务，如果不够小，就必须分割成两个子任务，每个子任务在调用 `fork` 方法时，又会进入 `compute` 方法，看看当前子任务是否需要继续分割成子任务，如果不继续分割，则执行当前任务并返回结果。使用 `join` 方法会等待任务执行完并得到其结果。

## Fork/Join 框架的异常处理

`ForkJoinTask` 在执行任务的时候可能会抛出异常，但是我们没有办法在主线程里直接捕获异常，所以 `ForkJoinTask` 提供了 `isCompletedAbnormally()` 方法来判断任务是否已经抛出异常或者已经取消了，并且可以通过 `ForkJoinTask` 的 `getException()` 方法获取异常。

```
                if (leftTask.isCompletedAbnormally()) {
                    if (leftTask.getException() != null ) {
                        leftTask.getException().printStackTrace();
                    }
                }
```

`getException()` 方法返回 `Throwable` 对象，如果任务取消了则返回 `CancellationException`。如果任务没有完成或者没有抛出异常则返回 null。

## Fork/Join 框架的实现原理

## 参考

《Java 并发编程的艺术》

