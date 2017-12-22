---
layout: page
title: ConfigurableListableBeanFactory接口定义
subtitle: ConfigurableListableBeanFactory接口及父接口定义
date: 2017-12-22 10:35:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言

ConfigurableApplicationContext具备应用上下文 *ApplicationContex* 相关操作以外，同时具有了生命周期和流属性。除此之外，
提供了设置应用id，设置父类上下文，设置环境 *ConfigurableEnvironment*，添加应用监听器，添加bean工厂后处理器 *BeanFactoryPostProcessor*，添加协议解决器 *ProtocolResolver*，刷新应用上下文，关闭应用上下文，判断上下文状态，以及注册虚拟机关闭Hook等操作，同时重写了获取环境操作，此操作返回的为可配置环境 *ConfigurableEnvironment*。最关键的是提供了获取内部bean工厂的访问操作，
方法返回为 *ConfigurableListableBeanFactory*。需要注意的是，调用关闭操作，并不关闭父类的应用上下文，应用上下文与父类的上下文生命周期，相互独立。

![ConfigurableApplicationContext](/image/spring-context/ConfigurableApplicationContext.png)

今天我们来看一下ConfigurableListableBeanFactory接口的定义。
# 目录
* [ConfigurableBeanFactory](#configurablebeanfactory)
* [SingletonBeanRegistry](#singletonbeanregistry)
* [ConfigurableApplicationContext接口定义](#configurableapplicationcontext接口定义)
* [总结](#总结)
* [附](#附)



### ConfigurableListableBeanFactory接口定义

源码参见：[ConfigurableListableBeanFactory][]

[ConfigurableListableBeanFactory]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/config/ConfigurableListableBeanFactory.java "ConfigurableListableBeanFactory"

```java

```
