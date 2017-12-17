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

```java
package org.springframework.beans.factory;

/**
 *HierarchicalBeanFactory作为bean工厂的实现，标志着为一个可继承的bean工厂。
 *在ConfigurableBeanFactory接口的setParentBeanFactory方法，中可以配置bean工厂的父类工厂。
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @since 07.07.2003
 * @see org.springframework.beans.factory.config.ConfigurableBeanFactory#setParentBeanFactory
 */
public interface HierarchicalBeanFactory extends BeanFactory {
	BeanFactory getParentBeanFactory();
	boolean containsLocalBean(String name);
}
```
从HierarchicalBeanFactory的定义来看，HierarchicalBeanFactory接口标志一个以可继承bean工厂，我们可以通过 *ConfigurableBeanFactory* 接口的 *setParentBeanFactory* 方法配置bean工厂的父类工厂，主要提供获取父工厂操作，以及判断在本地bean工厂中是否存在指定name对应的bean的操作，但忽略祖先上下文中的bean定义。

### ApplicationEventPublisher
ApplicationEventPublisher的源码，参见[ApplicationEventPublisher][]

[ApplicationEventPublisher]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/ApplicationEventPublisher.java "ApplicationEventPublisher"

```java
package org.springframework.context;

/**
 *ApplicationEventPublisher接口，封装了事件发布功能，作为ApplicationContext的父接口，
 *服务于应用上下文ApplicationContext。
 * @author Juergen Hoeller
 * @author Stephane Nicoll
 * @since 1.1.1
 * @see ApplicationContext
 * @see ApplicationEventPublisherAware
 * @see org.springframework.context.ApplicationEvent
 * @see org.springframework.context.event.EventPublicationInterceptor
 */
public interface ApplicationEventPublisher {
	void publishEvent(ApplicationEvent event);
	void publishEvent(Object event);

}
```
从上面可看出，事件发布接口ApplicationEventPublisher，主要作为ApplicationContext的父接口，封装了事件发布功能，提供了事件发布功能。当事件发布时，通知所有注册到当前应用关注ApplicationEvent事件event的监听器，如果发布的事件不是 *ApplicationEvent*，
则将会包装成 *PayloadApplicationEvent*。

再来简单看一下应用事件[ApplicationEvent][]

[ApplicationEvent]:  https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/ApplicationEvent.java "ApplicationEvent"

```java
package org.springframework.context;

import java.util.EventObject;

/**
 *所有的应用事件，将会继承此类。抽象不意味着，一般的应用事件不可以直接发布。
 * @author Rod Johnson
 * @author Juergen Hoeller
 */
public abstract class ApplicationEvent extends EventObject {
	/** use serialVersionUID from Spring 1.2 for interoperability */
	private static final long serialVersionUID = 7099057708183571937L;
	/** System time when the event happened 事件发生的系统时间*/
	private final long timestamp;
	public ApplicationEvent(Object source) {
		super(source);
		this.timestamp = System.currentTimeMillis();
	}
	public final long getTimestamp() {
		return this.timestamp;
	}

}
```

从上面可以看出，ApplicationEvent用于表示应用发生的事件，事件包括事件发生的时间和事件发生源。

我们再来看一下ApplicationEvent的父类EventObject，EventObject属于jdk的范畴。

```java
package java.util;

/**
 * <p>
 * The root class from which all event state objects shall be derived.
 所有事件状态对象的原始类。
 * <p>
 * All Events are constructed with a reference to the object, the "source",
 * that is logically deemed to be the object upon which the Event in question
 * initially occurred upon.
 所有事件构造有一个对象已用，此对象用于表示，事件发生源。
 *
 * @since JDK1.1
 */

public class EventObject implements java.io.Serializable {

    private static final long serialVersionUID = 5516075349620653480L;

    /**
     * The object on which the Event initially occurred.
     事件发生源
     */
    protected transient Object  source;

    /**
     * Constructs a prototypical Event.
     *
     * @param    source    The object on which the Event initially occurred.
     * @exception  IllegalArgumentException  if source is null.
     */
    public EventObject(Object source) {
        if (source == null)
            throw new IllegalArgumentException("null source");

        this.source = source;
    }
    public Object getSource() {
        return source;
    }
    ...
}
```

从上面来看EventObject，表示一个事件对象，同时记录事件发生源。

再来看一下应用监听器[ApplicationListener][]

[ApplicationListener]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/ApplicationListener.java "ApplicationListener"

```java
package org.springframework.context;

import java.util.EventListener;

/**
 * Interface to be implemented by application event listeners.
 * Based on the standard {@code java.util.EventListener} interface
 * for the Observer design pattern.。
 *所有应用事件监听器将实现ApplicationListener接口。此接口基于标准的观察者模式的接口java.util.EventListener。
 * <p>As of Spring 3.0, an ApplicationListener can generically declare the event type
 * that it is interested in. When registered with a Spring ApplicationContext, events
 * will be filtered accordingly, with the listener getting invoked for matching event
 * objects only.
 * 从spring3.0以后，应用监听器需要声明关注的应用事件类型。当监听器注册到spring的应用上下文ApplicationContext时，
 * 将会根据事件类型过滤监听器，匹配事件类型的监听器才会被通知。
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @param <E> the specific ApplicationEvent subclass to listen to
 * @see org.springframework.context.event.ApplicationEventMulticaster
 */
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	/**
	 * Handle an application event.
	 * 处理应用事件
	 * @param event the event to respond to
	 */
	void onApplicationEvent(E event);

}

```

从上面可以看出，监听器接口ApplicationListener，是基于标准JDK的观察者模式的接口java.util.EventListener，
从spring3.0以后，应用监听器需要声明关注的应用事件类型。当监听器注册到spring的应用上下文ApplicationContext时，
将会根据事件类型过滤监听器，匹配事件类型的监听器才会被通知。接口主要提供了处理事件操作。

来简单看一下jdk的事件监听器接口的声明：

```java
package java.util;

/**
 * A tagging interface that all event listener interfaces must extend.
 * @since JDK1.1
 */
public interface EventListener {
}
```
从上面可以看出事件监听器接口，从JDK1.1，就已经出现。

我们接着看EnvironmentCapable接口

### EnvironmentCapable
源码参见[EnvironmentCapable][]

[EnvironmentCapable]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/env/EnvironmentCapable.java "EnvironmentCapable"

```java
package org.springframework.core.env;

/**
 * EnvironmentCapable表示一个组件包括一个或暴露一个Environment环境引用。
 *Spring的所有应用上下文都是EnvironmentCapable接口实现，用于应用上下文与环境交互。
 * 需要注意的是，ApplicationContext扩展了EnvironmentCapable接口，通过getEnvironment方法暴露环境配置；
 * 然而ConfigurableApplicationContext将会重定义getEnvironment方法，返回一个ConfigurableEnvironment。
 * 两种方法带来的效果是，在环境配置Environment对象在ConfigurableApplicationContext可访问以前，都是自读的，
 * 可以理解为ConfigurableApplicationContext的getEnvironment方法返回的环境对象时可修改的。
 *
 * @author Chris Beams
 * @since 3.1
 * @see Environment
 * @see ConfigurableEnvironment
 * @see org.springframework.context.ConfigurableApplicationContext#getEnvironment()
 */
public interface EnvironmentCapable {

	/**
	 * Return the {@link Environment} associated with this component
	 * (may be {@code null} or a default environment).
	 * 返回组件关联的环境Environment，没有则为空。
	 */
	Environment getEnvironment();

}
```
从上面可以看出，接口从spring3.1才使用，EnvironmentCapable接口，表示包括或暴露一个Environment环境引用的组件。Spring的所有应用上下文都是EnvironmentCapable接口实现，用于应用上下文与环境交互。需要注意的是，ApplicationContext扩展了EnvironmentCapable接口，通过getEnvironment方法暴露环境配置；然而 *ConfigurableApplicationContext* 将会重定义getEnvironment方法，返回一个*ConfigurableEnvironment*。 两种方法带来的效果是，在环境配置Environment对象在 *ConfigurableApplicationContext* 可访问以前，都是自读的，可以理解为 *ConfigurableApplicationContext* 的getEnvironment方法返回的环境象时可修改的。接口提供了获取环境配置操作。

我们再来看一下环境配置[Environment][]接口的定义:

[Environment]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/env/Environment.java  "Environment"

```java
```



[PropertyResolver][]
[PropertyResolver]: ""

```java
```


### InitializingBean

### DisposableBean

### BeanNameAware

### ResourceLoader

### ResourcePatternResolver

### MessageSource

### Lifecycle

### Closeable


## 总结

BeanFactory接口主要是主要提供了根据名称或类型获取bean的相关方法，以及判断bean是否匹配指定类型或判断是否包含指定name的共享单实例或多实例bean。需要注意的是，如果bean工厂的实现是可继承工厂，那么调用这些方法，如果没有在当前bean工厂实例中找到，将会从父工厂中查到。另外还需要注意一点，判断一个bean是否为共享单例模式，可以使用isSingleton方法，返回true，即是，返回false，并不能表示bean是多实例bean，具体要用isPrototype方法判断，同理isPrototype方法也是如此。  

ListableBeanFactory接口主要提供了判断是否存在给定name的bean定义，获取bean定义数量，获取指定类型的bean定义的name集或name与bean实例的映射集，获取待指定注解的bean定义的name或name与bean实例的映射集，以及获取给定name对应的bean的注定注解实例。需要注意的是，提供的操作不会到可继承bean工厂中去搜索，但包括BeanFactoryUtils工具类获取bean工厂的祖先bean工厂。另外getBeanNamesForType和getBeanNamesForAnnotation方法可以通过includeNonSingletons和allowEagerInit，
控制搜索bean的作用域范围和是否初始化懒加载单例模式bean与工厂bean。   

HierarchicalBeanFactory接口标志一个以可继承bean工厂，我们可以通过 *ConfigurableBeanFactory* 接口的 *setParentBeanFactory* 方法配置bean工厂的父类工厂，主要提供获取父工厂操作，以及判断在本地bean工厂中是否存在指定name对应的bean的操作，但忽略祖先上下文中的bean定义。

事件发布接口ApplicationEventPublisher，主要作为ApplicationContext的父接口，封装了事件发布功能，提供了事件发布功能。当事件发布时，通知所有注册到当前应用关注ApplicationEvent事件event的监听器，如果发布的事件不是 *ApplicationEvent*，
则将会包装成 *PayloadApplicationEvent*。

ApplicationEvent用于表示应用发生的事件，事件包括事件发生的时间和事件发生源。

监听器接口ApplicationListener，是基于标准JDK的观察者模式的接口java.util.EventListener，
从spring3.0以后，应用监听器需要声明关注的应用事件类型。当监听器注册到spring的应用上下文ApplicationContext时，
将会根据事件类型过滤监听器，匹配事件类型的监听器才会被通知。接口主要提供了处理事件操作。

EnvironmentCapable接口，表示包括或暴露一个Environment环境引用的组件。Spring的所有应用上下文都是EnvironmentCapable接口实现，用于应用上下文与环境交互。需要注意的是，ApplicationContext扩展了EnvironmentCapable接口，通过getEnvironment方法暴露环境配置；然而 *ConfigurableApplicationContext* 将会重定义getEnvironment方法，返回一个*ConfigurableEnvironment*。 两种方法带来的效果是，在环境配置Environment对象在 *ConfigurableApplicationContext* 可访问以前，都是自读的，可以理解为 *ConfigurableApplicationContext* 的getEnvironment方法返回的环境象时可修改的。接口提供了获取环境配置操作。

## 附
