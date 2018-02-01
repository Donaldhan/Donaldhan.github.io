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

Netty 是一个利用Java 的高级网络的能力，隐藏其背后的复杂性而提供一个易于使用的API的客户端/服务器框架。同时是一个广泛使用的Java网络编程框架（Netty 在 2011 年获得了Duke's Choice Award。它活跃和成长于用户社区，像大型公司 Facebook 和 Instagram 以及流行 开源项目如 Infinispan, HornetQ, Vert.x, Apache Cassandra 和 Elasticsearch 等，都利用其强大的对于网络抽象的核心代码。



![netty](/image/Netty/netty.png)




# 目录
* [](#)
* [](#)







<!-- demo -->
netty 网络通信示例一 ：[url]http://donald-draper.iteye.com/blog/2383326[/url]
netty 网络通信示例二：[url]http://donald-draper.iteye.com/blog/2383328[/url]
netty 网络通信示例三：[url]http://donald-draper.iteye.com/blog/2383392[/url]
netty 网络通信示例四：[url]http://donald-draper.iteye.com/blog/2383472[/url]
Netty 构建HTTP服务器示例：[url]http://donald-draper.iteye.com/blog/2383527[/url]
Netty UDT网络通信示例：[url]http://donald-draper.iteye.com/blog/2383529[/url]
<!-- Channel Handler -->
Netty 通道处理器ChannelHandler和适配器定义ChannelHandlerAdapter：[url]http://donald-draper.iteye.com/blog/2386891[/url]
Netty Inbound/Outbound通道处理器定义：[url]http://donald-draper.iteye.com/blog/2387019[/url]
netty 简单Inbound通道处理器（SimpleChannelInboundHandler）：[url]http://donald-draper.iteye.com/blog/2387772[/url]
netty 消息编码器-MessageToByteEncoder:[url]http://donald-draper.iteye.com/blog/2387832[/url]
netty 消息解码器-ByteToMessageDecoder:[url]http://donald-draper.iteye.com/blog/2388088[/url]
<!-- Channel Pipeline -->
netty Inboudn/Outbound通道Invoker:[url]http://donald-draper.iteye.com/blog/2388233[/url]
netty 异步任务-ChannelFuture：[url]http://donald-draper.iteye.com/blog/2388297[/url]
netty 管道线定义-ChannelPipeline：[url]http://donald-draper.iteye.com/blog/2388453[/url]
netty 默认Channel管道线初始化：[url]http://donald-draper.iteye.com/blog/2388613[/url]
netty 默认Channel管道线-添加通道处理器：[url]http://donald-draper.iteye.com/blog/2388726[/url]
netty 默认Channel管道线-通道处理器移除与替换：[url]http://donald-draper.iteye.com/blog/2388793[/url]
netty 默认Channel管道线-Inbound和Outbound事件处理：[url]http://donald-draper.iteye.com/blog/2389148[/url]
<!-- ChannelHandlerContext -->
netty 通道处理器上下文定义：[url]http://donald-draper.iteye.com/blog/2389214[/url]
netty 通道处理器上下文：[url]http://donald-draper.iteye.com/blog/2389299[/url]
<!-- ChannelInitializer -->
netty 通道初始化器ChannelInitializer:[url]http://donald-draper.iteye.com/blog/2389352[/url]
<!-- EventLoopGroup -->
netty 事件执行器组和事件执行器定义及抽象实现：[url]http://donald-draper.iteye.com/blog/2391257[/url]
netty 多线程事件执行器组：[url]http://donald-draper.iteye.com/blog/2391270[/url]
netty 多线程事件循环组：[url]http://donald-draper.iteye.com/blog/2391276[/url]
netty 抽象调度事件执行器：[url]http://donald-draper.iteye.com/blog/2391379[/url]
netty 单线程事件执行器初始化：[url]http://donald-draper.iteye.com/blog/2391895[/url]
netty 单线程事件执行器执行任务与graceful方式关闭：[url]http://donald-draper.iteye.com/blog/2392051[/url]
netty 单线程事件循环：[url]http://donald-draper.iteye.com/blog/2392067[/url]
netty nio事件循环初始化：[url]http://donald-draper.iteye.com/blog/2392161[/url]
netty nio事件循环后续：[url]http://donald-draper.iteye.com/blog/2392264[/url]
netty nio事件循环组：[url]http://donald-draper.iteye.com/blog/2392300[/url]
<!-- BootStrap -->
netty 抽象BootStrap定义：[url]http://donald-draper.iteye.com/blog/2392492[/url]
netty ServerBootStrap解析：[url]http://donald-draper.iteye.com/blog/2392572[/url]
netty Bootstrap解析：[url]http://donald-draper.iteye.com/blog/2392593[/url]
<!-- SocketChannel -->
netty 通道接口定义:[url]http://donald-draper.iteye.com/blog/2392740[/url]
netty 抽象通道初始化：[url]http://donald-draper.iteye.com/blog/2392801[/url]
netty 抽象Unsafe定义：[url]http://donald-draper.iteye.com/blog/2393053[/url]
netty 通道Outbound缓冲区：[url]http://donald-draper.iteye.com/blog/2393098[/url]
netty 抽象通道后续：[url]http://donald-draper.iteye.com/blog/2393166[/url]
netty 抽象nio通道：[url]http://donald-draper.iteye.com/blog/2393269[/url]
netty 抽象nio字节通道：[url]http://donald-draper.iteye.com/blog/2393323[/url]
netty 抽象nio消息通道：[url]http://donald-draper.iteye.com/blog/2393364[/url]
netty NioServerSocketChannel解析：[url]http://donald-draper.iteye.com/blog/2393443[/url]
netty 通道配置接口定义：[url]http://donald-draper.iteye.com/blog/2393484[/url]
netty 默认通道配置初始化：[url]http://donald-draper.iteye.com/blog/2393504[/url]
netty 默认通道配置后续：[url]http://donald-draper.iteye.com/blog/2393510[/url]
netty NioSocketChannel解析：[url]http://donald-draper.iteye.com/blog/2394968[/url]
<!-- ByteBuf -->
netty 字节buf定义：[url]http://donald-draper.iteye.com/blog/2393813[/url]
netty 资源泄漏探测器：[url]http://donald-draper.iteye.com/blog/2393940[/url]
netty 抽象字节buf解析：[url]http://donald-draper.iteye.com/blog/2394078[/url]
netty 抽象字节buf引用计数器：[url]http://donald-draper.iteye.com/blog/2394109[/url]
netty 复合buf概念：[url]http://donald-draper.iteye.com/blog/2394408[/url]
netty 抽象字节buf分配器：[url]http://donald-draper.iteye.com/blog/2394419[/url]
netty Unpooled字节buf分配器：[url]http://donald-draper.iteye.com/blog/2394619[/url]
netty Pooled字节buf分配器：[url]http://donald-draper.iteye.com/blog/2394814[/url]
