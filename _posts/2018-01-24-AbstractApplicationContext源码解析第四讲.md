---
layout: page
title: AbstractApplicationContext源码解析第四讲
subtitle: AbstractApplicationContext源码解析第四讲
date: 2018-01-24 22:56:19
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
    * [配置环境](#配置环境)
    * [加发布事件](#发布事件)
    * [加载应用上下文配置](#加载应用上下文配置)
* [总结](#总结)

## AbstractApplicationContext定义
源码参见：[AbstractApplicationContext][]

[AbstractApplicationContext]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java "AbstractApplicationContext"

### 配置环境
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

### 发布事件
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
### 加载应用上下文配置
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
			//注册拦截bean创建的bean后处理器
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
			//完成上下文刷新
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
刷新应用上下文方法，我们有一下几点需要关注

1. 准备上下文刷新操作
```java
// Prepare this context for refreshing.
//准备上下文刷新操作
prepareRefresh();
```

2. 告诉子类刷新内部bean工厂
```java
// Tell the subclass to refresh the internal bean factory.
//告诉子类刷新内部bean工厂
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

3. 准备上下文使用的bean工厂
```java
//准备上下文使用的bean工厂
prepareBeanFactory(beanFactory);
```
4. 在上下文子类中，允许后处理bean工厂
```java
// Allows post-processing of the bean factory in context subclasses.
//在上下文子类中，允许后处理bean工厂。
postProcessBeanFactory(beanFactory);
```

5. 调用注册到上下文中的工厂后处理器
```java
// Invoke factory processors registered as beans in the context.
//调用注册到上下文中的工厂后处理器
invokeBeanFactoryPostProcessors(beanFactory);
```
6. 注册拦截bean创建的bean后处理器
```java
// Register bean processors that intercept bean creation.
//注册拦截bean创建的bean后处理器
registerBeanPostProcessors(beanFactory);
```

7. 初始化上下文的消息源
```java
// Initialize message source for this context.
//初始化上下文的消息源
initMessageSource();
```

8. 始化上下文的事件多播器
```java
// Initialize event multicaster for this context.
//初始化上下文的事件多播器
initApplicationEventMulticaster();
```

9. 初始化上下文子类中的其他特殊的bean
```java
// Initialize other special beans in specific context subclasses.
//初始化上下文子类中的其他特殊的bean
onRefresh();
```

10. 检查监听器bean，并注册
```java
// Check for listener beans and register them.
//检查监听器bean，并注册
registerListeners();
```

11. 初始化所有遗留的非懒加载单例bean
```java
// Instantiate all remaining (non-lazy-init) singletons.
//初始化所有遗留的非懒加载单例bean
finishBeanFactoryInitialization(beanFactory);
```

12. 完成上下文刷新
```java
// Last step: publish corresponding event.
//发布相关事件
finishRefresh();
```

13. 销毁已经创建的单例bean，避免资源空占
```java
// Destroy already created singletons to avoid dangling resources.
//销毁已经创建的单例bean，避免资源空占。
destroyBeans();
```

14. 置上下文激活状态标志
```java
// Reset 'active' flag.
//重置上下文激活状态标志
cancelRefresh(ex);
```

15. 由于我们不在需要单例bean的元数据，重置Spring核心的一般内省缓存
```java
// Reset common introspection caches in Spring's core, since we
// might not ever need metadata for singleton beans anymore...
//由于我们不在需要单例bean的元数据，重置Spring核心的一般内省缓存。
resetCommonCaches();
```

从上面可以看出，应用上下文刷新的过程为：
1. 准备上下文刷新操作；
2. 告诉子类刷新内部bean工厂；
3. 准备上下文使用的bean工厂；
4. 在上下文子类中，允许后处理bean工厂；
5. 调用注册到上下文中的工厂后处理器；
6. 注册拦截bean创建的bean后处理器；
7. 初始化上下文的消息源；
8. 始化上下文的事件多播器；
9. 初始化上下文子类中的其他特殊的bean；
10. 检查监听器bean，并注册；
11. 初始化所有遗留的非懒加载单例bean；
12. 完成上下文刷新；
如果以上过程出现异常则执行13,14步，
13. 销毁已经创建的单例bean，避免资源空占；
14. 重置上下文激活状态标志；
最后执行15步，
15. 由于我们不在需要单例bean的元数据，重置Spring核心的一般内省缓存。

下面我们分别来看以上几点：
1. 准备上下文刷新操作
```java
// Prepare this context for refreshing.
//准备上下文刷新操作
prepareRefresh();
```
```java
/**
 * Prepare this context for refreshing, setting its startup date and
 * active flag as well as performing any initialization of property sources.
 */
protected void prepareRefresh() {
	//初始化上下文启动时间，更新上下文关闭与激活状态
	this.startupDate = System.currentTimeMillis();
	this.closed.set(false);
	this.active.set(true);

	if (logger.isInfoEnabled()) {
		logger.info("Refreshing " + this);
	}

	// Initialize any placeholder property sources in the context environment
	//初始化应用上下文环境中的占位符属性源
	initPropertySources();

	// Validate that all properties marked as required are resolvable
	//验证所有需要可解决的标注属性
	// see ConfigurablePropertyResolver#setRequiredProperties
	getEnvironment().validateRequiredProperties();

	// Allow for the collection of early ApplicationEvents,
	// to be published once the multicaster is available...
	//创建预发布应用事件集，一旦多播器可用，则发布事件
	this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
}
/**
 * <p>Replace any stub property sources with actual instances.
 * 使用实际的属性源实例替换所有存根属性源
 * @see org.springframework.core.env.PropertySource.StubPropertySource
 * @see org.springframework.web.context.support.WebApplicationContextUtils#initServletPropertySources
 */
protected void initPropertySources() {
	// For subclasses: do nothing by default.
}
```
从上面可以看，准备上下文刷新操作主要初始化应用上下文环境中的占位符属性源，验证所有需要可解决的标注属性，创建预发布应用事件集earlyApplicationEvents（LinkedHashSet<ApplicationEvent>）。初始化属性源方法 *#initPropertySources* 待子类实现。

2. 告诉子类刷新内部bean工厂
```java
// Tell the subclass to refresh the internal bean factory.
//告诉子类刷新内部bean工厂
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```
```java
/**
 * Tell the subclass to refresh the internal bean factory.
 * 通知子类刷新内部bean工厂
 * @return the fresh BeanFactory instance
 * @see #refreshBeanFactory()
 * @see #getBeanFactory()
 */
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	refreshBeanFactory();//刷新bean工厂，加载配置
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (logger.isDebugEnabled()) {
		logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
	}
	return beanFactory;
}
/**
 * Subclasses must implement this method to perform the actual configuration load.
 * 子类必须实现此方法，执行实际的配置加载。
 * The method is invoked by {@link #refresh()} before any other initialization work.
 * 在任何初始化工作前，此方法被{@link #refresh()}方法调用。
 * <p>A subclass will either create a new bean factory and hold a reference to it,
 * or return a single BeanFactory instance that it holds. In the latter case, it will
 * usually throw an IllegalStateException if refreshing the context more than once.
 * 子类要么创建一个新的bean工厂，并持有工厂引用，要么返回持有的bean工厂实例。在后一种情况下，
 * 如果刷新上下文多余一次，将抛出非法状态异常。
 * @throws BeansException if initialization of the bean factory failed
 * @throws IllegalStateException if already initialized and multiple refresh
 * attempts are not supported
 */
protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException;
/**
 * Subclasses must return their internal bean factory here. They should implement the
 * lookup efficiently, so that it can be called repeatedly without a performance penalty.
 * 子类必须他们内部的bean工厂。同时应该实现有效的查找，以便重复调用时，不会影响性能。
 * <p>Note: Subclasses should check whether the context is still active before
 * returning the internal bean factory. The internal factory should generally be
 * considered unavailable once the context has been closed.
 * 注意：在返回内部bean工厂之前，子类应该检查上下文是否处于激活状态。一旦上下文关闭，内部bean工厂将不可用。
 * @return this application context's internal bean factory (never {@code null})
 * @throws IllegalStateException if the context does not hold an internal bean factory yet
 * (usually if {@link #refresh()} has never been called) or if the context has been
 * closed already
 * 如果上下文还没有只有内部bean工厂（通常情况下#refresh方法，还没有调用），或者上下文还没有关闭。
 * @see #refreshBeanFactory()
 * @see #closeBeanFactory()
 */
@Override
public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```
从上面可以看出，通知子类刷新内部bean工厂实际操作在 *#refreshBeanFactory* 中，刷新bean工厂操作待子类扩展，在刷新完bean工厂之后，返回当前上下文的bean工厂，
返回当前上下文的bean工厂 *#getBeanFactory* 待子类实现。

3. 准备上下文使用的bean工厂
```java
//准备上下文使用的bean工厂
prepareBeanFactory(beanFactory);
```
```java
/**
 * Configure the factory's standard context characteristics,
 * such as the context's ClassLoader and post-processors.
 * 配置bean工厂的标准上下文特征，比如上下文加载器和后处理器
 * @param beanFactory the BeanFactory to configure
 */
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	// Tell the internal bean factory to use the context's class loader etc.
	beanFactory.setBeanClassLoader(getClassLoader());//设置bean工厂的bean加载器
	//设置bean表达式解决器
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
	//设置bean工厂的属性编辑注册器
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

	// Configure the bean factory with context callbacks.
	//配置bean工厂上下文回调，忽略上下文回调Aware相关接口
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

	// BeanFactory interface not registered as resolvable type in a plain factory.
	// MessageSource registered (and found for autowiring) as a bean.
	//bean工厂接口在空白工厂中，没有作为可解决类型。
	//消息源作为bean注册到工厂中，以便自动装配。
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);

	// Register early post-processor for detecting inner beans as ApplicationListeners.
	//注册bean后处理器，用于探测内部应用监听器bean。
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

	// Detect a LoadTimeWeaver and prepare for weaving, if found.
	//探测加载时间织入器，如果发现，准备织入。
	if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		// Set a temporary ClassLoader for type matching.
		//配置类型匹配的临时类加载器
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}

	// Register default environment beans.
	//注册默认的环境，系统属性，系统环境配置bean。
	if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
	}
}
```
从上面可以看出，预备bean工厂畅主要的是设置工厂类加载器，Spring EL表达式解决器 *StandardBeanExpressionResolver*，资源编辑器 *ResourceEditorRegistrar*；
添加bean后处理器 *ApplicationContextAwareProcessor* 处理与应用上下文关联的Awarebean实例，比如 *EnvironmentAware，EmbeddedValueResolverAware，ResourceLoaderAware
ApplicationEventPublisherAware，MessageSourceAware，ApplicationContextAware*，并忽略掉这些依赖。同时注册 *BeanFactory，ResourceLoader，ApplicationEventPublisher，
ApplicationContext* 类到工厂可解决类。添加应用监听器探测器，探测应用监听器bean定义，并添加到应用上下文中。如果bean工厂中包含加载时间织入器，则设置加载时间织入器后处理器
*LoadTimeWeaverAwareProcessor* ， 并配置类型匹配的临时类加载器。最后如果需要，注册环境，系统属性，系统变量单例bean到bean工厂。
再来看第四点。

4. 在上下文子类中，允许后处理bean工厂
```java
// Allows post-processing of the bean factory in context subclasses.
//在上下文子类中，允许后处理bean工厂。
postProcessBeanFactory(beanFactory);
```
```java
/**
 * Modify the application context's internal bean factory after its standard
 * initialization. All bean definitions will have been loaded, but no beans
 * will have been instantiated yet. This allows for registering special
 * BeanPostProcessors etc in certain ApplicationContext implementations.
 * 在标准初始化完成后，修改应用上下文内部的bean工厂。所有的bean定义已经加载，但是还没bean
 * 已完成初始化。允许在确定的应用上下文实现中注册特殊的bean后处理器。
 *
 * @param beanFactory the bean factory used by the application context
 */
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
}
```
方法postProcessBeanFactory在标准初始化完成后，修改应用上下文内部的bean工厂。所有的bean定义已经加载，但是还没bean已完成初始化。允许在确定的应用上下文实现中注册特殊的bean后处理器。postProcessBeanFactory待子类扩展。

5. 调用注册到上下文中的工厂后处理器
```java
// Invoke factory processors registered as beans in the context.
//调用注册到上下文中的工厂后处理器
invokeBeanFactoryPostProcessors(beanFactory);
```
```java
/**
 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
 * respecting explicit order if given.
 * <p>Must be called before singleton instantiation.
 * 初始化和调用所有注册的bean工厂后处理器bean，如果显示配置顺序，则按顺序初始化、调用。
 * 在单例模式bean初始化前，必须调用。
 */
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

	// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
	// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
	//如果同时探测到加载时间织入器，准备织入，比如通过ConfigurationClassPostProcessor注册的bean
	if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}
}
```
[PostProcessorRegistrationDelegate][]用于代理抽象上下文的后处理器处理操作。这一步所有的做的事情为调用bean工厂后处理器，处理bean工厂，具体过程为：如果为bean工厂为bean定义注册器实例，则调用上下文中的bean定义注册器后处理器,处理工厂，然后调用实现了PriorityOrdered，Ordered接口和剩余的bean定义注册器后处理器，处理bean工厂；如果bean工厂不是bean定义注册器实例，则使用上下文中的bean工厂后处理器，处理bean工厂。从bean工厂内获取所有bean工厂处理器实例，
按实现了PriorityOrdered，Ordered接口和剩余的bean工厂后处理器顺序，处理bean工厂；最后清空bean工厂的元数据缓存。
如果bean工厂的临时加载器为空，且包含加载时间织入器，则设置bean工厂的加载时间织入器后处理器，设置bean工厂的临时类加载器为[ContextTypeMatchClassLoader][]。

6. 注册拦截bean创建的bean后处理器
```java
// Register bean processors that intercept bean creation.
//注册拦截bean创建的bean后处理器
registerBeanPostProcessors(beanFactory);
```
```java
/**
 * Instantiate and invoke all registered BeanPostProcessor beans,
 * respecting explicit order if given.
 * <p>Must be called before any instantiation of application beans.
 * 如果注册的bean后处理器有顺序，则以顺序，初始化、调用注册的bean后处理器。
 * 必须在应用bean初始化前调用。
 */
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```
注册bean工厂的bean后处理操作，主要委托给[PostProcessorRegistrationDelegate][],注册过程为：
注册实现PriorityOrdered，Ordered和剩余的bean后处理器到bean工厂，然后注册内部bean后处理器[MergedBeanDefinitionPostProcessor][];
最后添加应用监听器探测器[ApplicationListenerDetector][]。

7. 初始化上下文的消息源
```java
// Initialize message source for this context.
//初始化上下文的消息源
initMessageSource();
```
```java
/**
 * Initialize the MessageSource.
 * Use parent's if none defined in this context.
 * 初始化消息源，如果上下文中没有定义，则使用父类的
 */
protected void initMessageSource() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
		//如果当前bean工厂配置了消息源，则获取消息源
		this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
		// Make MessageSource aware of parent MessageSource.
		//如果消息源为层级消息源，则设置消息源的父类消息源
		if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
			HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
			if (hms.getParentMessageSource() == null) {
				// Only set parent context as parent MessageSource if no parent MessageSource
				// registered already.
				hms.setParentMessageSource(getInternalParentMessageSource());
			}
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Using MessageSource [" + this.messageSource + "]");
		}
	}
	else {
		//如果当前bean工厂没有配置消息源，使用代理消息源，代理父类消息源，并注册到工厂中
		// Use empty MessageSource to be able to accept getMessage calls.
		DelegatingMessageSource dms = new DelegatingMessageSource();
		dms.setParentMessageSource(getInternalParentMessageSource());
		this.messageSource = dms;
		beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate MessageSource with name '" + MESSAGE_SOURCE_BEAN_NAME +
					"': using default [" + this.messageSource + "]");
		}
	}
}
```
从上面可以看出，如果bean工厂包含消息源，则配置为上下文消息源，如果当前上下文父上下文不为空且消息源为层级消息源HierarchicalMessageSource，
配置当前消息源的父消息源为当前上下文内部父消息源。否则创建上下文内部的父消息源的代理DelegatingMessageSource，并注册消息源到bean工厂。

8. 始化上下文的事件多播器
```java
// Initialize event multicaster for this context.
//初始化上下文的事件多播器
initApplicationEventMulticaster();
```
```java
/**
 * Initialize the ApplicationEventMulticaster.
 * Uses SimpleApplicationEventMulticaster if none defined in the context.
 * 初始化应用事件多播器，如果上下文中没有定义，则使用SimpleApplicationEventMulticaster
 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
 */
protected void initApplicationEventMulticaster() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
		this.applicationEventMulticaster =
				beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
		}
	}
	else {
		this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
		beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
					APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
					"': using default [" + this.applicationEventMulticaster + "]");
		}
	}
}
```
从上面可看出，初始化上下文的事件多播器过程为，如果bean工厂包含应用事件多播器，则配置应用上下文的应用事件多播器为bean工厂内的应用事件多播器；
否则使用SimpleApplicationEventMulticaster，并注册到bean工厂。

9. 初始化上下文子类中的其他特殊的bean
```java
// Initialize other special beans in specific context subclasses.
//初始化上下文子类中的其他特殊的bean
onRefresh();
```
```java
/**
 * Template method which can be overridden to add context-specific refresh work.
 * Called on initialization of special beans, before instantiation of singletons.
 * <p>This implementation is empty.
 * 此模板方法可以被重写用于添加上下文的特殊刷新工作。在初始化单例bean前调用，初始化一些特殊的bean。
 * @throws BeansException in case of errors
 * @see #refresh()
 */
protected void onRefresh() throws BeansException {
	// For subclasses: do nothing by default.
}
```
初始化上下文子类中的其他特殊的bean方法onRefresh，待子类扩展使用。

10. 检查监听器bean，并注册
```java
// Check for listener beans and register them.
//检查监听器bean，并注册
registerListeners();
```
```java
/**
 * Add beans that implement ApplicationListener as listeners.
 * Doesn't affect other listeners, which can be added without being beans.
 * 添加应用事件监听器bean
 */
protected void registerListeners() {
	// Register statically specified listeners first.
	//注册静态的特殊监听器先
	for (ApplicationListener<?> listener : getApplicationListeners()) {
		getApplicationEventMulticaster().addApplicationListener(listener);
	}

	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let post-processors apply to them!
	//在这里不要初始化工厂bean，我们需要使用后处理器应用者写没有初始化的bean。
	String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
	for (String listenerBeanName : listenerBeanNames) {
		getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
	}

	// Publish early application events now that we finally have a multicaster...
	//现在bean工行已经有一事件多播器，可以发布预应用事件
	Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
	this.earlyApplicationEvents = null;
	if (earlyEventsToProcess != null) {
		for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
			getApplicationEventMulticaster().multicastEvent(earlyEvent);
		}
	}
}
@Override
public String[] getBeanNamesForType(Class<?> type, boolean includeNonSingletons, boolean allowEagerInit) {
	assertBeanFactoryActive();
	return getBeanFactory().getBeanNamesForType(type, includeNonSingletons, allowEagerInit);
}
```
从上面可看出，注册监听器过程，主要是注册应用上下文中的应用监听器到应用事件多播器，注册bean工厂内部的应用监听器
到应用事件多播器，如果存在预发布应用事件，则多播应用事件。

11. 初始化所有遗留的非懒加载单例bean
```java
// Instantiate all remaining (non-lazy-init) singletons.
//初始化所有遗留的非懒加载单例bean
finishBeanFactoryInitialization(beanFactory);
```
```java
/**
 * Finish the initialization of this context's bean factory,
 * initializing all remaining singleton beans.
 * 完成上下文bean工厂的初始化，初始化所有遗留的单例bean。
 */
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// Initialize conversion service for this context.
	//初始化上下文转换服务
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}

	// Register a default embedded value resolver if no bean post-processor
	// (such as a PropertyPlaceholderConfigurer bean) registered any before:
	// at this point, primarily for resolution in annotation attribute values.
	/*
	 * 如果没有bean后处理器（PropertyPlaceholderConfigurer）注册，则注册一个默认的
	 * 嵌入式值解决器：主要解决注解属性的值。
	 */
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
			@Override
			public String resolveStringValue(String strVal) {
				return getEnvironment().resolvePlaceholders(strVal);
			}
		});
	}

	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	//初始化先前加载时间织入器Aware bean，允许注册他们的变换器。
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}

	// Stop using the temporary ClassLoader for type matching.
	//停止使用临时类加载器用于类型匹配
	beanFactory.setTempClassLoader(null);

	// Allow for caching all bean definition metadata, not expecting further changes.
	//允许缓存所有的bean定义元数据，不期望进一步的改变。
	beanFactory.freezeConfiguration();

	// Instantiate all remaining (non-lazy-init) singletons.
	//初始化所有剩余的单例bean
	beanFactory.preInstantiateSingletons();
}
@Override
public Object getBean(String name) throws BeansException {
	assertBeanFactoryActive();
	return getBeanFactory().getBean(name);
}
```
从上面可以看出：完成上下文bean工厂的初始化，初始化所有遗留的单例bean，如果bean工厂内存在转换服务ConversionService，则配置bean工厂的转换服务；
如果没有bean后处理器（PropertyPlaceholderConfigurer）注册，则注册一个默认的嵌入式值解决器StringValueResolver，主要解决注解属性的值；
初始化先前加载时间织入器Aware bean，允许注册他们的变换器；停止使用临时类加载器用于类型匹配；冻结bean工厂配置；最后初始化所有剩余的单例bean。

12. 完成上下文刷新
```java
// Last step: publish corresponding event.
//完成上下文刷新
finishRefresh();
```
```java
/**
 * Finish the refresh of this context, invoking the LifecycleProcessor's
 * onRefresh() method and publishing the
 * {@link org.springframework.context.event.ContextRefreshedEvent}.
 * 完成上下文刷新操作，调用生命周期处理器的onRefresh方法，并发布ContextRefreshedEvent事件
 */
protected void finishRefresh() {
	// Initialize lifecycle processor for this context.
	//初始化上下文生命周期处理器
	initLifecycleProcessor();
	// Propagate refresh to lifecycle processor first.
	getLifecycleProcessor().onRefresh();
	// Publish the final event.发布上下文刷新事件
	publishEvent(new ContextRefreshedEvent(this));
	// Participate in LiveBeansView MBean, if active.
    //注册上下文到MBean
	LiveBeansView.registerApplicationContext(this);
}
```
从上面可以看出，完成上下文刷新主要所有的工作为，初始化上下文的生命周期处理器，默认为 [DefaultLifecycleProcessor][],生命周期处理器主要管理bean工厂内部的生命周期bean。然后启动bean工厂内的生命周期bean。然后发布上下文刷新事件给应用监听器，最后注册上下文到[LiveBeansView][] MBean，以便监控上下文中bean工厂内部的bean的定义及依赖。

我们分别来看以上两个关键点：
```java
// Initialize lifecycle processor for this context.
//初始化上下文生命周期处理器
initLifecycleProcessor();
```
```java
/**
 * Initialize the LifecycleProcessor.
 * Uses DefaultLifecycleProcessor if none defined in the context.
 * @see org.springframework.context.support.DefaultLifecycleProcessor
 * 初始化生命周期处理器，没有则使用DefaultLifecycleProcessor
 */
protected void initLifecycleProcessor() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
		this.lifecycleProcessor =
				beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
		}
	}
	else {
		DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
		defaultProcessor.setBeanFactory(beanFactory);
		this.lifecycleProcessor = defaultProcessor;
		beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate LifecycleProcessor with name '" +
					LIFECYCLE_PROCESSOR_BEAN_NAME +
					"': using default [" + this.lifecycleProcessor + "]");
		}
	}
}
```
再来看第二点发布上下文刷新事件
```java
// Publish the final event.发布上下文刷新事件
publishEvent(new ContextRefreshedEvent(this));
```
```java
/* Publish the given event to all listeners.
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
```
    13. 销毁已经创建的单例bean，避免资源空占
```java
// Destroy already created singletons to avoid dangling resources.
//销毁已经创建的单例bean，避免资源空占。
destroyBeans();
```

```java
/**
 * Template method for destroying all beans that this context manages.
 * The default implementation destroy all cached singletons in this context,
 * invoking {@code DisposableBean.destroy()} and/or the specified
 * "destroy-method".
 * 此模板方法用于销毁所有上下文管理的bean。默认实现，调用{@code DisposableBean.destroy()}
 * 方法，或者特殊的"destroy-method"，销毁所有上下文中缓存的单例bean.
 * <p>Can be overridden to add context-specific bean destruction steps
 * right before or right after standard singleton destruction,
 * while the context's BeanFactory is still active.
 * 当上下文bean工厂仍处于激活状态，在标准的单例bean析构前或后，可以重写上下文特殊bean的析构
 * @see #getBeanFactory()
 * @see org.springframework.beans.factory.config.ConfigurableBeanFactory#destroySingletons()
 */
protected void destroyBeans() {
	getBeanFactory().destroySingletons();
}
```
从上面可以看出，销毁bean操作，主要委托给应用上文的内部bean工厂，bean工厂完成销毁所有的单例bean。

    14. 重置上下文激活状态标志
```java
// Reset 'active' flag.
//重置上下文激活状态标志
cancelRefresh(ex);
```
```java
/**
 * Cancel this context's refresh attempt, resetting the {@code active} flag
 * after an exception got thrown.
 * 取消上下文刷新尝试，在异常抛出后，重置激活状态标志。
 * @param ex the exception that led to the cancellation
 */
protected void cancelRefresh(BeansException ex) {
	this.active.set(false);
}
```

    15. 由于我们不在需要单例bean的元数据，重置Spring核心的一般内省缓存
```java
// Reset common introspection caches in Spring's core, since we
// might not ever need metadata for singleton beans anymore...
//由于我们不在需要单例bean的元数据，重置Spring核心的一般内省缓存。
resetCommonCaches();
```

```java
/**
 * Reset Spring's common core caches, in particular the {@link ReflectionUtils},
 * {@link ResolvableType} and {@link CachedIntrospectionResults} caches.
 * 重置spring的一般核心缓存，特别是 {@link ReflectionUtils},{@link ResolvableType},{@link CachedIntrospectionResults}
 * 中缓存。
 * @since 4.2
 * @see ReflectionUtils#clearCache()
 * @see ResolvableType#clearCache()
 * @see CachedIntrospectionResults#clearClassLoader(ClassLoader)
 */
protected void resetCommonCaches() {
	ReflectionUtils.clearCache();
	ResolvableType.clearCache();
	CachedIntrospectionResults.clearClassLoader(getClassLoader());
}
```
主要清除，类的方法与属性缓存，可解决类型缓存，及类缓存。

从上面可以看出，应用上下文刷新的过程为：
1. 准备上下文刷新操作；
准备上下文刷新操作主要初始化应用上下文环境中的占位符属性源，验证所有需要可解决的标注属性，创建预发布应用事件集earlyApplicationEvents（LinkedHashSet<ApplicationEvent>）。
初始化属性源方法 *#initPropertySources* 待子类实现。

2. 告诉子类刷新内部bean工厂；
通知子类刷新内部bean工厂实际操作在 *#refreshBeanFactory* 中，刷新bean工厂操作待子类扩展，在刷新完bean工厂之后，返回当前上下文的bean工厂，
返回当前上下文的bean工厂 *#getBeanFactory* 待子类实现。

3. 准备上下文使用的bean工厂；
预备bean工厂畅主要的是设置工厂类加载器，Spring EL表达式解决器 *StandardBeanExpressionResolver*，资源编辑器 *ResourceEditorRegistrar*；
添加bean后处理器 *ApplicationContextAwareProcessor* 处理与应用上下文关联的Awarebean实例，比如 *EnvironmentAware，EmbeddedValueResolverAware，ResourceLoaderAware
ApplicationEventPublisherAware，MessageSourceAware，ApplicationContextAware*，并忽略掉这些依赖。同时注册 *BeanFactory，ResourceLoader，ApplicationEventPublisher，
ApplicationContext* 类到工厂可解决类。添加应用监听器探测器，探测应用监听器bean定义，并添加到应用上下文中。如果bean工厂中包含加载时间织入器，则设置加载时间织入器后处理器
*LoadTimeWeaverAwareProcessor* ， 并配置类型匹配的临时类加载器。最后如果需要，注册环境，系统属性，系统变量单例bean到bean工厂。

4. 在上下文子类中，允许后处理bean工厂；
在标准初始化完成后，修改应用上下文内部的bean工厂。所有的bean定义已经加载，但是还没bean已完成初始化。
允许在确定的应用上下文实现中注册特殊的bean后处理器。方法postProcessBeanFactory待子类扩展。

5. 调用注册到上下文中的工厂后处理器；
这一步所有的做的事情为调用bean工厂后处理器，处理bean工厂，具体过程为：如果为bean工厂为bean定义注册器实例，
则调用上下文中的bean定义注册器后处理器,处理工厂，然后调用实现了PriorityOrdered，Ordered接口和剩余的bean定义注册器后处理器，处理bean工厂；
如果bean工厂不是bean定义注册器实例，则使用上下文中的bean工厂后处理器，处理bean工厂。从bean工厂内获取所有bean工厂处理器实例，
按实现了PriorityOrdered，Ordered接口和剩余的bean工厂后处理器顺序，处理bean工厂；最后清空bean工厂的元数据缓存。
如果bean工厂的临时加载器为空，且包含加载时间织入器，则设置bean工厂的加载时间织入器后处理器，设置bean工厂的临时类加载器为[ContextTypeMatchClassLoader][]。

6. 注册拦截bean创建的bean后处理器；
注册bean工厂的bean后处理操作，主要委托给[PostProcessorRegistrationDelegate][],注册过程为：
注册实现PriorityOrdered，Ordered和剩余的bean后处理器到bean工厂，然后注册内部bean后处理器[MergedBeanDefinitionPostProcessor][];
最后添加应用监听器探测器[ApplicationListenerDetector][]。

7. 初始化上下文的消息源；
如果bean工厂包含消息源，则配置为上下文消息源，如果当前上下文父上下文不为空且消息源为层级消息源HierarchicalMessageSource，
配置当前消息源的父消息源为当前上下文内部父消息源。否则创建上下文内部的父消息源的代理DelegatingMessageSource，并注册消息源到bean工厂。

8. 始化上下文的事件多播器；
初始化上下文的事件多播器过程为，如果bean工厂包含应用事件多播器，则配置应用上下文的应用事件多播器为bean工厂内的应用事件多播器；
否则使用SimpleApplicationEventMulticaster，并注册到bean工厂。

9. 初始化上下文子类中的其他特殊的bean；
初始化上下文子类中的其他特殊的bean方法onRefresh，待子类扩展使用。

10. 检查监听器bean，并注册；
注册监听器过程，主要是注册应用上下文中的应用监听器到应用事件多播器，注册bean工厂内部的应用监听器
到应用事件多播器，如果存在预发布应用事件，则多播应用事件。

11. 初始化所有遗留的非懒加载单例bean；
完成上下文bean工厂的初始化，初始化所有遗留的单例bean，如果bean工厂内存在转换服务ConversionService，则配置bean工厂的转换服务；
如果没有bean后处理器（PropertyPlaceholderConfigurer）注册，则注册一个默认的嵌入式值解决器StringValueResolver，主要解决注解属性的值；
初始化先前加载时间织入器Aware bean，允许注册他们的变换器；停止使用临时类加载器用于类型匹配；冻结bean工厂配置；最后初始化所有剩余的单例bean。

12. 完成上下文刷新；
完成上下文刷新主要所有的工作为，初始化上下文的生命周期处理器，默认为 [DefaultLifecycleProcessor][],生命周期处理器主要管理
bean工厂内部的生命周期bean。然后启动bean工厂内的生命周期bean。然后发布上下文刷新事件给应用监听器，最后注册上下文到[LiveBeansView][] MBean，
以便监控上下文中bean工厂内部的bean的定义及依赖。
如果以上过程出现异常则执行13,14步，

13. 销毁已经创建的单例bean，避免资源空占；
销毁bean操作，主要委托给应用上文的内部bean工厂，bean工厂完成销毁所有的单例bean。

14. 重置上下文激活状态标志；
最后执行15步，

15. 由于我们不在需要单例bean的元数据，重置Spring核心的一般内省缓存。
主要清除，类的方法与属性缓存，可解决类型缓存，及类缓存。

今天我们到此为止，剩下的部分，我们放在下一篇来讲。

最后我们以AbstractApplicationContext的类图结束这篇文章。
![AbstractApplicationContext](/image/spring-context/AbstractApplicationContext.png)

## 总结

我们可以通过setEnvironment配置上下文环境，通过getEnvironment获取上下文位置，如果没有则上下文的环境默认为StandardEnvironment。
需要注意的是在修改应用上下文环境操作，应该在刷新上下文refresh操作之前。

获取自动装配bean工厂AutowireCapableBeanFactory，实际委托给获取bean工厂方法getBeanFactory，getBeanFactory方法待子类扩展。


抽象应用上下文发布应用事件实际委托给内部的应用事件多播器，应用事件多播器在上下文刷新的时候初始化，如果配置有应用事件多播器
，则使用配置的应用事件多播器发布应用事件，否则默认使用SimpleApplicationEventMulticaster。在发布应用事件的时候，如果预发布事件集不为空，
则发布事件集中的应用事件，如果当上下文的父上下文不为空，则调用父上下文的事件发布操作，将事件传播给父上下文中的应用监听器。

在设置上下文的父应用上文时，如果父上下文的环境配置为可配置环境，则合并父上下文环境到当前上下文环境。

添加bean工厂后处理器，实际为将bean工厂后处理器添加到上下文的bean工厂后处理器集中beanFactoryPostProcessors（ArrayList<BeanFactoryPostProcessor>）。添加应用监听器，实际上，是将监听器添加到上下文的监听器集合applicationListeners（LinkedHashSet<ApplicationListener<?>>）中。

应用上下文刷新的过程为：
1. 准备上下文刷新操作；
准备上下文刷新操作主要初始化应用上下文环境中的占位符属性源，验证所有需要可解决的标注属性，创建预发布应用事件集earlyApplicationEvents（LinkedHashSet<ApplicationEvent>）。
初始化属性源方法 *#initPropertySources* 待子类实现。

2. 告诉子类刷新内部bean工厂；
通知子类刷新内部bean工厂实际操作在 *#refreshBeanFactory* 中，刷新bean工厂操作待子类扩展，在刷新完bean工厂之后，返回当前上下文的bean工厂，
返回当前上下文的bean工厂 *#getBeanFactory* 待子类实现。

3. 准备上下文使用的bean工厂；
预备bean工厂畅主要的是设置工厂类加载器，Spring EL表达式解决器 *StandardBeanExpressionResolver*，资源编辑器 *ResourceEditorRegistrar*；
添加bean后处理器 *ApplicationContextAwareProcessor* 处理与应用上下文关联的Awarebean实例，比如 *EnvironmentAware，EmbeddedValueResolverAware，ResourceLoaderAware
ApplicationEventPublisherAware，MessageSourceAware，ApplicationContextAware*，并忽略掉这些依赖。同时注册 *BeanFactory，ResourceLoader，ApplicationEventPublisher，
ApplicationContext* 类到工厂可解决类。添加应用监听器探测器，探测应用监听器bean定义，并添加到应用上下文中。如果bean工厂中包含加载时间织入器，则设置加载时间织入器后处理器
*LoadTimeWeaverAwareProcessor* ， 并配置类型匹配的临时类加载器。最后如果需要，注册环境，系统属性，系统变量单例bean到bean工厂。

4. 在上下文子类中，允许后处理bean工厂；
在标准初始化完成后，修改应用上下文内部的bean工厂。所有的bean定义已经加载，但是还没bean已完成初始化。
允许在确定的应用上下文实现中注册特殊的bean后处理器。方法postProcessBeanFactory待子类扩展。

5. 调用注册到上下文中的工厂后处理器；
这一步所有的做的事情为调用bean工厂后处理器，处理bean工厂，具体过程为：如果为bean工厂为bean定义注册器实例，
则调用上下文中的bean定义注册器后处理器,处理工厂，然后调用实现了PriorityOrdered，Ordered接口和剩余的bean定义注册器后处理器，处理bean工厂；
如果bean工厂不是bean定义注册器实例，则使用上下文中的bean工厂后处理器，处理bean工厂。从bean工厂内获取所有bean工厂处理器实例，
按实现了PriorityOrdered，Ordered接口和剩余的bean工厂后处理器顺序，处理bean工厂；最后清空bean工厂的元数据缓存。
如果bean工厂的临时加载器为空，且包含加载时间织入器，则设置bean工厂的加载时间织入器后处理器，设置bean工厂的临时类加载器为[ContextTypeMatchClassLoader][]。

6. 注册拦截bean创建的bean后处理器；
注册bean工厂的bean后处理操作，主要委托给[PostProcessorRegistrationDelegate][],注册过程为：
注册实现PriorityOrdered，Ordered和剩余的bean后处理器到bean工厂，然后注册内部bean后处理器[MergedBeanDefinitionPostProcessor][];
最后添加应用监听器探测器[ApplicationListenerDetector][]。

7. 初始化上下文的消息源；
如果bean工厂包含消息源，则配置为上下文消息源，如果当前上下文父上下文不为空且消息源为层级消息源HierarchicalMessageSource，
配置当前消息源的父消息源为当前上下文内部父消息源。否则创建上下文内部的父消息源的代理DelegatingMessageSource，并注册消息源到bean工厂。

8. 始化上下文的事件多播器；
初始化上下文的事件多播器过程为，如果bean工厂包含应用事件多播器，则配置应用上下文的应用事件多播器为bean工厂内的应用事件多播器；
否则使用SimpleApplicationEventMulticaster，并注册到bean工厂。

9. 初始化上下文子类中的其他特殊的bean；
初始化上下文子类中的其他特殊的bean方法onRefresh，待子类扩展使用。

10. 检查监听器bean，并注册；
注册监听器过程，主要是注册应用上下文中的应用监听器到应用事件多播器，注册bean工厂内部的应用监听器
到应用事件多播器，如果存在预发布应用事件，则多播应用事件。

11. 初始化所有遗留的非懒加载单例bean；
完成上下文bean工厂的初始化，初始化所有遗留的单例bean，如果bean工厂内存在转换服务ConversionService，则配置bean工厂的转换服务；
如果没有bean后处理器（PropertyPlaceholderConfigurer）注册，则注册一个默认的嵌入式值解决器StringValueResolver，主要解决注解属性的值；
初始化先前加载时间织入器Aware bean，允许注册他们的变换器；停止使用临时类加载器用于类型匹配；冻结bean工厂配置；最后初始化所有剩余的单例bean。

12. 完成上下文刷新；
完成上下文刷新主要所有的工作为，初始化上下文的生命周期处理器，默认为 [DefaultLifecycleProcessor][],生命周期处理器主要管理
bean工厂内部的生命周期bean。然后启动bean工厂内的生命周期bean。然后发布上下文刷新事件给应用监听器，最后注册上下文到[LiveBeansView][] MBean，
以便监控上下文中bean工厂内部的bean的定义及依赖。
如果以上过程出现异常则执行13,14步：

13. 销毁已经创建的单例bean，避免资源空占；
销毁bean操作，主要委托给应用上文的内部bean工厂，bean工厂完成销毁所有的单例bean。

14. 重置上下文激活状态标志；
最后执行15步，

15. 由于我们不在需要单例bean的元数据，重置Spring核心的一般内省缓存。
主要清除，类的方法与属性缓存，可解决类型缓存，及类缓存。


[PostProcessorRegistrationDelegate]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/PostProcessorRegistrationDelegate.java "PostProcessorRegistrationDelegate"  
[ContextTypeMatchClassLoader]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/ContextTypeMatchClassLoader.java "ContextTypeMatchClassLoader"


[MergedBeanDefinitionPostProcessor]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/support/MergedBeanDefinitionPostProcessor.java "MergedBeanDefinitionPostProcessor"   

[ApplicationListenerDetector]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/ApplicationListenerDetector.java "ApplicationListenerDetector"


[DefaultLifecycleProcessor]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/DefaultLifecycleProcessor.java "DefaultLifecycleProcessor"   

[LiveBeansView]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/LiveBeansView.java "LiveBeansView"  
