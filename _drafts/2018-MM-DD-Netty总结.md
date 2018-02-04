---
layout: page
title: Netty总结
subtitle: Netty总结
date: 2018-11-04 15:17:19
author: donaldhan
catalog: true
category: Netty
categories:
    - Netty
tags:
    - Netty
---

# 引言

Netty 是一个易于使用网络通信的客户端/服务器框架,利用Java的高级网络的能力，隐藏其背后的复杂性。同时是一个使用广泛的Java网络编程框架（Netty 在 2011 年获得了Duke's Choice Award。它活跃和成长于用户社区，像大型公司 Facebook 和 Instagram 以及流行 开源项目如 Infinispan, HornetQ, Vert.x, Apache Cassandra 和 Elasticsearch 等，都利用其强大的网络抽象核心代码。

![Netty](/image/Netty/netty.png)

* [Netty 网络通信示例一][]
* [Netty 网络通信示例一][]
* [Netty 网络通信示例三][]
* [Netty 网络通信示例四][]
* [Netty 构建HTTP服务器示例][]
* [Netty UDT网络通信示例][]
* [Netty 通道处理器ChannelHandler和适配器定义ChannelHandlerAdapter][]
* [Netty Inbound/Outbound通道处理器定义][]
* [Netty 简单Inbound通道处理器（SimpleChannelInboundHandler）][]
* [Netty 消息编码器-MessageToByteEncoder][]
* [Netty Inbound/Outbound通道Invoker][]
* [Netty 异步任务-ChannelFuture][]
* [Netty 管道线定义-ChannelPipeline][]
* [Netty 默认Channel管道线初始化][]
* [Netty 默认Channel管道线-添加通道处理器][]
* [Netty 默认Channel管道线-通道处理器移除与替换][]
* [Netty 默认Channel管道线-Inbound和Outbound事件处理][]
* [Netty 通道处理器上下文定义][]
* [Netty 通道处理器上下文][]
* [Netty 通道初始化器ChannelInitializer][]
* [Netty 事件执行器组和事件执行器定义及抽象实现][]
* [Netty 多线程事件执行器组][]
* [Netty 多线程事件循环组][]
* [Netty 抽象调度事件执行器][]
* [Netty 单线程事件执行器初始化][]
* [Netty 单线程事件执行器执行任务与graceful方式关闭][]
* [Netty 单线程事件循环][]
* [Netty nio事件循环初始化][]
* [Netty nio事件循环后续][]
* [Netty nio事件循环组][]
* [Netty 抽象BootStrap定义][]
* [Netty ServerBootStrap解析][]
* [Netty Bootstrap解析][]
* [Netty 通道接口定义][]
* [Netty 抽象通道初始化][]
* [Netty 抽象Unsafe定义][]
* [Netty 通道Outbound缓冲区][]
* [Netty 抽象通道后续][]
* [Netty 抽象nio通道][]
* [Netty 抽象nio字节通道][]
* [Netty 抽象nio消息通道][]
* [Netty NioServerSocketChannel解析][]
* [Netty 通道配置接口定义][]
* [Netty 默认通道配置初始化][]
* [Netty 默认通道配置后续][]
* [Netty NioSocketChannel解析][]
* [Netty 字节buf定义][]
* [Netty 资源泄漏探测器][]
* [Netty 抽象字节buf解析][]
* [Netty 抽象字节buf引用计数器][]
* [Netty 复合buf概念][]
* [Netty 抽象字节buf分配器][]
* [Netty Unpooled字节buf分配器][]
* [Netty Pooled字节buf分配器][]

上面文章中的Netty源码为Netty的4.1分支,当时的源码版本为4.1.12。

# 目录

* [Netty 网络通信示例三](#netty 网络通信示例三)
* [Netty 网络通信示例四](#netty 网络通信示例四)
* [构建HTTP服务器示例](#构建http服务器示例)
* [Netty UDT网络通信示例](#netty udt网络通信示例)
* [Netty 通道处理器ChannelHandler和适配器定义ChannelHandlerAdapter](#Netty 通道处理器channelhandler和适配器定义channelhandleradapter)
* [Netty 简单Inbound通道处理器（SimpleChannelInboundHandler）](#netty 简单inbound通道处理器（simplechannelinboundhandler）)
* [Netty 消息编码器-MessageToByteEncoder](#netty 消息编码器-messagetobyteencoder)
* [Netty 消息解码器-ByteToMessageDecoder](#netty 消息解码器-bytetomessagedecoder)
* [Netty Inbound/Outbound通道Invoker](#netty inbound/outbound通道invoker)
* [Netty 异步任务-ChannelFuture](#netty 异步任务-channelfuture)
* [Netty 管道线定义-ChannelPipeline](#netty 管道线定义-channelpipeline)
* [Netty 默认Channel管道线初始化](#netty 默认channel管道线初始化)
* [Netty 默认Channel管道线-添加通道处理器](#netty 默认channel管道线-添加通道处理器)
* [Netty 默认Channel管道线-通道处理器移除与替换](#netty 默认channel管道线-通道处理器移除与替换)
* [Netty 默认Channel管道线-Inbound和Outbound事件处理](#netty 默认channel管道线-inbound和outbound事件处理)
* [Netty 通道处理器上下文定义][#Netty 通道处理器上下文定义]
* [Netty 通道处理器上下文](#netty 通道处理器上下文)
* [Netty 通道初始化器ChannelInitializer](#netty 通道初始化器channelinitializer)
* [Netty 事件执行器组和事件执行器定义及抽象实现](#netty 事件执行器组和事件执行器定义及抽象实现)
* [Netty 多线程事件执行器组](#netty 多线程事件执行器组)
* [Netty 多线程事件循环组](#netty 多线程事件循环组)
* [Netty 抽象调度事件执行器](#netty 抽象调度事件执行器)
* [Netty 单线程事件执行器初始化](#netty 单线程事件执行器初始化)
* [Netty 单线程事件执行器执行任务与graceful方式关闭](#netty 单线程事件执行器执行任务与graceful方式关闭)
* [Netty 单线程事件循环](#netty 单线程事件循环)
* [Netty nio事件循环初始化](#netty nio事件循环初始化)
* [Netty nio事件循环后续](#netty nio事件循环后续)
* [Netty nio事件循环组](#netty nio事件循环组)
* [Netty 抽象BootStrap定义](#netty 抽象bootstrap定义)
* [Netty ServerBootStrap解析](#netty serverbootstrap解析)
* [Netty Bootstrap解析](#netty bootstrap解析)
* [Netty 通道接口定义](#netty 通道接口定义)
* [Netty 抽象通道初始化](#netty 抽象通道初始化)
* [Netty 抽象Unsafe定义](#netty 抽象unsafe定义)
* [Netty 通道Outbound缓冲区](#netty 通道outbound缓冲区)
* [Netty 抽象通道后续](#netty 抽象通道后续)
* [Netty 抽象nio通道](#netty 抽象nio通道)
* [Netty 抽象nio字节通道](#netty 抽象nio字节通道)
* [Netty 抽象nio消息通道](#netty 抽象nio消息通道)
* [Netty NioServerSocketChannel解析](#netty nioserversocketchannel解析)
* [Netty 通道配置接口定义](#netty 通道配置接口定义)
* [Netty 默认通道配置初始化](#netty 默认通道配置初始化)
* [Netty 默认通道配置后续](#netty 默认通道配置后续)
* [Netty NioSocketChannel解析](#netty niosocketchannel解析)
* [Netty 字节buf定义](#netty 字节buf定义)
* [Netty 资源泄漏探测器](#netty 资源泄漏探测器)
* [Netty 抽象字节buf解析](#netty 抽象字节buf解析)
* [Netty 抽象字节buf引用计数器](#netty 抽象字节buf引用计数器)
* [Netty 复合buf概念](#netty 复合buf概念)
* [Netty 抽象字节buf分配器](#netty 抽象字节buf分配器)
* [Netty Unpooled字节buf分配器](#netty unpooled字节buf分配器)
* [Netty Pooled字节buf分配器](#netty pooled字节buf分配器)


## Netty 通道处理器ChannelHandler和适配器定义ChannelHandlerAdapter
通道处理器ChannelHandler，主要有两个事件方法分别为handlerAdded和handlerRemoved，handlerAdded在通道处理器添加到实际上下文后调用，通道处理器准备处理IO事件；handlerRemoved在通道处理器从实际上下文中移除后调用，通道处理器不再处理IO事件。

一个通道处理器关联一个通道处理器上下文ChannelHandlerContext。通道处理器通过一个上下文对象，与它所属的通道管道线交互。通道上下文对象，通道处理器上行或下行传递的事件，动态修改管道，或通过AttributeKey存储特殊的信息。通道处理器内部定义了一个共享注解Sharable，默认访问类型为Protected；添加共享注解的通道处理器，说明通道处理器中的变量可以共享，可以创建一个通道处理器实例，多次添加到通道管道线ChannlePipeline;对于没有共享注解的通道器，在每次添加到管道线上时，都要重新创建一个通道处理器实例。通道处理器只定义了简单的通道处理器添加到通道处理器上下文或从上下文移除的事件操作，没有具体定义读操作（上行UpStream，输入流Inbound，字节流到消息对象ByteToMessage），写操作（下行DownStream，输出流Outbound，消息到字节流MessageToByte）。这操作分别定义在，输入流处理器ChannelInboundHandler，输出流处理器ChannelOutboundHandler，并提供了处理的相应适配器，输入流处理器适配器ChannelInboundHandlerAdapter，输出流通道适配器ChannelOutboundHandlerAdapter，多路复用适配器ChannelDuplexHandler。

通道处理器适配器ChannelHandlerAdapter的设计模式为适配器，这个适配器模式中的 handlerAdded和handlerRemoved事件默认处理器，不做任何事情，这个与MINA中的适配器模式相同。处理IO操作异常，则调用ChannelHandlerContext#fireExceptionCaught方法，触发异常事件，并转发给通道管道线的下一个通道处理器。
看通道处理器适配器的判断通道处理器是否共享注解，首先获取线程的本地变量，从线程本地变量获取线程本地共享注解通道处理器探测结果缓存，如果缓存中存在通道处理器clazz，则返回缓存结果，否则将探测结果添加到缓存中。

## Netty Inbound/Outbound通道处理器定义
通道Inbound处理器，主要是处理从peer发送过来的字节流；通道处理器上下文关联的通道注册到事件循环EventLoop时，触发channelRegistered方法；通道处理器上下文关联的通道激活时，触发channelActive方法；通道从peer读取消息时，触发channelRead方法；当上一消息通过#channelRead方法，并被当先读操作消费时，触发channelReadComplete方法，如果通道配置项#AUTO_READ为关闭状态，没有进一步尝试从当前通道读取inbound数据时，直到ChannelHandlerContext#read调用，触发；当用户事件发生时，触发userEventTriggered方法；异常抛出时，触发exceptionCaught方法；当通道可写状态改变时，触发channelWritabilityChanged方法；通道处理器上下文关联的通道注册到事件循环EventLoop，但处于非激活状态，达到生命周期的末端时，触发channelInactive方法；通道处理器上下文关联的通道从事件循环EventLoop移除时，触发channelUnregistered方法。

Inbound通道handler适配器ChannelInboundHandlerAdapter，提供的Inbound通道处理器的所有方法的实现，但实现仅仅是，转发操作给Channel管道线的下一个通道处理器，子类必须重写方法。需要注意的是，在#channelRead方法自动返回后，消息并没有释放。如果你寻找ChannelInboundHandler的实现，可以自动释放接受的到消息可以使用SimpleChannelInboundHandler。

Outbound通道处理器ChannelOutboundHandler主要处理outbound IO操作。当绑定操作发生时，调用bind方法；当连接操作发生时，调用connect方法；read方法拦截通道处理器上下文读操作；当写操发生时，调用write方法，写操作通过Channel管道线写消息，当通道调用#flush方法时，消息将会被刷新，发送出去；当一个刷新操作发生时，调用flush方法，刷新操作将会刷新所有先前已经写，待发送的消息。

 Outbound通道Handler适配器ChannelOutboundHandlerAdapter为Outbound通道处理器的基本实现，这个实现仅仅通过通道处理器上下文转发方法的调用。子类必须重写Outbound通道Handler适配器的相关方法。

在Mina中，通道读写全部在一个通道Handler，Mina提供的通道Handler适配器，我们在使用通道处理器时继承它，实现我们需要关注的读写事件。而Netty使用InBound和OutBound将通道的读写分离，同时提供了InBound和OutBound通道Handler的适配器。

## Netty 简单Inbound通道处理器（SimpleChannelInboundHandler）
简单Inbound通道处理器SimpleChannelInboundHandler<I>，内部有连个变量一个为参数类型匹配器，用来判断通道是否可以处理消息，另一个变量autoRelease，用于控制是否在通道处理消息完毕时，释放消息。读取方法channelRead，首先判断跟定的消息类型是否可以被处理，如果是，则委托给channelRead0，channelRead0待子类实现；如果返回false，则将消息转递给Channel管道线的下一个通道处理器；最后，如果autoRelease为自动释放消息，且消息已处理则释放消息。

## Netty 消息编码器-MessageToByteEncoder
消息编码器MessageToByteEncoder实际上为一个Outbound通道处理器，内部有一个类型参数处理器TypeParameterMatcher，用于判断消息是否可以被当前编码器处理，不能则传给Channel管道线上的下一个通道处理器；一个preferDirect参数，用于决定，当将消息编码为字节序列时，应该存储在direct类型还是heap类型的字节buffer中。消息编码器主要方法为write方法，write方法首先，判断消息是否可以被当前编码器处理，如果消息可以被编码器处理，根据通道处理器上下文和preferDirect，分配一个字节buf，委托encode方法，编码消息对象到字节buf，encode方法待子类实现；释放消息对应引用参数，如果当前buffer可读，则通道处理器上下文写buffer，否则释放buffer，写空buf，最后释放buf。消息编码器MessageToByteEncoder实际上为一个Outbound通道处理器，这个与Mina中的消息编码器是有区别的，Mina中的消息编码器要和解码器组装成编解码工厂过滤器添加到过滤链上，且编解码工厂过滤器，在过滤链上是由先后顺序的，通道Mina中编码器和通道Handler是两个概念。而Netty中编码器实际为Outbound通道处理器，主要是通过类型参数匹配器TypeParameterMatcher，来判断消息是否可以被编码器处理。

## Netty 消息解码器-ByteToMessageDecoder
消息解码器ByteToMessageDecoder，内部有两个buf累计器，分别为MERGE_CUMULATOR累计器buf，累计buf的过程为，首先判断当前累计buf空间不够存储，需要整合的in buf，或当前buf的引用数大于1，或累计buf只可读，三个条件中，有一个满足，则扩展累计buf容量，然后写in buf字节序列到累计器，释放in buf；COMPOSITE_CUMULATOR累计器是将需要整合的buf放在，内部的Component集合中，每个Component描述一个buf信息。

解码器有一个累计buf cumulation用于存放接收的数据；一个累计器Cumulator，默认为MERGE_CUMULATOR，累计接收的字节数据到cumulatio；first表示是否第一次累计字节到累计buf；decodeWasNull表示当前解码器是否成功解码消息，后续调用ByteBuf#discardSomeReadBytes方法；singleDecode表示解码器是否只能解码一次；numReads表示当前已经读取的字节数；discardAfterReads表示在读取多少个字节后，调用ByteBuf#discardSomeReadBytes方法，释放buf空间；解码器有三种状态，初始化状态STATE_INIT，还没有解码，正在解码状态STATE_CALLING_CHILD_DECODE，解码器正在从通道处理器上下文移除状态STATE_HANDLER_REMOVED_PENDING。

需要注意的是，解码器不可共享。读取操作首先判断消息是否为字节buf，是则，创建解码消息List集合out，如果第一次累计字节buf，则累计buf为，消息buf，否则累计器，累计消息buf数据，然后调用解码器#callDecode解码累计buf中的数据，并将解码后的消息添加到out集合中，并遍历解码消息集合，转发消息到Channle管道线上的下一个通道处理器。如果消息类型不是字节buf，直接通知Channle管道线上的下一个通道处理器消息消息。在解码的过程中，如果解码器从通道处理器上下文移除，则处理移除事件。移除解码器，首先判断 解码器状态，如果解码器处于正在解码状态，则解码器状态置为正在移除，并返回，否则判断累计buf是否为空，如果为空，则置空，否则通知通道处理上下文，所属的Channle管道线上的下一通道Handler消费数据。委托给handlerRemoved0方法完成实际的handler移除工作。

解码器的channelReadComplete方法，主要是执行通道处理器上下文read操作，请求从通道读取数据到第一个Inbound字节buf中，如果读取到数据，触发ChannelInboundHandler#channelRead操作；并触发一个ChannelInboundHandler#channelReadComplete时间以便，处理能够决定是否继续读取数据，如果当前解码完成，则通知，管道中的下一个通道处理器的read方法处理数据，如果一个读操作正在放生，则此方法不做什么事情。并触发管道中下一个通道处理器的fireChannelReadComplete事件。

channelInactive事件操作，主要是关闭通道输入流，在关闭之前，如果累计buf不为空，调用callDecode方法，解码字节数，为消息对象，并放入解码消息集合out中，管道中的下一个通道处理器，消费解码消息集合中的消息；最后调用decodeRemovalReentryProtection做最后的解码工作和通道移除工作。

消息解码器ByteToMessageDecoder实际上为Inbound通道处理器，这个与Mina中的消息解码器是有区别的，Mina中的消息解码器要和编码器组装成编解码工厂过滤器添加到过滤链上，且编解码工厂过滤器，在过滤链上是有先后顺序的，通道Mina中解码器和通道Handler是两个概念。

## Netty Inbound/Outbound通道Invoker
每个通道Channel拥有自己的管道Pipeline，当通道创建时，管道自动创建,默认为DefaultChannelPipeline。Inbound通道Invoker ChannelInboundInvoker主要是触发管道线ChannelPipeline上的下一个Inbound通道处理器ChannelInboundHandler的相关方法。ChannelInboundInvoker有点Mina过滤器的意味。Outbound通道Invoker ChannelOutboundInvoker主要是触发触发管道线ChannelPipeline上的下一个Outbound通道处理器ChannelOnboundHandler的相关方法，同时增加了一下通道结果创建方法，
ChannelOutboundInvoker也有点Mina过滤器的意味，只不过不像ChannelInboundInvoker的方法命名那么相似。

## Netty 异步任务-ChannelFuture
netty的异步结果Future继承于JUC的Future，可以异步获取IO操作的结果信息，比如IO操作是否成功完成，如果失败，可以获取失败的原因，是否取消，同时可以使用cancel方法取消IO操作，添加异步结果监听器，、监听IO操作是否完成，并可以移除结果监听器，除这些之外我们还可以异步、同步等待或超时等待IO操作结果。异步结果监听器GenericFutureListener，主要监听一个IO操作是否完成，在异步结果有返回值时，通知监听器。ChannelFuture继承于空异步结果，即没有返回值，所以添加移除监听器，同步异步等待方法为空体。netty所有的IO操作都是异步的，当一个IO操作开始时，不管操作是否完成，一个新的异步操作结果将会被创建。如果因为IO操作没有完成，同时既没有成功，失败，也没有取消，新创建的那么，异步结果并没有完成初始化。如果IO操作完成，不论操作结果成功，失败或取消，异步结果将会标记为完成，同时携带更多的精确信息，比如失败的原因。需要注意的时，失败或取消也属于完成状态。强烈建议使用添加监听器的方式等待IO操作结果，而不await方法，因为监听器模式时非阻塞的，有更好的性能和资源利用率。

通道结果监听器ChannelFutureListener内部有3个监听器，分别为在操作完成时，关闭通道任务关联的通道的监听器CLOSE；当IO操作失败时，关闭通道任务关联的通道的监听器CLOSE_ON_FAILURE；转发通道任务异常到Channel管道的监听器FIRE_EXCEPTION_ON_FAILURE。Promise任务继承了任务Future，但多了以便标记成功、失败和不可取消的方法。
ChannelPromise与ChannelFuture的不同在于ChannelPromise可以标记任务结果。ChannelProgressivePromise与ProgressivePromise，ChannelProgressiveFuture的关系与ChannelPromise与Promise，ChannelFuture的关系类似，只不过ChannelPromise表示异步操作任务，ChannelProgressivePromise表示异步任务的进度，同时Promise类型异步任务都是可写的。

## Netty 管道线定义-ChannelPipeline
Channle管道线继承了Inbound、OutBound通道Invoker和Iterable<Entry<String, ChannelHandler>>接口，Channel管道线主要是管理Channel的通道处理器，每个通道有一个Channle管道线。Channle管道线主要定义了添加移除替换通道处理器的相关方法，在添加通道处理器的相关方法中，有一个事件执行器group参数，用于中Inbound和Outbound的相关事件，告诉管道，在不同于IO线程的事件执行器组中，执行通道处理器的事件执行方法，以保证IO线程不会被一个耗时任务阻塞，如果你的业务逻辑完全异步或能够快速的完成，可以添加一个事件执行器组。Channel管道线中的Inbound和Outbound通道处理器，主要通过通道处理器上下文的相关fire-INBOUND_ENT和OUTBOUND_OPR事件方法，传播Inbound和Outbound事件给管道中的下一个通道处理器

## Netty 默认Channel管道线初始化
netty的log实例，实际是从默认log工厂获取，默认log工厂顺序为SLF4J，LOG4j，JDKLog。
每个通道拥有一个Channel管道线；管道线用于管理，通道事件处理Handler ChannelHandler，管道线管理通道处理器的方式，为通道处理器器上下文模式，即每个通道处理器在管道中，是以通道上下文的形式存在；通道上下文关联一个通道处理器，通道上下文描述通道处理器的上下文，通道上下文拥有一个前驱和后继上下文，即通道上下文在管道线中是一个双向链表，通道处理器上下文通过inbound和oubound两个布尔标志，判断通道处理器是inbound还是outbound。

上下文链表的头部为HeadContext，尾部为TailContext。头部上下文HeadContext的outbound的相关操作，直接委托给管道线所属通道的unsafe（Native API），inbound事件直接触发通道处理器上下文的相关事件，以便通道处理器上下文关联的通道Handler处理相关事件，但读操作实际是通过Channel读取。HeadContext的通道注册方法channelRegistered，主要是执行通道处理器添加回调任务链中的任务。处理器添加回调任务主要是触发触发上下文关联通道处理器的handlerAdded事件，更新上下文状态为添加完毕状态，如果过程中有异常发生，则移除通道上下文。channelUnregistered方法，主要是在通道从选择器反注册时，清空管道线程的通道处理器上下文，并触发上下文关联的通道处理器handlerRemoved事件，更新上下文状态为已移除。

Channel的管道线的通道处理器上下文链的尾部TailContext是一个傀儡，不同于尾部上下文，头部上下文，在处理inbound事件时，触发通道处理器上下文相关的方法，在处理outbound事件时，委托给管道线关联的Channle的内部unsafe。默认Channel管道实现内部有两个回调任务PendingHandlerAdded/RemovedTask，一个是添加通道处理器上下文回调任务，一个是移除通道上下文回调任务，主要是触发上下文关联通道处理器的处理器添加移除事件，并更新相应的上下文状态为已添加或已移除。管道构造，主要是检查管道通道是否为空，初始化管道上下文链的头部与尾部上下文。

netty通道处理器上下文可以说，是Mina中Hanlder和过滤器的集合，整合两者功能，管道线有点Mina过滤链的意味，HeadContext相当于Mina过滤链的头部过滤器，TailContext相当于Mina过滤链的尾部过滤器

## Netty 默认Channel管道线-添加通道处理器
添加通道处理器到管道头部，首次检查通道处理器是否为共享模式，如果非共享，且已添加，则抛出异常；检查通道处理器名在管道内，是否存在对应通道处理器上下文，已存在抛出异常；根据事件执行器，处理器名，及处理器，构造处理器上下文；添加处理器上限文到管道上下文链头；如果通道没有注册到事件循环，上下文添加到管道时，创建添加通道处理器回调任务，并将任务添加管道的回调任务链中，当通道注册到事件循环时，触发通道处理器的handlerAdded事件，已注册则创建一个线程，用于调用通道处理器的handlerAdded事件方法，及更新上下文状态为已添加，并交由事件执行器执行;最好调用callHandlerAdded0方法，确保调用通道处理器的handlerAdded事件方法，更新上下文状态为已添加。其他last（添加到管道尾部），before（添加指定上下文的前面），after（添加指定上下文的后面）操作，基本上与addfirst思路基本相同，不同的是添加到管道上下文链的位置。

## Netty 默认Channel管道线-通道处理器移除与替换
无论是根据名称，处理器句柄，还是根据类型移除通道处理器，都是首先获取对应的处理器上下文，从管道中移除对应的上下文，如果通道已经从事件循环反注册，则添加移除回调任务到管道回调任务链，否则直接创建线程（触发上下文关联的处理器handlerRemoved事件，更新上下文状态为已移除），有上下文关联的事件执行器执行。

无论是根据名称，处理器句柄，还是根据类型替换通道处理器，都是首先获取对应的处理器上下文，然后添加新上下文到管道中原始上下文的位置，并将原始上下文的前驱和后继同时指向新上下文，以便转发剩余的buf内容；可以简单理解为添加新上下文，移除原始上下文，注意必须先添加，后移除，因为移除操作会触发channelRead和flush事件，而这些事件处理必须在handlerAdded事件后。

## Netty 默认Channel管道线-Inbound和Outbound事件处理
管道处理通道注册方法fireChannelRegistered，首先从头部上下文开始，如果上下文已经添加到管道，则触发上下文关联的通道处理器的channelRegistered事件，否则转发事件给上下文所属管道的下一个上下文;其他触发Inbound事件的处理过程与fireChannelRegistered方法思路相同，只不过触发的通道处理器的相应事件。管道处理Inbound事件首先从头部上下文开始，直到尾部上下文，最后默认直接丢弃。

管道处理通道处理器地址绑定bind事件，首先从管道上下文链的尾部开始，寻找Outbound上下文，获取上下文的事件执行器，如果事件执行器线程在当前事件循环中，则触发通道处理器地址绑定事件#invokeBind，否则创建一个线程，执行事件触发操作，并交由事件执行器执行；#invokeBind首先判断通道处理器是否已经添加到管道，如果以添加，则触发Outbound通道处理器的bind事件方法，否则，传递地址绑定事件给管道中的下一个Outbound上下文。管道处理Outbound相关事件，从尾部上下文到头部上下文，到达头部时，交由上下文所属管道关联的Channel的Unsafe处理。

## netty 通道处理器上下文定义
通道处理器上下文ChannelHandlerContext，使通道处理器可以与管道和管道中其他处理器进行交互。当IO事件发生时，处理可以将事件转发给所属管道的下一个通道处理器，同时可以动态修改处理器所属的管道。通过上下文可以获取关联通道，处理器，事件执行器，上下文名，所属管道等信息。同时可以通过AttributeKey存储上下文属性，用alloc方法获取通道上下文的字节buf分配器，用于分配buf。

## Netty 通道处理器上下文
抽象通道处理器上下文AbstractChannelHandlerContext，拥有一个前驱和后继上下文，用于在管道中传递IO事件；通道处理器总共有四个状态，分别为初始化，正在添加到管道，已添加管道和从管道移除状态；上下文同时关联一个管道；Inbound和Outbound两个用于判断上下文的类型，决定了上下文是处理器Inbound事件还是Outbound事件；一个事件执行器executor，当上下文执行器不在当前事务循环中时，用于执行IO事件操作；同时有一些延时任务,如上下文读任务，上下文刷新任务，读完成任务和通道可写状态改变任务。上下文构造，主要是初始化上下文name，所属管道，事件执行器，上下文类型。上下文关联的通道通道处理器在具体的实现中定义，比如通道处理器上下文默认实现为DefaultChannelHandlerContext，内部关联一个通道处理器。

上下文处理通道fireChannelRegistered事件，如果上下文事件执行器在当前事务循环，则直接在当前线程，执行触发上下文关联通道处理器的channelRegistered事件任务，否则，创建一个线程执行事件任务，并由上下文事务执行器运行；触发上下文关联通道处理器的channelRegistered事件任务，首先判断上下文是否已经添加到管道，已添加，则触发
上下文关联通道处理器的channelRegistered事件，否则转发事件给上下文所属管道的下一个Inbound上下文。其他Inbound事件的处理过程与fireChannelRegistered方法思路相同，只不过触发的是通道处理器的相应事件;如果Inbound事件处理过程中，异常发生，首先检查异常是不是通道处理器的exceptionCaught方法抛出，是，则直接log，否则触发上下文关联通道处理器的exceptionCaught事件。

上下文处理关联通道处理器的地址绑定bind事件，首先从所属管道上下文链的尾部开始，寻找Outbound上下文，找到后，获取上下文的事件执行器，如果事件执行器线程在当前事件循环中，则触发上下文关联通道处理器地址绑定事件，否则创建一个线程，执行事件触发操作，并交由事件执行器执行；触发上下文关联通道处理器地址绑定事件，首先判断上下文关联通道处理器是否已经添加到管道，如果以添加，则触发Outbound通道处理器的bind事件方法，否则，传递地址绑定事件给管道中的下一个Outbound上下文。如果在Outbound事件处理器过程中，出现异常则直接委托给异步任务结果通知工具PromiseNotificationUtil，通知异步任务失败，并log异常日志。

netty的通道处理器上下文和mina的会话有点像，都拥有描述通道Handler的Context；不同的时mina中的会话与通道直接关联，而netty上下文与通道间是通过Channel管道关联起来，mina中的过滤链是依附于会话，而netty上下文依附于Channel管道，mina中的IO事件执行器为IoProcessor，netty中的IO事件的处理委托给事件循环或上下文的子事件执行器。

## Netty 通道初始化器ChannelInitializer
通道初始化器ChannelInitializer实际上为Inbound通道处理器，当通道注册到事件循环中后，添加通道初始化器到通道，触发handlerAdded事件，然后将初始化器的上下文放入通道初始化器的上下文Map中，如果放入成功且先前不存在，initChannel(C ch)，初始化通道，其中C为当前通道，我们可以获取C的管道，添加通道处理器到管道，这就是通道初始化器的作用。添加完后，从通道的管道中移除初始化器上下文，并从通道初始化器的上下文Map中移除通道初始化器上下文。

## Netty 事件执行器组和事件执行器定义及抽象实现  
事件循环组EventLoopGroup为一个特殊的事件执行器组EventExecutorGroup，可以注册通道，以便在事件循环中，被后面的选择操作处理器。事件执行器组继承了JUC的调度执行器服务ScheduledExecutorService，用迭代器Iterable<EventExecutor>管理组内的事件执行器。事件执行器是一个特殊的事件执行器组。Nio多线程事件循环NioEventLoopGroup可以理解为多线程版MultithreadEventExecutorGroup的事件执行器组。

事件执行器组EventExecutorGroup主要提供了关闭事件执行器组管理的执行器的相关方法，获取事件执行器组管理的事件执行器和执行任务线程方法。事件执行器EventExecutor为一个特殊的事件执行器组EventExecutorGroup，提供了获取事件执行器组的下一个事件执行器方法，判断线程是否在当前事件循环中以及创建可写的异步任务结果和进度结果，及已经成功失败的异步结果。 抽象事件执行器组AbstractEventExecutorGroup，所有与调度执行器关联的提交任务和调度任务方法，直接委托给事件执行器组的下一个事件执行器相应方法执行。graceful方式关闭事件执行器组，默认关闭间隔为2s，超时时间为25s，具体定义在抽象事件执行器AbstractEventExecutor中。

抽象事件执行器，继承了抽象执行器服务AbstractExecutorService，提交任务线程，直接委托给父类抽象执行器服务，不支持延时调度的周期间歇性调度任务线程，多个一个安全地执行给定任务线程方法，捕捉执行过程中抛出的异常。由于抽象的事件执行器是一个特殊的事件执行器组，内部事件执行器selfCollection（Collections.<EventExecutor>singleton(this)），是自己单例集，next方法返回的是自己。

## Netty 多线程事件执行器组
多线程事件执行器组MultithreadEventExecutorGroup，内部有一个事件执行器数组存放组内的事件执行器；readonlyChildren为组内事件执行器集的可读包装集Set；terminatedChildren（AtomicInteger），用于记录已关闭的事件执行器数；termination为执行器组terminated异步任务结果；同时有一个事件执行器选择器chooser（EventExecutorChooser）。构造多线程执行器组，首先检查线程数参数，如果执行器不为空，则初始化线程执行器的线程工厂，创建事件执行器集，并根据执行器和相关参数创建事件执行器，实际创建方法为newChild，待子类实现，初始化事件执行器选择器，创建terminated事件执行器监听器，添加terminated事件执行器监听器到terminated异步任务结果，包装事件执行器集为只读集readonlyChildren。获取执行器组的下一个事件执行器方法委托个内存的事件执行器选择器chooser；返回的迭代器为内部只读执行器集的迭代器；而关闭执行器组方法，实际为遍历管理的事件执行器集，关闭执行器；判断执行器组是否关闭和Terminated，当且仅当组内的事件执行器都关闭和Terminated时，才返回true；超时等待Terminated执行器组方法，实际为遍历事件执行器组超时等待时间耗完，则停止Terminated执行器组，否则，超时剩余等待时间timeLeft，Terminated事件执行器。

## Netty 多线程事件循环组
事件循环组EventLoopGroup继承了事件执行器组EventExecutorGroup，next方法返回的为事件循环EventLoop，事件循环组主要所做的工作为通道注册。
事件循环EventLoop可理解为已顺序、串行的方式处理提交的任务的事件执行器EventExecutor。事件循环组EventLoopGroup可以理解为特殊的事件执行器组EventExecutorGroup；事件执行器组管理事件执行器，事件循环组管理事件循环。抽象事件循环AbstractEventLoop继承了抽象事件执行器AbstractEventExecutor，实现了事件循环接口。
多线程事件循环组MultithreadEventLoopGroup继承了多线程事件执行器组，实现了事件循环组接口，相关注册通道方法委托给多线程事件循环组的next事件循环，线程工程创建的线程优先级默认为最大线程优先级；默认事件循环线程数为1和可用处理器数的2倍中的最大者，这个线程数就是构造多线程事件执行器组事件执行器数量。

## Netty 抽象调度事件执行器
事件循环组EventLoopGroup为一个特殊的事件执行器组EventExecutorGroup，可以注册通道，以便在事件循环中，被后面的选择操作处理器。事件执行器组继承了JUC的调度执行器服务ScheduledExecutorService，用迭代器Iterable<EventExecutor>管理组内的事件执行器。事件执行器是一个特殊的事件执行器组。Nio多线程事件循环NioEventLoopGroup可以理解为多线程版MultithreadEventExecutorGroup的事件执行器组。

事件执行器组EventExecutorGroup主要提供了关闭事件执行器组管理的执行器的相关方法，获取事件执行器组管理的事件执行器和执行任务线程方法。事件执行器EventExecutor为一个特殊的事件执行器组EventExecutorGroup，提供了获取事件执行器组的下一个事件执行器方法，判断线程是否在当前事件循环中以及创建可写的异步任务结果和进度结果，及已经成功失败的异步结果。抽象事件执行器组AbstractEventExecutorGroup，所有与调度执行器关联的提交任务和调度任务方法，直接委托给事件执行器组的下一个事件执行器相应方法执行。graceful方式关闭事件执行器组，默认关闭间隔为2s，超时时间为25s，具体定义在抽象事件执行器AbstractEventExecutor中。抽象事件执行器，继承了抽象执行器服务AbstractExecutorService，提交任务线程，直接委托给父类抽象执行器服务，不支持延时调度的周期间歇性调度任务线程，多个一个安全地执行给定任务线程方法，捕捉执行过程中抛出的异常。由于抽象的事件执行器是一个特殊的事件执行器组，内部事件执行器selfCollection（Collections.<EventExecutor>singleton(this)），是自己单例集，next方法返回的是自己。

## Netty 单线程事件执行器初始化
单线程事件执行器SingleThreadEventExecutor，内部主要有一个状态变量STATE_UPDATER（AtomicIntegerFieldUpdater），执行器状态以供有4中就绪，开始，正在关闭，已关闭，终止；一个任务队列taskQueue存放待执行的任务线程；一个执行器执行任务taskQueue(LinkedBlockingQueue)；一个事件执行器关闭信号量threadLock控制事件执行器的关闭；一个是高可见线程thread，指定当前事件执行器线程，用于判断IO操作线程是否在当前事件循环中；单线程事件执行器构造，主要是初始化父事件执行器，最大任务数，事件执行器，任务队列和任务拒绝策略，默认拒绝策略为直接抛出拒绝执行器异常。由于单线程事件执行器为顺序执行器OrderedEventExecutor，其主要通过taskQueue为LinkedBlockQueue保证任务的顺序执行。

## Netty 单线程事件执行器执行任务与graceful方式关闭
单线程事件执行器，执行任务，首先判断任务是否为null，为空抛出空指针异常，否则，判断线程是否在当前事件循环中，在则添加任务到任务队列，否则开启当前单线程事件执行器，并添加任务到任务队列，如果此时事件执行器已关闭，并可以移除任务，则抛出拒绝执行器任务异常；如果需要启动事件执行器唤醒线程，则添加唤醒线程到任务队列。添加，移除，poll任务操作，实际委托给任务队列，添加，移除hook线程操作委托给关闭hooks线程集合。单线程事件执行器take任务，首先从调度任务队列peek头部调度任务，如果任务不为空，则获取调度任务延时时间，如果延时时间大于0，则从任务队列超时poll任务，否则从调度任务队列抓取调度任务，添加到任务队列，并从任务队列poll任务；如果调度任务为空，则从任务队列take一个任务，如果是唤醒任务，则忽略。关闭单线程执行器，首先检查间隔、超时时间，时间单元参数，并且间隔时间要小于超时时间，如果已经关闭，则返回异步关闭任务结果，否则检查线程是否在当前事务循环中，如果是则更新状态为正在关闭，并计算计算关闭间隔和超时时间。

## Netty 单线程事件循环
单线程事件循环SingleThreadEventLoop，继承了单线程事件执行器，实现了事件循环接口，内部一个事件循环任务队列，我们可以把单线程事件循环看为一个简单的事件执行器，单线程事件循环中多了一个通道注册的方法，实际注册工作委托给通道关联的UnSafe。

## Netty nio事件循环初始化
## Netty nio事件循环后续
## Netty nio事件循环组
## Netty 抽象BootStrap定义
## Netty ServerBootStrap解析
## Netty Bootstrap解析
## Netty 通道接口定义
## Netty 抽象通道初始化  
## Netty 抽象Unsafe定义
## Netty 通道Outbound缓冲区
## Netty 抽象通道后续
## Netty 抽象nio通道
## Netty 抽象nio字节通道
## Netty 抽象nio消息通道  
## Netty NioServerSocketChannel解析
## Netty 默认通道配置初始化
## Netty 默认通道配置后续
## Netty NioSocketChannel解析
## Netty 字节buf定义
## Netty 资源泄漏探测器
## Netty 抽象字节buf解析
## Netty 抽象字节buf引用计数器
## Netty 复合buf概念   
## Netty 抽象字节buf分配器
## Netty Unpooled字节buf分配器
## Netty Pooled字节buf分配器  








<!-- demo -->
[Netty 网络通信示例一]:http://donald-draper.iteye.com/blog/2383326  "Netty 网络通信示例一"  
[Netty 网络通信示例二]:http://donald-draper.iteye.com/blog/2383328  "Netty 网络通信示例二"  
[Netty 网络通信示例三]:http://donald-draper.iteye.com/blog/2383392  "Netty 网络通信示例三"  
[Netty 网络通信示例四]:http://donald-draper.iteye.com/blog/2383472  "Netty 网络通信示例四"  
[Netty 构建HTTP服务器示例]:http://donald-draper.iteye.com/blog/2383527  "构建HTTP服务器示例"  
[Netty UDT网络通信示例]:http://donald-draper.iteye.com/blog/2383529  "Netty UDT网络通信示例"  
<!-- Channel Handler -->
[Netty 通道处理器ChannelHandler和适配器定义ChannelHandlerAdapter]:http://donald-draper.iteye.com/blog/2386891  "Netty 通道处理器ChannelHandler和适配器定义ChannelHandlerAdapter"  
[Netty Inbound/Outbound通道处理器定义]:http://donald-draper.iteye.com/blog/2387019  "Netty Inbound/Outbound通道处理器定义"  
[Netty 简单Inbound通道处理器（SimpleChannelInboundHandler）]:http://donald-draper.iteye.com/blog/2387772  "Netty 简单Inbound通道处理器（SimpleChannelInboundHandler）"  
[Netty 消息编码器-MessageToByteEncoder]:http://donald-draper.iteye.com/blog/2387832  "Netty 消息编码器-MessageToByteEncoder"  
[Netty 消息解码器-ByteToMessageDecoder]:http://donald-draper.iteye.com/blog/2388088  "Netty 消息解码器-ByteToMessageDecoder"  
<!-- Channel Pipeline -->
[Netty Inbound/Outbound通道Invoker]:http://donald-draper.iteye.com/blog/2388233  "Netty Inbound/Outbound通道Invoker"  
[Netty 异步任务-ChannelFuture]:http://donald-draper.iteye.com/blog/2388297  "Netty 异步任务-ChannelFuture"  
[Netty 管道线定义-ChannelPipeline]:http://donald-draper.iteye.com/blog/2388453  "Netty 管道线定义-ChannelPipeline"  
[Netty 默认Channel管道线初始化]:http://donald-draper.iteye.com/blog/2388613  "Netty 默认Channel管道线初始化"  
[Netty 默认Channel管道线-添加通道处理器]:http://donald-draper.iteye.com/blog/2389299  "Netty 默认Channel管道线-添加通道处理器"  
[Netty 默认Channel管道线-通道处理器移除与替换]:http://donald-draper.iteye.com/blog/2388793 "Netty 默认Channel管道线-通道处理器移除与替换"
[Netty 默认Channel管道线-Inbound和Outbound事件处理]:http://donald-draper.iteye.com/blog/2389148 "Netty 默认Channel管道线-Inbound和Outbound事件处理"

[Netty 通道处理器上下文定义]:http://donald-draper.iteye.com/blog/2389214 "Netty 通道处理器上下文定义"
[Netty 通道处理器上下文]:http://donald-draper.iteye.com/blog/2389299 "Netty 通道处理器上下文"
<!-- ChannelInitializer -->
[Netty 通道初始化器ChannelInitializer]:http://donald-draper.iteye.com/blog/2389352  "Netty 通道初始化器ChannelInitializer"  
<!-- EventLoopGroup -->
[Netty 事件执行器组和事件执行器定义及抽象实现]:http://donald-draper.iteye.com/blog/2391257  "Netty 事件执行器组和事件执行器定义及抽象实现"  
[Netty 多线程事件执行器组]:http://donald-draper.iteye.com/blog/2391270  "Netty 多线程事件执行器组"  
[Netty 多线程事件循环组]:http://donald-draper.iteye.com/blog/2391276  "Netty 多线程事件循环组"  
[Netty 抽象调度事件执行器]:http://donald-draper.iteye.com/blog/2391379  "Netty 抽象调度事件执行器"   
[Netty 单线程事件执行器初始化]:http://donald-draper.iteye.com/blog/2391895  "Netty 单线程事件执行器初始化"  
[Netty 单线程事件执行器执行任务与graceful方式关闭]:http://donald-draper.iteye.com/blog/2392051  "Netty 单线程事件执行器执行任务与graceful方式关闭"  
[Netty 单线程事件循环]:http://donald-draper.iteye.com/blog/2392067  "Netty 单线程事件循环"   
[Netty nio事件循环初始化]:http://donald-draper.iteye.com/blog/2392161  "Netty nio事件循环初始化"  
[Netty nio事件循环后续]:http://donald-draper.iteye.com/blog/2392264  "Netty nio事件循环后续"  
[Netty nio事件循环组]:http://donald-draper.iteye.com/blog/2392300  "Netty nio事件循环组"  
<!-- BootStrap -->
[Netty 抽象BootStrap定义]:http://donald-draper.iteye.com/blog/2392492  "Netty 抽象BootStrap定义"  
[Netty ServerBootStrap解析]:http://donald-draper.iteye.com/blog/2392572  "Netty ServerBootStrap解析"  
[Netty Bootstrap解析]:http://donald-draper.iteye.com/blog/2392593  "Netty Bootstrap解析"  
<!-- SocketChannel -->
[Netty 通道接口定义]:http://donald-draper.iteye.com/blog/2392740  "Netty 通道接口定义"  
[Netty 抽象通道初始化]:http://donald-draper.iteye.com/blog/2392801  "Netty 抽象通道初始化"  
[Netty 抽象Unsafe定义]:http://donald-draper.iteye.com/blog/2393053  "Netty 抽象Unsafe定义"  
[Netty 通道Outbound缓冲区]:http://donald-draper.iteye.com/blog/2393098  "Netty 通道Outbound缓冲区"  
[Netty 抽象通道后续]:http://donald-draper.iteye.com/blog/2393166  "Netty 抽象通道后续"  
[Netty 抽象nio通道]:http://donald-draper.iteye.com/blog/2393269  "Netty 抽象nio通道"  
[Netty 抽象nio字节通道]:http://donald-draper.iteye.com/blog/2393323  "Netty 抽象nio字节通道"  
[Netty 抽象nio消息通道]:http://donald-draper.iteye.com/blog/2393364  "Netty 抽象nio消息通道"    
[Netty NioServerSocketChannel解析]:http://donald-draper.iteye.com/blog/2393443  "Netty NioServerSocketChannel解析"  
[Netty 通道配置接口定义]:http://donald-draper.iteye.com/blog/2393484  "Netty 通道配置接口定义"  
[Netty 默认通道配置初始化]:http://donald-draper.iteye.com/blog/2393504  "Netty 默认通道配置初始化"  
[Netty 默认通道配置后续]:http://donald-draper.iteye.com/blog/2393510  "Netty 默认通道配置后续"  
[Netty NioSocketChannel解析]:http://donald-draper.iteye.com/blog/2394968  "Netty NioSocketChannel解析"  
<!-- ByteBuf -->
[Netty 字节buf定义]:http://donald-draper.iteye.com/blog/2393813  "Netty 字节buf定义"  
[Netty 资源泄漏探测器]:http://donald-draper.iteye.com/blog/2393940  "Netty 资源泄漏探测器"  
[Netty 抽象字节buf解析]:http://donald-draper.iteye.com/blog/2394078  "Netty 抽象字节buf解析"  
[Netty 抽象字节buf引用计数器]:http://donald-draper.iteye.com/blog/2394109  "Netty 抽象字节buf引用计数器"  
[Netty 复合buf概念]:http://donald-draper.iteye.com/blog/2394408  "Netty 复合buf概念"  
[Netty 抽象字节buf分配器]:http://donald-draper.iteye.com/blog/2394419  "Netty 抽象字节buf分配器"  
[Netty Unpooled字节buf分配器]:http://donald-draper.iteye.com/blog/2394619  "Netty Unpooled字节buf分配器"  
[Netty Pooled字节buf分配器]:http://donald-draper.iteye.com/blog/2394814  "Netty Pooled字节buf分配器"  
