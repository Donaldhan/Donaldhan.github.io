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


## CountDownLatch使用场景
CountDownLatch同步化的一个辅助工具，允许一个或多个线程等待，直到所有线程中执行完成
比如主线程计算一个复杂的计算表达式，将表达式分为多个子表达式在线程中去计算，
主线程要计算表达式的最后值，必须等所有的线程计算完子表达式计算，方可计算表达式的值；
再比如一个团队赛跑游戏，最后要计算团队赛跑的成绩，主线程计算最后成绩，要等到所有
团队成员跑完，方可计算总成绩。使用情况两种：第一种，所有线程等待一个开始信息号，当开始信息号启动时，所有线程执行，等待所有线程执行完；第二种，所有线程放在线程池中，执行，等待所有线程执行完，方可执行主线程任务方可执行主线程任务。

## Lock和synchronized的性能的比较
在公平的锁上，线程按照他们发出请求的顺序获取锁，但在非公平锁上，则允许‘插队’：当一个线程请求非公平锁时，如果在发出请求的同时该锁变成可用状态，那么这个线程会跳过队列中所有的等待线程而获得锁。 非公平的ReentrantLock 并不提倡 插队行为，但是无法防止某个线程在合适的时候进行插队。在公平的锁中，如果有另一个线程持有锁或者有其他线程在等待队列中等待这个所，那么新发出的请求的线程将被放入到队列中。而非公平锁上，只有当锁被某个线程持有时，新发出请求的线程才会被放入队列中。非公平锁性能高于公平锁性能的原因：在恢复一个被挂起的线程与该线程真正运行之间存在着严重的延迟。假设线程A持有一个锁，并且线程B请求这个锁。由于锁被A持有，因此B将被挂起。当A释放锁时，B将被唤醒，因此B会再次尝试获取这个锁。与此同时，如果线程C也请求这个锁，那么C很可能会在B被完全唤醒之前获得、使用以及释放这个锁。这样就是一种双赢的局面：B获得锁的时刻并没有推迟，C更早的获得了锁，并且吞吐量也提高了。当持有锁的时间相对较长或者请求锁的平均时间间隔较长，应该使用公平锁。在这些情况下，插队带来的吞吐量提升（当锁处于可用状态时，线程却还处于被唤醒的过程中）可能不会出现。

## AQS详解-CLH队列，线程等待状态
AQS使用CAS原子操作，修改锁的状态state；待锁的线程被放入到等待队列（CLH队列）中，每个线程等待状态用NODE来描述。
NODE有共享模式和独占模式，独占模式为NULL。NODE有CANCELLED，SIGNAL，SIGNAL，PROPAGATE4中状态值。

1. SIGNAL：节点的后继由于park等原因被阻塞，当节点释放锁或取消时，要unpark后继节点。为了避免竞争，acquire方法必须，首先检查他们是否需要唤醒后继节点，再原子获取锁，获成功，失败，阻塞。简单说，节点释放锁，是否需要唤醒后继节点
2. CANCELLED:节点由于等待锁超时或者中断等原因，被取消，节点不会停留在这个状态。简单说，单节点处于这个状态，将被移除到等待队列

3. CONDITION: 处于这个状态的节点线程，放在条件队列中。它永远不会被用作一个同步队列节点，直到等待的条件发生，节点将被转移到同步队列中。（这个状态与其他状态，没有关联，只是一种简化的机制）。

4. PROPAGATE: 处于此模式下，释放共享锁具有传递性。头节点调用doReleaseShared方法，保证传递释放共享锁，即使有其他的操作干涉。这个时共享模式下的状态。

CLH队列由于虚头节点，队列中线程等待节点有一个前驱和一个后继节点，NODE有一个状态waitStatus，描述线程的当前状态，有一个线程field用于表示当前等待线程，同时还有nextWaiter节点，用于描述，节点时候有等待条件，或共享模式，获取锁时，需要通知其他线程。

Node nextWaiter：节点下一个等待条件或共享锁的节点。当线程持有独占锁时，只需要访问条件队列，所以我们只需要一个简单的连接队列，存储等待条件的线程。当他们转移到主队列时，可以重新获取锁。由于条件可以是互斥的，所以我们用，特殊的值，去表示共享模式。AQS有一个状态state表示锁的状态，一个CLH队列存放等待锁的线程节点。NODE还可以用于描述节点的等待条件节点线程，用nextWaiter去关联，组成的队列是条件队列。条件队列和等待队列并不冲突，当等待条件的线程被唤醒时，可以尝试获取锁，加入到等待对列。当一个等待队列节点线程获取独占锁时，可以访问条件队列，唤醒等待条件的线程。

## AQS-Condition详解
ConditionObject是AQS的一个内部类，其实现是基于AQS；ConditionObject中的条件等待队列中的节点与同步队列中的节点基本相同，最大的不同时，同步等待队列中的节点有先驱pre和后继next，而条件等待队列中的节点只有后继nextWaiter。条件等待有两种方法，一种为非中断等待awaitUninterruptibly，在线程等待条件时不可中断线程；另外一种，可中断等待await，等待后，确定中断当前线程，还是抛出异常。非中断模式等待，首先添加新的等待条件线程节点，到等待条件线程队列；释放节点状态，释放节点锁状态过程，待子类扩展，在过程中，同时唤醒等待队列头结点；再判断节点是在同步等待队列上，如果不在，则park当前线程；如果线程已经中断，则取消线程中断状态，即线程非中断等待条件。唤醒有两种，一种是唤醒等待条件队列中的头结点，另一种，唤醒条件等待队列中，所有等待此条件的节点线程。唤醒节点时，首先设置线程等待的节点状态的初始状态0，然后添加到同步等待队列，如果线程取消等待，则移除线程，否则通知节点前驱，当前驱节点线程释放锁时，unpark线程。在唤醒等待条件的线程时，为什么要将，节点移动同步等待线程节点能，这是因为条件时锁的一部分，先创建锁，再创建条件；当线程被唤醒时，只能说明，线程条件发生，能处于park状态，要获取锁，才能unpark线程。

## 可重入锁ReentrantLock详解
可重入自旋锁，当线程持有锁，可以多次获取锁，但最多只有2^31-1次；获取失败时，添加到同步等待队列自旋，直到获取锁成功；ReentrantLock关联一个同步锁SYNC，内部的SYNC是基于AQS实现的。同步锁SYNC有两种实现，公平锁与非公平锁；ReentrantLock默认创建的是非公平锁。比较非公平锁的尝试获取锁nonfairTryAcquire与公平锁TryAcquire的区别在于，非公平尝试获取锁时，如果锁为打开状态，则锁住锁；而公平锁，则先看有没有前驱节点，有前驱，则不能锁住锁，没有则可锁住。公平锁与公平锁lock，最大的不同是非公平锁，先以CAS的方式锁住锁，在进行acquire操作，而公平锁，直接acquire操作。acquire操作主要过程为，自旋，检查节点的前驱节点是否为头节点，如果是，当前节点为同步队列的第一个节点，则尝试获取锁，如果成功，设置头结点为当前节点，否则判断尝试获取锁失败，是否应该park，如果需要park，则park当前线程，park后，检查是否可中断当前线程，如果可，则中断当前线程。

## CountDownLatch详解
CountDownLatch本质上是一个共享锁，是一个多功能的同步工具，可用被用于很多场景。当count初始化为1时，CountDownLatch可以作为一个简单的on/off闭锁，或者可以理解为一扇门：所有调用await的线程，等待这扇门被线程调用countDown打开。CountDownLatch初始化为N时，这种情况，可以用作一下场景：1.一个线程等待，直到N个线程完成工作或任务；2.一个任务被完成N次。CountDownLatch内部有一个基于AQS实现的共享锁，用SYNC的状态status，来表示，锁可以被多少个线程所共享，当锁被所有的线程打开countDown，则其他线程可以获取锁。在锁没有被完全打开之前，其他线程，自旋，尝试获取共享锁，在这个过程中，线程可能被park，或者中断。

## CyclicBarrier详解
屏障点思想，当每个线程完成任务时，自旋等待条件Condition trip，释放共享锁，count减1；当线程代的最后一个线程到达屏障点时，唤醒线程代中所有等待的线程，
如果有action，执行action，然后创建下一代线程。如果在线程代未结束之前，有等待线程中断或超时，则结束当前代，唤醒所有等待线程，重置count为parties。

## Semaphore详解
信号量，维持着一个许可集。如果许可集中，无许可，线程acquire，将会阻塞，直到其他线程释放许可。线程每一次释放#release，则添加一个许可，潜在地
释放一个阻塞信号获取者。信号量的许可，实际上并不是一对象，仅仅保证一定数量的虚拟许可证。信号量经常被用于，只有一定数量的线程访问一些物理或逻辑资源。比如用信号量控制池对象的获取。当信号量被初始化为1时，作为互斥锁，可以用于最多只有一个 permit可以用的场景。这种方式比较有名的一种是二进制信号量，因为它只有两种状态，1表示可利用，0表示无permits可利用。二进制信号量，有一个属性，锁可以被其他非持有锁的线程释放。这种特性在一些特殊的上下文场景中，比较拥有，比如恢复死锁。信号量中的锁有公平和非公平方式，这个和我们前面讲的可重入锁，有点相似。非公平方式，获取锁，首先检查锁是否可用，可用则获取，公平锁获取锁，先检查有没有前驱节点，有则等待，没有则获取。获取锁的方法，有多种，有公平的和非公平，则个要看需求。

## ReadWriteLock实现ConcurrentMap
ReadWriteLock需要严格区分读写操作，如果读操作使用了写入锁，那么降低读操作的吞吐量，如果写操作使用了读取锁，那么就可能发生数据错误。另外ReentrantReadWriteLock还有以下几个特性：
* 公平性
非公平锁（默认） 这个和独占锁的非公平性一样，由于读线程之间没有锁竞争，所以读操作没有公平性和非公平性，写操作时，由于写操作可能立即获取到锁，所以会推迟一个或多个读操作或者写操作。因此非公平锁的吞吐量要高于公平锁。公平锁 利用AQS的CLH队列，释放当前保持的锁（读锁或者写锁）时，优先为等待时间最长的那个写线程分配写入锁，当前前提是写线程的等待时间要比所有读线程的等待时间要长。同样一个线程持有写入锁或者有一个写线程已经在等待了，那么试图获取公平锁的（非重入）所有线程（包括读写线程）都将被阻塞，
直到最先的写线程释放锁。如果读线程的等待时间比写线程的等待时间还有长，那么一旦上一个写线程释放锁，这一组读线程将获取锁。
* 重入性
读写锁允许读线程和写线程按照请求锁的顺序重新获取读取锁或者写入锁。当然了只有写线程释放了锁，读线程才能获取重入锁。写线程获取写入锁后可以再次获取读取锁，但是读线程获取读取锁后却不能获取写入锁。另外读写锁最多支持65535个递归写入锁和65535个递归读取锁。
* 锁降级
写线程获取写入锁后可以获取读取锁，然后释放写入锁，这样就从写入锁变成了读取锁，从而实现锁降级的特性。
* 锁升级
读取锁是不能直接升级为写入锁的。因为获取一个写入锁需要释放所有读取锁，所以如果有两个读取锁尝试获取写入锁而都不释放读取锁时就会发生死锁。
* 锁获取中断
读取锁和写入锁都支持获取锁期间被中断。这个和独占锁一致。
* 条件变量
写入锁提供了条件变量(Condition)的支持，这个和独占锁一致，但是读取锁却不允许获取条件变量，将得到一个UnsupportedOperationException异常。
* 重入数
读取锁和写入锁的数量最大分别只能是65535（包括重入数）。


## ReentrantReadWriteLock详解一
Sync是ReentrantReadWriteLock实现读写锁的基础，Sync是基于AQS的实现，内部有一个各读锁计数器（ThreadLocal），每个线程拥有自己的读写计数器，存储着线程持有读锁的数量。同时有一个缓存计数器，用于记录当前线程拥有的读锁数量。有一个firstReader用于记录第一个获取读锁的线程，firstReaderHoldCount记录第一个获取读锁线程的持有锁数量。SYNC将锁的状态int state分成两部分，分为高16位和低16位，高位表示共享读锁，低位是独占写锁；所以读锁和写锁的最大持有量为65535。线程只有在没有线程持有读锁且写锁状态为打开，即state为打开状态，或当前线程持有写锁的数量小于65535的情况下，获取写锁成功，否则失败。线程只有在其他线程没有持有写锁，且读锁的持有数量未达到65535，或当前线程持有写锁且没有读锁的持有数量未达到65535（即锁降级），则当前线程获取读锁成功，否则自旋等待。

## ReentrantReadWriteLock详解后续
ReentrantReadWriteLock的锁机制的实现通过内部SYNC，而SYNC是基于AQS的实现。SYNC有两种实现公平和非公平两个版本。公平锁时先判断队列中是否有线程在等待，有则阻塞获取读锁和写锁。而非公平锁，则直接获取写锁；获取读锁时，判断等待队列第一个节点线程是不是在等待独占模式线程，即如果等待队列的第一个节点线程，在等待写锁，则阻塞锁的获取，否则不阻塞。ReentrantReadWriteLock默认为非公平锁可以提高吞吐量，ReentrantReadWriteLock的构造带有一个公平性参数，用于控制内部锁机制是公平锁还是非公平锁。ReadLock和WriteLock都是通过SYNC来实现，在构造函数中通过ReentrantReadWriteLock，将锁sync交给ReadLock和WriteLock。ReadLock是共享模式锁和我们前面讲的CountDownLatch原理较像，WriteLock是独占锁与ReentrantLock的原理较像。
ReadLock和WriteLock还可以创建Condition，用于控制共享锁和独占锁获取时的条件等待。

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
