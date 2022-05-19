---
title: 数据结构 -- 堆
categories: 数据结构与算法
comments: true
tags: [数据结构]
description: 介绍堆的基础知识
date: 2014-7-5 10:00:00
---

## 概述

堆也是一种树形结构，我们平时接触到的堆一般时二叉堆，它是二叉树的一种。除了二叉堆，还有二项堆，斐波那契堆等等。本文下面说的堆都是指二叉堆。

二叉堆具有如下的特点：

 - 堆是一种完全二叉树
 - 它的子结点的值大于或者等于父结点的值，或者子结点的值小于或者等于父结点的值

堆是局部有序的，任何一个结点与其兄弟结点之间都没有必然的顺序关系，但它与其父子结点有大小顺序关系。
根据它的特点我们可以将堆分为两类：

 - 最大堆：根结点的值是所有堆结点值中最大的
 - 最小堆：根结点的值是所有堆结点值中最小的

![效果图](/images/data-structure-basic-heap/big-small-heap.png)

堆是左平衡的树，所以随着结点的增加，树会逐级从左至右增长，因此对于堆来说，一个比较好的表示左平衡二叉树的方式是，将结点通过水平遍历的方式存储到一个数组中。数组中处于i处的结点，其父结点位于(i-1)/2处。其左右子结点分别位于2i+1和2i+2位置上。这样的组织结构对于堆来说非常重要，因为通过它我们能迅速定位堆的最后一个结点：最后一个结点指处于树中最深层最右端的结点，这在实现某些堆操作时非常重要。
比如，上图中的最大堆我们可以用数组来表示：

![效果图](/images/data-structure-basic-heap/heap-array.png)

## 堆的操作

堆的操作可以分为下面几种：

 - 创建堆
 - 插入结点
 - 删除结点
 - 堆排序

## 创建堆

### 图解

首先，假设给定无序数列如下，并用数组如下表示：

![效果图](/images/data-structure-basic-heap/heap-int-1.png)

接着，从最后一个非叶子结点开始，即下图中的第 10/2 - 1 = 4 个结点，从右到左，从下到上进行调整。如图1所示。
先和它的左右叶子结点中的最大值即第10个结点进行比较，发现比第10个小，那么交换他们的位置，那么较大的值13位置上移。如图2所示。
接着看第3个结点，它和它的左右叶子结点大小符合最大堆特征，不用交换。
接着看第2个结点，按照前面的方法交换第2、5结点。如图3所示。
接着看第1个结点，先和它的叶子中最大值也就是左叶子结点交换，如图4所示。
交换后，对左子树的最大堆特性造成了影响，那么就要对左子树进行重新构建，交换第3、8结点后，符合了最大堆特性。如图5所示。
继续进行排列，直到进行到根节点位置。如图6所示。构造最大堆完成。
**注意**：一定要牢记，每次交换都要把改变了的那个节点所在的树重新构建一下直到符合最大堆特性。

![效果图](/images/data-structure-basic-heap/heap-int-2.png)

### 代码实现

```
    private void heapInit(int[] array) {
        // 根据数组构造一个最大堆
        // 从最后一个非叶子结点开始，对堆进行调整。最后一个非叶子结点的右结点为 2*i+1 = array.length
        // 这个调整是自上而下的，意思是针对某个结点进行调整时，它的左右子树的已经符合最大堆特性
        for(int i = array.length/2 - 1; i >=0; i--) {
            heapAdjust(array, i, array.length);
        }
    }

    private void heapAdjust(int[] array, int parent, int length) {
        int tmp = 0;

        int j = 2 * parent + 1;// 左子结点

        while (j < length) {
            if(j + 1 < length && array[j + 1] > array[j]) {
                j++;
            }
            // 如果顺序已经排好，跳出循环，表示当前子树已经排好序
            if (array[parent] >= array[j]) {
                break;
            }

            //较小节点下移
            tmp = array[parent];
            array[parent] = array[j];
            array[j] = tmp;

            // 交换数据后，可能对子树的顺序产生影响，因此还要继续对子树进行调整
            // 将即将进行调整的子树的父结点设置为前面被交换的子结点
            parent = j;
            // 被移动结点的左子树
            j = 2 * parent +1;
        }
    }
```

## 插入结点

对于堆的插入，处理思想很简单，首先在堆的最后添加一个新结点，然后对这个新的堆重新调整即可。这个过程涉及结点的向上筛选，直到新结点找到自己合适的位置为止。

### 代码实现

针对最大堆的代码实现：

```
    private void shiftUp(int[] array, int position) {
        int num = array[position];
        int father = (position - 1)/2;

        while (position > 0 && array[father] < num) {
            array[position] = array[father];
            position = father;
            father = (position - 1)/2;
        }
        array[position] = num;
    }
```

## 删除结点

堆的结点删除的处理思想是把堆尾元素剪切，覆盖到删除的结点位置，然后对堆进行一轮调整，这个过程涉及到元素的向下筛选，知道该堆尾元素重新找到自己的位置。

### 代码实现

针对最大堆的代码实现，可以利用前面的构建堆时的 heapAdjust 方法：

```
    private void shiftDown(int[] array, int position) {
        array[position] = array[array.length - 1];
        array[array.length - 1] = -1;
        heapAdjust(array, position, array.length - 1);
    }
```

## 堆排序

参考 [算法系列 -- 排序算法](http://www.heqiangfly.com/2014/06/02/algorithms-basic-sort/) 一文中的堆排序。

<!-- 
https://blog.csdn.net/qq_41117236/article/details/81029618
https://blog.csdn.net/u013384984/article/details/79496052

-->