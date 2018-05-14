---
layout: post
title: ThreadLocal深入了解
date: 2018-05-09
categories: high concurrency
tags: [concurrency]
description: java并发编程中ThreadLocal深入了解
---

**对ThreadLocal的理解**
> 称之为线程本地变量，ThreadLocal为变量在每个线程中都创建一个变量副本，
每个线程都可以访问自己内部的副本变量

举例：
```java
public class ConnectionManager {
 
     //共享变量，可被多个对象共享
     private static Connection connect = null;
     
     public static Connection openConnection() {
     if(connect == null){
     connect = DriverManager.getConnection();
     }
     return connect;
     }
     
     public static void closeConnection() {
     if(connect!=null)
     connect.close();
     }
}
```
以上代码类似一个数据库连接管理类，在单线程中是没有问题的，多线程中可能出现以下情况：<br/>
1)由于该变量是共享变量，需要在调用connection时设置同步来保证线程安全，因为很可能一个线程在创建完
connect后没有被调用之前就被其他线程close连接，设置同步会影响程序执行效率，因为一个线程在创建connect时，
其他线程只能等待

解决方法：
- 将Connection connect设置为非共享变量，在使用数据库连接时才创建数据库连接对象，但是这样会频繁的
创建和关闭数据库连接，会导致服务器压力很大
- 使用ThreadLocal在每个线程中创建connect变量副本，既保证了线程安全，也不会频繁的创建和销毁数据库连接对服务器造成很大的压力

**深入了解ThreadLocal类**<br/>
1.ThreadLocal提供的几个常用方法介绍：
```html
//获取当前线程中副本变量
public T get() { }

//设置当前线程中的副本变量
public void set(T value) { }

//移除当前线程中的副本变量
public void remove() { }
```
2.线程创建副本过程：<br/>
1）Thread线程类中有ThreadLocal.ThreadLocalMap类型的成员变量threadLocals,键值为ThreadLocal,value值为
变量副本（T变量）<br/>
2）初始时，Thread中的threadLocals为空,当通过ThreadLocal调用get(),set()方法，就会对threadLocals变量
进行初始化，并且以LocalThread为键值，副本变量为value

```java
public class ThreadLocalDemo {

    private static ThreadLocal<String> threadLocalName = new ThreadLocal <>();
    private static ThreadLocal<Long> threadLocalId = new ThreadLocal<>();

    /**
     * 为每个线程设置两个线程副本变量 
     */
    private static void set() {
        threadLocalName.set(Thread.currentThread().getName());
        threadLocalId.set(Thread.currentThread().getId());
    }

    /**
     * 移除线程中的线程副本 
     */
    private static void remove() {
        threadLocalName.remove();
        threadLocalId.remove();
    }

    public static void main(String[] args) throws InterruptedException {
        set();
        System.out.println("thread name:" + threadLocalName.get() + ", thread id:" + threadLocalId.get());
        remove();
        Thread thread1 = new Thread(){
            @Override
            public void run() {
                set();
                System.out.println("thread name:" + threadLocalName.get() + ", thread id:" + threadLocalId.get());
                remove();
            };
        };
        thread1.start();
        
        //防止主线程优于子线程执行完
        thread1.join();
        System.out.println("thread name:" + threadLocalName.get() + ", thread id:" + threadLocalId.get());
    }
   
}
```
```html
    thread name:main, thread id:1
    thread name:Thread-0, thread id:10
    thread name:null, thread id:null
```
3.总结：<br/>
- 实际通过ThreadLocal创建的线程副本变量是存储在每个线程中的threadLocals变量中
- 每个线程中可以存在多个线程副本变量
- 通过ThreadLocal创建的线程副本变量不会共享，只属于单个线程
- 线程副本用来之后及时remove，不然会造成内存泄漏

4.ThreadLocal应用场景：高并发过程中用来解决数据库连接，session管理等需要频繁创建和销毁连接场景





