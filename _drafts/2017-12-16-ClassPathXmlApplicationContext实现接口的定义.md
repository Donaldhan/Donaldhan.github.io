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
BeanFactory接口源码参见[BeanFactory][]


[BeanFactory]: https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/BeanFactory.java  "BeanFactory"

```java
package org.springframework.beans.factory;

import org.springframework.beans.BeansException;
import org.springframework.core.ResolvableType;

/**
 * <p>Bean factory implementations should support the standard bean lifecycle interfaces
 * as far as possible. The full set of initialization methods and their standard order is:
 * 尽可能地bean工厂的实现，应该支持标准的bean生命周期接口，比如lifecycle。所有初始化方法和他们的标准顺序如下：
 * <ol>
 * <li>BeanNameAware's {@code setBeanName}
 * <li>BeanClassLoaderAware's {@code setBeanClassLoader}
 * <li>BeanFactoryAware's {@code setBeanFactory}
 * <li>EnvironmentAware's {@code setEnvironment}
 * <li>EmbeddedValueResolverAware's {@code setEmbeddedValueResolver}
 * <li>ResourceLoaderAware's {@code setResourceLoader}
 * (only applicable when running in an application context)
 * <li>ApplicationEventPublisherAware's {@code setApplicationEventPublisher}
 * (only applicable when running in an application context)
 * <li>MessageSourceAware's {@code setMessageSource}
 * (only applicable when running in an application context)
 * <li>ApplicationContextAware's {@code setApplicationContext}
 * (only applicable when running in an application context)
 * <li>ServletContextAware's {@code setServletContext}
 * (only applicable when running in a web application context)
 * <li>{@code postProcessBeforeInitialization} methods of BeanPostProcessors
 * <li>InitializingBean's {@code afterPropertiesSet}
 * <li>a custom init-method definition
 * <li>{@code postProcessAfterInitialization} methods of BeanPostProcessors
 * </ol>
 *
 * <p>On shutdown of a bean factory, the following lifecycle methods apply:
 * <ol>在关闭bean工厂之后，下面方法回调用
 * <li>{@code postProcessBeforeDestruction} methods of DestructionAwareBeanPostProcessors
 * <li>DisposableBean's {@code destroy}
 * <li>a custom destroy-method definition
 * </ol>
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @author Chris Beams
 * @since 13 April 2001
 * @see BeanNameAware#setBeanName
 * @see BeanClassLoaderAware#setBeanClassLoader
 * @see BeanFactoryAware#setBeanFactory
 * @see org.springframework.context.ResourceLoaderAware#setResourceLoader
 * @see org.springframework.context.ApplicationEventPublisherAware#setApplicationEventPublisher
 * @see org.springframework.context.MessageSourceAware#setMessageSource
 * @see org.springframework.context.ApplicationContextAware#setApplicationContext
 * @see org.springframework.web.context.ServletContextAware#setServletContext
 * @see org.springframework.beans.factory.config.BeanPostProcessor#postProcessBeforeInitialization
 * @see InitializingBean#afterPropertiesSet
 * @see org.springframework.beans.factory.support.RootBeanDefinition#getInitMethodName
 * @see org.springframework.beans.factory.config.BeanPostProcessor#postProcessAfterInitialization
 * @see DisposableBean#destroy
 * @see org.springframework.beans.factory.support.RootBeanDefinition#getDestroyMethodName
 */
public interface BeanFactory {
	String FACTORY_BEAN_PREFIX = "&";
	Object getBean(String name) throws BeansException;
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;
	boolean containsBean(String name);
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;
	String[] getAliases(String name);

}
```
从上面可以看出，BeanFactory接口主要是主要提供了根据名称或类型获取bean的相关方法，以及判断bean是否匹配指定类型或判断
是否包含指定name的共享单实例或多实例bean。需要注意的是，如果bean工厂的实现是可继承工厂 *HierarchicalBeanFactory*，那么调用这些方法，如果没有在当前bean工厂实例中找到，将会从父工厂中查到。另外还需要注意一点，判断一个bean是否为共享单例模式，可以使用isSingleton方法，返回true，即是，返回false，并不能表示bean是多实例bean，具体要用isPrototype方法判断，同理isPrototype方法也是如此。

再来看ListableBeanFactory接口的定义：
### ListableBeanFactory

具体ListableBeanFactory的源码，参见[ListableBeanFactory][]

[ListableBeanFactory]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/ListableBeanFactory.java  "ListableBeanFactory"


```java
package org.springframework.beans.factory;
import java.lang.annotation.Annotation;
import java.util.Map;
import org.springframework.beans.BeansException;
import org.springframework.core.ResolvableType;

/**
 *ListableBeanFactory扩展了bean工厂接口，ListableBeanFactory接口的实现可以列举所有bean实例，
 *而不是作为一个请求客户端尝试以bean的名称，搜索bean。BeanFactory实现将会预先加载所有配置中BeanFactory
 *的所有bean定义。
 *如果bean工厂是一个可继承的bean工厂HierarchicalBeanFactory，返回值将不会考虑父类工厂bean，
 *仅仅考虑与当前bean工厂关联的bean。BeanFactoryUtils工具类获取的祖先bean工厂，将考虑在内。
 *接口方法，只会搜索当前bean工厂的bean定义。将会忽略所有通过ConfigurableBeanFactory的registerSingleton方法，
 *注册到bean工厂的单例模式bean。
 *注意：getBeanDefinitionCount和containsBeanDefinition方法异常情况，此方法不是设计为频繁调用的。
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @since 16 April 2001
 * @see HierarchicalBeanFactory
 * @see BeanFactoryUtils
 */
public interface ListableBeanFactory extends BeanFactory {
	boolean containsBeanDefinition(String beanName);
	int getBeanDefinitionCount();
	String[] getBeanDefinitionNames();
	String[] getBeanNamesForType(ResolvableType type);
	String[] getBeanNamesForType(Class<?> type);
	String[] getBeanNamesForType(Class<?> type, boolean includeNonSingletons, boolean allowEagerInit);
	<T> Map<String, T> getBeansOfType(Class<T> type) throws BeansException;
	<T> Map<String, T> getBeansOfType(Class<T> type, boolean includeNonSingletons, boolean allowEagerInit)
			throws BeansException;
	String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType);
	Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType) throws BeansException;
	<A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType)
			throws NoSuchBeanDefinitionException;
}
```
从ListableBeanFactory的定义，可以看出ListableBeanFactory接口主要提供了判断是否存在给定name的bean定义，获取bean定义数量，获取指定类型的bean定义的name集或name与bean实例的映射集，获取待指定注解的bean定义的name或name与bean实例的映射集，以及获取给定name对应的bean的注定注解实例。需要注意的是，提供的操作不会到可继承bean工厂中去搜索，但包括BeanFactoryUtils工具类获取bean工厂的祖先bean工厂。另外getBeanNamesForType和getBeanNamesForAnnotation方法可以通过includeNonSingletons和allowEagerInit，
控制搜索bean的作用域范围和是否初始化懒加载单例模式bean与工厂bean。  

再来看HierarchicalBeanFactory接口的定义：
### HierarchicalBeanFactory
具体HierarchicalBeanFactory源码，参见[HierarchicalBeanFactory][]
[HierarchicalBeanFactory]：https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/HierarchicalBeanFactory.java "HierarchicalBeanFactory"

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

BeanFactory接口主要是主要提供了根据名称或类型获取bean的相关方法，以及判断bean是否匹配指定类型或判断是否包含指定name的共享单实例或多实例bean。需要注意的是，如果bean工厂的实现是可继承工厂，那么调用这些方法，如果没有在当前bean工厂实例中找到，将会从父工厂中查到。另外还需要注意一点，判断一个bean是否为共享单例模式，可以使用isSingleton方法，返回true，即是，返回false，并不能表示bean是多实例bean，具体要用isPrototype方法判断，同理isPrototype方法也是如此。  
ListableBeanFactory接口主要提供了判断是否存在给定name的bean定义，获取bean定义数量，获取指定类型的bean定义的name集或name与bean实例的映射集，获取待指定注解的bean定义的name或name与bean实例的映射集，以及获取给定name对应的bean的注定注解实例。需要注意的是，提供的操作不会到可继承bean工厂中去搜索，但包括BeanFactoryUtils工具类获取bean工厂的祖先bean工厂。另外getBeanNamesForType和getBeanNamesForAnnotation方法可以通过includeNonSingletons和allowEagerInit，
控制搜索bean的作用域范围和是否初始化懒加载单例模式bean与工厂bean。

## 附
