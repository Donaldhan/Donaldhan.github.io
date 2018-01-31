---
layout: page
title: NIO包总结
subtitle: NIO包总结
date: 2018-11-04 15:17:19
author: donaldhan
catalog: true
category: Java
categories:
    - Java
tags:
    - JUC
---

# 引言

在 Java 5.0 提供了 java.util.concurrent(简称JUC)包,在此包中增加了在并发编程中很常用的工具类,
用于定义类似于线程的自定义子系统,包括线程池,异步 IO 和轻量级任务框架;还提供了设计用于多线程上下文中
的 Collection ,Queue实现等;相关文章列表如下：

* [Callable与Future,FutureTask][]
* [CountDownLatch使用场景][]
* [AtomicInteger解析][]
* [Lock和synchronized的性能的比较][]
* [Condition实现消费生产者模型][]
* [JAVA assert测试][]
* [锁持有者管理器AbstractOwnableSynchronizer][]
* [AQS线程挂起辅助类LockSupport][]
* [AQS详解-CLH队列，线程等待状态][]
* [AQS-Condition详解][]
* [可重入锁ReentrantLock详解][]
* [CountDownLatch详解][]
* [CyclicBarrier使用实例][]
* [CyclicBarrier详解][]
* [用Semaphore实现对象池][]
* [Semaphore详解][]
* [ReadWriteLock实现ConcurrentMap][]
* [ReentrantReadWriteLock详解一][]
* [ReentrantReadWriteLock详解后续][]
* [HashMap父类Map][]
* [Map的简单实现AbstractMap][]
* [HashMap详解][]
* [ConcurrentMap介绍][]
* [ConcurrentHashMap解析-Segment][]
* [ConcurrentHashMap解析后续][]
* [Queue接口定义][]
* [AbstractQueue简介][]
* [ConcurrentLinkedQueue解析][]
* [BlockingQueue接口的定义][]
* [LinkedBlockingQueue解析][]
* [ArrayBlockingQueue解析][]
* [PriorityBlockingQueue解析][]
* [SynchronousQueue解析上-TransferStack][]
* [SynchronousQueue解析下-TransferQueue][]
* [DelayQueue解析][]
* [JAVA集合类简单综述][]
* [简单测试线程池拒绝执行任务策略][]
* [Executor接口的定义][]
* [ExecutorService接口定义][]
* [Future接口定义][]
* [FutureTask解析][]
* [CompletionService接口定义][]
* [ExecutorCompletionService解析][]
* [AbstractExecutorService解析][]
* [ScheduledExecutorService接口定义][]
* [ThreadPoolExecutor解析一（核心线程池数量、线程池状态等）][]
* [ThreadPoolExecutor解析二（线程工厂、工作线程，拒绝策略等）][]
* [ThreadPoolExecutor解析三（线程池执行提交任务）][]
* [ThreadPoolExecutor解析四（线程池关闭）][]
* [ScheduledThreadPoolExecutor解析一（调度任务，任务队列）][]
* [ScheduledThreadPoolExecutor解析二（任务调度）][]
* [ScheduledThreadPoolExecutor解析三（关闭线程池）][]
* [Executors解析][]


![JUC](/image/JUC/juc.png)

# 目录

* [Callable与Future,FutureTask](#Callable与Future,FutureTask)
* [CountDownLatch使用场景](#CountDownLatch使用场景)
* [AtomicInteger解析](#AtomicInteger解析)
* [Lock和synchronized的性能的比较](#Lock和synchronized的性能的比较)
* [Condition实现消费生产者模型](#Condition实现消费生产者模型)
* [JAVA assert测试](#JAVA assert测试)
* [锁持有者管理器AbstractOwnableSynchronizer](#锁持有者管理器AbstractOwnableSynchronizer)
* [AQS线程挂起辅助类LockSupport](#AQS线程挂起辅助类LockSupport)
* [AQS详解-CLH队列，线程等待状态](#AQS详解-CLH队列，线程等待状态)
* [AQS-Condition详解](#AQS-Condition详解)
* [可重入锁ReentrantLock详解](#可重入锁ReentrantLock详解)
* [CountDownLatch详解](#CountDownLatch详解)
* [CyclicBarrier使用实例](#CyclicBarrier使用实例)
* [CyclicBarrier详解](#CyclicBarrier详解)
* [用Semaphore实现对象池](#用Semaphore实现对象池)
* [Semaphore详解](#Semaphore详解)
* [ReadWriteLock实现ConcurrentMap](#ReadWriteLock实现ConcurrentMap)
* [ReentrantReadWriteLock详解一](#ReentrantReadWriteLock详解一)
* [ReentrantReadWriteLock详解后续](#ReentrantReadWriteLock详解后续)
* [HashMap父类Map](#HashMap父类Map)
* [Map的简单实现AbstractMap](#Map的简单实现AbstractMap)
* [HashMap详解](#HashMap详解)
* [ConcurrentMap介绍](#ConcurrentMap介绍)
* [ConcurrentHashMap解析-Segment](#ConcurrentHashMap解析-Segment)
* [ConcurrentHashMap解析后续](#ConcurrentHashMap解析后续)
* [Queue接口定义](#Queue接口定义)
* [AbstractQueue简介](#AbstractQueue简介)
* [ConcurrentLinkedQueue解析](#ConcurrentLinkedQueue解析)
* [BlockingQueue接口的定义](#BlockingQueue接口的定义)
* [LinkedBlockingQueue解析](#)
* [ArrayBlockingQueue解析](#ArrayBlockingQueue解析)
* [PriorityBlockingQueue解析](#PriorityBlockingQueue解析)
* [SynchronousQueue解析上-TransferStack](#SynchronousQueue解析上-TransferStack)
* [SynchronousQueue解析下-TransferQueue](#SynchronousQueue解析下-TransferQueue)
* [DelayQueue解析](#DelayQueue解析)
* [JAVA集合类简单综述](#JAVA集合类简单综述)
* [简单测试线程池拒绝执行任务策略](#简单测试线程池拒绝执行任务策略)
* [Executor接口的定义](#Executor接口的定义)
* [ExecutorService接口定义](#ExecutorService接口定义)
* [Future接口定义](#Future接口定义)
* [FutureTask解析](#FutureTask解析)
* [CompletionService接口定义](#CompletionService接口定义)
* [ExecutorCompletionService解析](#ExecutorCompletionService解析)
* [AbstractExecutorService解析](#AbstractExecutorService解析)
* [ScheduledExecutorService接口定义](#ScheduledExecutorService接口定义)
* [ThreadPoolExecutor解析一（核心线程池数量、线程池状态等）](#ThreadPoolExecutor解析一（核心线程池数量、线程池状态等）)
* [ThreadPoolExecutor解析二（线程工厂、工作线程，拒绝策略等）](#ThreadPoolExecutor解析二（线程工厂、工作线程，拒绝策略等）)
* [ThreadPoolExecutor解析三（线程池执行提交任务）](#ThreadPoolExecutor解析三（线程池执行提交任务）)
* [ThreadPoolExecutor解析四（线程池关闭）](#ThreadPoolExecutor解析四（线程池关闭）)
* [ScheduledThreadPoolExecutor解析一（调度任务，任务队列）](#ScheduledThreadPoolExecutor解析一（调度任务，任务队列）)
* [ScheduledThreadPoolExecutor解析二（任务调度）](#ScheduledThreadPoolExecutor解析二（任务调度）)
* [ScheduledThreadPoolExecutor解析三（关闭线程池）](#ScheduledThreadPoolExecutor解析三（关闭线程池）)
* [Executors解析](#Executors解析)


## Callable与Future,FutureTask
## CountDownLatch使用场景
## AtomicInteger解析
## Lock和synchronized的性能的比较
## Condition实现消费生产者模型
## JAVA assert测试
## 锁持有者管理器AbstractOwnableSynchronizer
## AQS线程挂起辅助类LockSupport
## AQS详解-CLH队列，线程等待状态
## AQS-Condition详解
## 可重入锁ReentrantLock详解
## CountDownLatch详解
## CyclicBarrier使用实例
## CyclicBarrier详解
## 用Semaphore实现对象池
## Semaphore详解
## ReadWriteLock实现ConcurrentMap
## ReentrantReadWriteLock详解一
## ReentrantReadWriteLock详解后续
## HashMap父类Map
## Map的简单实现AbstractMap
## HashMap详解
## ConcurrentMap介绍
## ConcurrentHashMap解析-Segment
## ConcurrentHashMap解析后续
## Queue接口定义
## AbstractQueue简介
## ConcurrentLinkedQueue解析
## BlockingQueue接口的定义
## LinkedBlockingQueue解析
## ArrayBlockingQueue解析
## PriorityBlockingQueue解析
## SynchronousQueue解析上-TransferStack
## SynchronousQueue解析下-TransferQueue
## DelayQueue解析
## JAVA集合类简单综述
## 简单测试线程池拒绝执行任务策略
## Executor接口的定义
## ExecutorService接口定义
## Future接口定义
## FutureTask解析
## CompletionService接口定义
## ExecutorCompletionService解析
## AbstractExecutorService解析
## ScheduledExecutorService接口定义
## ThreadPoolExecutor解析一（核心线程池数量、线程池状态等）
## ThreadPoolExecutor解析二（线程工厂、工作线程，拒绝策略等）
## ThreadPoolExecutor解析三（线程池执行提交任务）
## ThreadPoolExecutor解析四（线程池关闭）
## ScheduledThreadPoolExecutor解析一（调度任务，任务队列）
## ScheduledThreadPoolExecutor解析二（任务调度）
## ScheduledThreadPoolExecutor解析三（关闭线程池）
## Executors解析
##
##
##
##



















[Callable与Future,FutureTask]:http://donald-draper.iteye.com/blog/2311738 "Callable与Future,FutureTask"  
[CountDownLatch使用场景]:http://donald-draper.iteye.com/blog/2348106 "CountDownLatch使用场景"  
[AtomicInteger解析]:http://donald-draper.iteye.com/blog/2359555 "AtomicInteger解析"  
[Lock和synchronized的性能的比较]:http://donald-draper.iteye.com/blog/2359564 "Lock和synchronized的性能的比较"  
[Condition实现消费生产者模型]: "Condition实现消费生产者模型"  
[JAVA assert测试]:http://donald-draper.iteye.com/blog/2360001 "JAVA assert测试"  
[锁持有者管理器AbstractOwnableSynchronizer]:http://donald-draper.iteye.com/blog/2360109 "锁持有者管理器AbstractOwnableSynchronizer"  
[AQS线程挂起辅助类LockSupport]:http://donald-draper.iteye.com/blog/2360206 "AQS线程挂起辅助类LockSupport"  
[AQS详解-CLH队列，线程等待状态]:http://donald-draper.iteye.com/blog/2360256 "AQS详解-CLH队列，线程等待状态"  
[AQS-Condition详解]:http://donald-draper.iteye.com/blog/2360381 "AQS-Condition详解"  
[可重入锁ReentrantLock详解]:http://donald-draper.iteye.com/blog/2360411 "可重入锁ReentrantLock详解"  
[CountDownLatch详解]:http://donald-draper.iteye.com/blog/2360597 "CountDownLatch详解"  
[CyclicBarrier使用实例]:http://donald-draper.iteye.com/blog/2360609 "CyclicBarrier使用实例"  
[CyclicBarrier详解]:http://donald-draper.iteye.com/blog/2360812 "CyclicBarrier详解"  
[用Semaphore实现对象池]:http://donald-draper.iteye.com/blog/2360817 "用Semaphore实现对象池"  
[Semaphore详解]:http://donald-draper.iteye.com/blog/2361033 "Semaphore详解"  
[ReadWriteLock实现ConcurrentMap]: "ReadWriteLock实现ConcurrentMap"  
[ReentrantReadWriteLock详解一]:http://donald-draper.iteye.com/blog/2361521 "ReentrantReadWriteLock详解一"  
[ReentrantReadWriteLock详解后续]:http://donald-draper.iteye.com/blog/2361528 "ReentrantReadWriteLock详解后续"  
[HashMap父类Map]:http://donald-draper.iteye.com/blog/2361603 "HashMap父类Map"  
[Map的简单实现AbstractMap]:http://donald-draper.iteye.com/blog/2361627 "Map的简单实现AbstractMap"  
[HashMap详解]:http://donald-draper.iteye.com/blog/2361702 "HashMap详解"  
[ConcurrentMap介绍]:http://donald-draper.iteye.com/blog/2361719 "ConcurrentMap介绍"  
[ConcurrentHashMap解析-Segment]:http://donald-draper.iteye.com/blog/2363200 "ConcurrentHashMap解析-Segment"  
[ConcurrentHashMap解析后续]:http://donald-draper.iteye.com/blog/2363201 "ConcurrentHashMap解析后续"  
[Queue接口定义]:http://donald-draper.iteye.com/blog/2363491 "Queue接口定义"  
[AbstractQueue简介]:http://donald-draper.iteye.com/blog/2363608 "AbstractQueue简介"  
[ConcurrentLinkedQueue解析]:http://donald-draper.iteye.com/blog/2363874 "ConcurrentLinkedQueue解析"  
[BlockingQueue接口的定义]:http://donald-draper.iteye.com/blog/2363942 "BlockingQueue接口的定义"  
[LinkedBlockingQueue解析]:http://donald-draper.iteye.com/blog/2364007 "LinkedBlockingQueue解析"  
[ArrayBlockingQueue解析]:http://donald-draper.iteye.com/blog/2364034 "ArrayBlockingQueue解析"  
[PriorityBlockingQueue解析]:http://donald-draper.iteye.com/blog/2364100 "PriorityBlockingQueue解析"  
[SynchronousQueue解析上-TransferStack]:http://donald-draper.iteye.com/blog/2364622 "SynchronousQueue解析上-TransferStack"  
[SynchronousQueue解析下-TransferQueue]:http://donald-draper.iteye.com/blog/2364842 "SynchronousQueue解析下-TransferQueue"  
[DelayQueue解析]:http://donald-draper.iteye.com/blog/2364978 "DelayQueue解析"  
[JAVA集合类简单综述]:http://donald-draper.iteye.com/blog/2365238 "JAVA集合类简单综述"  
[简单测试线程池拒绝执行任务策略]:http://donald-draper.iteye.com/blog/2365617 "简单测试线程池拒绝执行任务策略"  
[Executor接口的定义]:http://donald-draper.iteye.com/blog/2365625 "Executor接口的定义"  
[ExecutorService接口定义]:http://donald-draper.iteye.com/blog/2365738 "ExecutorService接口定义"  
[Future接口定义]:http://donald-draper.iteye.com/blog/2365798 "Future接口定义"  
[FutureTask解析]:http://donald-draper.iteye.com/blog/2365980 "FutureTask解析"  
[CompletionService接口定义]:http://donald-draper.iteye.com/blog/2366239 "CompletionService接口定义"  
[ExecutorCompletionService解析]:http://donald-draper.iteye.com/blog/2366254 "ExecutorCompletionService解析"  
[AbstractExecutorService解析]:http://donald-draper.iteye.com/blog/2366348 "AbstractExecutorService解析"  
[ScheduledExecutorService接口定义]:http://donald-draper.iteye.com/blog/2366436 "ScheduledExecutorService接口定义"  
[ThreadPoolExecutor解析一（核心线程池数量、线程池状态等）]:http://donald-draper.iteye.com/blog/2366934 "ThreadPoolExecutor解析一（核心线程池数量、线程池状态等）"  
[ThreadPoolExecutor解析二（线程工厂、工作线程，拒绝策略等）]:http://donald-draper.iteye.com/blog/2367064 "ThreadPoolExecutor解析二（线程工厂、工作线程，拒绝策略等）"  
[ThreadPoolExecutor解析三（线程池执行提交任务）]:http://donald-draper.iteye.com/blog/2367199 "ThreadPoolExecutor解析三（线程池执行提交任务）"  
[ThreadPoolExecutor解析四（线程池关闭）]:http://donald-draper.iteye.com/blog/2367246 "ThreadPoolExecutor解析四（线程池关闭）"  
[ScheduledThreadPoolExecutor解析一（调度任务，任务队列）]:http://donald-draper.iteye.com/blog/2367332 "ScheduledThreadPoolExecutor解析一（调度任务，任务队列）"  
[ScheduledThreadPoolExecutor解析二（任务调度）]:http://donald-draper.iteye.com/blog/2367593 "ScheduledThreadPoolExecutor解析二（任务调度）"  
[ScheduledThreadPoolExecutor解析三（关闭线程池）]:http://donald-draper.iteye.com/blog/2367698 "ScheduledThreadPoolExecutor解析三（关闭线程池）"  
[Executors解析]:http://donald-draper.iteye.com/blog/2367793 "Executors解析"  
