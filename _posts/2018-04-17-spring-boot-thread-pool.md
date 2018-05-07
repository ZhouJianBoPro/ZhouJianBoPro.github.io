---
layout: post
title: 线程池实现多线程
date: 2018-04-17
categories: thread
tags: [thread]
description: java线程池实现多线程
---

[简单实现多线程](http://boopro.cn/java/grammar/2018/03/16/thread/)

**传统实现与线程池之间的区别**
1. 传统实现需要线程的时候就去创建线程，如果并发线程数量较多时，频繁的创建和销毁线程需要时间，会大大降低系统的性能。
2. 线程池中执行完的线程会被重新释放并回到线程池，并不会频繁的创建和销毁。

**线程池中的核心类ThreadPoolExecutor**<br/>
1.部分源码解析
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    
    //拒绝策略
    private static final RejectedExecutionHandler defaultHandler = new AbortPolicy();
    
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, Executors.defaultThreadFactory(), defaultHandler);
    }
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, defaultHandler);
    }
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, Executors.defaultThreadFactory(), handler);
    }
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler) {
        if (corePoolSize < 0 || maximumPoolSize <= 0 || maximumPoolSize < corePoolSize || keepAliveTime < 0)
                throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
}
```
其实前三个构造器都是调用第四个构造器进行初始化工作的

2.各个参数的含义：
- corePoolSize:核心池的大小，当线程池中的线程数量达到corePoolSize时，会将到达的任务放入缓存队列中（默认情况下，在创建了线程池之后，线程池中的线程数量为0，当有任务达到时，会逐步创建线程去执行任务）
- maximumPoolSize:线程池中的最大线程数量，线程池中最多能创建多少个线程
- keepAliveTime:线程池中的线程数量大于corePoolSize，多余部分的空闲线程在这个时间过后会被销毁
- unit:参数keepAliveTime单位，有七种取值，在TimeUnit中有7中静态属性
    ```html
    TimeUnit.DAYS;               //天
    TimeUnit.HOURS;             //小时
    TimeUnit.MINUTES;           //分钟
    TimeUnit.SECONDS;           //秒
    TimeUnit.MILLISECONDS;      //毫秒
    TimeUnit.MICROSECONDS;      //微妙
    TimeUnit.NANOSECONDS;       //纳秒
    ```
- workQueue:阻塞队列，用于存储等待执行的任务，常用的阻塞队列有以下三种
    ```html
    ArrayBlockingQueue;
    LinkedBlockingQueue;
    SynchronousQueue;
    ```
    ArrayBlockingQueue使用较少
- threadFactory:线程工厂，用于创建线程
- handler:拒绝处理任务时的策略，有以下四种取值
    ```html
    ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
    ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
    ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
    ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
    ```
    
    
