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
我们先来看一下，DisposableBean接口和默认的资源加载器DefaultResourceLoader

### DisposableBean
源码参见：[DisposableBean][]

[DisposableBean]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/DisposableBean.java "DisposableBean"

```java
package org.springframework.beans.factory;

/**
 *DisposableBean接口的实现用于在析构时，释放资源。如果bean工厂销毁一个缓存单例bean，应该调用#destroy方法。
 *应用上下文在关闭时，应该销毁所有的单例bean。
 *DisposableBean的一种可选实现为，在基于XML的bean定义中，配置bean的destroy-method。更多关于所有的bean的
 *生命周期方法，见BeanFactory的javadocs。
 *
 * @author Juergen Hoeller
 * @since 12.08.2003
 * @see org.springframework.beans.factory.support.RootBeanDefinition#getDestroyMethodName
 * @see org.springframework.context.ConfigurableApplicationContext#close
 */
public interface DisposableBean {

	/**
	 * bean工厂在析构单例bean的时候调用此方法。
	 * @throws Exception in case of shutdown errors.
	 * Exceptions will get logged but not rethrown to allow
	 * other beans to release their resources too.
	 * 在关闭错误的情况下，异常将被log输出，而不是重新抛出以允许其他bean释放资源。
	 */
	void destroy() throws Exception;

}

```
从上面可以看出，DisposableBean主要提供的销毁操作，一般用于在bean析构单例bean的时候调用，以释放bean关联的资源。


### DefaultResourceLoader
源码参见：[DefaultResourceLoader][]

[DefaultResourceLoader]: "DefaultResourceLoader"

```java
```



源码参见：[AbstractApplicationContext][]

[AbstractApplicationContext]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java "AbstractApplicationContext"

```java
```




最后我们以BeanDefinition的类图结束这篇文章。
![BeanDefinition](/image/spring-context/BeanDefinition.png)




## 总结

DisposableBean主要提供的销毁操作，一般用于在bean析构单例bean的时候调用，以释放bean关联的资源。
