---
title: Java InvocationHandler 实现动态代理
categories: Java
comments: true
tags: [Java, 动态代理]
description: 介绍使用 InvocationHandler 实现 Java 动态代理及其原理介绍
date: 2015-4-10 10:00:00
---

## 概述

代理是设计模式的一种，设计模式是指调用者并不直接调用实际的对象，而是通过调用代理，来间接的调用实际的对象。代理类与真正实现的类都是继承或者实现一个相同的类或这接口，这样的好处在于代理类可以与实际的类有相同的方法，可以保证客户端使用的透明性。
既然有动态代理，那么相对应的肯定也存在静态代理。

 - 静态代理：由程序员手工编写代理类。
 - 动态代理：在程序运行时运用反射机制动态创建生成代理类。

静态代理略去不介绍，下面介绍一下动态代理的实现。我们可以使用JDK的 InvocationHandler 和 Proxy 实现，也可以通过 CGLIB 来实现，他们的区别后面介绍，这里之介绍 InvocationHandler 和 Proxy 的实现。

## 动态代理

Java的动态代理主要涉及两个类，`Proxy` 和 `InvocationHandler`。

 - `Proxy` 用于为接口生成动态代理类及其对象。
 - `InvocationHandler` 提供了 `invoke` 方法，负责集中处理动态代理类上的所有方法调用。

代理类会负责将所有的方法调用分派到委托对象上反射执行，在分派执行的过程中，开发人员还可以按需调整委托类对象及其功能，这是一套非常灵活有弹性的代理框架。
下面通过一个实例来演示一下：

```java
public interface Fruit {
    String getName();
}
```

```java
public class Apple implements Fruit {
    @Override
    public String getName() {
        Log.e("Test","I am apple!");
        return "Apple";
    }
}
```

```java
    public static void testProxy(){

        final Apple apple = new Apple();

        InvocationHandler invocationHandler = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                if("getName".equals(method.getName())){
                    Log.e("Test","Let Apple introduce himself.");
                    // 返回方法的执行结果
                    Object result = method.invoke(apple, args);
                    Log.e("Test","Thank you!");
                    return result;
                }
                return null;
            }
        };

        Fruit newApple = (Fruit) Proxy.newProxyInstance(invocationHandler.getClass().getClassLoader(),
                apple.getClass().getInterfaces(),invocationHandler);
        String name = newApple.getName();
        Log.e("Test",newApple.getClass().getName() + ",name = " + name);

    }
```

结果：

```
E/Test: Let Apple introduce himself.
E/Test: I am apple!
E/Test: Thank you!
E/Test: $Proxy0,name = Apple
```

实现动态代理的方法：

 1. 获取需要代理的类的实例。
 2. 实现 InvocationHandler 接口创建自己的调用处理器，实现 invoke 方法，通过 `method.invoke(apple, args)` 调用被代理类。
 3. 通过 Proxy.newProxyInstance 方法并指定 ClassLoader 对象和一组 interface 以及 指定 invocationHandler 对象来创建动态代理类。
 4. 通过反射机制获得动态代理类的构造函数。
 5. 通过构造函数创建动态代理类实例，构造时调用处理器对象作为参数被传入。

其中 `Proxy.newProxyInstance()` 方法封装了 3~5 步骤的实现。下面一节将会简单介绍它的原理。

## 动态代理原理

下来看一下 `Proxy.newProxyInstance()` 源码。

```java
    public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,
                                          InvocationHandler invocationHandler)
            throws IllegalArgumentException {

        if (invocationHandler == null) {
            throw new NullPointerException("invocationHandler == null");
        }
        Exception cause;
        try {
            // getProxyClass 获得与指定类装载器和一组接口相关的代理类类型对象
            // getConstructor 通过反射获取构造函数对象
            // newInstance 生成代理类实例
            return getProxyClass(loader, interfaces)
                    .getConstructor(InvocationHandler.class)
                    .newInstance(invocationHandler);
        } catch (NoSuchMethodException e) {
            cause = e;
        } catch (IllegalAccessException e) {
            cause = e;
        } catch (InstantiationException e) {
            cause = e;
        } catch (InvocationTargetException e) {
            cause = e;
        }
        AssertionError error = new AssertionError();
        error.initCause(cause);
        throw error;
    }
```

来看一下 `Proxy.getProxyClass()`这个方法：

```java
    public static Class<?> getProxyClass(ClassLoader loader, Class<?>... interfaces)
            throws IllegalArgumentException {
        if (loader == null) {
            loader = ClassLoader.getSystemClassLoader();
        }

        if (interfaces == null) {
            throw new NullPointerException("interfaces == null");
        }

        final List<Class<?>> interfaceList = new ArrayList<Class<?>>(interfaces.length);
        Collections.addAll(interfaceList, interfaces);

        final Set<Class<?>> interfaceSet = new HashSet<Class<?>>(interfaceList);
        if (interfaceSet.contains(null)) {
            throw new NullPointerException("interface list contains null: " + interfaceList);
        }

        if (interfaceSet.size() != interfaces.length) {
            throw new IllegalArgumentException("duplicate interface in list: " + interfaceList);
        }

        synchronized (loader.proxyCache) {
            Class<?> proxy = loader.proxyCache.get(interfaceList);
            if (proxy != null) {
                return proxy;
            }
        }

        String commonPackageName = null;
        for (Class<?> c : interfaces) {
            // 如果不是interface，抛出异常
            if (!c.isInterface()) {
                throw new IllegalArgumentException(c + " is not an interface");
            }
            // 判断是否对类装载器可见
            if (!isVisibleToClassLoader(loader, c)) {
                throw new IllegalArgumentException(c + " is not visible from class loader");
            }
            if (!Modifier.isPublic(c.getModifiers())) {
                String packageName = c.getPackageName$();
                if (packageName == null) {
                    packageName = "";
                }
                if (commonPackageName != null && !commonPackageName.equals(packageName)) {
                    throw new IllegalArgumentException(
                            "non-public interfaces must be in the same package");
                }
                commonPackageName = packageName;
            }
        }

        List<Method> methods = getMethods(interfaces);
        Collections.sort(methods, ORDER_BY_SIGNATURE_AND_SUBTYPE);
        validateReturnTypes(methods);
        List<Class<?>[]> exceptions = deduplicateAndGetExceptions(methods);

        ArtMethod[] methodsArray = new ArtMethod[methods.size()];
        for (int i = 0; i < methodsArray.length; i++) {
            methodsArray[i] = methods.get(i).getArtMethod();
        }
        Class<?>[][] exceptionsArray = exceptions.toArray(new Class<?>[exceptions.size()][]);

        // 确定类名
        String baseName = commonPackageName != null && !commonPackageName.isEmpty()
                ? commonPackageName + ".$Proxy"
                : "$Proxy";

        Class<?> result;
        synchronized (loader.proxyCache) {
            result = loader.proxyCache.get(interfaceSet);
            if (result == null) {
                String name = baseName + nextClassNameIndex++;
                // 生成代理类
                result = generateProxy(name, interfaces, loader, methodsArray, exceptionsArray);
                // 放入缓存
                loader.proxyCache.put(interfaceList, result);
            }
        }

        return result;
    }
```

## 小结

Proxy 虽然使用起来很方便，但它的设计使得它只能支持 interface 的代理，Java 的继承机制注定了动态代理类无法实现对 class 的动态代理，因为多继承在 Java 中本质上就行不通。因为代理类需要继承 Proxy 类，这样它就不能再继承其他的类了。因此不支持实现类的代理，只支持接口的代理。
但是目前也有了针对 class 的动态代理解决方案，比如 CGLib。

## 相关推荐文章

https://www.ibm.com/developerworks/cn/java/j-lo-proxy1/
http://blog.csdn.net/chjttony/article/details/7934776
http://blog.csdn.net/rock_ray/article/details/22491763
http://rejoy.iteye.com/blog/1627405
http://www.cnblogs.com/xiaoluo501395377/p/3383130.html
