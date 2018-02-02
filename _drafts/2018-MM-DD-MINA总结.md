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

上面文章中的Mina源码为Mina的1.0分支。

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
过滤器链IoFilterChain用Entry存放过滤器对，即每个过滤器IoFilter关联一个后继过滤器NextFilter。我们可以通过滤器名name或过滤器实例ioFilter或过滤器类型获取相应的过滤器或过滤器对应的Entry。fireMessage*/exceptionCaught相关方法为触发IoHandler的相关事件,fireFilterWrite/Close触发的是，会话的相关事件IoSession#write/close。
DefaultIoFilterChainBuilder用entries列表（CopyOnWriteArrayList<DefaultIoFilterChainBuilder.EntryImpl>）来管理过滤器；添加过滤器，移除过滤器，及判断是否包含过滤器都是依赖于CopyOnWriteArrayList的相关功能。buildFilterChain方法是将默认过滤器链构建器的过滤器集合中的过滤器添加到指定的过滤链IoFilterChain上。DefaultIoFilterChainBuilder的过滤器EntryImpl中的getNextFilter并没有实际作用，即无效，这就说明了DefaultIoFilterChainBuilder只用于在创建会话时，构建过滤器链。创建完毕后，对过滤器链构建器的修改不会影响到会话实际的过滤器链IoFilterChain（SocketFilterChain,DatagramFilterChain...）。

## Mina 过滤器定义
IoFilter添加到过滤链时，一般以ReferenceCountingIoFilter包装，添加到过滤链，init方法在添加到过滤链时，由ReferenceCountingIoFilter调用，所以可以在init方法初始化一些共享资源。如果过滤器没有包装成ReferenceCountingIoFilter，init方法将不会调用。然后调用onPreAdd通知过滤器将会添加到过滤连上，当过滤器添加到过滤链上时，所有IoHandler事件和IO请求，将会被过滤器拦截当过滤器添加到过滤链上后，将会调用onPostAdd，如果方法有异常，过滤器在过滤链的链尾，ReferenceCountingIoFilter将调用#destroy方法释放共享资源。过滤器IoFilter，主要是监听会话IoSession相关事件（创建，打开，空闲，异常，关闭，接受数据，发送数据） 及IoSesssion的Write与close事件；过滤器后继NextFilter关注的事件与过滤器IoFilter相同，主要是转发相关事件。WriteRequest是会话IoSession写操作write的包装，内部有一个消息对象用于存放write的内容，一个socket地址，即会话写请求的目的socket地址，一个写请求结果返回值WriteFuture，用于获取会话write消息的操作结果。当一个过滤器从过滤链上移除时，#onPreRemove被调用，用于通知过滤器将从过滤链上移除；如果过滤器从过滤链上移除，所有IoHandler事件和IO请求，过滤器不再拦截；#onPostRemove调用通知过滤器已经从过滤链上移除；移除后，如果过滤器在过滤链的链尾，ReferenceCountingIoFilter将调用#destroy方法释放共享资源。

IoFilter生命周期如下：
inti->onPreAdd->onPostAdd->(拦截IoHandler相关事件：sessionCreated，Opened，Idle，exceptionCaught，Closed，messageSent，messageReceived;会话相关事件：filterWrite，filterClose)->onPreRemove->onPostRemove->destroy。

## Mina 日志过滤器与引用计数过滤器
LoggingFilter拦截IoHandler的sessionCreated事件，将日志输出委托给SessionLog，然后将相关事件传递给过滤器后继，拦截sessionOpened，sessionClosed，
filterClose事件思路基本相同。对于sessionIdle，messageSent，messageReceived，filterWrite先判断会话log的info级别是否开启，开启则输出相应事件日志。上面这些事件的日志级别都是Info；exceptionCaught则先判断会话log的warn级别是否开启，开启则输出相应事件日志。

一个过滤器可以多次添加到过滤链上，ReferenceCountingIoFilter用于保证过滤器第一次添加到过滤链上时，初始化过滤器，完全从过滤链上移除时，销毁过滤器，释放资；ReferenceCountingIoFilter内部一个count用于记录包装过滤器filter添加到过滤链上的次数，即过滤链上存在filter的个数，一个filter，即ReferenceCountingIoFilter包装的过滤器，ReferenceCountingIoFilter触发IoHandler和IoSession的相关事件，直接交给包装的内部filter处理。

## Mina 过滤链抽象实现
AbstractIoFilterChain内部关联一个IoSession，用EntryImp来包装过滤器，过滤链中用HashMap<String,EntryImpl>来存放过滤器Entry,key为过滤器名，value为过滤器Entry。EntryImpl是过滤器在过滤链上存在的形式，EntryImpl有一个前驱和一个后继，内部包裹一个过滤器 with name，及过滤器的后继过滤器NextFilter。后继过滤器NextFilter的传递IoHandler和IoSession事件的方法，主要是将事件转发给后继Entry对应的过滤器。过滤链头为HeadFilter，链尾为TailFilter。HeadFilter触发IoHandler和IoSession事件时，将事件传递给后继过滤器；但对于IoSession write/close事件除了传递事件外，需要调用实际的事件操作doWrite/doClose，这两个方法需要子类扩展实现。TailFilter触发IoHandler和IoSession事件时，直接调用会话处理器IoHandler的相关事件方法。在sessionOpened事件中，最后如果是SocketConnector创建的会话，则要通知相关ConnectFuture；在sessionClosed事件中，最后还要清空过滤链；messageSent和messageReceived事件，如果消息对象为ByteBuffer，则释放buffer。添加过滤器到过滤链，首先检查过滤链上是否存在过滤器，不存在，才添加；添加过滤器到头部即，插入过滤器到链头的后面，添加过滤器到尾部，即插入过滤器到链尾的前面；添加到指定过滤器前后，思路基本相同；添加前触发过滤器onPreAdd事件，添加后触发过滤器onPostAdd事件;移除过滤器，首先获取过滤器对应的Entry，然后触发过滤器onPreRemove事件，从过滤链name2entry移除Entry，然后触发过滤器onPostRemove事件。过滤链处理相关事件策略为：与IoHanler的相关事件(Session*)处理的顺序为，从链头到链尾-》Iohanlder（这个过程handler处理相关事件）；对于会话相关的事件（FilterWrite/close）,处理顺序为Iohanlder-》从链尾到链头（这是会话事件，只是在handler的方法中使用会话发送消息，关闭会话，handler并不处理会话事件）。

## Mina Socket与报文过滤链
SocketFilterChain发送消息首先获取Socket会话的的写请求队列；mark buffer的位置，主要因为buffer要传给messageSent事件，待消息发送完成，SocketIoProcessor.doFlush方法将会reset buffer到当前mark的位置；根据buffer的实际数据容量来判断是更新调度请求计数器还是更新调度写字节计数器；将写请求添加到session写请求队列中，如果session运行写操作，获取session关联的IoProcessor完成实际的消息发送工作。关闭session，即将会话添加到关联的IoProcessor待移除会话队列。DatagramFilterChain发送消息首先获取报文会话的的写请求队列；mark buffer的位置，主要因为buffer要传给messageSent事件，待消息发送完成，SocketIoProcessor.doFlush方法将会reset buffer到当前mark的位置；根据buffer的实际数据容量来判断是更新调度请求计数器还是更新调度写字节计数器；将写请求添加到session写请求队列中，如果session允许写操作，获取session关联的managerDelegate(DatagramService)完成实际的消息发送工作。
关闭会话委托给session关联的managerDelegate(DatagramService)，如果managerDelegate为DatagramConnectorDelegate者直接关闭，如果为DatagramAcceptorDelegate，通知DatagramAcceptorDelegate的监听器会话已关闭，设置会话CloseFuture为已关闭状态。

## Mina 协议编解码过滤器一（协议编解码工厂、协议编码器）
协议编解码过滤器ProtocolCodecFilter关联一个协议编解码工厂ProtocolCodecFactory，协议编解码工厂提供协议编码器ProtocolEncoder和解码器ProtocolDecoder；编码器将消息对象编码成二进制或协议数据，解码器将二进制或协议数据解码成消息对象。编码器ProtocolEncoder主要有两个方法，encode和dispose；encode用于，编码上层的消息对象为二进制或协议数据。mina调用编码器的#encode方法将从会话写请求队列中pop的消息，然后调用ProtocolEncoderOutput#write方法将编码后的消息放在ByteBuffer；dispose方法释放编码器资源。ProtocolEncoderAdapter为编码器抽象实现，默认实现了dispose，不做任何事情，对于不需要释放资源的编码器继承ProtocolEncoderAdapter。ProtocolEncoderOutput主要的工作是将协议编码器编码后的字节buffer，缓存起来，等待flush方法调用时，则将数据发送出去。SimpleProtocolEncoderOutput为ProtocolEncoderOutput的简单实现内部有一个buffer队列bufferQueue（Queue），用于存放write（ByteBuffer）方法，传入的字节buffer；mergeAll方法为合并buffer队列的所有buffer数据到一个buffer；flush方法为发送buffer队列中的所有buffer，实际发送工作委托给doFlush方法待子类实现。ProtocolEncoderOutputImpl为协议编解码过滤器的内部类，ProtocolEncoderOutputImpl的doFlush，首先将会话包装成DefaultWriteFuture，将会话，写请求信息传递给NextFilter。

## Mina 协议编解码过滤器二（协议解码器）
解码器ProtocolDecoder将二级制或协议内存解码成上层消息对象。mina在读取数据时，调用解码器的#decode，将解码后的消息放到ProtocolDecoderOutput；当会话关闭时，调用finishDecode解码那些在#decode方法中没有处理完的数据。dispose主要是 释放所有与解码器有关的资源。针对不需要#finishDecode和#dispose的解码器，我们可以继承协议解码适配器ProtocolDecoderAdapter。ProtocolDecoderOutput有两个方法，一个write方法，用于解码器，解完消息后回调；一个flush方法，用于刷新所有解码器写到协议解码输出的消息对象。简单协议解码输出SimpleProtocolDecoderOutput，关联一个会话session，一个后继过滤器nextFilter，及一个消息队列messageQueue；所有解码器解码后的消息，回调协议解码输出的write方法，将消息暂时放在消息队列messageQueue中；flush方法主要是将消息队列中的消息传给后继过滤器的messageReceived方法。

## Mina 队列Queue
队列Queue实际上为List，用一个对象数组items存放元素，一个实际容量计数器size，一个队头first和一个队尾索引last，默认队列容量为4。push元素时，先判断队列是否已满，如果已满，则扩容队列为原来的两倍。Queue实际为具有队列特性的List。Queue可以随机地访问队列索引。

## Mina 协议编解码过滤器三（会话write与消息接收过滤）
协议编解码过滤器构造，主要是初始化协议编码器工厂。一个过滤链上不能存在两个协议编解码器，即唯一。会话写操作来看session#write(filterWrite),首先从写请求获取消息，如果消息为字节buffer，则直接传给后继过滤器，否则从协议编解码工厂获取协议编码器和协议编码器输出，协议编码器encode编码消息，写到协议编码器输出字节buffer队列，
然后协议编码器输出flush字节buffer队列。会话接收消息messageReceived，如果消息非字节buffer，则直接传给后继过滤器，否则获取协议解码器，及协议解码器输出，协议解码器解码字节buffer为上传消息对象，写到协议解码器输出消息队列，最后解码器输出flush消息队列。会话关闭，主要是解码器解码会话未解码数据，写到解码器输出消息队列，最后从会话移除编码器，解码器及解码器输出属性，释放编码器，解码器及解码器输出相关资源，flush解码器输出消息队列。协议编解码器过滤器从过滤链移除后，要释放会话编解码器，及解码输出属性，并释放相关的资源。

## Mina 累计协议解码器
累积性协议解码器decode方法，首先从会话获取属性BUFFER对应的buffer，如果会话缓存buffer不为null，则将当前in（ByteBuffer）的内容放到会话缓存buffer中；否直接使用当前in（ByteBuffer），将是否开启会话缓存buffer，置为否；循环尝试使用doDecode方法解码数据，直到doDecode方法返回false，如果doDecode方法返回true，则继续调用doDecode方法解码数据，如果在一次解码数据返回true，但buffer中还有数据，则根据是否开启会话缓存buffer决定将buffer数据进行压缩还是存储在会话中；如果在一次解码数据返回true，但buffer中没有数据，则从会话中移除属性BUFFER，并释放对应的buffer空间。doDecode方法放抽象方法待类型实现。CumulativeProtocolDecoder实现可累性方法主要是通过将buffer存储在会话中，以实现接收数据的可累计性

## MINA 多路复用协议编解码器工厂一（多路复用协议编码器）
多路复用的协议编码器添加消息编码器，如果参数为消息类型和消息编码器类型，添加消息类型与默认构造消息编码工厂到消息编码器Map映射type2encoderFactory；如果参数为消息类型和消息编码器实例，添加消息类型与单例消息编码器工厂到消息编码器Map映射type2encoderFactory。多路复用的协议编码器编码消息，首先从会话获取消息编码器状态，从消息编码器状态获取消息对应的编码器（先从消息编码器查找，没有则从消息编码器映射type2encoder查找），如果没有消息对应的解码，则查找消息父接口和类对应的编码器，编码消息。

## MINA 多路复用协议编解码器工厂二（多路复用协议解码器）
消息解码器MessageDecoder，有3个方法分别为，decodable方法用于判断解码器是否可以解码buffer数据，返回值（MessageDecoderResult）#OK表示可以解码buffer中的数据，#NOT_OK表示解码器不可解码buffer数据， #NEED_DATA表示需要更多的数据来确认解码器是否可以解码buffer数据；decode用于解码buffer数据，返回值#OK表示解码消息成功，#NEED_DATA，表示需要更多的数据完成消息解码，#NOT_OK由于内容与协议不一致，不能解码当前消息；finishDecode主要是处理#decode方法没有解码完的数据。添加解码器到多路复用协议解码器，如果添加的解码器为Class，则添加默认构造消息解码器工厂到到多路复用协议解码器工厂集decoderFactories；如果添加的解码器为实例，则添加单例解码器工厂，到多路复用协议解码器工厂集。多路复用协议解码器，解码消息的过程为，首先从会话获取解码器状态集，遍历解码器集，找到可以解码消息的解码器，并赋给会话解码器状态当前解码器currentDecoder，最后由currentDecoder解码消息。
  多路复用协议编解码器工厂内部关联一个多路复用协议编码器和解码器器，添加消息编码器和解码器到多路复用协议编解码器工厂，实际是添加多路复用协议编解码器中。
  
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
