---
layout:     post
title:      "Android线程池"
subtitle:   "如何合理的创建线程池"
date:       2019-03-23
author:     "xiao"
header-img: "img/android_header.jpg"
tags:
    - Android
    - Framework
---

***

线程的调度模型分为两种

 * 分时调度模型：异步任务轮流获取CPU时间片执行
 * 占式调度模型：根据线程的优先级来分配时间片，JVM采用此种调度模型

使用线程池有以下好处

 * 可以重用线程池中的已经存在的线程，避免重复创建和销毁线程的性能开销；
 * 能够方便的控制最大并发数，避免大量的线程抢占系统资源带来的调度消耗；
 * 能够对线程进行简单的管理，提供定时执行以及指定间隔循环执行任务；

 Android中的线程池的概念来源于Java中的Executor，Executor是一个接口，真正的实现是ThreadPoolExecutor，其提供了一些配置参数用来创建不同的线程池，从功能上来说Android的线程池分为四类，都是通过Executors所提供的工厂方法来得到的，Android中的线程也都是通过对ThreadPoolExecutor进行不同的配置来实现的；

 ``` java

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


   ThreadPoolExecutor executor = new ThreadPoolExecutor();

   //默认情况下核心线程池是不允许销毁的，及时在空闲状态下也会保持存活状态，除非调用以下方法，核心线程会有超时策略，当超出keepAliveTime时长空闲时，核心线程会被终止
   executor.allowCoreThreadTimeOut(true);

 ```

 corePoolSize 线程池的核心线程数，默认情况下核心线程会一直存活，仅收到 allowCoreThreadTimeOut 的影响，当值为true时，超出keepAliveTime时长空闲时，核心线程会被终止

 maximumPoolSize  线程池中所能容纳的最大线程数，当超出最大值时，新添加的任务将会被拒绝执行

 keepAliveTime 非核心线程的限制等待时长，allowCoreThreadTimeOut为true时也会影响到核心线程的闲置等待时间

 unit 指定 keepAliveTime 参数的时间单位

 workQueue BlockingQueue<Runnable> 线程池的任务队列，通过execute方法提交的任务对象会存储到这个队列中

 threadFactory 线程工厂，用于为线程池提供新线程
```java
new ThreadFactory() {
    @Override
    public Thread newThread(Runnable r) {

    Thread thread = new Thread(r);
    //为新创建的线程设置线程名  设置线程优先级
    thread.setName("ThreadPoolUtils");
    thread.setPriority(Process.THREAD_PRIORITY_DEFAULT);
    return thread;
    }
})

```

RejectedExecutionHandler 当线程无法执行新任务时，可能是任务队列已满或无法成功执行任务，会回调此方法通知调用者

 ### ThreadPoolExecutor 执行任务遵循以下原则

 - 如果线程池中的线程数量未达到核心线程的数量，则直接开启一个核心线程执行任务
 - 如果线程池中的线程数量已达到核心线程的数量，则将新添加的任务添加到任务队列当中等到执行
 - 如果任务队列已满且核心线程数量已达到最大数量，则开启一个非核心线程来执行这个任务
 - 如果非核心线程数量已达最大值，且任务队列和核心线程数量都满了，则调用 RejectedExecutionHandler 的 rejectedExecution 方法通知调用者

```java
new RejectedExecutionHandler(){
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {

    }
};
```


