---
layout: page
title: Super-Diamond配置管理服务器
subtitle: Super-Diamond配置管理服务器
date: 2018-04-20 19：08：36
author: donaldhan
catalog: true
category:  Super-Diamond
categories:
    -  Super-Diamond
tags:
    -  Super-Diamond
---

# 引言

在我们配置服务器属性时候，一般常使用方法，将配置属性放在在一个配置文件中，当应用上线时，需要修改配置文件，这样容易导致手动修改配置文件出错；
自从maven的出现使我们可以，我们可以将不同环境的配置，写到不同的文件中，比如（dev，test ，exp，prod）等环境，在项目上线时，我们只需要根据
Profile属性，打包相应的属性文件，这样避免的手动修改配置引起的认为问题，但无法解决修改配置文件需要重新启动或重新打包的问题。历史的车轮，终是向前推动的。
一些配置管理服务顺应出现，比如Spring的[spring-cloud-config][], 淘宝的淘宝[diamond][],另外还有一种轻量级的配置管理服务器[ Super-Diamond][]。Spring家族的
spring-cloud-config的文档，分支版本管理比较规范，毕竟是专业滴。淘宝的diamond，不知现在淘宝现在，还在不在用，不过，在github上，搜不到淘宝的diamond，但是有个
diamond的分支，不过文档很少，这也是阿里开源产品通病。今天我们来看轻量级的配置管理服务器[Super-Diamond][]，从源码的版权声明来看是苏州科大国创信息技术有限公司的产品。


![BeanDefinition](/image/spring-context/BeanDefinition.png)

[BeanDefinition接口]:https://donaldhan.github.io/spring-framework/2017/12/26/BeanDefinition%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "BeanDefinition接口"

上一篇文章我们看了，BeanDefinition接口的定义，截止到上一篇文章我们将应用上下文和可配置应用上下文已看完，从这篇文章开始，我们将进入应用上下文的实现。

[spring-cloud-config]:https://github.com/Donaldhan/spring-cloud-config "spring-cloud-config"  
[diamond]:https://github.com/takeseem/diamond "diamond"  
[Super-Diamond]:https://github.com/Donaldhan/super-diamond "Super-Diamond"

# 目录
* [Super-Diamond架构设计](super-diamond架构设计)
    * [Super-Diamond服务端](#)
    * [Super-Diamond客户端](#)
* [总结](#总结)

## Super-Diamond架构设计
源码参见：[AbstractApplicationContext][]

[AbstractApplicationContext]: "AbstractApplicationContext"

```java
```
