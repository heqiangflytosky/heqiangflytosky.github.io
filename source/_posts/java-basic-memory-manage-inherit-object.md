---
title: Java 内存管理 -- 父子实例的内存控制
categories: Java
comments: true
tags: [Java]
description: 介绍 java 的父子实例的内存控制
date: 2014-2-22 10:00:00
---

## 概述

继承是面向对象的3大特征之一，也是Java语言的重要特性。接下来我们分析一下父子实例的内存控制。

## 继承成员变量和继承方法的区别

当子类继承父类时，子类会获得父类中定义的成员变量和方法。当访问权限允许的情况下，子类可以直接访问父类中定义的成员变量和方法。其实这种说法比较笼统，Java 继承中对成员变量和方法的处理是不同的。
下面通过一些特殊情况下的例子来介绍一下：

```
public class Base {
    int i = 2;
    public void display() {
        Log.e("Test","Base i = "+i);
    }
}

public class Derived extends Base {
    int i = 22;
    @Override
    public void display() {
        Log.e("Test","Derived i = "+i);
    }
}

```

测试代码：

```
        //1.声明并创建一个Base对象，并调用i变量和display方法
        Base b = new Base();
        Log.e("Test","i = "+b.i);
        b.display();

        //2.声明并创建一个Derived对象，并调用i变量和display方法
        Derived d = new Derived();
        Log.e("Test","i = "+d.i);
        d.display();

        //3.声明Base变量，创建一个Derived对象赋值给bd，并调用i变量和display方法
        Base bd = new Derived();
        Log.e("Test","i = "+bd.i);
        bd.display();

        //4.声明Base变量，并指向前面创建的Derived对象
        Base b2d = d;
        Log.e("Test","i = "+b2d.i);
        b2d.display();
```

输出结果：

```
E Test    : i = 2
E Test    : Base i = 2

E Test    : i = 22
E Test    : Derived i = 22

E Test    : i = 2
E Test    : Derived i = 22

E Test    : i = 2
E Test    : Derived i = 22
```

程序1和2中情况分别声明和创建了Base和Derived变量和对象，它们的输出结果我们是没有疑问的。
程序3声明了一个Base变量bd，却将一个Derived对象赋给该变量。此时系统将自动进行向上转型来保证程序正确性。问题是直接通过bd访问i变量，输出的是Base（声明时的类型）对象的i实例变量的值；如果通过db来调用display方法，该方法却是Derived（运行时类型）对象的行为方式。
程序4是直接将d变量赋值给d2b变量，只是d2b变量的类型是Base。这意味着d和d2b两个变量指向同一个Java对象，因此，如果在程序中判断d2b == d，将返回true。但是，访问d.i输出22，访问d2b.i时却输出2。这一点看上去有点奇怪：两个指向同一个对象的变量，分别访问它们的实例变量时却输出不同的值。这表明在d2b、d变量所指向的Java对象中包含了两块内存，分别存放值为2的i实例变量和值为22的i实例变量。
但是不管是d变量，还是bd变量、b2d变量，只要它们实际指向一个Derived变量，不管声明它们时用什么类型，当通过这些变量调用方法时，方法的行为总是表现出它们实际类型的行为。但是如果通过这些变量来访问它们所指对象的实例变量，这些实例变量的值总是表现出声明这些变量所用类型的行为。由此可见，Java继承在处理成员变量和方法是有区别的。
看下面代码：

```
class Fruit{
    public String name;
    public void info(){
    }
}

public class Apple extends Fruit {
    private double weight;
}
```

上面代码中，Apple类继承了Fruit类，因此它会获得Apple类中声明的成员变量和方法，下面使用javap工具分析一下Apple类。
$ javap -c -private Apple.class

```
Compiled from "Apple.java"
public class com.example.heqiang.testsomething.commontest.Apple extends com.example.heqiang.testsomething.commontest.Fruit {
  private double weight;

  public com.example.heqiang.testsomething.commontest.Apple();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method com/example/heqiang/testsomething/commontest/Fruit."<init>":()V
       4: return

  public void info();
    Code:
       0: aload_0
       1: invokespecial #2                  // Method com/example/heqiang/testsomething/commontest/Fruit.info:()V
       4: return
}
```

可以看出，当Apple继承Fruit类时，编译器会直接将Fruit里的info方法转移到Apple类中。这意味着，如果Apple也有info方法，那么编译器就无法将Fruit的info方法转移到Apple类中。
**注意**：只有当父类不适用pulic修饰符修饰，子类使用public修饰符修饰时，才可以通过javap看到编译器将父类的public方法直接转移到子类。
如果子类重写了父类的方法，那么子类定义的方法彻底覆盖了父类里面的同名方法，系统将不可能把父类的方法转移到子类了。
但是对于成员变量而言，我们并没有在Apple的字节码中看到Fruit的name成员变量，因此，成员变量不会直接转移到子类，那么使得父类和子类可以拥有同名的实例变量，子类的实例变量不能覆盖父类的同名实例变量，这就是我们所说的隐藏的概念。
因为继承成员变量和继承父类方法之间存在这一的差别，所以对于一个引用类型的变量而言，当通过该变量访问它所引用的对象的实例变量时，该实例变量的值取决于声明该变量时的类型；当通过该变量来调用它所引用的对象的方法时，该方法行为取决于它所实际引用的对象的类型。

## 内存中的子类实例以及对super的理解

上一节我们分析了继承时对于成员变量和方法的不同处理，现在我们引申出来一个问题是：Derived对象在内存中到底是如何存储的，很明显它有两个不同的i实例变量，这意味着必须用两块内存保存它们。
Derived对象不仅存储了它自身i实例变量，还需要存储从Base那里继承到的i实例变量，但是这两个i实例变量在底层是有区别的，程序通过Base型变量来访问该对象的i实例变量时，将输出2，通过Derived型的变量访问对象的i变量时，将输出22。
下面我们在Derived中添加下面的方法：

```
    public void displaySuper() {
        Log.e("Test","Derived i = "+super.i);
    }
```

如果我们访问 displaySuper 方法，输出的是父类Base的成员变量i的值2。
那么此处的super代表什么？绝大部分可能会说：super代表父类的默认实例。这个说法比较笼统，如果super代表父类的默认实例，那么这个默认实例在哪里？
事实上，系统中只有一个Base对象，而这个对象持有两个i实例变量。当创建一个Derived对象之后，它不仅保存了自己定义的所有实例变量，还保存了它的所有父类所定义的全部实例变量。（private类型的也包含吗？）
通过下面的代码来深入了解一下super。

```
public class Fruit{
    String colr = "not defined";
    //返回调用该方法的对象
    public Fruit getThis() {
        return this;
    }
    public void info(){
        Log.e("Test","Fruit info");
    }
}

public class Apple extends Fruit {
    String color = "red";

    @Override
    public void info() {
        Log.e("Test","Apple info");
    }

    // 通过super调用父类的info方法
    public void accessSuperinfo() {
        super.info();
    }

    // 返回super关键字代表的内容
    public Fruit getSuper() {
        // 这里不能直接return super；否则编译器报错
        return super.getThis();
    }
}
```

测试代码：

```
        //创建一个Apple对象
        Apple apple = new Apple();
        //调用getSuper方法获取Apple对象关联的super引用
        Fruit fruit = apple.getSuper();
        // 1.判断apple和fruit的关系
        Log.e("Test",""+(apple == fruit));
        // 2.判断apple和fruit所引用的color实例变量
        Log.e("Test",""+apple.color);
        Log.e("Test",""+fruit.colr);
        // 3.分别通过apple和fruit两个变量调用info方法
        apple.info();
        fruit.info();
        // 4.通过accessSuperinfo调用父类的info方法
        apple.accessSuperinfo();
```

输出：

```
E Test    : true
E Test    : red
E Test    : not defined
E Test    : Apple info
E Test    : Apple info
E Test    : Fruit info
```

通过1处的结果，我们可以断定，Apple对象的getSuper方法中的super.getThis()实际上返回的是该Apple对象本身。
2处的结果，虽然Apple对象的getSuper方法返回的是Apple对象本身，但是它的声明类型是Fruit，因此通过fruit变量访问color实例变量时，该实例变量的值由Fruit类决定。
3处的结果，调用info方法由变量实际所引用的Java对象决定。
4处的结果，当程序在Apple类的accessSuperinfo使用super作为限定调用info方法，该info才真正变现出Fruit类的行为。
另外关于super，如果我们对比getClass().getName() 和super.getClass().getName()的结果，发现它们的输出是一样的。
super.getClass()只是表示调用父类的方法而已。getClass方法来自Object类，它返回对象在运行时的类型。
如果想获得父类的Class，请用getClass().getSuperclass().getName()。

关于super：

 - 子类方法不能直接使用reture super，但使用return this返回调用该方法的对象时允许的。
 - 程序不允许直接把super当成变量使用，如果判断super和a变量是否引用同一个对象（super == a），这条代码会引起编译器错误。

通过上面的分析可以看出：super关键字本身并没有引用任何对象，它并没有代表超类的一个引用的能力，只是代表调用父类的方法而已。它甚至不能被当做一个真正的引用变量引用。

现在我们给出关于父、子对象在内存中存储的结论：当程序创建一个子类对象时，系统不仅会为该类定义的实例变量分配内存，也会为其所有父类中定义的所有实例变量分配内存，即使子类定义了与父类中同名实例变量。也就是说，当系统创建一个Java对象时，如果该Java类有两个父类（直接父类A，间接父类B），假设A类中定义了2个实例变量，B类中定义了3个实例变量，当前类定义了2个实例变量，那么这个Java对象将会保存2+3+2=7个实例变量。
如果在子类中定义了与父类中已有变量同名的变量，那么子类中定义的变量会隐藏父类中定义的变量。注意，不是完全覆盖，因此系统创建子类时，依然会为父类中定义的、被隐藏的变量分配内存空间。
为了在子类方法中访问父类中定义的、被隐藏的变量，或者为了在子类方法中调用父类中定义的、被覆盖（override）的方法，可以通过super作为限定来调用父类的变量或者方法。


## 参考

《疯狂Java-突破程序员基本功的16课》