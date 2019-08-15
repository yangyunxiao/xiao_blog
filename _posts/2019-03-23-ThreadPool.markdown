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

 ```
 java


 ```