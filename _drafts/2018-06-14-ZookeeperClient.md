---
layout: page
title: ZookeeperClient
subtitle: ZookeeperClient
date: 2018-06-14 17:07:15
author: donaldhan
catalog: true
category: Zookeeper
categories:
    - Zookeeper
tags:
    - Zookeeper
---

# 引言
随着分布式应用的发展，对配置的管理变得越来越繁琐，单机配置中心，容易导致配置中心挂掉的情况下，出现整体应用瘫痪的场景。Zookeeper出现解决了单点配置的缺点，
同时我们可以很容易使用Zookeeper建立配置集群，当集群中的某台配置服务器宕机时，不会对应用造成任务影响。更重要的是ZK提供了观察者机制，可以动态监听配置的变更。
Zookeeper以文件目录的方式存储数据，使我们可以非常方便的管理配置。

这篇文章，我们不打算深入的研究Zookeeper，我们关注的是Zookeeper原生API， ZkClient和Curator3客户端的优缺点，以后有时间我们在来探究Zookeeper的源码。这篇文章所有使用的示例代码可以参考[zookeeper-demo][]。





# 目录
* [定义](abstractapplicationcontext定义)
    * [](#)
    * [](#)
* [总结](#总结)

## AbstractApplicationContext定义
源码参见：[AbstractApplicationContext][]

[AbstractApplicationContext]: "AbstractApplicationContext"

```java
```


###
源码参见：[][]

[]: ""

```java
```


###
源码参见：[][]

[]: ""

```java
```

[zookeeper-demo]:https://github.com/Donaldhan/zookeeper-demo "zookeeper-demo"

[BeanDefinition接口][]

![BeanDefinition](/image/spring-context/BeanDefinition.png)

[BeanDefinition接口]:https://donaldhan.github.io/spring-framework/2017/12/26/BeanDefinition%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "BeanDefinition接口"

## 总结
