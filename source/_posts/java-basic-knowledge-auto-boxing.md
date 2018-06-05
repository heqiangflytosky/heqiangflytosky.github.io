---
title: Java 基础 -- 关于自动装箱和拆箱机制
categories: Java
comments: true
tags: [Java]
description: 介绍 Java 的自动装箱和拆箱机制
date: 2014-2-12 10:00:00
---


## 先看一段代码

```
        Integer f1 = 127, f2 = 127, f3 = 128, f4 = 128;
        Integer f5 = new Integer(10);
        Integer f6 = new Integer(10);

        int i1 = f1, i2 = f2, i3 = f3, i4 = f4;

        System.out.println(f1 == f2);
        System.out.println(f3 == f4);
        System.out.println(f5 == f6);
        System.out.println(i1 == i2);
        System.out.println(i3 == i4);
```

结果是：true false false true true

## 自动装箱和拆箱的概念

Java 中引入了基本数据类型，针对着集中基本数据类型，又引入了相对应的包装类型。
Java 为每个原始类型提供了包装类型：

 - 原始类型: boolean，char，byte，short，int，long，float，double
 - 包装类型：Boolean，Character，Byte，Short，Integer，Long，Float，Double

从Java 5开始引入了自动装箱/拆箱机制，使得原始类型和包装类型可以相互转换。
前面代码中就用到了自动装箱/拆箱机制：

```
Integer f1 = 127, f2 = 127, f3 = 128, f4 = 128; // 自动装箱成Integer类型
int i1 = f1, i2 = f2, i3 = f3, i4 = f4;//自动拆箱成int类型
```

自动装箱来得到一个引用数据类型时，jvm 实际上调用了 `valueOf()` 方法

## 问题解析

因为 == 比较的是两个对象是否相同，明显 f1和f2 指向的是同一个对象，f3 和 f4 指向的不是同一个对象。
为什么会这样呢？
这就归结于 Java 对于 Integer 与 int 的自动装箱与拆箱的设计，运用了一种设计模式：享元模式。
为了加大对简单数字的重利用，java定义：在自动装箱时对于值从–128到127之间的值，它们被装箱为Integer对象后，会存在内存中被重用，始终只存在一个对象。
看一下 `Integer.valueOf` 的代码实现：

```
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```

在 `IntegerCache.low` 和 `IntegerCache.high` 之间的数会直接向缓存池 `IntegerCache` 中获取缓存对象，如果在这个范围之外则重新创建对象。
再来看一下  `IntegerCache` 的实现：

```
    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```
其实就是创建了个 [-128, 127] 的数字的缓存池。
那么，为什么要这么设计呢？一般来说，小数字的使用频率很高，将小数字保存起来，让其始终仅有一个对象可以节约内存，提高效率。
而如果超过了从 –128 到 127 之间的值，被装箱后的 `Integer` 对象并不会被重用，即相当于每次装箱时都新建一个 `Integer` 对象。这就是为什么 f5 == f6 为false的原因了。
以上的现象是由于使用了自动装箱所引起的，如果你没有使用自动装箱，而是跟一般类一样，用 new 来进行实例化，就会每次 new 就都一个新的对象；
至于 i1 == i2 和i3 == i4，这属于基本数据类型的比较。

## String 类的自动装箱拆箱

```
        String s1 = "str", s2 = "str";
        String s3 = new String("str");
        String s4 = new String("str");
        System.out.println(s1 == s2);
        System.out.println(s3 == s4);
```

结果为 true false。

String 有个字符串常量池 - String Pool，通过自动装箱得到的对象都存储在这个字符串常量池里面，相同的字符串不会重复创建。
通过显式 new 出来的对象不在这里面，每次都会创建一个新的对象。

## Long 类的自动装箱拆箱

Long 的自动装箱和 Integer 类似：

```
    public static Long valueOf(long l) {
        final int offset = 128;
        if (l >= -128 && l <= 127) { // will cache
            return LongCache.cache[(int)l + offset];
        }
        return new Long(l);
    }
```

```
    private static class LongCache {
        private LongCache(){}

        static final Long cache[] = new Long[-(-128) + 127 + 1];

        static {
            for(int i = 0; i < cache.length; i++)
                cache[i] = new Long(i - 128);
        }
    }
```
