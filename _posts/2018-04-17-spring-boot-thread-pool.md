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
    
**深入剖析线程池实现原理**
1.线程池的状态：
```html
running状态：当创建线程池之后，线程池处于running状态；
shutdown状态：如果调用了shutdown()，线程池处于shutdown状态，线程池不能接收新的任务，它会等待其他任务执行完；
stop状态：如果调用了shutdownNow()，线程池处于stop状态，线程池不能接收新任务，而且正在执行的任务也会立即被终止
terminated状态：当线程池处于shutdown或stop状态，并且所有的工作线程都已销毁，任务缓存队列已清空或执行完，线程池被设置为termitnated
```
2.任务的执行：
- 首先判断任务是否为null，抛出NullpointException
- 当poolSize <= corePoolSize时，每来一个任务就创建一个线程去执行
- poolSize > corePoolSize时，将后续到来的任务放入缓存队列中，执行完任务的线程主动去缓存队列中取出任务执行，
而不是重新开一个任务负责将队列中的任务分配给线程，降低了复杂度；添加失败一般是缓存队列已满。
- 当poolSize = maximumPoolSize时，且缓存队列已满，会采取任务拒绝策略进行处理
- 当corePoolSize < poolSize < maximumPoolSize时，空闲线程会在keepAliveTime后终止
 
3.线程池中线程的初始化：
- 默认情况下，线程池创建之后不会立即创建线程，只有当有任务到达后才会创建线程
- prestartCoreThread():初始化一个核心线程
- prestartAllCoreThreads():初始化所有核心线程

4.任务缓存队列及排队策略<br/>
任务缓存队列，即workQueue,用来存放等待执行的任务，workQueue的类型为BlockingQueue<Runnable>,通常可以去以下三种类型：
- ArrayBlockingQueue:基于数组的先进先出队列，此队列创建需要指定大小
- LinkedBlockingQueue:基于链表的先进先出队列，默认大小为Integer.maxValue()
- synchronousQueue:不会保存提交的任务，直接创建一个线程来执行新任务

5.任务拒绝策略<br/>
当poolSize = maximumPoolSize且缓存队列已满，如果还有任务到达时会采取任务拒绝策略，通常有四种任务策略：
```html
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
```
6.线程池的关闭<br/>
- shutdown():线程池处于shutdown状态，不会立即关闭线程池，会等待所有缓存队列中的任务执行完才关闭
- shutdownNow():线程池处于stop状态，立即关闭线程池，尝试打断正在执行的任务，并且清空缓存队列，返回尚未执行的任务




