---
title: Java 基础 -- 反射
categories: Java
comments: true
tags: [Java]
description: 介绍反射的用法
date: 2014-11-2 10:00:00
---

## 概述

什么叫反射？
在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性；这种运行时动态获取信息以及动态调用对象方法的功能称为 Java 语言的反射机制。
反射可以“看透”程序的运行情况，可以让我们在运行期知晓一个类或者实例的运行状况，可以动态地加载和调用，虽然有一定的性能忧患，但它带给我们的便利远远大于其性能缺陷。

实现反射的几个步骤：

 - 获取 Class 类
 - 通过 Class 创建对象
 - 获取类中的方法
 - 获取类中的属性、属性值或者类型

## 反射相关的类

要实现反射机制一般要使用下面四个类：

 - Class
 - Constructor
 - Method
 - Field

### Class 

理解 Class 对象：
https://blog.csdn.net/javazejian/article/details/70768369?utm_source=gold_browser_extension
https://www.cnblogs.com/yaoyinglong/p/Class%E5%AF%B9%E8%B1%A1.html

Java 语言是先把 Java 源文件编译成后缀为 class 的字节码文件，然后再通过 ClassLoader 机制把这些类文件加载到内存中，最后生成实例来执行的，这是 Java 处理的基本机制，但是加载到内存中的数据是如何描述一个类的呢？比如 Dog.class 文件中定义的是一个 Dog 类，那么它在内存中是如何展现的呢？
Java 使用一个元类（MetaClass）来描述加载到内存中的类数据，这就是 Class 类，它是一个描述类的类对象，比如 Dog.classs 文件加载到内存中后就会一个Class 的实例对象来描述它。因为Class类是“类中类”，也就预示着它有很多特殊的地方：

 - 无构造函数：Java 中的类一般都有构造函数，用于创建实例对象。但是 Class 类却没有构造函数，不能实例化，Class 对象是在加载类时由 Java 虚拟机通过调用类的加载器中的defineClass方法自动构造的。
 - 可以描述基本类型：虽然8个基本类型在 JVM 中并不是一个对象，它们一般存在于栈内存中，但是 Class 类仍然可以描述它们。例如：可以使用 int.class 标识int类型的类对象。
 - 其对象都是单例模式：一个Class的实例对象描述一个类，并且只描述一个类，反过来也成立，一个类只有一个Class实例对象。
 
下面的代码都是返回true的：

```
// 类的属性class所引用的对象与实例对象的getClass返回值相同
String.class.equals(new String().class);
"ABC".getClass.equals(String.class);
// class 实例对象不区分泛型
ArrayList.class.equals(new ArrayList<String>().getClass());
```

Class 类是 Java 反射的入口，只有在获得了一个类的描述对象后才能动态地加载、调用。
下面介绍一下获取 Class 的三种方法：

第一种方式：通过对象的 getClass 方法获取：

```
        OuterClass outerClass = new OuterClass();
        Class cls = outerClass.getClass();
```

第二种方式：直接通过类名获取，也就是通过类属性的方式：

```
        Class cls = OuterClass.class;
```

第三种方式：通过类的全称获取，使用 forName 方法加载：

```
        try {
            Class cls = Class.forName("com.example.hq.testsomething.OuterClass");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
```

获得了 Class 对象后，就可以通过下面Class 的几个方法来获取构造方法、属性、方法以及注解，为后续的反射代码铺平了道路：

 - getConstructors()/getDeclaredConstructors()：获取类中所有的构造方法
 - getFields()/getDeclaredFields()：获取类中所有的属性
 - getMethods()/getDeclaredMethods()：获取类中所有的方法
 - getAnnotations()/getDeclaredAnnotationsv()：获取类所有的注解
 - getSuperclass()：获取直接继承的父类
 - getGenericSuperclass()：获取直接继承的父类（包含泛型参数）


### `get*` 和 `getDeclared*` 方法的区别

Java Class 类提供了很多的 `get*` 和 `getDeclared*` 方法，比如 getDeclaredMethods() 和 getMethods()，那么它们之间有什么差别呢？
getMethods() 方法获取的是所有的 public 访问级别的方法，包括从父类继承的方法。
getDeclaredMethods() 方法获得的是自身类的所有方法，包括公用方法、私有方法等，而且不受限于访问权限。但不包括继承的方法。
getDeclaredField（s）和getField（s）同上。
Java 之所以如此处理，是因为反射本身只是正常代码逻辑的一种补充，而不是让正常代码逻辑产生翻天覆地的变动，所以public得属性和方法最容易获取，私有属性和方法也可以获取，但要先定本类。

### Method

Method 的几个方法：

 - getModifiers()：获取方法修饰符，比如Modifier.PUBLIC、Modifier.STATIC等
 - getDeclaringClass()：返回表示声明由此Method对象表示的方法的类的Class对象


### Field

Field 的几个方法：

 - getDeclaringClass()：返回表示声明由Field对象表示的字段的类或接口的 Class 对象。比如 mName 是 Student 类的对象，那么这个方法就返回 Student 对应的 Class 对象。
 - getClass()：返回 Field Class 对象。
 - getType()：返回一个Class对象，用于标识Field对象所表示的字段的声明类型。比如 mName 是String 类型的，那么就返回 String 对应的 Class 对象。

## 实例化对象

实例化对象的方法：

 - Class.newInstance()：只能够调用无参的构造函数，即默认的构造函数。要求被调用的构造函数是可见的，也即必须是public类型的。
 - Constructor.newInstance()：可以根据传入的参数，调用任意构造构造函数。可以调用私有的构造函数，需要通过setAccessible（true）实现。

通过 Class.newInstance 方法实例化对象：

```
        try {
            OuterClass oc = (OuterClass) cls.newInstance();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
```

通过 Constructor.newInstance() 方法实例化对象：

```
        try {
            Constructor constructor = cls.getDeclaredConstructor(int.class, String.class);
            constructor.setAccessible(true);
            OuterClass oc = (OuterClass) constructor.newInstance("1","test");
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
```

## 关于 setAccessible

我们平时在反射代码时，尤其时反射调用私有的 Field 或者 Method 时，会设置 `setAccessible(true)`，如果不设置，就会抛出异常：

```
java.lang.IllegalAccessException: Class java.lang.Class<com.example.heqiang.testsomething.commontest.OtherTestActivity> cannot access private  field boolean com.example.heqiang.testsomething.reflect.TestA.mTestBoolean of class java.lang.Class<com.example.heqiang.testsomething.reflect.TestA>
```

那么可能大家就会认为 Accessible 代表的是访问权限的控制。其实并不是这样，我们可以访问一个公有public的属性或者方法，不去设置 `setAccessible(true)`，此时是可以访问或者执行的。但是我们打印下面的代码：

```
field.isAccessible()
method.isAccessible()
```

发现返回的是 false。
为什么返回了 false 还能执行呢？这也说明，属性和方法对象的Accessible属性并不是用来决定是否可以访问的。
Accessible 其实指的是是否更容易获得，是否进行安全检查。
我们知道，动态修改一个类或者方法或者执行方法时都会受到Java安全体系的制约，而安全的处理是非常消耗资源的（性能非常低），因此对于运行期要执行的方法或要修改的属性就提供了 Accessible 可选项：由开发者决定是否要逃避安全体系的检查。
查看源码，我们发现 Field 、Method 和 Constructor 都是继承自 AccessibleObject 的，isAccessible() 和 setAccessible 都是 AccessibleObject 的方法。

```
public class AccessibleObject implements AnnotatedElement {
    boolean override;
    
    public void setAccessible(boolean flag) throws SecurityException {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) sm.checkPermission(ACCESS_PERMISSION);
        setAccessible0(this, flag);
    }

    /* Check that you aren't exposing java.lang.Class.<init>. */
    private static void setAccessible0(AccessibleObject obj, boolean flag)
        throws SecurityException
    {
        if (obj instanceof Constructor && flag == true) {
            Constructor<?> c = (Constructor<?>)obj;
            if (c.getDeclaringClass() == Class.class) {
                throw new SecurityException("Can not make a java.lang.Class" +
                                            " constructor accessible");
            }
        }
        obj.override = flag;
    }
}
```

设置 setAccessible 其实就是设置 override 属性。
再来看一下 Method 的 invoke 方法：

```
    public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass(1);

                checkAccess(caller, clazz, obj, modifiers);
            }
        }
        MethodAccessor ma = methodAccessor;             // read volatile
        if (ma == null) {
            ma = acquireMethodAccessor();
        }
        return ma.invoke(obj, args);
}
```

如果 override 为 true，那么就不需要执行安全检查，这就可以大幅度地提升了系统性能（当然，由于取消了安全检查，也就可以允许private方法，访问private私有属性了）。经过有人测试，在大量的反射情况下，设置 Accessible 为true可以提升性能 20 倍以上。
因此，我们在进行反射编程时，设置 Accessible 为true，不仅仅是因为操作习惯的问题，这是在为我们的系统性能考虑。

## 是否需要太关注反射效率

我们平时编程，总会有人经常说：反射的效率是非常低的，不到万不得已就不要使用。事实上，这句话的前半句是对的，后半句是错的。
反射的效率相对于正常的代码执行确实效率低很多（有人经过测试，相差15倍左右），但它是一个非常有效的运行期工具类，只要代码结构清晰、可读性好那就先开发起来，等到性能测试时证明此处性能确实有问题再修改也不迟（一般情况下反射并不是性能的终极杀手，而代码结构混乱、可读性差则很可能埋下性能隐患）。很少有项目是因为反射问题引起系统效率故障的。而根据二八原则，80%的性能消耗在20%的代码上，这20%的代码才是我们关注的重点，不要单单把反射作为重点关注对象。

## 反射和private属性

我们经常会被问到这样的问题：我们通过反射恶意访问private成员和方法，既然能访问为什么要private，它的使用意义是什么？
首先，private的属性和方法是可以通过反射来访问的。private 并不是来解决安全问题的，它只是OOP（面向对象编程）的封装概念，想做到的是不暴露内部的实现。如果你想解决安全的问题，就要想想别的办法了。
如果在特殊的情况下，你需要关注甚至修改内部实现（反射），这种例外的存在并不影响封装的理念。这更多的是一种“知道这么多就够了”，而不是“这是机密，坚决不能让外人知道”。
setAccessible也是一种hack，它可以取消安全检查，它的意义是：你清楚内部实现，你知道你在做什么，相信你不会搞砸。

## 反射和final属性

关于反射修改final变量的问题，网上有文章说不能按照修改普通变量的方式来修改，要修改 Filed 的 modifiers 属性，把final属性去掉后才能修改，对此我做过测试，不修改 modifiers 属性也是可以对final变量进行修改的，但是和普通的变量修改也有区别：就是不管final是不是public的都要调用 `setAccessible(true)`，否则还是报错。
我是在 Android 平台上测试的，可能是底层虚拟机的差异导致的，先记录于此，后面有时间再详细了解。

## 反射实现代码

### 测试类

TestA

```
public class TestA {
    private TestB mTestB;
    private boolean mTestBoolean = false;
    public TestA(){
        mTestB = new TestB();
    }

    public void getTestBoolean(){
        Log.e("Test","mTestBoolean = "+mTestBoolean);
    }

    private void testPrivateMethod(){
        Log.e("Test","testPrivateMethod!");
    }

    private String testPrivateMethodWithArgs(String arg1,int arg2){
        Log.e("Test","testPrivateMethodWithArgs! arg1 = "+arg1+",arg2 = "+arg2);
        return "return "+arg1;
    }
}
```

TestB

```
public class TestB {
    public TestB(){

    }

    private void functionA(){
        Log.e("Test","functionA()");
    }
}
```

OutterClass

```
public class OuterClass {
    private InnerClass mInnerClass;
    public OuterClass(){
        mInnerClass = new InnerClass();
        mInnerClass.printInt();
    }

    private class InnerClass{
        private int mInt = 1;
        private String mStr = "Test";
        public void printInt(){
            Log.d("Test","mInt = "+mInt);
        }
    }
}
```

### 获取属性值

```
    public void getFiled(){
        TestA a = new TestA();

        try {
            //或者a.getClass().getDeclaredField("mTestBoolean");
            Field field = TestA.class.getDeclaredField("mTestBoolean");
            field.setAccessible(true);
            Object value = field.get(a);
            if(value != null && value instanceof Boolean){
                Log.e("Test"," value = "+value);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void getFiled1(){
        TestA a = new TestA();

        try {
            Class classObj = Class.forName("com.example.heqiang.testsomething.TestA");
            Field field = classObj.getDeclaredField("mTestBoolean");
            field.setAccessible(true);
            Object value = field.get(a);
            if(value != null && value instanceof Boolean){
                Log.e("Test"," value = "+value);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

### 修改属性值

```
    public void setField(){
        TestA a = new TestA();
        a.getTestBoolean();

        try {
            Field field = TestA.class.getDeclaredField("mTestBoolean");
            field.setAccessible(true);
            field.set(a,true);
        } catch (Exception e) {
            e.printStackTrace();
        }

        a.getTestBoolean();
    }

```

### 调用方法

```
    public void invokeMethod(){
        TestA a = new TestA();
        try {
            Method method = TestA.class.getDeclaredMethod("testPrivateMethod");
            method.setAccessible(true);
            method.invoke(a);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

### 调用带参数方法并获取返回值

```
    public void invokeMethodWithArgs(){
        TestA a = new TestA();
        Object result = null;
        try {
            Method method = TestA.class.getDeclaredMethod("testPrivateMethodWithArgs",String.class,int class);
            method.setAccessible(true);
            result = method.invoke(a,"Test",1);
        } catch (Exception e) {
            e.printStackTrace();
        }
        if(result != null && result instanceof String){
            Log.e("Test"," result = "+result);
        }
    }
```

### 调用私有属性的方法

```
    public void invokePirvateFieldMethod(){
        TestA a = new TestA();
        Method functionA = null;
        try {
            Field field = TestA.class.getDeclaredField("mTestB");
            field.setAccessible(true);
            Object testB = field.get(a);
            if(testB != null){
                Class testBClass = testB.getClass();
                functionA = testBClass.getDeclaredMethod("functionA");
                functionA.setAccessible(true);
                if(functionA != null){
                    functionA.invoke(testB);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

### 内部类

```
    public void testReflectInnerClass(){
        OuterClass outerClass = new OuterClass();
        try{
            Field outerInnerClassFiled =Class.forName("com.example.heqiang.testsomething.OuterClass").getDeclaredField("mInnerClass");
            outerInnerClassFiled.setAccessible(true);
            Class innerClass = Class.forName("com.example.heqiang.testsomething.OuterClass$InnerClass");
            Object outerInnerClassObject = outerInnerClassFiled.get(outerClass);
            if (outerInnerClassObject.getClass().getName().equals(innerClass.getName())){
                Field innerIntFiled =innerClass.getDeclaredField("mInt");
                innerIntFiled.setAccessible(true);
                Field innerStrFiled =innerClass.getDeclaredField("mStr");
                innerStrFiled.setAccessible(true);
                Method innerFucMethod =innerClass.getDeclaredMethod("printInt");
                innerFucMethod.setAccessible(true);

                innerFucMethod.invoke(outerInnerClassObject);
                innerIntFiled.set(outerInnerClassObject, 8);
                innerFucMethod.invoke(outerInnerClassObject);

                Object innerStrObject = innerStrFiled.get(outerInnerClassObject);
                Log.e("Test","mStr length = "+((String)innerStrObject).length());
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```

### 获取方法的类型和参数

```
public class ReflectTest {
    public static void testFunc(String grg0,int arg1){

    }

    public final int testFunc1(){
        return 0;
    }
}
```

```
    public void testRefect(){
        Method[] methods = ReflectTest.class.getMethods();
        for (Method method : methods) {
            Log.d("Test","---------------------->");
            String methodName = method.getName();
            Log.d("Test","methodName = "+methodName);
            int modifiers = method.getModifiers();
            if((modifiers & Modifier.PUBLIC) != 0)
                Log.d("Test","public");
            if((modifiers & Modifier.PRIVATE) != 0)
                Log.d("Test","private");
            if((modifiers & Modifier.STATIC) != 0)
                Log.d("Test","static");
            if((modifiers & Modifier.FINAL) != 0)
                Log.d("Test","final");
            Class<?>[] parameterTypes = method.getParameterTypes();
            Log.d("Test","parameter num = "+parameterTypes.length);
            for(Class c : parameterTypes){
                Log.d("Test","parameter name = "+c.getName());
            }
            Log.d("Test","<---------------------->");
        }
    }
```
结果：

```
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = equals
D/Test    (23196): public
D/Test    (23196): parameter num = 1
D/Test    (23196): parameter name = java.lang.Object
D/Test    (23196): <---------------------->
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = getClass
D/Test    (23196): public
D/Test    (23196): final
D/Test    (23196): parameter num = 0
D/Test    (23196): <---------------------->
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = hashCode
D/Test    (23196): public
D/Test    (23196): parameter num = 0
D/Test    (23196): <---------------------->
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = notify
D/Test    (23196): public
D/Test    (23196): final
D/Test    (23196): parameter num = 0
D/Test    (23196): <---------------------->
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = notifyAll
D/Test    (23196): public
D/Test    (23196): final
D/Test    (23196): parameter num = 0
D/Test    (23196): <---------------------->
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = testFunc
D/Test    (23196): public
D/Test    (23196): static
D/Test    (23196): parameter num = 2
D/Test    (23196): parameter name = java.lang.String
D/Test    (23196): parameter name = int
D/Test    (23196): <---------------------->
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = testFunc1
D/Test    (23196): public
D/Test    (23196): final
D/Test    (23196): parameter num = 0
D/Test    (23196): <---------------------->
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = toString
D/Test    (23196): public
D/Test    (23196): parameter num = 0
D/Test    (23196): <---------------------->
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = wait
D/Test    (23196): public
D/Test    (23196): final
D/Test    (23196): parameter num = 0
D/Test    (23196): <---------------------->
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = wait
D/Test    (23196): public
D/Test    (23196): final
D/Test    (23196): parameter num = 1
D/Test    (23196): parameter name = long
D/Test    (23196): <---------------------->
D/Test    (23196): ---------------------->
D/Test    (23196): methodName = wait
D/Test    (23196): public
D/Test    (23196): final
D/Test    (23196): parameter num = 2
D/Test    (23196): parameter name = long
D/Test    (23196): parameter name = int
D/Test    (23196): <---------------------->
```



