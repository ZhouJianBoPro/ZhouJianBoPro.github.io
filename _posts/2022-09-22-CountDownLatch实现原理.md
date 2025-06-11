---
layout: post
title: CountDownLatch实现原理
date: 2022-09-22
tags: [concurrency]
---

#### CountDownLatch
> CountDownLatch是一个线程等待工具类，可以实现 一个或多个线程 等待 其他N个线程 执行完成，其核心实现基于 AQS共享锁（多个线程并发访问资源）。

#### 计数机制
1. 初始化时指定一个计数值（如任务数量），使用AQS state状态字段记录计数值
2. 每个任务执行完成后调用一次countDown()方法将计数值减一
3. 等待线程调用await()方法进入阻塞状态，直到计数值变为0时，唤醒所有阻塞线程

#### 应用场景
1. 判断线程池是否执行完成
    ```java
    public class CountDownLatchTest {
    
        private static final ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 60, TimeUnit.SECONDS, new ArrayBlockingQueue<>(2000));
    
        public static void main(String[] args) throws InterruptedException {
    
            int taskNum = 20;
            CountDownLatch countDownLatch = new CountDownLatch(taskNum);
    
            for(int i = 0; i < taskNum; i++) {
                executor.execute(() -> {
                    try {
                        System.out.println("线程" + Thread.currentThread().getName() + "开始执行");
                    } finally {
                        countDownLatch.countDown();
                    }
                });
            }
    
            // 等待线程池执行完所有任务
            countDownLatch.await();
            System.out.println("线程池任务执行完毕");
    
            executor.shutdown();
        }
    }
    ```
2. 多线程计算后汇总（主线程await阻塞，等待子线程返回结果后汇总）