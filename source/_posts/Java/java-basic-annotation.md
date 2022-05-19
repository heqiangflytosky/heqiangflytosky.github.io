---
title: Java基础 -- 注解
categories: Java
comments: true
tags: [Java]
description: 介绍注解的用法
date: 2016-12-22 10:00:00
---

## 概述

Java 从1.5 版本后开始引入了注解（Annotation），其目的是不影响代码语义的情况下增强代码的可读性，并且不改变代码的执行逻辑。
注解相当于一种标记，在程序中加上了注解就等于为程序加上了某种标记，以后，JAVAC编译器，开发工具和其他程序可以用反射来了解你的类以及各种元素上有无任何标记，看你有什么标记，就去干相应的事。
注解可以用来注解类、方法、属性和方法参数等。

其实关于注解的使用有两派的争论，正方认为注解有助于数据和代码的耦合，在有代码的周边集合数据，反方认为注解把代码和数据混淆在一起，增加了代码的易变性，削弱了代码的健壮性和稳定性。
我们姑且不去看这些争论，在合适的地方去使用注解，确实能给我们带来不少便利。
目前看到的很多开源框架大部分都会有注解使用的影子，本文就简单的介绍一下注解的使用。

## 常用注解

### Java 注解

 - @Override
 - @JavascriptInterface
 - @Deprecated
 - @SuppressWarnings
 - @Nullable
 - @NonNull
 - @StringRes
 - @DrawableRes
 - @ColorRes
 - @InterpolatorRes
 - @AnyRes
 - @IntDef

### Android 注解

使用时要在build.gradle中加上 `compile 'com.android.support:support-annotations:22.0.0'`

 - @StringDef
 - @UiThread
 - @MainThread
 - @WorkerThread
 - @BinderThread
 - @ColorInt
 - @Size
 - @IntRange
 - @FloatRange
 - @RequiresPermission
 - @CallSuper
 - @CheckResult
 - @VisibleForTesting
 - @Keep
 - @CheckResult


## 元注解

元注解是 java API 提供，是专门用来定义注解的注解。
元注解有四个：

 - @Retention
 - @Target
 - @Document
 - @Inherited

### Retention

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.ANNOTATION_TYPE})
public @interface Retention {
    RetentionPolicy value();
}
```

```java
public enum RetentionPolicy {
    SOURCE,
    CLASS,
    RUNTIME;

    private RetentionPolicy() {
    }
}
```

`Retention` 表示该注解信息保存在什么级别，可选的参数值在枚举类型 RetentionPolicy 中，包括：

 - RetentionPolicy.SOURCE：在编译阶段丢弃。这些注解在编译结束之后就不再有任何意义，所以它们不会写入字节码。`@Override`, `@SuppressWarnings` 都属于这类注解。 
 - RetentionPolicy.CLASS：默认值，编译器仅把注解保存在 class 文件中，在运行 Java 程序时，JVM 不会保留注释，即不能用反射（在运行期）来获取注释。`@StringRes`，`@DrawableRes`，`@MainThread` 都属于这类注解。 
 - RetentionPolicy.RUNTIME：编译器不仅把注解保存在 class 文件中，同时在运行 Java 程序时，JVM 也会保留注释，即可以通过反射来获取注释。`@Nullable`，`@NonNull` 都属于这类注解。 

### Target

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.ANNOTATION_TYPE})
public @interface Target {
    ElementType[] value();
}
```

```java
public enum ElementType {
    TYPE,
    FIELD,
    METHOD,
    PARAMETER,
    CONSTRUCTOR,
    LOCAL_VARIABLE,
    ANNOTATION_TYPE,
    PACKAGE,
    TYPE_PARAMETER,
    TYPE_USE;

    private ElementType() {
    }
}
```

`Target` 表示该注解用于什么地方，可能的值在枚举类 ElemenetType 中，包括：

 - ElemenetType.TYPE：修饰类、接口或枚举（enum) 
 - ElemenetType.FIELD：注解成员变量
 - ElemenetType.METHOD：注解方法
 - ElemenetType.PARAMETER：注解方法参数 
 - ElemenetType.CONSTRUCTOR：注解构造函数
 - ElemenetType.LOCAL_VARIABLE：注解局部变量 
 - ElemenetType.ANNOTATION_TYPE：注解另一个自定义注解 
 - ElemenetType.PACKAGE：注解包
 - ElemenetType.TYPE_PARAMETER：该注解能写在类型变量的声明语句中
 - ElemenetType.TYPE_USE：该注解能写在使用类型的任何语句中

### Document

将此注解包含在 javadoc 中 ，它代表着此注解会被javadoc工具提取成文档。在 doc 文档中的内容会因为此注解的信息内容不同而不同。相当与@see,@param 等。

### Inherited

允许子类继承父类中的该注解。如果不标记@Inherited，那么父类的该注解子类将无法继承。

## 自定义注解

使用@interface自定义注解时，自动继承了 `java.lang.annotation.Annotation` 接口，由编译程序自动完成其他细节。在定义注解时，不能继承其他的注解或接口。`@interface` 用来声明一个注解，其中的每一个方法实际上是声明了一个配置参数。方法的名称就是参数的名称，返回值类型就是参数的类型。可以通过default来声明参数的默认值。
自定义注解格式：
public @interface 注解名 {定义体}
注解参数的可支持数据类型：

 - 所有基本数据类型（int,float,boolean,byte,double,char,long,short)
 - String类型
 - Class类型
 - enum类型
 - Annotation类型
 - 以上所有类型的数组

参数设定：

 - 只能用public或默认(default)这两个访问权修饰.例如,String value();这里把方法设为defaul默认类型；
 - 参数成员只能用上面列举的集中类型
 - 参数可以设置默认值，如果不设置，必须在添加注解时指定。

```java
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface Fruit{
    String name();
    int grade() default 0;
    String country() default "China";
    String[] saleTo() default {};
}
```

## 注解解析

我们添加自定义注解的目的就是对标注的类或者方法进行一些特殊处理，这个就需要我们添加注解处理器。
`Class` 类提供了下面的方法来获取类的注解信息：

 - `<A extends Annotation> A getAnnotation(Class<A> annotationType)`：获取指定的注解，该注解可以是自己声明的，也可以是继承的
 - `Annotation[] getAnnotations()`：获取所有的注解，包括自己声明的以及继承的
 - `Annotation[] getDeclaredAnnotations()`：获取自己声明的注解
 - `isAnnotationPresent(Class<? extends Annotation> annotationType)`：检查类是否有注解
 - `<A extends Annotation> A[] getAnnotationsByType(Class<A> annotationClass)`
 - `<A extends Annotation> A getDeclaredAnnotation(Class<A> var1)`

`Method` 类提供了下面的方法来获取方法的注解信息：

 - `<A extends Annotation> A getAnnotation(Class<A> annotationType)`
 - `Annotation[][] getParameterAnnotations()`

`Field` 类提供了下面的方法来获取属性的注解信息：

 - `isAnnotationPresent(Class<? extends Annotation> annotationType)`
 - `<A extends Annotation> A getAnnotation(Class<A> annotationType)`
 - `Annotation[] getDeclaredAnnotations()`

测试代码：

```
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface Fruit{
    String name();
    int grade() default 0;
    String country() default "China";
    String[] saleTo() default {};
}

```

```
@Fruit(name = "Apple", saleTo = {"China","USA"})
public class TestAnnotation {
    public void commonMethod() {

    }

    @InternalAPI(name = "annotationAPI", callerList = {"MainActivity","Custom"})
    public void annotationAPI(String para) {

    }
}
```

```

@Fruit(name = "Apple", saleTo = {"China","USA"})
public class TestAnnotation {
    public void commonMethod() {

    }

    @InternalAPI(name = "annotationAPI", callerList = {"MainActivity","Custom"})
    public void annotationAPI(String para) {

    }
}
```

### 类的注解处理

```
    public void classAnnotationProcessor(Object object) {
        Class<?> cls = object.getClass();

        if (cls.isAnnotationPresent(Fruit.class)) {
            Fruit fruit = cls.getAnnotation(Fruit.class);
            if (fruit != null) {
                String name = fruit.name();
                String country = fruit.country();
                int grade = fruit.grade();
                String [] saleTo = fruit.saleTo();

                Log.e("Test","name = "+name+",country = "+country+", grade = "+grade+", saleTo = "+Arrays.toString(saleTo));
            }
        }

    }
```

### 方法的注解处理

```
    public void methodAnnotationProcessor(Object object) {
        Class<?> cls = object.getClass();

        Method[] methods = cls.getDeclaredMethods();

        for (Method method : methods) {
            // 获取方法修饰符
            int modifiers = method.getModifiers();
            // 获取方法参数列表
            Class<?>[] parameterTypes = method.getParameterTypes();
            Log.e("Test","modifiers = "+modifiers+", parameterTypes="+ Arrays.toString(parameterTypes));
            // 获取方法的制定注解，如果没有被指定注解修饰，则返回 null
            InternalAPI internalAPI = method.getAnnotation(InternalAPI.class);

            if (internalAPI != null) {
                // 获取注解参数
                String name = internalAPI.name();
                boolean isHide = internalAPI.isHide();
                String[] list = internalAPI.callerList();
                Log.e("Test","name = "+name+",isHide = "+isHide+", caller = "+Arrays.toString(list));
            }
        }
    }
```

**注意：**

 - 子类重写的方法，注解无法被继承

## annotationProcessor

annotationProcessor 是处理注解的工具，它用来在编译时扫描和处理注解。
在编译的时候就把注解信息给保存下来，这就省了我们在代码中寻找注解的开销，这也是注解类框架非常流行的原因。
后面会有专门写一篇博客园来介绍。

## 注解的运用

### 使用注解改进代码检查 

[Android 官方文档：使用注解改进代码检查](https://developer.android.com/studio/write/annotations)

### 注解代替Enum

[我的博客：Java 基础 -- enum](http://www.heqiangfly.com/2013/06/26/java-basic-enum/)