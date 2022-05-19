---
title: Java 基础 -- enum
categories: Java
comments: true
tags: [Java]
description: 介绍 enum 的用法
date: 2013-6-26 10:00:00
---

## 概述

枚举类型是指由一组固定的常量组成合法值的类型，它是在 Java 1.5 引入的一种类型，在此之前，我们只有两种声明常量的方式：类常量和接口常量。
枚举的引入改变了常量的声明方式，在枚举之前，我们表示枚举类型的常用模式是声明一组具名的 int 常量，每个类型成员一个常量：

```
interface Fruit{
    int APPLE = 0;
    int PEAR = 1;
    int LEMON = 2;
    int BANANA = 3;
    int ORANGE = 4;
}
```

这种方式称为 int 枚举模式，它存在着类型安全性和使用方便性等诸多不足。
我们可以用下面方式声明枚举：

```
enum Fruit{APPLE, PEAR, LEMON, BANANA,ORANGE}
```

使用枚举时，每个枚举项都是该枚举的实例对象。比如 APPLE 就是 Fruit 的一个实例对象。
枚举无法被继承，但可以继承其他类或者实现其他接口。

## 枚举的优势

我们为什么要使用枚举呢？下面我们来说一下它和静态常量的优势。

### 枚举常量更简单

枚举常量只需要定义每个枚举项，不需要定义枚举值，而接口常量或者类常量则必须定义值，否则编译不过，即使我们不需要关心其值是多少也必须定义。
其次，虽然两者被引用方式相同（都是类名.属性，比如 Fruit.APPLE），但是枚举表示的是一个枚举项，字面含义是苹果，而接口常量其字面含义也是苹果，但是在运算中我们势必要关注其 int 值。

### 枚举常量属于稳态型

如果我们用 switch 语句来使用接口常量，我们必须对输入值进行检查，确定其是否越界，而且容易造成一些在编译期通过，但是运行期产生不符合预期的后果。
使用枚举不用关心这些校验问题，它在编译器限定类型，不允许发生越界的情况。


### 枚举具有内置方法

枚举的内置方法有：

 - valueOf():返回具有指定名称的枚举类型的枚举常量，比如 Fruit.valueOf("ORANGE") 返回的就是枚举值 ORANGE。
 - values():列出所有枚举项

每个枚举都是 java.lang.Enum 的子类，该类也提供了很多方法：

 - ordinal():获得枚举的排序值，从 0 开始的 int 类型；
 - compareTo()：比较方法

### 枚举可以自定义方法

枚举不仅可以定义静态方法，还可以定义非静态方法，而且还能够从根本上杜绝常量类被实例化。

```
public enum Fruit {
    APPLE("apple",3),
    PEAR("pear",4),
    LEMON("lemon",5),
    BANANA("banana",6),
    ORANGE("orange",7);

    private String name;

    //带参数的构造方法
    Fruit(String name, int id) {
        this.name = name;
    }

    public static Fruit getFavourite() {
        return APPLE;
    }

    public String getName() {
        return name;
    }
}
```

## 使用构造函数协助描述枚举项

一般来说，我们经常使用的枚举项只有一个属性，即排序号，其默认值是从0、1、2、3……，这一点我们很熟悉。
但是枚举还可以有一个或多个属性：枚举描述，它的含义是通过枚举的构造函数，声明每个枚举项（也就是枚举实例）必须具有的属性和行为，这是对枚举项的描述或者补充，目的是使枚举项表述的意义更加清晰准确。
比如：

```
public enum Fruit {
    APPLE("apple",3),
    PEAR("pear",4),
    LEMON("lemon",5),
    BANANA("banana",6),
    ORANGE("orange",7);

    private String name;
    private int id;

    //带参数的构造方法
    Fruit(String name, int id) {
        this.name = name;
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public int getID() {
        return id;
    }
}
```

上面代码添加了 name 和 id 的描述信息。

## 使用枚举实现工厂方法模式

工厂方法模式是“创建对象的接口，让子类决定实例化哪一个类，并使一个类的实例化延迟到其子类”。
工厂方法模式一般可以通过下面实现：

```
    interface Car{

    }

    class FordCar implements Car{

    }

    class BuickCar implements Car{

    }

    class CarFactory{
        public static Car creat(Class<? extends Car> c) {
            try {
                return c.newInstance();
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
            return null;
        }
    }
```

有了工厂方法模式，我们就不关心一辆车具体是怎么生成的了，只要告诉工厂“我要生产一辆福特汽车”就行了。

```
Car car = CarFactory.creat(FordCar.class);
```

其实通过枚举也可以实现工厂方法模式：

### 枚举非静态方法实现工厂方法模式

我们知道每个枚举项都是该枚举的实例对象，那是不是定义一个方法可以生成每个枚举项的对应产品来实现此模式呢？

```
    enum CarFactory {
        FordCar,
        BuickCar;

        public Car create() {
            switch (this) {
                case FordCar:
                    return new FordCar();
                case BuickCar:
                    return new BuickCar();
                default:
                    // TODO
                    return null;
            }
        }
    }
```

create 是一个非静态方法，也就是只有通过 FordCar、BuickCar枚举项才能访问。采用这种方式实现工厂方法模式时，客户端要生产一辆汽车就很简单了，代码如下：

```
Car car = CarFactory.FordCar.create();
```

### 通过抽象方法生成成品

枚举类型虽然不能继承，但是可以用 abstract 修饰其方法，此时就表示该枚举是一个抽象枚举，需要每个枚举项自行实现该方法，也就是说枚举项的类型是该枚举的一个子类：

```
    enum CarFactory {
        FordCar {
            public Car create() {
                return new FordCar();
            }
        },
        BuickCar{
            public Car create() {
                return new BuickCar();
            }
        };

        public abstract Car create();
    }
```

### 为什么使用枚举类型的工厂方法

看了上面的实现，大家可能会问：为什么要使用枚举类型的工厂方法模式呢？
那是因为它有下面三个有点：

#### 避免错误调用发生

一般工厂方法模式中的生产方法可以接收三种类型的参数：类型参数（我们的例子）、String 参数（生产方法根据String参数判断需要生产什么产品）、int 参数（生产方法根据int值判断需要生产什么产品），这三种参数都是宽泛的数据类型，很容易产生错误，而且这种错误编译器不会报警，只有到运行期间才能发现。
而使用枚举类型的工厂模式就不存在该问题了，不需要传递任何参数，只需要选择好生产什么类型的产品即可。

#### 使用便捷

#### 降低类间耦合

不管生产方法接收的是Class、String还是int，都会成为客户端的负担，这些类并不是客户端需要的，而是因为工厂方法的限制必须输入的，这个违背了最少知识原则：一个对象应该对其他对象有最少的了解。
二枚举类型的工厂方法就没有这种问题了，它只需要依赖工厂类就行了，完全可以无视具体汽车类的存在。

## 使用枚举实现单例模式

```
public enum Singleton {
    INSTANCE;
    Singleton() {
    }
    public void testMethod() {
        Log.e("Test","enum Singleton test mothod");
    }
}
```

使用方法：

```
        Singleton.INSTANCE.testMethod();
```

关于枚举实现单例的优点会有专门文章来介绍。

## 使用枚举时需要注意的问题

### 小心 switch 带来的空指针异常

```
    {
        setFruit(null);
    }

    private void setFruit(Fruit fruit) {
        switch (fruit) {
            case APPLE:

                break;
            case PEAR:

                break;
            default:
                break;
        }
    }
```

上面的代码执行起来会报下面异常：

```
Caused by: java.lang.NullPointerException: Attempt to invoke virtual method 'int java.lang.Enum.ordinal()' on a null object reference
```

这里怎么会有空指针呢？这就与枚举和 switch 的特性有关了。
我们知道，目前 Java 中的 switch 只能判断 byte、short、char、int
String 类型，这是 Java 编译器的限制，那么为什么枚举可以跟在 switch 后面呢？
很简单，因为编译时，编译器判断出 switch 语句后的参数时枚举类型，然后就会根据枚举的排序值继续匹配，也就是上面的代码和下面相同：

```
    private void setFruit(Fruit fruit) {
        switch (fruit.ordinal()) {
            case Fruit.APPLE.ordinal():
                break;
            case Fruit.PEAR.ordinal():
                break;
            default:
                break;
        }
    }
```

明白了吧？switch语句是先计算 fruit 变量的排序值，然后与枚举常量的每个排序值进行比较的。如果我们传入 null 参数，调用 ordinal 方法时就会报空指针了。
正确的做法应该在setFruit方法里面对 fruit 参数进行 null 判断。

### 调用 valueOf 方法注意校验问题

valueOf 方法会把一个 String 类型的名称转换为枚举项，也就是在枚举项中查找出字面值与该参数相等的枚举项。
那么如果我们输入一个不匹配的值会出现什么结果呢？

```
Fruit fruit = Fruit.valueOf("test");
```

会有下面的异常：

```
Caused by: java.lang.IllegalArgumentException: No enum constant com.example.hq.testsomething.commontest.Fruit.test
```

来看一下源码：

```
    public static <T extends Enum<T>> T valueOf(Class<T> enumType,
                                                String name) {
        if (enumType == null)
            throw new NullPointerException("enumType == null");
        if (name == null)
            throw new NullPointerException("Name is null");
        T[] values = getSharedConstants(enumType);
        T result = null;
        if (values != null) {
            for (T value : values) {
                if (name.equals(value.name())) {
                    result = value;
                }
            }
        } else {
            throw new IllegalArgumentException(enumType.toString() + " is not an enum type.");
        }

        if (result != null)
            return result;
        throw new IllegalArgumentException(
            "No enum constant " + enumType.getCanonicalName() + "." + name);
    }
```

如果找不到匹配值就会抛出异常。
下面有两个方法可以解决：

 - 使用 try cathc 捕获异常；
 - 扩展枚举类；

可以通过增加一个 contains 方法来判断是否包含指定的枚举项，然后再继续进行转换。

```
public enum Fruit {
    APPLE("apple",3),
    PEAR("pear",4),
    LEMON("lemon",5),
    BANANA("banana",6),
    ORANGE("orange",7);

    private String name;
    private int id;

    Fruit(String name, int id) {
        this.name = name;
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public int getID() {
        return id;
    }

    public static boolean contains(String name) {
        for(Fruit fruit: values()) {
            if(fruit.name.equals(name)) {
                return true
            }
        }
        return false;
    }
}
```

## 枚举无法继承

一个枚举常量定义完毕后，除非修改重构，否则无法扩展。而接口常量或者类常量则可通过继承进行扩展。但是可以通过上面的方法实现 abstract 方法。

## 枚举的缺点

枚举变量非常方便，但不幸的是它会牺牲执行的速度和并大幅增加文件体积。例如：

```
public class Foo {
 public enum Fruit { APPLE, PEAR, LEMON }
}
```

会产生一个900字节的 .class 文件(`Foo$Fruit.class`)。在它被首次调用时，这个类会调用初始化方法来准备每个枚举变量。每个枚举项都会被声明成一个静态变量，并被赋值。然后将这些静态变量放在一个名为 "$VALUES" 的静态数组变量中。而这么一大堆代码，仅仅是为了使用三个整数。
这样: `Fruit fruit = Fruit.APPLE;` 会引起一个对静态变量的引用，如果这个静态变量是final int，那么编译器会直接内联这个常数。
一方面说，使用枚举变量可以让你的 API 更出色，并能提供编译时的检查。所以在通常的时候你毫无疑问应该为公共 API 选择枚举变量。但是当性能方面有所限制的时候，你就应该避免这种做法了。尤其是在移动端，对内存的要求比较严格。
鉴于 Enum 会比使用常量带来更大的内存开销，因此 Android 官方也建议，不是在必要的情况下应尽量避免过多使用 Enum，下面介绍使用注解来代替 Enum 的方法：

```
    public static final int APPLE = 0;
    public static final int PEAR = 1;
    public static final int LEMON = 2;
    public static final int BANANA = 3;
    public static final int ORANGE = 4;

    @IntDef ({APPLE, PEAR, LEMON, BANANA, ORANGE})
    @Retention(RetentionPolicy.SOURCE)
    public @interface Fruit{}

    private @Fruit int mFruit;

    ……
    {
            mFruit = APPLE;
    }

    private void setFruit(@Fruit int fruit) {
        mFruit = fruit;
    }
```

