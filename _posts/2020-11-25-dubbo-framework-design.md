---
layout: page
title: Dubbo框架设计及源码解读
subtitle: Dubbo框架设计及源码解读
date: 2020-11-23 23:06:00
author: valuewithTime
catalog: true
category: Dubbo
categories:
    - Dubbo
tags:
    - Dubbo
---

# 引言
随着互联网应用体量不断的增加，为了对应大量用户访问的体验效果，及提供应用的可用性，微服务应运而生，微服务使应用的业务具有更高的内聚性，同时对整体应用进行服务化的解耦；微服务之间除了通过异步的MQ进行通信，常用的还有基于同步的RPC通知机制，比如基于HTTP的GRPC, BRPC及在国内使用比较广泛的[Dubbo][]。一致在使用DUBOO，对其中的功能组件有大致的了解，本计划看一下源码的，一致没有抽出时间，最近终于忙里抽闲，一探Dubbo芳容。

[Dubbo]: https://github.com/apache/dubbo "Dubbo"



# 目录
* [概要框架设计](#概要框架设计)
* [源码分析](#源码分析)
    * [应用协议](#应用协议)
        * [注册器协议](#注册器协议) 
            * [服务导出Exporter](#服务导出exporter) 
            * [服务注册](#注册器协议) 
            * [订阅服务](#订阅服务)
            * [服务Invoker](#服务invoker) 
        * [Dubbo协议](#dubbo协议) 
    * [数据传输器Transport](#数据传输器transport)
        * [服务端](#服务端) 
        * [客户端](#客户端) 
    * [消息编解码](#消息编解码)
        * [消息编码器](#消息编码器)
        * [消息解码器](#消息解码器)
* [总结](#总结)
* [附](#附)


# 概要框架设计


![dubbo-framework](/image/dubbo/dubbo-framework.png)  


dubbo框架主要包括序列化，消息层，传输层，协议层。序列化主要是请求消息和响应消息的序列化，比如基于Javad的ObjectOut/InputStream序列化、基于JSON的序列化。消息层提供消费者调用服务请求消息、服务提供方处理
结果响应消息的编解码；传输层主要建立消费者和服务者的通信通道，传输服务请求响应数据，比如基于Netty和Mina的，默认为Netty；协议层首先是基于相关协议将服提供者，和消费者通过export暴露出去，即注册器Registry中，消费者通过Registry订阅响应的服务提供者，消费者发现有服务
提供者，则与服务提供者建立连接，注册协议有基于Zookeeper，Redis等，在注册协议中还有一个注册器目录服务，用于提供消费者和服务者列表，及根据负载均衡策略选择服务者。服务提供者接受的消费者的服务请求后，根据相关协议，调用相应的Invoker服务。 消费者和服务者的RPC调用协议，实际在DubboProtocol中，协议首先导出服务，消费者发送RPC请求，调用Exporter服务容器中的Invoker。

# 源码分析

[Dubbo框架设计源码解读第一篇（服务和引用bean初始化）](https://donaldhan.github.io/dubbo/2020/11/17/dubbo-framework-service-reference-bean-init.html)

dubbo框架主要包括消息层，传输层，协议层。消息层提供消费者调用服务请求消息、服务提供方处理
结果响应消息的编解码；传输层主要建立消费者和服务者的通信通道，传输服务请求响应数据；协议层首先是基于相关协议将服
提供者，和消费者通过export暴露出去，即注册的Registry中，消费者通过Registry订阅响应的服务提供者，消费者发现有服务
提供者，则与服务提供者建立连接。服务提供者接受的消费者的服务请求后，根据相关协议，调用响应的Invoker服务。


ServiceAnnotationBeanPostProcessor主要做的事情是扫描 应用先的Service注解bean，并构造ServiceBean，注册到bean注册器中。导出服务实际委托给相应的协议RegistryProtocol


ReferenceBean后处理主要扫描Reference注解的bean，并构造ReferenceBean，ReferenceBean通过Invoker去调用服务提供者的服务，Invoker为服务的包装类。实际通过相应的协议创建。

## 应用协议

### 注册器协议

[Dubbo框架设计源码解读二（注册器，服务注册，订阅）](https://donaldhan.github.io/dubbo/2020/11/19/dubbo-framework-application-registry-protocol.html)


注册协议，导出服务主要有注册服务和订阅服务。注册服务，实际是将基于Dubbo协议的服务URL写到ZK上，如何在注册的过程中，由于Dubbo自身机制导致的注册失败，将加入的失败注册集，并有定时钟，进行重试注册。订阅服务，监听服务提供者的节点路径。消费者注册到ZK上的订阅服务节点上，具体的订阅委托给目录服务。

注册目录服务依赖于注册器，消费者从注册器获取服务提供者，实际为从注册目录服务获取服务列表（zk注册器为，服务节点下的提供者），并根据路由策略，选择可用的服务提供者Invoker。注册器目录处理提供服务路由，同时监听服务的变化。如果注册器节点信息存在变化，则重新刷新服务，建立服务Invoker索引。

### Dubbo协议

[Dubbo框架设计源码解读三（Dubbo协议，服务导出，引用）](https://donaldhan.github.io/dubbo/2020/11/19/dubbo-framework-dubbo-protocol.html)  

Dubbo协议是消费者和服务者通信的基础，包括服务的调用。注册器协议中，有如下一个功能，导出服务到本地， 实际委托的相应的协议，比如Dubbo协议DubboProtocol的export操作。注册器目录，当前监听注册器节点变化是，重新索引服务，在转换URL为Invoker，实际委托的相应的协议，比如Dubbo协议DubboProtocol的 refer操作。 dubbo协议的导出服务，实际上创建一个服务Server，根据dubbo协议配置，可为NettyServer，或MinaServer。默认为NettyServer。

## 数据传输器Transport

[Dubbo框架设计源码解读四(Dubbo基于Netty的传输器Transport)](https://donaldhan.github.io/dubbo/2020/11/23/dubbo-framework-netty-server-client.html)  

Netty服务端是基于经典的bootstrap，事件，worker, 编解码器，消息处理器的实现。netty处理器实际为一个共享的SimpleChannelHandler， 所有操作委托内部通道处理器DecodeHandler。DecodeHandler在接受消息后，解码相应的消息，将交由内部的HeaderExchangeHandler处理。HeaderExchangeHandler， 所有的操作委托给内部的ExchangeHander，实际为DubboProtocol的中ExchangeHandleAdater。

消费者调用服务提供者实际上发送的一个Invocation消息，服务端接受到消息，根据Invoker上下文，从Dubbo协议的Exporter容器中获取对应的Invoker，调用相关服务，将调用结果返回给消费者。

netty客户端也没有多少新鲜的动心，编解码器，Netty客户端处理器NettyClientHandler。NettyClientHandler实际为一个共享的ChannelDuplexHandler，所有操作委托内部通道处理器DecodeHandler。DecodeHandler在接受消息后，解码相应的消息，将交由内部的HeaderExchangeHandler处理。HeanderExchangeHandler，HeanderExchangeHandler所有的操作委托给内部的ExchangeHander，实际为DubboProtocol的中ExchangeHandleAdater。一个请求，主要包括请求Id，版本，及数据及Invocation。服务响应，主要包含消息id，消息版本，状态，相应结果，及错误信息，如果有的话。


## 消息编解码



NettyCodecAdapter为编解码器的适配，内部编码器实际为ByteToMessageDecoder， 内部的编码操作委托给内部编解码器，根据SPI机制，实际为DubboCodec；内部解码器实际为ByteToMessageDecoder，解码操作委托给DubboCodec。


编码器编码主要是针对请求Request和响应Response，编码首先写头部，针对请求主要有魔数、序列化标志，请求id，然后是请求数据,主要有RpcInvocation的服务URL，版本信息，服务方法，方法参数类型，方法参数类型通过ReflectUtils进行编码，最后还有方法参数；针对响应，首先写头部，魔数，请求id，然后是响应数据。需要注意在编码请求和响应的时候，有一部分是Attament，这部分是可以扩展的地方。序列化器，有FastJsonSerialization，JavaSerialization, 默认为JavaSerialization。JavaSerialization的序列化和反序列化实际是基于ObjectOutput/InputStream。


解码消息首先解码消息头部魔数等数据，然后是请求id，如果是消费者请求则解码出消息体，实际为DecodeableRpcInvocation，包括方面名，参数以及参数值；如果是请求响应，解码出返回结果，实际为DecodeableRpcResult，包含服务响应数据。




# 附

[dubbo offical site](https://dubbo.apache.org/zh-cn/index.html)    
[dubbo github](https://github.com/apache/dubbo)   
[dubbo github vt](https://github.com/Donaldhan/incubator-dubbo)  
[incubator-dubbo-spring-boot-project github vt](https://github.com/Donaldhan/incubator-dubbo-spring-boot-project)  
