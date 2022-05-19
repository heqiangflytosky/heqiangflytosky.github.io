---
title: 算法系列 -- 排序算法
categories: 数据结构与算法
comments: true
tags: [算法]
description: 介绍常用的排序算法，面试必备系列
date: 2014-6-2 10:00:00
---

## 各种排序算法的时间复杂度

| 排序算法 | 时间复杂度 |
|:-------------:|:-------------:|
| 冒泡排序 | O(n²) |
| 选择排序 | O(n²) |
| 插入排序 | O(n²) |
| 希尔排序 | O(n1.5) |
| 快速排序 | O(N*logN) |
| 归并排序 | O(N*logN) |
| 堆排序 | O(N*logN) |

## 冒泡排序

### 基本思想

冒泡排序是一种简单直观的排序算法，它要重复地走访要排序的数列，依次比较两个相邻数的大小，如果它们的顺序不符合我们预期，那么就交换他们。
重复地走访数列，直到没有必要再进行交换为止。那么数列的排序工作就完成了。
**注意**：这里的走访进行到“直到没有必要再进行交换为止”是需要我们优化的点。
经过这样逐轮进行排序，最大或者最小的数就排到了数列的末尾或者队首。

### 步骤

按照升序排列，从数列队首开始进行排序：

 - 第一轮，比较相邻的元素。如果第一个比第二个大，就交换他们两个。经过第一轮，最大的数就移动到了数列的末尾。
 - 重复第一轮，只进行到倒数第二个，那么就把第二大的数排到了倒数第二。
 - 依次进行，直到某轮再没有进行过数据的交换。表示已经排列好。

### 图解

![效果图](http://www.runoob.com/wp-content/uploads/2015/09/1240)

### 代码实现

```
    private void bubbleSort(int [] array) {
        int tmp;
        boolean sort = false;
        for(int i = 0;i<array.length;i++){
            sort = false;
            for (int j = 0; j<array.length - 1 - i; j++) {
                if (array[j] > array[j+1]) {
                    tmp = array[j+1];
                    array[j+1] = array[j];
                    array[j] = tmp;

                    sort = true;
                }
            }
            if(!sort) {
                break;
            }
        }
    }
```

## 选择排序

### 基本思想

选择排序也是一种简单直观的排序算法。排序过程中可以把数列分为两部分，前面部分是已经排序好的数列，后部分是无序数列。它需要N次遍历无序数列部分，取到最大或者最小的数与无序数列队首的数进行交换。这种遍历只在无序数列部分进行。

### 步骤

按升序排序来讲解：

 - 第一轮从第二个元素开始，找出第二个到第N个元素中最小的数与第一个数交换，这样总数列中最小的数就排到了队首第一个。
 - 第二轮从三个元素开始，找出第三个到第N个元素中最小的数与第二个数交换，这样第二小的数就排到了第二个为止。
 - 重复进行N-1轮排序。

### 图解

![效果图](http://www.runoob.com/wp-content/uploads/2015/09/12401)

### 代码实现

```
    private void selectionSort(int [] array) {
        for(int i = 0;i<array.length - 1;i++) {
            int k = i;
            for(int j = i+1;j<array.length;j++){
                if (array[j] < array[k]) {
                    k = j;
                }
            }
            int tmp;
            if (k != i) {
                tmp = array[i];
                array[i] = array[k];
                array[k] = tmp;
            }
        }
    }
```


## 插入排序

### 基本思想

插入排序在排序过程中也把数列分为两部分，依次把前面i个数排好顺序，那么第n+1个数就按给定的顺序插入到前面i个数中。进行 n 轮的循环，直到把所有 n 个数全部排到指定位置。
如果数列是基本有序的数列，那么用插入排序会比较高效。

### 步骤

按升序排列：

 - 第一轮先将第二个数和第一个数比较，如果第二个数小于第一个数，那么交换他们位置。否则不交换。
 - 第二轮先从第三个数和第二个数比较，如果需要交换，交换后第二个再和第一个数比较。否则结束本次循环。
 - 重复前面的比较规则，注意当i个数和i-1个数不需要交换时，说明找到插入的位置了，这时结束本次循环。
 - n轮比较后，所有数列排好序。

### 图解

![效果图](http://www.runoob.com/wp-content/uploads/2015/09/33403)

就像我们平时的打扑克一样：

![效果图](http://www.runoob.com/wp-content/uploads/2015/09/22402)

### 代码实现

```
    private void insertionSort(int [] array) {
        for(int i = 1;i<array.length; i++) {
            for(int j = i; j > 0; j--) {
                if (array[j-1] > array[j]) {
                    int tmp = array[j];
                    array[j] = array[j-1];
                    array[j-1] = tmp;
                } else {
                    break;
                }
            }
        }
    }
```

```
    private void insertionSort2(int [] array) {
        for(int i = 1;i<array.length; i++) {
            int t = array[i];
            int j;
            for(j = i; j > 0 && t < array[j-1]; j--) {
                array[j] = array[j-1];
            }
            array[j] = t;
        }
    }
```

## 希尔排序

希尔排序是插入排序的一种，又称“缩小增量排序”。是直接插入排序算法的一种更高效的改进版本。

### 基本思想

希尔排序是把记录按下标的一定增量分组，对每组使用直接插入排序算法排序；随着增量逐渐减少，每组包含的关键词越来越多，当增量减至1时，整个文件恰被分成一组，此时的数列基本变的有序，然后再执行一次插入排序，由于数列基本有序，此时的插入算法会非常高效。此次排完序后算法便终止。

### 步骤

进行8个数的数列的升序排列：

 - 第一步，将8个数字平均分成8/2=4个组，第1、5个数字一组，2、6，3、7，4、8这样分成四组，然后小组内按插入排序法排序。
 - 第二步，将8个数字分成4/2=2个组，第1、3、5、7一组，2、4、6、8一组，小组内按插入排序法排序。
 - 第三部，将8个数组分成2/2=1个组，这是数字基本是有序的，然后再按照插入排序排序一次。结束排序。

### 图解

![效果图](http://www.runoob.com/wp-content/uploads/2015/09/44404)

### 代码实现

```
    private void shellSort(int [] array) {
        int n = array.length/2;
        while (n>0){
            for (int i = 0; i < n;i++){
                for(int j = n; j < array.length; j ++) {
                    for (int k = j; k > 0 && k - n >= 0; k --) {
                        if (array[k - n] > array[k]) {
                            int tmp = array[k];
                            array[k] = array[k - n];
                            array[k - n] = tmp;
                        } else {
                            break;
                        }
                    }
                }
            }
            n/=2;
        }
    }
```

## 快速排序

快速排序是对冒泡排序的一种改进。是分治法的一种应用。

### 基本思想

首先找到一个基准值（通常可以指定最左边的数据），通过一趟排序将要排序的数据分割成独立的两部分，其中一部分的所有数据都比基准值小，另外一部分的所有数据都要大，然后再按此方法对这两部分数据分别进行快速排序。

### 步骤

基准值为最左边数的升序排列：

 - 选最左边为基准值，标记left为第1个数，right为n-1个数。从right开始和基准值比较，直到遇到比基准值小的数，把这个数放到基准值的位置，然后标记这个空位置为j。再从left开始和基准值比较，知道遇到比基准值大的数，把这个数放到j的位置，然后标记这个空位置为i；然后再从j-1处开始比较重复上面做法。直到 i==j，然后把基准值放到这个位置。
 - 第二轮，把上面排好的序列分为 [left，i-1]，[i+1,right]两个序列进行上面的排序步骤。
 - 分为四个序列进行排序。
 - 直到各个子序列不能再分位置。排序结束。 

### 图解

### 代码实现

递归法实现：

```
    private void quickSort(int [] array, int left, int right) {
        if (left >= right){
            return;
        }
        int base = array[left];
        int i = left, j = right;
        while (i < j) {
            while (i<j && array[j] > base) {
                j--;
            }
            if (i<j) {
                array[i] = array[j];
                i++ ;
            }
            while (i < j && array[i] < base) {
                i++;
            }
            if (i < j) {
                array[j] = array[i];
                j--;
            }
        }
        array[i] = base;
        quickSort(array, left, i-1);
        quickSort(array, i+1,right);
    }
```

栈实现：

```
    private void quickSort2(int [] array) {
        Stack<Integer> stack = new Stack<>();
        stack.push(0);
        stack.push(array.length - 1);
        while (!stack.isEmpty()) {
            int right = stack.pop();
            int left = stack.pop();

            int i = left, j = right;
            int base = array[i];
            while (i < j) {
                while (i < j && array[j] > base) {
                    j--;
                }
                if (i < j) {
                    array[i] = array[j];
                    i++ ;
                }
                while (i < j && array[i] < base) {
                    i++;
                }
                if (i < j) {
                    array[j] = array[i];
                    j--;
                }
            }
            array[i] = base;

            if(i - 1 > left){
                stack.push(left);
                stack.push(i - 1);
            }
            if(i + 1 < right){
                stack.push(i +1);
                stack.push(right);
            }
        }
    }
```


## 归并排序

归并排序是分治法的一种应用。

### 基本思想

归并操作就是将已有序的子序列合并，得到完全有序的序列。那么首先就要把序列分为多个有序的序列。我们采用的方法是将序列从中间进行切割，然后再将得到的子序列从中间进行切割，直到子序列变成只有两个元素的子序列位置，然后把这个两个元素的子序列进行排序。

### 步骤

升序排列：

 - 首先，把原序列从中间一分为二，如果得到子序列元素大于2，继续切分，直到子序列变为2个元素。
 - 首先对这些2个元素的序列排序。
 - 在对相邻的两个有序子序列进行合并，得到一个有序子序列。
 - 直到所有的子序列完全合并。

### 图解

![效果图](https://images2015.cnblogs.com/blog/1024555/201612/1024555-20161218163120151-452283750.png)

![效果图](https://upload.wikimedia.org/wikipedia/commons/c/c5/Merge_sort_animation2.gif)


### 代码实现

递归实现：

```
    private void mergeSort(int [] array) {
        int [] tmp = new int[array.length];
        doMergeSort(array, tmp, 0, array.length -1);
    }
    private void doMergeSort(int [] array, int [] tmp, int first, int end) {
        if (first >= end){
            return;
        }
        int middle = (first + end)/2;
        doMergeSort(array, tmp, first, middle);
        doMergeSort(array, tmp, middle + 1, end);

        // 合并数组
        int i = first,l = first,  j = middle + 1;
        while(i <= middle && j <= end) {
            if (array[i] <= array[j]) {
                tmp[l++] = array[i++];
            } else {
                tmp[l++] = array[j++];
            }
        }

        while (i <= middle){
            tmp[l++] = array[i++];
        }

        while (j <= end) {
            tmp[l++] = array[j++];
        }

        for (i = first; i <= end; i++) {
            array[i] = tmp[i];
        }
    }
```

## 堆排序

### 基本思想

堆排序利用了最大堆和最小堆特性，现将根据数列构建的最大堆或者最小堆的根结点元素和最后一个结点进行数据交换，这个最大值或者最小值就排到了队尾。然后再对剩余的元素构建最大堆或者最小堆，然后再把根结点和当前堆的最后一个元素交换，重复进行，直到堆只剩一个元素。那么此时已经是一个有序的数组。

### 步骤

降序排序：

 - 把数列构建一个最小堆
 - 把根节点和堆的最后一个元素交换，那么最小元素就排到了数列的最后一个位置
 - 再把剩余的 n-1 个树构建一个最小堆
 - 然后再把跟结点和当前堆的最后一个元素交换，即第 n-1 个元素交换，那么有把当前堆最小元素排到了倒数第二的位置
 - 重复进行，直到堆只剩下一个元素
 - 那么数组已经是一个有序的数组

### 图解

假设带排序的数列为：[72,6,57,88,60,42,83,73,48,85]
首先根据数列构建一个最大堆，如下图1。

把根节点的值和最后一个叶子结点交换，那么，最大值就放在了数列的尾部。如下图2.

这时，原来的堆已经不符合最大堆的特性，那么需要重新构建最大堆。此时最后一个结点已经是数列的最大值，可以不参与最大堆的构建。方法是从根节点开始，比较它和左右子树的大小，把最大值放在根位置，如果期间发生交换，那么需要重新对发生交换的子树进行构建。如下图3.

再次交换剩余堆的根节点和最后一个叶子结点。如下图4。

然后对除去最后一个叶子结点之外的结点重新构建最大堆。如下图5。

直到只剩余最后一个结点。如下图6。

此时数列已经按照预期排好顺序。

![效果图](/images/algorithms-basic-sort/heapSort1.png)

### 代码实现

```
    private void heapSort(int [] array) {
        // 根据数组构造一个最大堆
        // 从最后一个非叶子结点开始，对堆进行调整。最后一个非叶子结点的右结点为 2*i+1 = array.length
        // 这个调整是自上而下的，意思是针对某个结点进行调整时，它的左右子树的已经符合最大堆特性
        for(int i = array.length/2 - 1; i >=0; i--) {
            heapAdjust(array, i, array.length);
        }

        // 开始进行排序，从最后一个元素开始，由于前面已经是最大堆，先交换根节点的元素和数组最后一个元素
        // 这样最大的元素就排到了数组最后面
        // 然后再对除去最后一个元素的剩余元素组成的堆进行调整
        // ......直到剩余最后一个元素
        int tmp = 0;
        for (int j = array.length - 1; j > 0; j--) {
            // 元素交换，把根节点的元素即最大元素和当前最大堆的最后一个元素交换
            tmp = array[0];
            array[0] = array[j];
            array[j] = tmp;

            // 从根结点开始再进行一轮的调整
            heapAdjust(array, 0, j);
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



<!-- 
http://www.runoob.com/w3cnote/sort-algorithm-summary.html
https://blog.csdn.net/u013384984/article/details/79496052

-->

