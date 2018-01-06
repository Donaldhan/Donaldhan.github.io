---
layout: page
title: my blog
subtitle: sub title
date: 2018-11-04 15:17:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言

应用事件多播器ApplicationEventMulticaster主要提供了应用事件监听器的管理操作（添加、移除），同时提供了发布应用事件到所管理的应用监听器的操作。应用事件多播器典型应用，为代理应用上下文，发布相关应用事件。BeanClassLoaderAware主要体用了设置bean类加载器的操作，主要用于框架实现类想用根据的name获取bean的应用类型的场景。

AbstractApplicationEventMulticaster内部有一个存放监听器的集合 *ListenerRetriever*，事件监听器缓存retrieverCache（*ConcurrentHashMap<ListenerCacheKey, ListenerRetriever>*）用于存放应用事件与监听器映射关系，bean类加载器 *ClassLoader*，所属bean工厂BeanFactory
用于获取监听器bean name对应的监听器。所有的监听器注册操作实际由 *ListenerRetriever* 来完成，*ListenerRetriever* 使用LinkedHashSet来管理监听器。注意在每次添加和移除监听器之后，将会清除监听器缓存。抽象应用事件多播器除了管理监听器相关的实现此外，提供了获取注册到多播器监听器的方法，实际为ListenerRetriever整合
内部监听器集和监听器bean name对应的监听器；同时还有获取给定事件类型的对应的监听器，即关注给定事件类型的监听器，这过程首先从监听器缓存
中获取事件相关的监听器，如果存在，则从监听器检索器中检索出关闭事件的监听器，并封装在监听器检索器ListenerRetriever中，然后添加到监听器缓存中。
监听器缓存键ListenerCacheKey为事件类型与事件源的封装。

简单事件多播器[][SimpleApplicationEventMulticaster]，主要实现了多播器的多播事件操作，即将应用事件传递给相应的应用监听器，非关注
此事件的监听器，将会被忽略。默认情况下，简单事件多播器在当前线程下调用监听器的事件处理器操作，当然我们也可以设置多播器的任务执行器 *Executor*，委托任务执行器
调用监听器的事件处理器操作，同时我们也可以设置异常处理器 *ErrorHandler* 用于处理调用监听器过程中异常。

![SimpleApplicationEventMulticaster](/image/spring-context/SimpleApplicationEventMulticaster.png)



上一篇文章我们分析应用事件多播器的作用及默认实现SimpleApplicationEventMulticaster，[抽象应用上下文][]默认使用的多播器为SimpleApplicationEventMulticaster。
今天我们来看抽象应用上下为文的设计的另外一个模块的实现，及标准环境配置StandardEnvironment定义。

[SimpleApplicationEventMulticaster]:https://donaldhan.github.io/spring-framework/2018/01/06/SimpleApplicationEventMulticaster%E8%A7%A3%E6%9E%90.html "SimpleApplicationEventMulticaster解析"

[抽象应用上下文]:https://donaldhan.github.io/spring-framework/2018/01/04/AbstractApplicationContext%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%AC%AC%E4%B8%89%E8%AE%B2.html "抽象应用上下文第三讲"

# 目录
* [StandardEnvironment定义](standardenvironment定义)
    * [AbstractEnvironment](#abstractenvironment)
    * [](#)
* [总结](#总结)


## StandardEnvironment定义
源码参见：[StandardEnvironment][]

[StandardEnvironment]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/env/StandardEnvironment.java "StandardEnvironment"

```java
```


### AbstractEnvironment
源码参见：[AbstractEnvironment][]

[AbstractEnvironment]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/env/AbstractEnvironment.java "AbstractEnvironment"

```java
```


###
源码参见：[][]

[]: ""

```java
```


最后我们以StandardEnvironment的类图结束这篇文章。
![StandardEnvironment](/image/spring-context/StandardEnvironment.png)

## 总结
