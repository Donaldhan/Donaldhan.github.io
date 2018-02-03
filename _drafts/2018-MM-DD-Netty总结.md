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

## Netty 消息编码器-MessageToByteEncoder
## Netty 消息解码器-ByteToMessageDecoder
## Netty Inbound/Outbound通道Invoker
## Netty 异步任务-ChannelFuture
## Netty 管道线定义-ChannelPipeline
## Netty 默认Channel管道线初始化
## Netty 默认Channel管道线-添加通道处理器
## Netty 通道初始化器ChannelInitializer
## Netty 事件执行器组和事件执行器定义及抽象实现  
## Netty 多线程事件执行器组
## Netty 多线程事件循环组
## Netty 抽象调度事件执行器
## Netty 单线程事件执行器初始化
## Netty 单线程事件执行器执行任务与graceful方式关闭
## Netty 单线程事件循环
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
