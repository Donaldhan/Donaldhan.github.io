---
layout: page
title: AbstractApplicationContext源码解析第一讲
subtitle: AbstractApplicationContext解析
date: 2017-11-04 15:17:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---
# 引言

[BeanDefinition接口][]用于描述一个bean实例的属性及构造参数等元数据；主要提供了父beanname，bean类型名，作用域，懒加载，
bean依赖，自动注入候选bean，自动注入候选主要bean熟悉的设置与获取操作。同时提供了判断bean是否为单例、原型模式、抽象bean的操作，及获取bean的描述，资源描述，属性源，构造参数，原始bean定义等操作。

![BeanDefinition](/image/spring-context/BeanDefinition.png)

[BeanDefinition接口]:https://donaldhan.github.io/spring-framework/2017/12/26/BeanDefinition%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "BeanDefinition接口"

上一篇文章我们看了，BeanDefinition接口的定义，截止到上一篇文章我们将应用上下文和可配置应用上下文已看完，从这篇文章开始，我们将进入应用上下文的实现。


# 目录
* [AbstractApplicationContext定义](abstractapplicationcontext定义)
    * [DisposableBean](#disposablebean)
    * [DefaultResourceLoader](#defaultresourceloader)
* [总结](#总结)

## AbstractApplicationContext定义
源码参见：[AbstractApplicationContext][]

[AbstractApplicationContext]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java "AbstractApplicationContext"

```java
```


### DisposableBean
源码参见：[DisposableBean][]

[DisposableBean]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/DisposableBean.java "DisposableBean"

```java
```


### DefaultResourceLoader
源码参见：[DefaultResourceLoader][]

[DefaultResourceLoader]: "DefaultResourceLoader"

```java
```



最后我们以BeanDefinition的类图结束这篇文章。
![BeanDefinition](/image/spring-context/BeanDefinition.png)

## 总结
