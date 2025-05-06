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

#### 线程池的工作流程图
![线程池工作流程图](/images/threadpool.png)

#### 线程池的五种状态
```java
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```
- RUNNING: 可接受新任务并处理队列中的任务
- SHUTDOWN: 不接受新任务，但可以处理队列中的任务
- STOP：不接受新任务，不处理队列中的任务，中断正在进行中的任务
- TIDYING：线程池内线程全部执行完毕，等待回收，进入TIDYING状态
- TERMINATED：终止状态

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
- corePoolSize
>核心池大小。当一个任务提交到线程池时，首先会判断当前核心池的大小，若当前线程数量小于corePoolSize时，
则会创建新的线程执行此任务，反之不会创建新线程，而是去判断阻塞队列
- maximumPoolSize
>线程池最大线程数。当阻塞队列已满，并且当前池内线程数小于maximumPoolSize时，也会创建新的线程执行此任务
- keepAliveTime
>空闲线程存活时间。当线程池内的线程数超过了corePoolSize，并且空闲线程的存活时间超过了keepAliveTime，
会将这些空闲线程销毁
- unit
>keepAliveTime存活时间单位
- workQueue
>阻塞队列。当池内线程数超过corePoolSize时，且workQueue中的数量未满，将任务放入到阻塞队列中
- handler
>拒绝策略。当workQueue阻塞队列已满，且当前池内线程数已达到maximumPoolSize，此时作拒绝策略处理
- threadFactory
> 默认Executors.defaultThreadFactory()

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

#### BlockingQueue阻塞队列
- SynchronousQueue（同步移交队列）
>默认的阻塞队列，该队列容量为0不存储任务，每次会直接创建新的线程执行任务。若线程池内线程数超过了maximumPoolSize时，会导致任务加入队列失败执行拒绝策略。该阻塞队列适用于低延迟任务。Executors.newCachedThreadPool()方式创建线程池使用到SynchronousQueue阻塞队列
- ArrayBlockingQueue（有界阻塞队列）
>基于数组的阻塞队列，需要设置队列的初始容量，并且基于先进先出（FIFO）处理队列中的任务。当线程池内的线程数达到corePoolSize时，新到达的任务会放入该队列中。当该队列容量已满时，且池内线程数不超过maximumPoolSize，此时会创建新线程来执行任务，否则会对任务作拒绝处理。该阻塞队列适用于CPU密集型的任务
- LinkedBlockingQueue（无界阻塞队列）
>基于链表的阻塞队列，该阻塞队列容量默认为Integer.MAX。当线程池内线程数达到corePoolSize时，新任务到达后，放入该队列进行等待执行。由于该队列是无界的（不存在队列已满的情况），因此maximumPoolSize配置无效（池内线程数永远不会超过corePoolSize），同时也不会存在空闲线程销毁的场景。适用于IO密集型的任务，Executors.newFixedThreadPool()，Executors.newSingleThreadExecutor()方式创建的线程池使用到LinkedBlockingQueue阻塞队列
- PriorityBlockingQueue（优先级阻塞队列）
>该队列为链表结构，并且也是一个无界队列，该队列可以按照优先级处理任务。无界队列可能由于队列中的任务过多导致OOM
- DelayQueue（延迟队列）
>ScheduledThreadPoolExecutor专用，适用于定时/周期性的任务

#### RejectedExecutionHandler任务拒绝策略
针对有界队列，当阻塞队列和线程池内最大线程数已满的情况下会使用任务拒绝策略
- AbortPolicy：默认拒绝策略，拒绝提交的任务，并抛出RejectExecutionException异常
- DiscardPolicy：不处理丢弃任务，适用于日志或监控系统
- DiscardOldestPolicy：丢弃阻塞队列中最早的任务
- CallerRunsPolicy：由调用者执行，会影响任务的提交效率