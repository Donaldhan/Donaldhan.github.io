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

[抽象应用上下文][] *AbstractApplicationContext* 实际为一个可配置上下文 *ConfigurableApplicationContext* 和可销毁的bean（DisposableBean），同时拥有了资源加载功能（DefaultResourceLoader）。我们通过一个唯一的id标注抽象上下文，同时抽象上下文拥有一个展示名。除此身份识别属性之前，抽象应用上下文，有一个父上下文 *ApplicationContext* ，可配的环境配置 *ConfigurableEnvironment* ，bean工厂后处理器集（List<BeanFactoryPostProcessor>），资源模式解决器（ResourcePatternResolver），声明周期处理器（LifecycleProcessor),消息源 *MessageSource* ，事件发布器 *ApplicationEventMulticaster* ，应用监听器集（LinkedHashSet<ApplicationListener<?>>），预发布的应用事件集（LinkedHashSet<ApplicationEvent>）。除了上述的功能性属性外，抽象应用上下文，还有一个一些状态属性，如果启动时间，激活状态（AtomicBoolean），关闭状态（AtomicBoolean）。最后还有一个上下为刷新和销毁的同步监控对象和一虚拟机关闭hook线程。


路径匹配资源模式解决器PathMatchingResourcePatternResolver内部有一个Ant路径匹配器 *AntPathMatcher*，和一个资源类加载器，资源加载器可以
使用所属上下文中的资源加载器，也可以为给定类加载器的DefaultResourceLoader。路径匹配资源模式解决器主要提供了加载给定路径位置的资源方法，此方法可以解决无通配符的路径位置模式（{@code file:C:/context.xml}，{@code classpath:/context.xml}，{@code /WEB-INF/context.xml}"），也可以解决包含Ant风格的通配符路径位置模式资源（{@code classpath*:META-INF/beans.xml}），主要以classpath*为前缀的路径位置模式，资源加载器将会查找类路径下所有相同name对应的资源文件，包括子目录和jar包。如果明确的加载资源，可以使用{@code classpath:/context.xml}形式路径模式，如果想要探测类路径下的所有name对应的资源文件，可以使用形式路径模式。

BeanFactoryAware接口主要提供设置bean工厂操作。LifecycleProcessor接口主要提供了通知上下文刷新和关闭的操作。Phased主要提供了获取组件阶段值操作。
SmartLifecycle接口主要提供关闭回调操作，在组件停止后，调用回调接口。并提供了判断组件在容器上下文刷新时，组件是否自动刷新的操作。

默认生命周期处理器DefaultLifecycleProcessor，内部主要有3个成员变量，一个是运行状态标识，一个是生命周期bean关闭超时时间，还有一个是所属的bean工厂。默认生命周期处理器，启动生命周期bean的过程为，将生命周期bean，按阶段值分组，并从阶段值从小到大，启动生命周期bean分组中bean。默认生命周期处理器，关闭生命周期bean的过程为，将生命周期bean，按阶段值分组，并从阶段值从大到小，关闭生命周期bean分组中bean。关闭生命周期bean的顺序与启动顺序正好相反。需要注意的是无论是启动还是关闭，生命周期bean所依赖的bean都是在其之前启动或关闭，忽略掉被依赖bean的Phase阶段值。对于非生命周期bean，其阶段值默认为0。

![DefaultLifecycleProcessor](/image/spring-context/DefaultLifecycleProcessor.png)

[抽象应用上下文]:https://donaldhan.github.io/spring-framework/2018/01/04/AbstractApplicationContext%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%AC%AC%E4%B8%89%E8%AE%B2.html "抽象应用上下文第三讲"

上一篇文章我们看了抽象应用上下文的内部变量声明与构造函数，同时看了一下默认的路径匹配资源模式解决器PathMatchingResourcePatternResolver和生命周期处理器DefaultLifecycleProcessor。
由于抽象应用上下文所设计的模块较多，我们不得不分模块来分析，今天我们来看另一个模块应用事件多播器。


# 目录
* [SimpleApplicationEventMulticaster定义](simpleapplicationeventmulticaster定义)
    * [ApplicationEventMulticaster](#applicationeventmulticaster)
    * [BeanClassLoaderAware](#beanclassloaderaware)
    * [AbstractApplicationEventMulticaster](#abstractapplicationeventmulticaster)
* [总结](#总结)



## SimpleApplicationEventMulticaster定义
源码参见：[SimpleApplicationEventMulticaster][]

[SimpleApplicationEventMulticaster]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/event/SimpleApplicationEventMulticaster.java "SimpleApplicationEventMulticaster"

在看SimpleApplicationEventMulticaster定义之前，我们来看一下其父接口定义

### ApplicationEventMulticaster
源码参见：ApplicationEventMulticaster][]

[ApplicationEventMulticaster]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/event/ApplicationEventMulticaster.java "ApplicationEventMulticaster"

```java
package org.springframework.context.event;

import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.core.ResolvableType;

/**
 *应用事件多播器ApplicationEventMulticaster的实现可以管理多个应用监听器对象，
 *并发布事件到相关监听器。
 *应用事件多播器的使用典型场景，应用上下文可以使用应用事件多播器代理事件的发布事件操作。
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @author Stephane Nicoll
 */
public interface ApplicationEventMulticaster {

	/**
	 * 添加监听器
	 * @param listener the listener to add
	 */
	void addApplicationListener(ApplicationListener<?> listener);

	/**
	 * 添加监听器bean
	 * @param listenerBeanName the name of the listener bean to add
	 */
	void addApplicationListenerBean(String listenerBeanName);

	/**
	 * 移除监听器
	 * @param listener the listener to remove
	 */
	void removeApplicationListener(ApplicationListener<?> listener);

	/**
	 * 移除监听器bean
	 * @param listenerBeanName the name of the listener bean to add
	 */
	void removeApplicationListenerBean(String listenerBeanName);

	/**
	 * 移除所有注册到多播器的监听器。
	 * 在移除所有监听器操作调用后，多播器对于发生的事件不做任何处理，直到有新的监听器注册
	 */
	void removeAllListeners();

	/**
	 * 多播给定的应用事件到相关监听器
	 * 如果想要尽可能中的支持一般的事件，可以考虑使用{@link #multicastEvent(ApplicationEvent, ResolvableType)}
	 * 方法。
	 * @param event the event to multicast
	 */
	void multicastEvent(ApplicationEvent event);

	/**
	 * Multicast the given application event to appropriate listeners.
	 * 多播给定的事件到关联监听器。
	 * 如果eventType类型为空，则将基于event实例 构建一个默认的类型
	 * @param event the event to multicast
	 * @param eventType the type of event (can be null)
	 * @since 4.2
	 */
	void multicastEvent(ApplicationEvent event, ResolvableType eventType);

}

```
从上面可以看出，应用事件多播器ApplicationEventMulticaster主要提供了应用事件监听器的管理操作（添加、移除），同时提供了发布应用事件到所管理的应用监听器的操作。
应用事件多播器典型应用，为代理应用上下文，发布相关应用事件。

### BeanClassLoaderAware
源码参见：[BeanClassLoaderAware][]

[BeanClassLoaderAware]: "BeanClassLoaderAware"

```java
```


### AbstractApplicationEventMulticaster
源码参见：[AbstractApplicationEventMulticaster][]

[AbstractApplicationEventMulticaster]: "AbstractApplicationEventMulticaster"

```java
```



我们回到SimpleApplicationEventMulticaster
```java
```

最后我们以BeanDefinition的类图结束这篇文章。
![BeanDefinition](/image/spring-context/BeanDefinition.png)

## 总结

应用事件多播器ApplicationEventMulticaster主要提供了应用事件监听器的管理操作（添加、移除），同时提供了发布应用事件到所管理的应用监听器的操作。
应用事件多播器典型应用，为代理应用上下文，发布相关应用事件。
