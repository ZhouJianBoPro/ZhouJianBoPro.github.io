---
layout: post
title: ThreadPoolExecutor原理
date: 2022-09-22
tags: [thread]
---

#### 使用线程池的好处
使用线程池可以减少线程的创建及销毁的次数，使得线程池内的线程可以复用
1. 无需每次创建新的线程，提升了系统响应速度
2. 减少了线程创建及销毁时的系统资源占用
3. 线程池内维护了最大的线程数，不能无限制的创建线程，保证系统稳定性

#### 线程池的工作流程图
![线程池工作流程图](/images/threadpool.png)

#### 线程池的创建
线程池的创建是通过ThreadPoolExecutor来完成的，ThreadPoolExecutor继承AbstractExecutorService，而AbstractExecutor实现ExecutorService接口
```java
public abstract class AbstractExecutorService implements ExecutorService {

    public ThreadPoolExecutor(int corePoolSize,
                                  int maximumPoolSize,
                                  long keepAliveTime,
                                  TimeUnit unit,
                                  BlockingQueue<Runnable> workQueue,
                                  ThreadFactory threadFactory,
                                  RejectedExecutionHandler handler) {
            if (corePoolSize < 0 ||
                maximumPoolSize <= 0 ||
                maximumPoolSize < corePoolSize ||
                keepAliveTime < 0)
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
- corePoolSize：核心池大小。当一个任务提交到线程池时，首先会判断当前核心池的大小，若当前核心池数量小于corePoolSize时，
则会创建新的线程执行此任务，反之不会创建新线程，而是去判断阻塞队列
- maximumPoolSize：线程池最大线程数。当阻塞队列已满，并且当前池内线程数小于maximumPoolSize时，也会创建新的线程执行此任务
- keepAliveTime：空闲线程存活时间。当线程池内的线程数超过了corePoolSize，并且空闲线程的存活时间超过了keepAliveTime，
会将这些空闲线程销毁
- unit：keepAliveTime存活时间单位
- workQueue：阻塞队列。当池内线程数超过corePoolSize时，且workQueue中的数量未满，将任务放入到阻塞队列中
- handler：拒绝策略。当workQueue阻塞队列已满，且当前池内线程数已达到maximumPoolSize，此时作拒绝策略处理

#### ThreadPoolExecutor#execute
```java
public abstract class AbstractExecutorService implements ExecutorService {

    public void execute(Runnable command) {

            if (command == null)
                throw new NullPointerException();

            int c = ctl.get();

            //池内当前线程数小于核心池数量时，创建新线程执行任务
            if (workerCountOf(c) < corePoolSize) {
                //addWorker第二个参数用于区分限制添加线程数是根据corePoolSize还是maximumPoolSize。 true表示通过corePoolSize判断，false表示通过maximumPoolSize判断
                if (addWorker(command, true))
                    return;
                c = ctl.get();
            }
            
            //当前线程数超过核心池数量时，将任务放入阻塞队列中
            if (isRunning(c) && workQueue.offer(command)) {
                int recheck = ctl.get();
                //若线程池不是运行状态，则从阻塞队列中移除该任务，并且对该任务作拒绝策略处理
                if (! isRunning(recheck) && remove(command))
                    reject(command);
                else if (workerCountOf(recheck) == 0)
                    addWorker(null, false);
            //阻塞队列已满添加失败，但池内线程数没超过最大线程数，开启新线程执行任务
            } else if (!addWorker(command, false))
                //池内线程数超过最大线程数，任务拒绝处理
                reject(command);
        }
}
```
