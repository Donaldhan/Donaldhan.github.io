---
layout: page
title: AutowireCapableBeanFactory接口定义
subtitle: AutowireCapableBeanFactory接口定义
date: 2017-12-19 15:17:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

[ApplicationContext接口定义][]
[ApplicationContext接口定义]:  "ApplicationContext接口定义"

# 引言

上一篇文章我们看了[ApplicationContext接口定义][]接口及其父类接口的定义，先来回顾一下：   
ApplicationContext接口主要提供了获取父上下文，自动装配bean工厂 *AutowireCapableBeanFactory*，应用上下文name，展示name，启动时间戳及应用id的操作。应用上下文继承了 *EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,MessageSource, ApplicationEventPublisher, ResourcePatternResolver* ，具有了访问bean容器中组件，配置环境，加载文件或类路径资源，发布应用事件到监听器，已经解决国际化消息的功能。另外需要注意的是，应用上下文具有，父上下文的继承性（HierarchicalBeanFactory）。定义在子孙上下文中的bean定义将会有限考虑。这意味着，一个单独的父上下文可以被整个web应用上下文所使用。这一点体现在，当我们使用spring的核心容器特性和spring mvc时，在web.xml中，我们有两个配置一个是上下文监听器（org.springframework.web.context.ContextLoaderListener），同时需要配置应用上下文bean的定义配置，一般是ApplicationContext.xml，另一个是Servlet分发器（org.springframework.web.servlet.DispatcherServlet），同时需要配置WebMVC相关配置，一般是springmvc.xml。应用一般运行的在Web容器中，Web容器可以访问应用上下文，同时Web容器的Servlet也可以访问应用上下文，然而每个servlet有自己的上下文，独立于其他servlet。

![ApplicationContext](/image/spring-context/ApplicationContext.png)

今天我么来看一下*AutowireCapableBeanFactory* 接口的定义

# 目录



[AutowireCapableBeanFactory][]
[AutowireCapableBeanFactory]:
https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/config/AutowireCapableBeanFactory.java "AutowireCapableBeanFactory"
