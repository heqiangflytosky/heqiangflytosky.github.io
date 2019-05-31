---
title: Java 设计模式 -- 工厂方法模式
categories: Java
comments: true
tags: [Java]
description: 介绍单例模式的几种实现
date: 2014-11-10 10:00:00
---

## 概述

本文主要介绍一下设计模式中的工厂方法模式，在介绍工厂方法模式之前，先介绍一下简单工厂模式，简单工厂模式并不在我们常说的23种设计模式之列。但它可以帮助我们更好地理解工厂方法模式。
它们和抽象工厂模式的关系可以理解为：简单工厂进阶为工厂方法，工厂方法进阶为抽象工厂。


## 简单工厂模式

简单工厂模式是属于创建型模式，又叫做静态工厂方法模式。简单工厂模式是由一个工厂对象决定创建出哪一种产品类的实例。
通常的做法是把要创建的产品特征通过参数传递给工厂类，由工厂类动态决定创建那个产品。
主要角色：

 - 工厂
 - 抽象产品
 - 具体产品

```
/**
 * 抽象产品类
 * 手机类
 * 描述手机名称的公共接口
 */
public abstract class Phone{
    protected String mConfig;
    public Phone(String config) {
        mConfig = config;
    }
    public abstract void introduce();
}

/**
 * 具体产品类
 * 魅族手机
 */
public class MeizuPhone extends Phone {
    public MeizuPhone(String config) {
        super(config);
    }

    @Override
    public void introduce() {
        Log.e("Test","MeizuPhone "+mConfig);
    }
}

/**
 * 具体产品类
 * 小米手机
 */
public class XiaoMiPhone extends Phone {
    public XiaoMiPhone(String config) {
        super(config);
    }

    @Override
    public void introduce() {
        Log.e("Test","XiaoMiPhone "+mConfig);
    }
}

/**
 * 手机工厂类
 * 负责生产具体的手机
 * 提供一个方法，根据手机品牌名称生产不同品牌手机
 */
public class SimplePhoneFactory {
    public static Phone getPhone(String name,String config) {
        switch (name) {
            case "MeizuPhone":
                return new MeizuPhone(config);
            case "XiaoMiPhone":
                return new XiaoMiPhone(config);
        }
        return null;
    }
}
```

```
        String config = "ram = 8G;sdcard = 128G";
        Phone meizu = SimplePhoneFactory.getPhone("MeizuPhone",config);
        meizu.introduce();
        Phone xiaomi = SimplePhoneFactory.getPhone("XiaoMiPhone",config);
        xiaomi.introduce();
```

优点：工厂类中包含了必要的逻辑判断，根据客户端的选择条件来动态实例化相关产品类。对于客户端来说，去除了与具体产品的依赖，将创建和使用分开，客户端不必关系产品如何创建，实现了解耦。
缺点：简单工厂的缺点也很明显，违背“开放-关闭原则”，一旦添加新产品就不得不修改工厂类的逻辑，这样就会造成工厂逻辑过于复杂。

## 工厂方法模式

工厂方法模式（Factory Method）定义一个工厂的父类，它负责定义一个创建具体产品的公共接口，而工厂子类负责生成具体的产品对象。这样就把产品类的实例化延迟到了工厂类的子类（具体工厂），有具体工厂决定实例化哪一个具体产品类。
比如我们要生成魅族手机，这样就需要一个专门生成魅族手机的工厂类与之对应。

主要角色：

 - 抽象工厂
 - 具体工厂
 - 抽象产品
 - 具体产品

```
/**
 * 抽象产品类
 * 手机类
 * 描述手机名称的公共接口
 */
public abstract class Phone{
    protected String mConfig;
    public Phone(String config) {
        mConfig = config;
    }
    public abstract void introduce();
}

/**
 * 具体产品类
 * 魅族手机
 */
public class MeizuPhone extends Phone {
    public MeizuPhone(String config) {
        super(config);
    }

    @Override
    public void introduce() {
        Log.e("Test","MeizuPhone "+mConfig);
    }
}

/**
 * 具体产品类
 * 小米手机
 */
public class XiaoMiPhone extends Phone {
    public XiaoMiPhone(String config) {
        super(config);
    }

    @Override
    public void introduce() {
        Log.e("Test","XiaoMiPhone "+mConfig);
    }
}

/**
 * 抽象工厂类
 * 提供一个抽象方法，生产某个品牌手机
 */
public abstract class Factory {
    public abstract Phone getPhone(String config);
}

/**
 * 具体工厂类
 * 生产魅族手机
 */
public class MeizuFactory extends Factory {
    @Override
    public Phone getPhone(String config) {
        return new MeizuPhone(config);
    }
}

/**
 * 具体工厂类
 * 生产小米手机
 */
public class XiaoMiFactory extends Factory {
    @Override
    public Phone getPhone(String config) {
        return new XiaoMiPhone(config);
    }
}
```

```
        String config = "ram = 8G;sdcard = 128G";
        MeizuFactory meizuFactory = new MeizuFactory();
        Phone meizu = meizuFactory.getPhone(config);
        meizu.introduce();

        XiaoMiFactory xiaoMiFactory = new XiaoMiFactory();
        Phone xiaomi = xiaoMiFactory.getPhone(config);
        xiaomi.introduce();
```

优点：解决了简单工厂的问题，符合“开放-关闭原则”，新增一种产品时，只需要增加相应的具体产品类和相应的工厂子类即可。符合“单一职责原则”，每个具体工厂类只负责创建对应的产品
缺点：一个具体工厂只能创建一种具体产品，如果需要很多的产品，类的数量需要成对的增加。