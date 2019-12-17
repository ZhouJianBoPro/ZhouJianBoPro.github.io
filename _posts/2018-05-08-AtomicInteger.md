---
layout: post
title: AcomicInteger原子操作类
date: 2018-05-08
tags: [java]
description: 并发操作Integer操作类
---

**AtomicInteger类的理解与使用**<br/>
首先看两段代码Integer、AtomicInteger类的理解与使用
```java
public class Sample1 {

    private static Integer count = 0;

    synchronized public static void increment() {
        count++;
    }
}
```
```java
public class Sample2 {

    private static AtomicInteger count = new AtomicInteger(0);

    public static void increment() {
        count.getAndIncrement();
    }

}
```
以上两段代码，在使用Integer的时候，并发访问时需要synchronized来保证不会有多个线程同时访问的情况，
AtomicInteger在并发访问时保证了变量的原子性。

**AtomicInteger使用场景**<br/>
AtomicInteger使用非阻塞算法实现并发控制，适用于高并发情况下对变量进行增减

**AtomicInteger的使用**<br/>
AtomicInteger提供的接口：
```html
//获取当前的值
public final int get()

//取当前的值，并设置新的值
 public final int getAndSet(int newValue)

//获取当前的值，并自增
 public final int getAndIncrement()

//获取当前的值，并自减
public final int getAndDecrement()
```



