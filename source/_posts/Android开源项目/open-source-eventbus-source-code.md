---
title: EventBus -- 源码分析
categories: Android开源项目
comments: true
tags: [EventBus]
description: 介绍EventBus的源码
date: 2017-12-12 10:00:00
---

## 概述

介绍完了 EventBus 的基本用法之后，照例要进行一下源码分析。
本文基于 EventBus V3.0.0 进行分析，并尽量保持同步更新。

[github 源码地址](https://github.com/greenrobot/EventBus)

未完待续......

## 消息订阅

```
    public void register(Object subscriber) {
        // 获取订阅对象的 class 对象
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

### SubscriberMethodFinder

SubscriberMethodFinder 作用是解析并找出订阅对象中所有的订阅方法。

```
    SubscriberMethodFinder(List<SubscriberInfoIndex> subscriberInfoIndexes, boolean strictMethodVerification,
                           boolean ignoreGeneratedIndex) {
        this.subscriberInfoIndexes = subscriberInfoIndexes;
        this.strictMethodVerification = strictMethodVerification;
        this.ignoreGeneratedIndex = ignoreGeneratedIndex;
    }
```

```
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        // 先找出是否有缓存
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        // 如果有缓存直接返回缓存的订阅方法
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        
        if (ignoreGeneratedIndex) {
            // 如果设置了忽略生成的索引，关于索引，后面专门介绍
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            // 没有设置会走到这里，默认是没有设置的
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            // 如果没有找到订阅方法，抛出异常
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            // 找到订阅方法，加入缓存
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
        }
    }    
```

获取订阅对象中的订阅方法列表：

```
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

解析订阅对象的注解信息。
因为订阅方法以及它的一些属性都是通过注解来标识的，只有解析到了这些方法才能向它们发布消息事件。

```
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // 获取订阅者对象的所有方法
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            // 方法的修饰符必须符合下面的类型
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                // 获取方法的参数
                Class<?>[] parameterTypes = method.getParameterTypes();
                // 订阅方法必须接受一个参数，也就是表示消息事件的类
                if (parameterTypes.length == 1) {
                    // 获取该方法的 Subscribe 注解信息
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    // 如果有这个注解
                    if (subscribeAnnotation != null) {
                        // 获取参数的 class 对象
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            // 获取 ThreadMode 类型
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            // 保存到subscriberMethods
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

## 消息发布

```
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```

未完待续......