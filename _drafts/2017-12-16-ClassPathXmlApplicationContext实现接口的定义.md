---
layout: page
title: ClassPathXmlApplicationContext实现接口的定义
subtitle: Spring基于xml的类型路径应用上下文的实现接口的定义
date: 2017-12-16 13:16:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言
[ClassPathXmlApplicationContext声明][]

[ClassPathXmlApplicationContext声明]: https://donaldhan.github.io/spring-framework/2017/12/16/ClassPathXmlApplicationContext%E5%A3%B0%E6%98%8E.html "ClassPathXmlApplicationContext声明"

上一篇文中我们，我们看了ClassPathXmlApplicationContext声明，并整理出ClassPathXmlApplicationContext的类图，ClassPathXmlApplicationContext直接或间接地实现了 *EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,MessageSource, ApplicationEventPublisher, ResourceLoader，Lifecycle，Closeable，BeanNameAware，InitializingBean，DisposableBean* 。

![ClassPathXmlApplicationContext](/image/spring-context/ClassPathXmlApplicationContext.png)


# 目录

* [实现接口定义](#实现接口定义)
* [总结](#总结)
* [](#)


## 实现接口定义
我们先来看一下 *BeanFactory* 两个子类接口 *ListableBeanFactory，HierarchicalBeanFactory* 的定义，看之前，先看BeanFactory接口定义：

### BeanFactory

### ListableBeanFactory

### HierarchicalBeanFactory


### InitializingBean

### DisposableBean

### BeanNameAware

### EnvironmentCapable

### ApplicationEventPublisher

### ResourceLoader

### ResourcePatternResolver


### MessageSource

### Lifecycle

### Closeable


## 总结

## 附
