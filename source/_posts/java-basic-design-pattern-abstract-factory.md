---
title: Java 设计模式 -- 抽象工厂模式
categories: 设计模式
comments: true
tags: [设计模式]
description: 介绍单例模式的几种实现
date: 2014-11-12 10:00:00
---

## 概述

在前面介绍的工厂方法时，每个工厂方法只能生产一种产品，那么如果我想增加一个产品线，生产电视，该如何做呢？这是就可以用到抽象工厂模式。

## 抽象工厂模式

抽象工厂模式（Abstract Factory）提供一个创建一系列相关或互相依赖的对象的接口，而无需指定它们具体的类。
也就是说，抽象工厂中的每个工厂提供了创建多中产品的方法。
主要角色：

 - 抽象工厂
 - 具体工厂
 - 抽象产品族
 - 抽象产品
 - 具体产品

对比工厂方法，多了抽象产品族，指代多个抽象产品。

下面来修改代码。
新增一个抽象产品类和两个具体产品类来代表电视：

```
/**
 * 抽象产品类
 * 电视类
 * 描述电视名称的公共接口
 */
public abstract class TV {
    public abstract void getTVInfo();
}

/**
 * 具体产品类
 * 魅族电视
 */
public class MeizuTV extends TV {
    @Override
    public void getTVInfo() {
        Log.e("Test","MeizuTV");
    }
}

/**
 * 具体产品类
 * 小米电视
 */
public class XiaoMiTV extends TV {
    @Override
    public void getTVInfo() {
        Log.e("Test","XiaoMiTV");
    }
}
```

然后修改工厂类，新增生产电视产品的方法：

```
/**
 * 抽象工厂类
 * 提供一个抽象方法，生产某个品牌手机
 * 提供一个抽象方法，生产某个品牌电视
 */
public abstract class Factory {
    public abstract Phone getPhone(String config);
    public abstract TV getTV();
}

/**
 * 具体工厂类
 * 生产魅族手机
 * 生产魅族电视
 */
public class MeizuFactory extends Factory {
    @Override
    public Phone getPhone(String config) {
        return new MeizuPhone(config);
    }

    @Override
    public TV getTV() {
        return new MeizuTV();
    }
}

/**
 * 具体工厂类
 * 生产小米手机
 * 生产小米电视
 */
public class XiaoMiFactory extends Factory {
    @Override
    public Phone getPhone(String config) {
        return new XiaoMiPhone(config);
    }

    @Override
    public TV getTV() {
        return new XiaoMiTV();
    }
}
```

客户端代码：

```
        String config = "ram = 8G;sdcard = 128G";
        MeizuFactory meizuFactory = new MeizuFactory();
        Phone meizu = meizuFactory.getPhone(config);
        meizu.introduce();
        TV meizeTV = meizuFactory.getTV();
        meizeTV.getTVInfo();

        XiaoMiFactory xiaoMiFactory = new XiaoMiFactory();
        Phone xiaomi = xiaoMiFactory.getPhone(config);
        xiaomi.introduce();
        TV xiaomiTV = xiaoMiFactory.getTV();
        xiaomiTV.getTVInfo();
```

上面的 Phone 和 TV 就是两个抽象产品，之所以抽象，是因为它们可能有不同的是实现，比如MeizuPhone，MeizuTV，XiaoMiPhone和XiaoMiTV。
Factory 是个抽象工厂，包含了创建两个产品的方法。MeizuFactory和XiaoMiFactory及时具体工厂，可以生产具体的产品。

## 抽象工厂的特点

和工厂方法的区别：如果使用工厂方法实现时如果增加需求，需要多加一个产品线时，由于产品类构成了不同类型的产品族，它就变成抽象工厂模式了；而对于抽象工厂模式，当减少一个方法使的提供的产品不再构成产品族之后，它就演变成了工厂方法模式。
优点：易于扩展和交换产品系列，只需要改变具体工厂即可使用不同的产品配置。具体产品的创建实例过程和客户端分离。
缺点：产品族的扩展比较麻烦。如果要新增加一条手表产品线，至少要新增加三个类，抽象产品类Watch，具体产品类MeizuWatch和XiaoMiWatch，还要修改Factory、MeizuFactory和XiaoMiFactory来新增一个方法来生产手表。

## 用简单工厂对抽象工厂的改善

设计模式没有孰好孰坏，适合自己的才是最好的。
去掉 Factory、MeizuFactory 和 XiaoMiFactory，添加 SimpleFactory 类。

```
public class SimpleFactory {
    public static Phone getPhone(String name,String config) {
        switch (name) {
            case "MeizuPhone":
                return new MeizuPhone(config);
            case "XiaoMiPhone":
                return new XiaoMiPhone(config);
        }
        return null;
    }

    public static TV getTV(String name) {
        switch (name) {
            case "MeizuTV":
                return new MeizuTV();
            case "XiaoMiTV":
                return new XiaoMiTV();
        }
        return null;
    }
}
```

## 用反射对抽象工厂的改进