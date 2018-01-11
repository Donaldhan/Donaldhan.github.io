---
layout: page
title: AbstractApplicationContext源码解析第三讲
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

HierarchicalMessageSource主要提供了设置父消息的操作。此接口用于，当消息源不能解决给定消息时，尝试使用父消息源解决消息，即层级解决消息。

MessageSourceSupport内部主要成员为消息格式缓存messageFormatsPerLocale（HashMap<String, Map<Locale, MessageFormat>>()）用于存储消息的消息格式，以及是否应用消息格式规则，解析没有参数的消息标志alwaysUseMessageFormat。此消息源实现基础类主要提供了根据给定的消息，消息参数和本地化渲染消息操作，即使用参数替代默认消息中的占位符。

DelegatingMessageSource内部有一父消息源，获取编码消息操作直接委托给父类数据源，如果没有父消息源，同时有默认消息，则使用消息参数渲染默认消息，并返回。如果没有父消息可用，不会解决任何消息，抛出异常。代理消息源用于，上下文没有定义消息源情况下，AbstractApplicationContext使用 DelegatingMessageSource为消息源的占位符，此代理消息源不能被应用直接使用。

![DelegatingMessageSource](/image/spring-context/DelegatingMessageSource.png)

这是我们上一篇[DelegatingMessageSource解析][]文章所讲内容,自此我们将抽象应用上下文的资源加载模块，应用事件多播，环境配置，消息源（DelegatingMessageSource）已将完，我们接着抽象应用上下文的成员变量和构造继续来讲，如果遗忘了，可以重新查看相关的文章[抽象应用上下文第三讲][]。

先来回顾下抽象应用上下文第三讲：  

抽象应用上下文 *AbstractApplicationContext* 实际为一个可配置上下文 *ConfigurableApplicationContext* 和可销毁的bean（DisposableBean），同时拥有了资源加载功能（DefaultResourceLoader）。我们通过一个唯一的id标注抽象上下文，同时抽象上下文拥有一个展示名。除此身份识别属性之前，抽象应用上下文，有一个父上下文 *ApplicationContext* ，可配的环境配置 *ConfigurableEnvironment* ，bean工厂后处理器集（List<BeanFactoryPostProcessor>），资源模式解决器（ResourcePatternResolver），声明周期处理器（LifecycleProcessor),消息源 *MessageSource* ，事件发布器 *ApplicationEventMulticaster* ，应用监听器集（LinkedHashSet<ApplicationListener<?>>），预发布的应用事件集（LinkedHashSet<ApplicationEvent>）。除了上述的功能性属性外，抽象应用上下文，还有一个一些状态属性，如果启动时间，激活状态（AtomicBoolean），关闭状态（AtomicBoolean）。最后还有一个上下为刷新和销毁的同步监控对象和一虚拟机关闭hook线程。

现在我们来继续看应用上下文的其他方法。

[DelegatingMessageSource解析]: "DelegatingMessageSource解析"  

[抽象应用上下文第三讲]:https://donaldhan.github.io/spring-framework/2018/01/04/AbstractApplicationContext%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%AC%AC%E4%B8%89%E8%AE%B2.html "抽象应用上下文第三讲"





# 目录
* [AbstractApplicationContext定义](abstractapplicationcontext定义)
    * [](#)
    * [](#)
* [总结](#总结)

## AbstractApplicationContext定义
源码参见：[AbstractApplicationContext][]

[AbstractApplicationContext]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java "AbstractApplicationContext"

```java
/**
	 * 设置应用上下文的环境。
	 * 默认为{@link #createEnvironment()}返回的环境实例。使用此方法可以替换默认的环境配置，
	 * 可以通过{@link#getEnvironment()}获取环境配置。修改环境应该在{@link #refresh()}操作之前
	 * @see org.springframework.context.support.AbstractApplicationContext#createEnvironment
	 */
	@Override
	public void setEnvironment(ConfigurableEnvironment environment) {
		this.environment = environment;
	}

	/**
	 * 返回应用上下文的可配置环境，允许进一步定制，如果没有，则通过createEnvironment方法，创建一个
	 * 标准的环境。
	 */
	@Override
	public ConfigurableEnvironment getEnvironment() {
		if (this.environment == null) {
			this.environment = createEnvironment();
		}
		return this.environment;
	}
	/**
	 * 创建，并返回一个标准的环境StandardEnvironment，为了使用定制ConfigurableEnvironment，
	 * 的子类可以重写这个方法
	 */
	protected ConfigurableEnvironment createEnvironment() {
		return new StandardEnvironment();
	}

```
从上面可以，我们可以通过setEnvironment配置上下文环境，通过getEnvironment获取上下文位置，如果没有则上下文的环境默认为StandardEnvironment。
需要注意的是在修改应用上下文环境操作，应该在刷新上下文refresh操作之前。

再来看获取bean工厂
```java
/**
	 * 如果可利用，则返回上下文内部的bean工厂AutowireCapableBeanFactory
	 * @see #getBeanFactory()
	 */
	@Override
	public AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException {
		return getBeanFactory();
	}
    /**
	 * 子类必须他们内部的bean工厂。同时应该实现有效的查找，以便重复调用时，不会影响性能。
	 * 注意：在返回内部bean工厂之前，子类应该检查上下文是否处于激活状态。一旦上下文关闭，内部bean工厂将不可用。
	 * 如果上下文还没有只有内部bean工厂（通常情况下#refresh方法，还没有调用），或者上下文还没有关闭。
	 * @see #refreshBeanFactory()
	 * @see #closeBeanFactory()
	 */
	@Override
	public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```
从上面可以看，获取自动装配bean工厂AutowireCapableBeanFactory，实际委托给获取bean工厂方法getBeanFactory，getBeanFactory方法待子类扩展。

再来看发布应用事件：
```java
/**
 * Publish the given event to all listeners.
 * <p>Note: Listeners get initialized after the MessageSource, to be able
 * to access it within listener implementations. Thus, MessageSource
 * implementations cannot publish events.
 * 发布给定的应用事件给所有事件监听器。
 * 注意：在消息源可以访问其内部的监听器实现后，监听器才被初始化。因此消息源实现不能够发布应用事件。
 * @param event the event to publish (may be application-specific or a
 * standard framework event)
 * 可以是一个特殊的应用事件，也可以是标准框架事件
 */
@Override
public void publishEvent(ApplicationEvent event) {
	publishEvent(event, null);
}

/**
 * Publish the given event to all listeners.
 * <p>Note: Listeners get initialized after the MessageSource, to be able
 * to access it within listener implementations. Thus, MessageSource
 * implementations cannot publish events.
 * 发布给定的应用事件给所有事件监听器。
 * 注意：在消息源可以访问其内部的监听器实现后，监听器才被初始化。因此消息源实现不能够发布应用事件。
 * @param event the event to publish (may be an {@link ApplicationEvent}
 * or a payload object to be turned into a {@link PayloadApplicationEvent})
 * 可以是一个ApplicationEvent，亦可以是PayloadApplicationEvent
 */
@Override
public void publishEvent(Object event) {
	publishEvent(event, null);
}

/**
 * Publish the given event to all listeners.
 * 发布给定的事件到监听器
 * @param event the event to publish (may be an {@link ApplicationEvent}
 * or a payload object to be turned into a {@link PayloadApplicationEvent})
 * 事件可以为应用事件，获取负载对象，负载对象需要转化为PayloadApplicationEvent
 * @param eventType the resolved event type, if known
 * @since 4.2
 */
protected void publishEvent(Object event, ResolvableType eventType) {
	Assert.notNull(event, "Event must not be null");
	if (logger.isTraceEnabled()) {
		logger.trace("Publishing event in " + getDisplayName() + ": " + event);
	}

	// Decorate event as an ApplicationEvent if necessary
	ApplicationEvent applicationEvent;
	if (event instanceof ApplicationEvent) {
		applicationEvent = (ApplicationEvent) event;
	}
	else {
		//否则将事件对象，转化负载应用事件
		applicationEvent = new PayloadApplicationEvent<Object>(this, event);
		if (eventType == null) {
			eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
		}
	}

	// Multicast right now if possible - or lazily once the multicaster is initialized
	//如果预发布事件集不为空，则添加到应用事件到多播集
	if (this.earlyApplicationEvents != null) {
		this.earlyApplicationEvents.add(applicationEvent);
	}
	else {
		//应用事件多播器，多播事件
		getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
	}

	// Publish event via parent context as well...
	//发布事件到父上下文
	if (this.parent != null) {
		//如果父类为抽象应用上下文的实例，则直接调用publishEvent操作，发布事件。
		if (this.parent instanceof AbstractApplicationContext) {
			((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
		}
		else {
			//否则调用应用上下文的ApplicationEventPublisher发布事件。
			this.parent.publishEvent(event);
		}
	}
}
/**
 * Return the internal ApplicationEventMulticaster used by the context.
 * 获取上下文内部应用事件多播器ApplicationEventMulticaster。
 * @return the internal ApplicationEventMulticaster (never {@code null})
 * @throws IllegalStateException if the context has not been initialized yet
 */
ApplicationEventMulticaster getApplicationEventMulticaster() throws IllegalStateException {
	if (this.applicationEventMulticaster == null) {
		throw new IllegalStateException("ApplicationEventMulticaster not initialized - " +
				"call 'refresh' before multicasting events via the context: " + this);
	}
	return this.applicationEventMulticaster;
}
```
从上面可以看出，抽象应用上下文发布应用事件实际委托给内部的应用事件多播器，应用事件多播器在上下文刷新的时候初始化，如果配置有应用事件多播器
，则使用配置的应用事件多播器发布应用事件，否则默认使用SimpleApplicationEventMulticaster。在发布应用事件的时候，如果预发布事件集不为空，
则发布事件集中的应用事件，如果当上下文的父上下文不为空，则调用父上下文的事件发布操作，将事件传播给父上下文中的应用监听器。
应用事件多播器初始化，我们在应用上下文刷新时一起再讲。

我们再来抽象应用上下文关于ConfigurableApplicationContext接口的实现
```java
/**
 * Set the parent of this application context.
 * 设置父类上下文
 * <p>The parent {@linkplain ApplicationContext#getEnvironment() environment} is
 * {@linkplain ConfigurableEnvironment#merge(ConfigurableEnvironment) merged} with
 * this (child) application context environment if the parent is non-{@code null} and
 * its environment is an instance of {@link ConfigurableEnvironment}.
 * 如果父类上下文不为空，且其环境配置为ConfigurableEnvironment实例，则合并父环境实例到当前环境配置。
 * @see ConfigurableEnvironment#merge(ConfigurableEnvironment)
 */
@Override
public void setParent(ApplicationContext parent) {
	this.parent = parent;
	if (parent != null) {
		//如果父上下文不为空，则整合父上下文的配置环境到当前环境
		Environment parentEnvironment = parent.getEnvironment();
		if (parentEnvironment instanceof ConfigurableEnvironment) {
			getEnvironment().merge((ConfigurableEnvironment) parentEnvironment);
		}
	}
}
```
从上面可以看出，在设置上下文的父应用上文时，如果父上下文的环境配置为可配置环境，则合并父上下文环境到当前上下文环境。


```java
/**
 * 添加bean工厂后处理器到上下文
 */
@Override
public void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor) {
	Assert.notNull(postProcessor, "BeanFactoryPostProcessor must not be null");
	this.beanFactoryPostProcessors.add(postProcessor);
}
/**
 * 如果事件多播器不为空，则添加监听器到应用事件多播器，否则添加监听器到当前上下文应用监听器集。
 */
@Override
public void addApplicationListener(ApplicationListener<?> listener) {
	Assert.notNull(listener, "ApplicationListener must not be null");
	if (this.applicationEventMulticaster != null) {
		this.applicationEventMulticaster.addApplicationListener(listener);
	}
	else {
		this.applicationListeners.add(listener);
	}
}
```
从上面来看，添加bean工厂后处理器，实际为将bean工厂后处理器添加到上下文的bean工厂后处理器集中beanFactoryPostProcessors（ArrayList<BeanFactoryPostProcessor>）。添加应用监听器，实际上，是将监听器添加到上下文的监听器集合applicationListeners（LinkedHashSet<ApplicationListener<?>>）中。

下面我们讲到上下文的关键部分刷新应用上下文：
```java
/**
 * 加载应用上下文配置
 */
@Override
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		//准备上下文刷新操作
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		//告诉子类刷新内部bean工厂
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		//准备上下文使用的bean工厂
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			//在上下文子类中，允许后处理bean工厂。
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			//调用注册到上下文中的工厂后处理器
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			//注册bean后处理器拦截bean的创建
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			//初始化上下文的消息源
			initMessageSource();

			// Initialize event multicaster for this context.
			//初始化上下文的时间多播器
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			//初始化上下文子类中的其他特殊的bean
			onRefresh();

			// Check for listener beans and register them.
			//检查监听器bean，并注册
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			//初始化所有遗留的非懒加载单例bean
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			//发布相关事件
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			//销毁已经创建的单例bean，避免资源空占。
			destroyBeans();

			// Reset 'active' flag.
			//重置上下文激活状态标志
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			//由于我们不在需要单例bean的元数据，重置Spring核心的一般内省缓存。
			resetCommonCaches();
		}
	}
}
```
###
源码参见：[][]

[]: ""

```java
```


###
源码参见：[][]

[]: ""

```java
```


最后我们以BeanDefinition的类图结束这篇文章。
![BeanDefinition](/image/spring-context/BeanDefinition.png)

## 总结

我们可以通过setEnvironment配置上下文环境，通过getEnvironment获取上下文位置，如果没有则上下文的环境默认为StandardEnvironment。
需要注意的是在修改应用上下文环境操作，应该在刷新上下文refresh操作之前。

获取自动装配bean工厂AutowireCapableBeanFactory，实际委托给获取bean工厂方法getBeanFactory，getBeanFactory方法待子类扩展。


抽象应用上下文发布应用事件实际委托给内部的应用事件多播器，应用事件多播器在上下文刷新的时候初始化，如果配置有应用事件多播器
，则使用配置的应用事件多播器发布应用事件，否则默认使用SimpleApplicationEventMulticaster。在发布应用事件的时候，如果预发布事件集不为空，
则发布事件集中的应用事件，如果当上下文的父上下文不为空，则调用父上下文的事件发布操作，将事件传播给父上下文中的应用监听器。

在设置上下文的父应用上文时，如果父上下文的环境配置为可配置环境，则合并父上下文环境到当前上下文环境。

添加bean工厂后处理器，实际为将bean工厂后处理器添加到上下文的bean工厂后处理器集中beanFactoryPostProcessors（ArrayList<BeanFactoryPostProcessor>）。添加应用监听器，实际上，是将监听器添加到上下文的监听器集合applicationListeners（LinkedHashSet<ApplicationListener<?>>）中。
