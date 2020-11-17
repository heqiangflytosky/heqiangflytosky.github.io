---
title: Kotlin -- 集合
categories: Kotlin
comments: true
tags: [Kotlin]
description: 介绍 Kotlin 集合的一些 API
date: 2020-1-2 10:00:00
---

## 概述

本节介绍一下 kotlin 中一些集合类的用法。
Kotlin 中的集合是完全基于 Java 的集合实现的，并在其基础上进行了扩充和增强。Kotlin支持3种类型的集合，即Set、List和Map。与Java不同的是，Kotlin将只读集合、可变集合分开实现，程序员根据编程意图选择使用哪种集合，避免集合数据发生无意识修改而产生一些非可控结果或Bug。使用Set、List、Map类型声明的集合都是只读集合，前面冠以Mutable的是可变集合。只读集合中的函数、方法、属性，对可变集合来说都可用，反之则不行。集合和数组非常相似，都是由许多数据构成的列表。数组由有序的、可重复的数据组成，数组的长度是固定的，向数组中间插入数据的操作非常麻烦。数组和集合可以使用函数进行相互转换。
Set和List这两类集合的数据形式相同，有很多相同的函数，本小节就介绍这两类集合的常用函数。
先定义一个数组：`val myList = listOf(2,4,5,6,8)`

## 总数操作类函数

### any函数

any 函数判断集合中是否有满足条件的数据，结果返回布尔类型。
比如，下面的语句判断myList中是否有能够被2整除的数据，运行结果是 true。

```
Log.e("Test","${myList.any({it % 2 == 0})}")
```



### all函数

all函数判断集合中的数值是否都满足指定的条件，结果返回布尔类型。
比如，下面的语句判断myList中的数据是否都能够被2整除，运行结果是false。

```
Log.e("Test","${myList.all({it % 2 == 0})}")
```

### count函数

count函数计算集合中满足指定条件的数据的个数，结果返回Int类型。
比如，下面的语句计算myList中能够被2整除的数值的个数，运行结果是4。

```
Log.e("Test","${myList.count({it % 2 == 0})}")
```

### fold和foldRight函数

在给定初始值的基础上，fold函数从第一项到最后一项进行累加，而foldRight函数从最后一项到第一项进行累加。
比如，下面的语句计算myList中所有数值之和再加上10，输出结果是35。

```
Log.e("Test","${myList.fold(10, {t, n -> t+n})}")
```

### forEach函数

forEach函数遍历集合中的每个数值，对其进行相关操作。
比如，下面的语句遍历myList，输出所有大于5的数据，输出结果是6和8。

```
Log.e("Test","${myList.forEach({n -> if(n > 5) Log.e("Test","$n")})}")
```

### forEachIndexed函数

forEachIndexed 函数循环遍历集合，同时得到数值的索引值，并执行指定的操作。
比如，下面的语句遍历myList，输出所有数值大于5的索引值和数值。
3,6
4,8

```
Log.e("Test","${myList.forEachIndexed({i, v -> if(v > 5) Log.e("Test","$i, $v")})}")
```

### max、min函数


如果集合中的数据属于同一种类型，并且是可以比较的数据，则可以使用函数max（）获取其中的最大值，使用函数min（）获取其中的最小值。

### maxBy、minBy函数

如果集合中的数据属于同一种类型，并且是可比较的，就可以使用该函数。该函数的作用是，根据指定的方法计算集合中的每个数值，并对计算之后的数值进行比较，最大或最小的数值对应的集合数值就是函数的返回值，返回值的类型和集合中的数据类型相同。如果没有，则返回null。
下面是示例代码：

```
Log.e("Test","${myList.maxBy { -it }}") // 得到的最大值时-2，因此返回2
```

### none函数

none函数按照指定的方法对集合中的所有数值进行判断。如果存在指定方法为true的数据，则函数返回false；否则返回true。
下面的语句判断是否不存在能够被6整除的数据，显然存在，所以下面语句的执行结果为false。

```
Log.e("Test","${myList.none { it % 6 == 0 }}")
```

### reduce和reduceRight函数

reduce 函数对集合中的数据从第一项到最后一项按照指定的方法进行累加计算，reduceRight函数对集合中的数据从最后一项到第一项按照指定的方法进行累加计算。这两个函数与fold函数的区别在于，这两者都没有初始值。
reduce函数和reduceRight函数的计算顺序不同，返回的结果未必是相同的。如果指定的方法简单地求累加和，则肯定返回相同的结果。逻辑相同的Lambda表达式，reduceRight函数和foldRight函数的计算结果不见得相同，这一点需要注意。

### sumBy函数

sumBy函数使用指定的方法对集合中的各项数据进行计算，并对计算后的结果求和，返回值的数据类型与集合中的数据类型相同。


## 过滤操作类函数

### drop、dropWhile、dropLastWhile函数

这3个函数以不同的方式对集合中的元素进行删除操作。drop函数删除集合中从第一项开始算起的指定数目的集合元素；dropWhile 函数按照表达式从第一项开始逐项判断集合中的数值，遇到使表达式为 true 的元素就删除，遇到第一个使表达式为 false 的元素就结束；dropLastWhile函数与dropWhile函数类似，只是它从最后一项开始判断。

```
        Log.e("Test","${myList.drop(4)}")   // [8]
        Log.e("Test","${myList.dropWhile({it % 4==0})}")  // [2, 4, 5, 6, 8]
        Log.e("Test","${myList.dropWhile({it % 2==0})}")  // [5, 6, 8]
        Log.e("Test","${myList.dropLastWhile({it > 6})}") // [2, 4, 5, 6]
```

### filter、filterNot和filterNotNull函数

这3个函数以不同的方式对集合中的元素进行过滤操作。filter函数只保留能够让表达式为true的元素；filterNot函数只保留能够让表达式为false的元素；filterNotNull函数只保留不为null的元素。

### slice函数

slice函数只保留集合中指定下标的元素，下标超界会抛出ArrayIndexOutOfBoundsException异常。
下面是示例代码：

```
        Log.e("Test","${myList.slice(listOf(0,1,3))}") // [2, 4, 6]
        Log.e("Test","${myList.slice(0..3)}")   // [2, 4, 5, 6]
```

集合下标是从0开始记数的，代码中的“0..3”表示0、1、2、3，是范围的表达方式。

### take、takeLast和takeWhile函数

这3个函数以不同的方式获取集合中的一部分，形成一个子集合。take函数获取集合中从第一个元素开始的指定数目的元素；takeLast函数与 take 函数类似，只是从最后一个元素开始获取；takeWhile 函数根据表达式从第一个元素开始计算，直到遇到能够使表达式取值为 false的元素为止，获取之前的数据。

```
        Log.e("Test","${myList.take(2)}")  //[2, 4]
        Log.e("Test","${myList.takeLast(2)}")  //[6, 8]
        Log.e("Test","${myList.takeWhile({it<3})}")  //[2]
```

## 映射类操作函数

### flatMap函数

flatMap函数按照指定的规则对集合进行合并。
下面是示例代码：

```
Log.e("Test","${myList.flatMap { listOf(it,it+1,it+2) }}") // [2, 3, 4, 4, 5, 6, 5, 6, 7, 6, 7, 8, 8, 9, 10]
```

myList的初始值是[2,4,5,6,8]，共5个元素。在上面的示例代码中，flatMap函数遍历myList集合，首先读取第一个数值2，然后根据规则以2、3、4构成List；接着读取第2个数值4，根据规则以4、5、6构成 List，并与前面得到的集合合并。依次处理各个元素，就得到了上面的输出内容。

### groupBy函数

groupBy函数用判断表达式指定规则，根据该规则对集合中的数据进行分组。Kotlin中的判断表达式有if和when两种。
下面的示例代码是一个非常简单的if表达式：

```
Log.e("Test","${myList.groupBy { if (it % 2 ==0) "偶数" else "奇数"  }}")
```

输出：{偶数=[2, 4, 6, 8], 奇数=[5]}

### map、mapIndexed和mapNotNull函数

这3个函数用不同的方式进行集合映像，得到一个新的集合。map函数根据指定的规则对集合中的每个元素进行运算，得到一个新的集合；mapIndexed函数使用集合的索引、数值构成的规则对集合中的每个元素进行运算，得到一个新的集合；mapNotNull函数按照指定的规则对集合中的非空元素进行运算，得到一个新的集合。

```
        Log.e("Test","${myList.map { it *2  }}") // [4, 8, 10, 12, 16]
        Log.e("Test","${myList.mapIndexed { i,v -> i *v  }}")  //[0, 4, 10, 18, 32]
        Log.e("Test","${myList.mapNotNull { it *2  }}")  //[4, 8, 10, 12, 16]
```

## 元素操作类函数

### contains函数

contains函数判断集合中是否包含指定的元素，如果有，则返回true；否则返回false。

### elementAt、elementAtOrElse和elementAtOrNull函数

这3个函数以不同的形式获取集合中指定下标的元素。elementAt函数获取集合中指定下标的元素，下标超界时抛出ArrayIndexOutOfBoundsException异常；elementAtOrElse函数获取集合中指定下标的元素，下标超界时根据规则对该指定的下标进行运算，并返回结果；elementAtOrNull函数获取集合中指定下标的元素，下标超界时返回null。
下面是示例代码：

```
        Log.e("Test", "${myList.elementAt(1)}")      // 4
        Log.e("Test", "${myList.elementAtOrElse(4,{2*it})}")   // 8
        Log.e("Test", "${myList.elementAtOrElse(5,{2*it})}")   // 10
        Log.e("Test", "${myList.elementAtOrNull(10)}")    // null
```

### first和firstOrNull函数

这两个函数返回集合中符合条件的第一个元素，没有符合条件的元素时，firstOrNull 函数会返回null,first函数会抛出NoSuchElementException异常。

### indexOf、indexOfFirst和indexOfLast函数

这3个函数分别返回集合中指定元素的下标、集合中第一个符合条件的元素的下标、最后一个符合条件的元素的下标，没有则返回−1。

### last、lastIndexOf和lastOrNull函数

last 函数返回集合中符合指定条件的最后一个元素，如果没有符合条件的元素，则抛出NoSuchElementException 异常；lastIndexOf 函数返回集合中某元素最后一次出现的下标，如果没有符合条件的元素，则返回−1;lastOrNull函数返回集合中符合指定条件的最后一个元素，如果没有符合条件的元素，则返回null。

### single和singleOrNull函数

这两个函数的功能相同，都是返回集合中符合条件的一个元素。如果没有符合条件的元素，或者符合条件的元素超过一个，则 single 函数会分别抛出NoSuchElementException 异常和IllegalArgumentException异常；singleOrNull函数会返回null。

### partition函数

partition函数根据表达式对集合中的元素进行拆分，表达式是true的为一组，表达式是false的为另一组。

```
        Log.e("Test", "${myList.partition { it%4==0 }}")  // ([4, 8], [2, 5, 6])
```

### plus函数

plus函数可以合并两个集合，也可以使用运算符“+”代替函数。

```
        Log.e("Test", "${myList.plus(listOf(10,11))}") //[2, 4, 5, 6, 8, 10, 11]
```

### zip和unzip函数

zip 函数的作用是在两个集合中各取一个数值形成数对，直到长度最短的集合中的数值取完；unzip函数是在数对中各取一个元素形成两个集合。

```
        Log.e("Test", "${myList.zip(listOf(11,12,13,14,15))}")  // [(2, 11), (4, 12), (5, 13), (6, 14), (8, 15)]
        Log.e("Test", "${listOf(Pair(2,11),Pair(5,13),Pair(6,14),Pair(8,15)).unzip()}") // ([2, 5, 6, 8], [11, 13, 14, 15])
```

### reversed函数

reversed函数的功能是将集合中的数值翻转，注意不是降序排列。

### sorted和sortedBy函数

这两个函数的作用是将集合中的数值按升序排列。sorted函数不支持方法，sortedBy函数支持按照表达式取值进行升序排列。

### sortedDescending和sortedByDescending函数

这两个函数的作用是将集合中的数值按降序排列。sortedDescending 函数不支持方法，sortedByDescending函数支持按照表达式取值进行降序排列。

细心的读者可能发现了，排序类操作函数使用的都是过去时（带有 ed），因为这些函数针对的是不可变集合。它们会临时创建一个新的集合，将排好序的集合赋值给临时创建的集合。以上排序函数的特点就是不改变原有集合中数据的存储顺序，即使应用在可变集合上。不带有ed的排序类函数可以改变集合数据的存储顺序，所以不能应用在不可变集合上，只能应用在可变集合上，这些函数有reverse（）、sort（）、sortBy（）、sortDescending（）、sortByDescending（）。

## 参考

《零基础学Kotlin之Android项目开发实战》