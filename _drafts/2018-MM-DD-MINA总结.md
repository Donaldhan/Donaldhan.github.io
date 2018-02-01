---
layout: page
title: Mina总结
subtitle: MINA总结
date: 2018-11-04 15:17:19
author: donaldhan
catalog: true
category: Mina
categories:
    - Mina
tags:
    - Mina
---

# 引言

Mina是一个网络通信应用框架，是基于TCP/IP、UDP/IP协议栈的通信框架（当然，也可以提供JAVA 对象的序列化服务、虚拟机管道通信服务等），Mina 可以帮助我们快速开发高性能、高扩展性的网络通信应用，Mina提供了事件驱动、异步（Mina的异步IO，默认使用的是JAVA NIO 作为底层支持）操作的编程模型。Mina的相关组件有IoService，IoProcessor,
IoFilter,IoHanler,IoBuffer。

* [MINA TCP简单通信实例][]
* [MINA 编解码器实例][]
* [MINA 多路分离解码器实例][]
* [Mina Socket会话配置][]
* [Mina 过滤链默认构建器][]
* [Mina 过滤器定义][]
* [Mina 日志过滤器与引用计数过滤器][]
* [Mina 过滤链抽象实现][]
* [Mina Socket与报文过滤链][]
* [Mina 协议编解码过滤器一（协议编解码工厂、协议编码器）][]
* [Mina 协议编解码过滤器二（协议解码器）][]
* [Mina 队列Queue][]
* [Mina 协议编解码过滤器三（会话write与消息接收过滤）][]
* [Mina 累计协议解码器][]
* [MINA 多路复用协议编解码器工厂一（多路复用协议编码器）][]
* [MINA 多路复用协议编解码器工厂二（多路复用协议解码器）][]
* [Mina IoHandler接口定义][]
* [Mina Io处理器抽象实现][]
* [Mina Nio处理器][]
* [Mina Io会话接口定义][]
* [Mina 抽象Io会话][]
* [Mina Nio会话（Socket，DataGram）][]
* [Mina IoService接口定义及抽象实现][]
* [Mina Io监听器接口定义及抽象实现][]
* [Mina 抽象polling监听器][]
* [Mina socket监听器（NioSocketAcceptor）][]
* [Mina 连接器接口定义及抽象实现（IoConnector ）][]
* [Mina 抽象Polling连接器（AbstractPollingIoConnector）][]
* [Mina socket连接器（NioSocketConnector）][]
* [Mina 报文通信简单示例][]
* [Mina 报文监听器NioDatagramAcceptor一（初始化，Io处理器）][]
* [Mina 报文监听器NioDatagramAcceptor二（发送会话消息据等）][]
* [Mina 报文连接器（NioDatagramConnector）][Mina 报文连接器（NioDatagramConnector）]



![mina](/image/Mina/mina.png)



# 目录

* [Mina Socket会话配置](#Mina Socket会话配置)
* [Mina 过滤链默认构建器](#Mina 过滤链默认构建器)
* [Mina 过滤器定义](#Mina 过滤器定义)
* [Mina 日志过滤器与引用计数过滤器](#Mina 日志过滤器与引用计数过滤器)
* [Mina 过滤链抽象实现](#Mina 过滤链抽象实现)
* [Mina Socket与报文过滤链](#Mina Socket与报文过滤链)
* [Mina 协议编解码过滤器一（协议编解码工厂、协议编码器）](#Mina 协议编解码过滤器一（协议编解码工厂、协议编码器）)
* [Mina 协议编解码过滤器二（协议解码器）](#Mina 协议编解码过滤器二（协议解码器）)
* [Mina 队列Queue](#Mina 队列Queue)
* [Mina 协议编解码过滤器三（会话write与消息接收过滤）](#)
* [Mina 累计协议解码器](#Mina 累计协议解码器)
* [MINA 多路复用协议编解码器工厂一（多路复用协议编码器）](#MINA 多路复用协议编解码器工厂一（多路复用协议编码器）)
* [MINA 多路复用协议编解码器工厂二（多路复用协议解码器）](#MINA 多路复用协议编解码器工厂二（多路复用协议解码器）)
* [Mina IoHandler接口定义](#Mina IoHandler接口定义)
* [Mina Io处理器抽象实现](#Mina Io处理器抽象实现)
* [Mina Nio处理器](#Mina Nio处理器)
* [Mina Io会话接口定义](#Mina Io会话接口定义)
* [Mina 抽象Io会话](#Mina 抽象Io会话)
* [Mina Nio会话（Socket，DataGram）](#Mina Nio会话（Socket，DataGram）)
* [Mina IoService接口定义及抽象实现](#Mina IoService接口定义及抽象实现)
* [Mina Io监听器接口定义及抽象实现](#Mina Io监听器接口定义及抽象实现)
* [Mina 抽象polling监听器](#Mina 抽象polling监听器)
* [Mina socket监听器（NioSocketAcceptor）](#Mina socket监听器（NioSocketAcceptor）)
* [Mina 连接器接口定义及抽象实现（IoConnector ）](#Mina 连接器接口定义及抽象实现（IoConnector ）)
* [Mina 抽象Polling连接器（AbstractPollingIoConnector）](#Mina 抽象Polling连接器（AbstractPollingIoConnector）)
* [Mina socket连接器（NioSocketConnector）](#Mina socket连接器（NioSocketConnector）)
* [Mina 报文通信简单示例](#Mina 报文通信简单示例)
* [Mina 报文监听器NioDatagramAcceptor一（初始化，Io处理器）](#Mina 报文监听器NioDatagramAcceptor一（初始化，Io处理器）)
* [Mina 报文监听器NioDatagramAcceptor二（发送会话消息据等）](#Mina 报文监听器NioDatagramAcceptor二（发送会话消息据等）)
* [Mina 报文连接器（NioDatagramConnector）](#Mina 报文连接器（NioDatagramConnector）)




## Mina Socket会话配置
NioSocketAcceptor构造是传输入的DefaultSocketSessionConfig参数实际上是初始化AbstractIoService的会话配置选项sessionConfig（IoSessionConfig）。会话配置IoSessionConfig主要是配置IoProcessor每次读操作分配的buffer的容量，一般情况下，不用手动设置这个属性，因为IoProcessor经常会自动调整；设置空闲状态IdleStatus（READER_IDLE,WRITER_IDLE 或 BOTH_IDLE）的空闲时间；UseReadOperation配置项用于优化客户端的读取操作，开启这个选项对服务器应用无效，并可能引起内存泄漏，因此默认为关闭状态。SocketSessionConfig主要是配置Socket，这些属性在java.net.Socket中都可以找到相似或类似的属性，比如发送缓冲区大小，是否保活，地址是否可重用等。从AbstractSocketSessionConfig来看SocketSessionConfig的发送、接收缓冲区大小，是否保活，地址是否可重用等配置默认为true。DefaultSocketSessionConfig构造时，发送和接受缓存区默认为-1，所以在创建NioSocketAcceptor要配置发送和接收缓存区大小；
DefaultSocketSessionConfig关联一个parent父对象IoService，即配置的依附对象。#init方法初始化的parent父对象IoService，如果parent父对象为IoService（SocketAcceptor），
则地址默认可重用，否则不可。

## Mina 过滤链默认构建器
## Mina 过滤器定义
## Mina 日志过滤器与引用计数过滤器
## Mina 过滤链抽象实现
## Mina Socket与报文过滤链
## Mina 协议编解码过滤器一（协议编解码工厂、协议编码器）
## Mina 协议编解码过滤器二（协议解码器）
## Mina 队列Queue
## Mina 协议编解码过滤器三（会话write与消息接收过滤）
## Mina 累计协议解码器
## MINA 多路复用协议编解码器工厂一（多路复用协议编码器）
## MINA 多路复用协议编解码器工厂二（多路复用协议解码器）
## Mina IoHandler接口定义
## Mina Io处理器抽象实现
## Mina Nio处理器
## Mina Io会话接口定义
## Mina 抽象Io会话
## Mina Nio会话（Socket，DataGram）
## Mina IoService接口定义及抽象实现
## Mina Io监听器接口定义及抽象实现
## Mina 抽象polling监听器
## Mina socket监听器（NioSocketAcceptor）
## Mina 连接器接口定义及抽象实现（IoConnector ）
## Mina 抽象Polling连接器（AbstractPollingIoConnector）
## Mina socket连接器（NioSocketConnector）
## Mina 报文通信简单示例
## Mina 报文监听器NioDatagramAcceptor一（初始化，Io处理器）
## Mina 报文监听器NioDatagramAcceptor二（发送会话消息据等）
## Mina 报文连接器（NioDatagramConnector）

















[MINA TCP简单通信实例]:http://donald-draper.iteye.com/blog/2375297  "MINA TCP简单通信实例"
[MINA 编解码器实例]:http://donald-draper.iteye.com/blog/2375317  "MINA 编解码器实例"
[MINA 多路分离解码器实例]:http://donald-draper.iteye.com/blog/2375324  "MINA 多路分离解码器实例"

[Mina Socket会话配置]:http://donald-draper.iteye.com/blog/2375529  "Mina Socket会话配置"
[Mina 过滤链默认构建器]:http://donald-draper.iteye.com/blog/2375985  "Mina 过滤链默认构建器"
[Mina 过滤器定义]:http://donald-draper.iteye.com/blog/2376161  "Mina 过滤器定义"
[Mina 日志过滤器与引用计数过滤器]:http://donald-draper.iteye.com/blog/2376226  "Mina 日志过滤器与引用计数过滤器"
[Mina 过滤链抽象实现]:http://donald-draper.iteye.com/blog/2376335  "Mina 过滤链抽象实现"
[Mina Socket与报文过滤链]:http://donald-draper.iteye.com/blog/2376440  "Mina Socket与报文过滤链"
[Mina 协议编解码过滤器一（协议编解码工厂、协议编码器）]:http://donald-draper.iteye.com/blog/2376663  "Mina 协议编解码过滤器一（协议编解码工厂、协议编码器）"
[Mina 协议编解码过滤器二（协议解码器）]:http://donald-draper.iteye.com/blog/2376679  "Mina 协议编解码过滤器二（协议解码器）"
[Mina 队列Queue]:http://donald-draper.iteye.com/blog/2376712  "Mina 队列Queue"
[Mina 协议编解码过滤器三（会话write与消息接收过滤）]:http://donald-draper.iteye.com/blog/2376818  "Mina 协议编解码过滤器三（会话write与消息接收过滤"
[Mina 累计协议解码器]:http://donald-draper.iteye.com/blog/2377029  "Mina 累计协议解码器"
[MINA 多路复用协议编解码器工厂一（多路复用协议编码器）]:http://donald-draper.iteye.com/blog/2377170  "MINA 多路复用协议编解码器工厂一（多路复用协议编码器）"
[MINA 多路复用协议编解码器工厂二（多路复用协议解码器）]:http://donald-draper.iteye.com/blog/2377324  "MINA 多路复用协议编解码器工厂二（多路复用协议解码器）"
[Mina IoHandler接口定义]:http://donald-draper.iteye.com/blog/2377419  "Mina IoHandler接口定义"

[Mina Io处理器抽象实现]:http://donald-draper.iteye.com/blog/2377663  "Mina Io处理器抽象实现"
[Mina Nio处理器]:http://donald-draper.iteye.com/blog/2377725  "Mina Nio处理器"


[Mina Io会话接口定义]:http://donald-draper.iteye.com/blog/2377737  "Mina Io会话接口定义"
[Mina 抽象Io会话]:http://donald-draper.iteye.com/blog/2377880  "Mina 抽象Io会话"
[Mina Nio会话（Socket，DataGram）]:http://donald-draper.iteye.com/blog/2378169  "Mina Nio会话（Socket，DataGram）"


[Mina IoService接口定义及抽象实现]:http://donald-draper.iteye.com/blog/2378271  "Mina IoService接口定义及抽象实现"
[Mina Io监听器接口定义及抽象实现]:http://donald-draper.iteye.com/blog/2378315  "Mina Io监听器接口定义及抽象实现"
[Mina 抽象polling监听器]:http://donald-draper.iteye.com/blog/2378649  "Mina 抽象polling监听器"
[Mina socket监听器（NioSocketAcceptor）]:http://donald-draper.iteye.com/blog/2378668  "Mina socket监听器（NioSocketAcceptor）"


[Mina 连接器接口定义及抽象实现（IoConnector ）]:http://donald-draper.iteye.com/blog/2378936  "Mina 连接器接口定义及抽象实现（IoConnector ）"
[Mina 抽象Polling连接器（AbstractPollingIoConnector）]:http://donald-draper.iteye.com/blog/2378978  "Mina 抽象Polling连接器（AbstractPollingIoConnector）"
[Mina socket连接器（NioSocketConnector）]:http://donald-draper.iteye.com/blog/2379000  "Mina socket连接器（NioSocketConnector）"

[Mina 报文通信简单示例]:http://donald-draper.iteye.com/blog/2379002  "Mina 报文通信简单示例"
[Mina 报文监听器NioDatagramAcceptor一（初始化，Io处理器）]:http://donald-draper.iteye.com/blog/2379152  "Mina 报文监听器NioDatagramAcceptor一（初始化，Io处理器）"
[Mina 报文监听器NioDatagramAcceptor二（发送会话消息据等）]:http://donald-draper.iteye.com/blog/2379228  "Mina 报文监听器NioDatagramAcceptor二（发送会话消息据等）"
[Mina 报文连接器（NioDatagramConnector）]:http://donald-draper.iteye.com/blog/2379292 "Mina 报文连接器（NioDatagramConnector）"
