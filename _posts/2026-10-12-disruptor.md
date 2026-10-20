---
layout: page
title: Disruptor高性能框架设计
subtitle: Disruptor高性能线程间消息传递库框架设计
date: 2026-10-12 15:00:00
author: 0xTTEPX
catalog: true
category: MQ
categories:
    - MQ
tags:
    - Disruptor
---

# 引言

随着金融交易系统、撮合引擎、实时风控等场景对延迟和吞吐量的要求越来越高，传统的基于`BlockingQueue`的生产消费模型逐渐暴露出诸多瓶颈：锁争用、对象频繁创建带来的GC压力、缓存行伪共享等问题，使得其在极端性能场景下难以满足需求。LMAX交易所团队开发的Disruptor框架，以环形缓冲区（RingBuffer）为基础，以序列号（Sequence）协调机制为核心，辅以可配置的等待策略和消费者依赖拓扑编排，实现了JVM进程内低延迟、高吞吐的事件传递。Disruptor在撮合、风控、日志聚合、异步流水线等对性能敏感的场景中得到了广泛应用。


# 目录
* [Disruptor框架设计](#Disruptor框架设计)
* [核心组件分层架构](#核心组件分层架构)
* [事件生产流程](#事件生产流程)
* [事件消费流程](#事件消费流程)
* [消费者依赖拓扑](#消费者依赖拓扑)
* [性能优化设计](#性能优化设计)
* [与BlockingQueue对比](#与BlockingQueue对比)
* [局限性与适用场景](#局限性与适用场景)
* [总结](#总结)
* [附](#附)


# Disruptor框架设计


Disruptor框架的核心组件可以分为五个层次：**事件存储层**、**序列协调层**、**等待协调层**、**消费执行层**和**消费拓扑层**。

**事件存储层**的核心数据结构为`RingBuffer`，它是一个固定大小的循环数组，大小必须是2的幂。初始化时通过`EventFactory`预分配事件对象，运行期间生产者复用这些事件槽位，避免频繁创建对象带来的GC压力。`RingBuffer`通过位运算`sequence & (bufferSize - 1)`将逻辑序列号映射到物理数组下标。

**序列协调层**的核心抽象为`Sequencer`接口，负责序列号分配、事件发布状态管理和gating sequence管理。`Sequencer`有两个实现：`SingleProducerSequencer`针对单生产者场景优化，采用更简单的协调逻辑；`MultiProducerSequencer`支持多生产者并发写入，通过`availableBuffer`机制追踪每个序列的发布状态，确保消费者只能消费连续可用的事件序列。

**等待协调层**由`SequenceBarrier`（具体实现为`ProcessingSequenceBarrier`）和`WaitStrategy`组成。`SequenceBarrier`负责判断消费者所需序列是否可安全消费，同时结合上游依赖消费者实现拓扑约束。`WaitStrategy`决定等待事件可用时的具体策略，包括`BlockingWaitStrategy`、`BusySpinWaitStrategy`、`YieldingWaitStrategy`、`SleepingWaitStrategy`、`PhasedBackoffWaitStrategy`等，在延迟与CPU开销之间提供不同的权衡。

**消费执行层**的核心为`BatchEventProcessor`，它是默认的事件循环执行器，批量拉取可用序号范围并回调用户定义的`EventHandler`。此外还有`WorkProcessor`配合`WorkerPool`实现多工作者竞争消费模式。

**消费拓扑层**由`dsl.Disruptor`、`EventHandlerGroup`和`ConsumerRepository`组成。`Disruptor`负责将`RingBuffer`、`Processor`、`Barrier`和依赖关系装配在一起，支持`handleEventsWith(...).then(...)`链式编排。`EventHandlerGroup`表达消费者之间的依赖关系，`ConsumerRepository`维护消费者注册信息和组件映射。

贯穿整个框架的核心抽象是`Sequence`，它用序号追踪每个参与者（生产者、消费者）的逻辑进度，并通过填充（Padding）设计规避伪共享（False Sharing）问题。


# 核心组件分层架构

| 层次 | 核心组件 | 主要职责 |
|------|----------|----------|
| 事件存储层 | `RingBuffer` | 存储和复用事件对象，序列号到数组下标的映射 |
| 序列协调层 | `Sequencer`（`SingleProducerSequencer`/`MultiProducerSequencer`） | 管理序列申请、事件发布和槽位复用 |
| 进度追踪 | `Sequence` | 记录生产者/消费者的逻辑进度，规避伪共享 |
| 等待协调层 | `ProcessingSequenceBarrier`、`WaitStrategy` | 等待事件可用，协调消费者依赖 |
| 消费执行层 | `BatchEventProcessor`、`WorkProcessor` | 驱动事件消费，批量回调`EventHandler` |
| 消费拓扑层 | `Disruptor`、`EventHandlerGroup`、`ConsumerRepository` | 注册消费者，构建依赖关系DAG |

其中，`RingBuffer`、`Sequencer`和`Sequence`是理解整个系统的基础：

- `RingBuffer`：事件放在哪里
- `Sequencer`：谁可以使用哪个序列，以及事件什么时候发布
- `Sequence`：各个参与者目前推进到哪里


# 事件生产流程

## 单生产者模式

```
Producer Thread
      |
      v
ringBuffer.next()
      |
      v
SingleProducerSequencer.next()
      |
      v
检查RingBuffer剩余容量（gating sequences）
      |
      v
获取可用序列号 sequence
      |
      v
ringBuffer.get(sequence)
      |
      v
写入事件数据到Event对象
      |
      v
ringBuffer.publish(sequence)
      |
      v
Sequencer标记序列已发布（volatile write / CAS）
```

单生产者模式下只有一个线程申请序列，因此不需要解决多线程并发争抢序列的问题。`next()`方法内部通过比较当前序列与最慢gating sequence的距离，判断RingBuffer是否有足够空间。如果空间不足，生产者线程将根据所选`WaitStrategy`等待消费者推进。

典型代码模式：
```java
long sequence = ringBuffer.next();
try {
    OrderEvent event = ringBuffer.get(sequence);
    event.setOrderId(orderId);
    event.setPrice(price);
} finally {
    ringBuffer.publish(sequence);
}
```

**关键点**：`next()`和`publish()`必须成对出现。申请到序列号不代表事件已准备好供消费者读取，只有`publish()`之后消费者才能看到该事件。

## 多生产者模式

```
Producer A ──┐
Producer B ──┼──> ringBuffer.next() / next(batchSize)
Producer C ──┘
      |
      v
MultiProducerSequencer（CAS竞争序列号）
      |
      v
Producer A -> claim sequence 101（写入慢）
Producer B -> claim sequence 102（写入快，已publish）
Producer C -> claim sequence 103（写入快，已publish）
      |
      v
availableBuffer记录发布状态：
  101: 未发布
  102: 已发布
  103: 已发布
      |
      v
消费者必须等待101发布后，才能消费102、103（连续可用约束）
```

多生产者模式下，多个线程通过CAS操作并发申请序列号。由于各生产者写入速度不同，可能出现序列号"空洞"。`MultiProducerSequencer`通过`availableBuffer`数组记录每个序列的发布状态，消费者在推进消费进度时必须遵守连续可用序列的约束。

**关键点**：不能仅依赖Cursor（游标）判断事件是否可用。Cursor表示生产者已发布的最大进度，但在多生产者模式下，Cursor到达的位置并不意味着之前的所有序列都已发布。

## 批量发布

Disruptor支持一次申请一段连续的序列号：

```java
long hi = ringBuffer.next(batchSize);
long lo = hi - batchSize + 1;
for (long sequence = lo; sequence <= hi; sequence++) {
    OrderEvent event = ringBuffer.get(sequence);
    event.set(orders.get((int)(sequence - lo)));
}
ringBuffer.publish(lo, hi);
```

批量发布减少逐条申请的固定开销，配合消费端的批量消费可以进一步降低整体延迟。


# 事件消费流程

```
BatchEventProcessor（事件循环）
      |
      v
读取当前消费进度 sequence.get()
      |
      v
计算下一个待处理序列 nextSequence
      |
      v
sequenceBarrier.waitFor(nextSequence)
      |
      +----> ProcessingSequenceBarrier
      |         |
      |         +----> 检查上游依赖消费者的Sequence
      |         +----> 检查Sequencer的可用性（cursor / availableBuffer）
      |         +----> 如果不可用，调用WaitStrategy等待
      |
      v
返回可用序列范围 availableSequence
      |
      v
循环消费：
  while (nextSequence <= availableSequence) {
      T event = dataProvider.get(nextSequence);
      eventHandler.onEvent(event, nextSequence,
                           nextSequence == availableSequence);
      nextSequence++;
  }
      |
      v
更新消费进度 sequence.set(availableSequence)
      |
      v
继续下一轮循环
```

`BatchEventProcessor`是消费端的核心引擎，其本质是一个运行在独立线程上的事件循环。每一轮循环中：

1. 通过`SequenceBarrier`等待事件可用
2. 批量获取可用事件并逐一回调`EventHandler`
3. 处理完一批后更新自身`Sequence`

**批量消费的优势**：
- 减少重复等待的开销
- 降低消费进度更新频率
- 利用`endOfBatch`标志触发批量业务操作（如批量写入数据库、批量发送RPC）

**`endOfBatch`标志**：`onEvent`方法的第三个参数标识当前事件是否为当前批次的最后一个。业务代码可以据此累积事件，在批次结束时批量处理，从而减少外部系统调用次数。


# 消费者依赖拓扑

Disruptor支持通过`EventHandlerGroup`构建消费者之间的依赖关系DAG（有向无环图）。

## 并行消费（广播模型）

```
              RingBuffer
                  |
          +-------+-------+
          |       |       |
          v       v       v
       HandlerA HandlerB HandlerC
```

三个独立的`EventHandler`，每个处理器都能看到事件流中的每一个事件，并行处理。

```java
disruptor.handleEventsWith(handlerA, handlerB, handlerC);
```

## 流水线消费（依赖模型）

```
              RingBuffer
                  |
                  v
             HandlerA（阶段1）
                  |
                  v
             HandlerB（阶段2）
                  |
                  v
             HandlerC（阶段3）
```

HandlerB依赖HandlerA，HandlerC依赖HandlerB，形成严格的处理顺序。

```java
disruptor.handleEventsWith(handlerA)
         .then(handlerB)
         .then(handlerC);
```

## 混合拓扑（并行+汇聚）

```
              TradeEvent
                  |
        +---------+---------+
        |         |         |
        v         v         v
     Order     Asset     Market
        |         |
        +----+----+
             |
             v
        Notification
```

订单处理、资产处理、行情处理可以并行执行；通知处理依赖订单和资产更新完成。

```java
EventHandlerGroup<TradeEvent> orderAndAsset =
    disruptor.handleEventsWith(orderHandler, assetHandler);
disruptor.handleEventsWith(marketHandler);
orderAndAsset.then(notificationHandler);
```

**关键点**：消费者之间的依赖不是通过业务线程互相调用实现的，而是通过`Sequence`和`SequenceBarrier`表达并协调。`ProcessingSequenceBarrier`在等待事件可用时，同时检查上游依赖消费者的`Sequence`进度，确保下游消费者不会在上游处理完成之前提前消费。


# 性能优化设计

## 1. 固定容量RingBuffer

固定大小的循环数组避免了为每个事件动态创建链表节点的需求，减少对象分配和内存管理开销。数组大小必须是2的幂，使得序列号到下标的映射可以用位运算`sequence & (bufferSize - 1)`替代取模运算。

## 2. 事件对象预分配

通过`EventFactory`在初始化时预创建所有事件对象，运行期间生产者复用这些槽位，避免反复创建和销毁事件对象，降低GC压力。

## 3. 序列号协调机制

Disruptor使用序列号表达生产者和消费者的逻辑进度，避免了将所有协调逻辑建立在单一的队列锁操作上。通过`Sequence`的原子操作和内存屏障实现线程间的进度可见性。

## 4. 伪共享规避（Cache Line Padding）

在高性能并发场景中，多个线程频繁更新不同变量时，如果这些变量位于同一缓存行（Cache Line），缓存一致性协议会导致缓存行在CPU核心之间反复转移，即使线程操作的是不同变量也会相互影响性能。Disruptor的`Sequence`实现使用填充设计，将频繁更新的序列变量隔离到独立的缓存行中，降低伪共享风险。

```
Cache Line
+----------------------+----------------------+
| Producer Sequence    |  (Padding)           |
+----------------------+----------------------+
| Consumer A Sequence  |  (Padding)           |
+----------------------+----------------------+
| Consumer B Sequence  |  (Padding)           |
+----------------------+----------------------+
```

## 5. 可配置的等待策略

| 策略 | 主要特点 | 典型场景 |
|------|----------|----------|
| `BlockingWaitStrategy` | 等待时阻塞线程，使用锁和条件变量 | 通用默认策略，CPU开销低 |
| `BusySpinWaitStrategy` | 持续自旋检查 | 延迟敏感、CPU资源充足 |
| `YieldingWaitStrategy` | 自旋后让出CPU执行机会 | 延迟与CPU之间折中 |
| `SleepingWaitStrategy` | 自旋后进入短暂休眠 | 不要求极低等待延迟 |
| `PhasedBackoffWaitStrategy` | 分阶段退避 | 复杂场景，适配不同负载阶段 |

选择等待策略需要综合考虑事件到达频率、消费者空闲比例、CPU核心数量、线程绑定与调度情况、延迟分布（尾延迟而非仅平均延迟）以及整体系统的CPU预算。

## 6. 批量生产与批量消费

批量发布（`next(n)` + `publish(lo, hi)`）减少逐条发布的固定开销；批量消费（`endOfBatch`标志）减少重复等待和进度更新频率，配合业务层的批量写入可以显著降低外部系统调用次数。


# 与BlockingQueue对比

| 对比维度 | Disruptor | BlockingQueue |
|----------|-----------|---------------|
| 核心结构 | 固定容量RingBuffer | 依具体实现（数组、链表等） |
| 事件对象复用 | 预分配和复用 | 取决于使用方式 |
| 消费者模型 | 原生支持多播和复杂依赖拓扑 | 通常需要自行编排 |
| 批量消费 | 原生支持批次处理语义 | 依实现及消费代码而定 |
| 等待策略 | 多种可选策略 | 取决于具体队列实现 |
| 生产消费协调 | 基于序列号和发布状态机制 | 取决于具体实现（锁/CAS） |
| 伪共享处理 | 缓存行填充 | 通常不处理 |
| 使用复杂度 | 较高 | 通常较低 |
| 持久化 | 不提供 | 通常不提供 |
| 跨进程传输 | 不支持 | 普通队列也不支持 |

**注意**：`BlockingQueue`是接口，`ArrayBlockingQueue`、`LinkedBlockingQueue`和`SynchronousQueue`的内部机制存在明显区别，选型时不能一概而论。


# 局限性与适用场景

## 局限性

| 局限 | 说明 |
|------|------|
| 仅解决进程内传递 | 不提供Kafka那样的跨进程消息传输能力 |
| 不自带持久化日志 | 进程异常退出后，不能依赖RingBuffer恢复未处理事件 |
| 固定容量 | 生产者可能受到慢消费者的限制 |
| 依赖关系设计复杂 | 消费拓扑不合理时，可能出现不必要的等待 |
| 不自动提供业务事务 | 不保证多个消费者的外部副作用原子提交 |
| 消费失败需额外设计 | 重试、补偿、幂等需要应用层负责 |
| 自旋可能消耗大量CPU | 低延迟策略可能增加资源占用 |
| 业务代码仍可能产生GC | 事件预分配不能消除所有对象分配 |

## 适用场景

- JVM内部的订单事件分发
- 交易事件的低延迟处理
- 多模块共享事件流（撮合、风控、行情）
- 实时统计与监控
- 高性能计算管道
- 需要显式控制消费者依赖关系的处理流程
- MQ与业务层之间的进程内低延迟缓冲层

## 不适用场景

- 跨进程消息中间件（应使用Kafka、RocketMQ等）
- 持久化事件日志
- 可靠的分布式事务协调器
- 自动保证Exactly-once的处理框架


# 总结

Disruptor是一个以**预分配环形缓冲区**为基础、以**序列号**为协调机制、以**可配置等待策略**为辅助、支持**批量消费**和**消费者依赖拓扑**的高性能JVM进程内事件处理框架。

它的核心设计可以归纳为：

```
RingBuffer
    |
    v
Sequencer（Single/Multi Producer）
    |
    v
Sequence / 发布状态（availableBuffer）
    |
    v
SequenceBarrier + WaitStrategy
    |
    v
BatchEventProcessor
    |
    v
EventHandler（业务处理）
```

其中：
- `RingBuffer`负责事件存储与复用
- `Sequencer`负责序列分配、发布协调与容量管理
- `Sequence`记录各参与者的进度，通过填充规避伪共享
- `SequenceBarrier`协调消费者的等待和依赖关系
- `WaitStrategy`决定等待事件的具体方式
- `BatchEventProcessor`驱动批量事件消费
- `EventHandlerGroup`帮助构建消费者依赖拓扑

对于架构师而言，最值得掌握的核心问题是：**生产者、消费者和消费者依赖关系如何通过Sequence协同工作**。理解了这一点，再分析Disruptor的源码、性能瓶颈和实际应用场景，就会容易得多。


# 附
源码代码阅读入口

## RingBuffer（事件存储层）

### RingBuffer核心方法
com.lmax.disruptor.RingBuffer#get(long sequence)

com.lmax.disruptor.RingBuffer#next()

com.lmax.disruptor.RingBuffer#next(int n)

com.lmax.disruptor.RingBuffer#publish(long sequence)

com.lmax.disruptor.RingBuffer#publish(long lo, long hi)

com.lmax.disruptor.RingBuffer#createMultiProducer

com.lmax.disruptor.RingBuffer#createSingleProducer

## Sequencer（序列协调层）

### Sequencer接口
com.lmax.disruptor.Sequencer

### SingleProducerSequencer
com.lmax.disruptor.SingleProducerSequencer#next()

com.lmax.disruptor.SingleProducerSequencer#next(int n)

com.lmax.disruptor.SingleProducerSequencer#publish(long sequence)

com.lmax.disruptor.SingleProducerSequencerPadding（填充字段，规避伪共享）

### MultiProducerSequencer
com.lmax.disruptor.MultiProducerSequencer#next()

com.lmax.disruptor.MultiProducerSequencer#next(int n)

com.lmax.disruptor.MultiProducerSequencer#publish(long sequence)

com.lmax.disruptor.MultiProducerSequencer#publish(long lo, long hi)

com.lmax.disruptor.MultiProducerSequencer#isAvailable(long sequence)

com.lmax.disruptor.MultiProducerSequencer#getHighestPublishedSequence(long lowerSequence, long availableSequence)

availableBuffer：发布状态追踪数组

## Sequence（进度追踪）

### Sequence
com.lmax.disruptor.Sequence#Sequence()

com.lmax.disruptor.Sequence#get()

com.lmax.disruptor.Sequence#set(long value)

com.lmax.disruptor.Sequence#compareAndSet(long expectedValue, long newValue)

填充字段（Padding）：规避缓存行伪共享

## SequenceBarrier（等待协调层）

### ProcessingSequenceBarrier
com.lmax.disruptor.ProcessingSequenceBarrier#waitFor(long sequence)

com.lmax.disruptor.ProcessingSequenceBarrier#checkAlert()

com.lmax.disruptor.ProcessingSequenceBarrier#clearAlert

### WaitStrategy
com.lmax.disruptor.WaitStrategy#waitFor(long sequence, Sequence cursor, Sequence dependentSequence, SequenceBarrier barrier)

### BlockingWaitStrategy
com.lmax.disruptor.BlockingWaitStrategy#waitFor

com.lmax.disruptor.BlockingWaitStrategy#signalAllWhenBlocking

### BusySpinWaitStrategy
com.lmax.disruptor.BusySpinWaitStrategy#waitFor

### YieldingWaitStrategy
com.lmax.disruptor.YieldingWaitStrategy#waitFor

### SleepingWaitStrategy
com.lmax.disruptor.SleepingWaitStrategy#waitFor

### PhasedBackoffWaitStrategy
com.lmax.disruptor.PhasedBackoffWaitStrategy#waitFor

## BatchEventProcessor（消费执行层）

### BatchEventProcessor
com.lmax.disruptor.BatchEventProcessor#run()

com.lmax.disruptor.BatchEventProcessor#processEvent

com.lmax.disruptor.BatchEventProcessor#notifyStart()

com.lmax.disruptor.BatchEventProcessor#notifyShutdown()

### EventHandler
com.lmax.disruptor.EventHandler#onEvent(T event, long sequence, boolean endOfBatch)

### RewindableEventHandler
com.lmax.disruptor.RewindableEventHandler

### ExceptionHandler
com.lmax.disruptor.ExceptionHandler#handleEventException

## 消费拓扑层（DSL）

### Disruptor
com.lmax.disruptor.dsl.Disruptor#handleEventsWith(EventHandler...)

com.lmax.disruptor.dsl.Disruptor#handleEventsWithWorkerPool(WorkHandler...)

com.lmax.disruptor.dsl.Disruptor#start()

com.lmax.disruptor.dsl.Disruptor#shutdown()

com.lmax.disruptor.dsl.Disruptor#halt()

### EventHandlerGroup
com.lmax.disruptor.dsl.EventHandlerGroup#then(EventHandler...)

com.lmax.disruptor.dsl.EventHandlerGroup#and(EventHandler...)

### ConsumerRepository
com.lmax.disruptor.dsl.ConsumerRepository

### ProducerType
com.lmax.disruptor.dsl.ProducerType#SINGLE

com.lmax.disruptor.dsl.ProducerType#MULTI

## WorkHandler模式（竞争消费）

### WorkerPool
com.lmax.disruptor.WorkerPool

### WorkProcessor
com.lmax.disruptor.WorkProcessor#run()

### WorkHandler
com.lmax.disruptor.WorkHandler#onEvent(T event)
