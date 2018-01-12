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


12. 发布相关事件
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
12. 发布相关事件；
如果刷新过程出现异常则执行13,14步，
13. 销毁已经创建的单例bean，避免资源空占；
14. 置上下文激活状态标志；
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


12. 发布相关事件
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
准备上下文刷新操作主要初始化应用上下文环境中的占位符属性源，验证所有需要可解决的标注属性，创建预发布应用事件集earlyApplicationEvents（LinkedHashSet<ApplicationEvent>）。
初始化属性源方法 *#initPropertySources* 待子类实现。
2. 告诉子类刷新内部bean工厂；
通知子类刷新内部bean工厂实际操作在 *#refreshBeanFactory* 中，刷新bean工厂操作待子类扩展，在刷新完bean工厂之后，返回当前上下文的bean工厂，
返回当前上下文的bean工厂 *#getBeanFactory* 待子类实现。
3. 准备上下文使用的bean工厂；
4. 在上下文子类中，允许后处理bean工厂；
5. 调用注册到上下文中的工厂后处理器；
6. 注册拦截bean创建的bean后处理器；
7. 初始化上下文的消息源；
8. 始化上下文的事件多播器；
9. 初始化上下文子类中的其他特殊的bean；
10. 检查监听器bean，并注册；
11. 初始化所有遗留的非懒加载单例bean；
12. 发布相关事件；
如果刷新过程出现异常则执行13,14步，
13. 销毁已经创建的单例bean，避免资源空占；
14. 置上下文激活状态标志；
最后执行15步，
15. 由于我们不在需要单例bean的元数据，重置Spring核心的一般内省缓存。



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
