---
layout: page
title: my blog
subtitle: sub title
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

ClassPathResource内部有3变量，一个为类资源路径path（String），一个类机载器classLoader（ClassLoader），一个为资源类clazz（Class<?> ），
同时提供根据3个内部变量构成类路径资源的构造。获取类路径资源URL，如果资源类不为空，从资源类的类加载器获取资源，否则从从类加载器加载资源，如果还不能加载资源，则从从系统类加载器加载资源。针对类的加载器不存在的情况，则获取系统类加载器加载资源，如果系统类加载器为空，则使用Bootstrap类加载器加载资源。打开类路径资源输入流的思路和获取文件URL的方法类似，如果资源类不为空，从资源类的类加载器打开输入流，否则从类加载器打开输入流，如果类加载器为空，则从系统类加载器加载资源，打开输入流。打开类路径资源输入流，先获取类路径资源URL，在委托URL打开输入流。

默认资源加载器DefaultResourceLoader的根据给定位置加载资源的方法，当给定资源的位置以资源位置以"/"开头，加载的资源类型为ClassPathContextResource。
ClassPathContextResource表示一个上下文相对路径的类路径资源。

UrlResource内部有3个变量，一个为资源的URI，一个为资源URL，另外一个为干净的URL，提供提供了根据资源URL，URI和资源协议、位置、分片来构建UrlResource
资源的构造方法，获取资源输入流，及获取文件都是委托给内部的URL。

![ClassPathResource](/image/spring-context/ClassPathResource.png)

上述为我们上一篇[AbstractApplicationContext源码解析第二讲][]所讲的内容，今天我们正式进入

[AbstractApplicationContext源码解析第二讲]:https://donaldhan.github.io/spring-framework/2017/12/27/AbstractApplicationContext%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%AC%AC%E4%BA%8C%E8%AE%B2.html "AbstractApplicationContext源码解析第二讲"

上一篇文章我们看了，BeanDefinition接口的定义，截止到上一篇文章我们将应用上下文和可配置应用上下文已看完，从这篇文章开始，我们将进入应用上下文的实现。


# 目录
* [AbstractApplicationContext定义](abstractapplicationcontext定义)
    * [PathMatchingResourcePatternResolver](#pathmatchingresourcepatternresolver)
    * [DefaultLifecycleProcessor](#defaultlifecycleprocessor)
    * [ApplicationEventMulticaster](#applicationeventmulticaster)
    * [SimpleApplicationEventMulticaster](#simpleapplicationeventmulticaster)
    * [StandardEnvironment](#standardenvironment)
    * [DelegatingMessageSource](#delegatingmessagesource)
* [总结](#总结)

## AbstractApplicationContext定义
源码参见：[AbstractApplicationContext][]

[AbstractApplicationContext]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java "AbstractApplicationContext"

```java
package org.springframework.context.support;

import java.io.IOException;
import java.lang.annotation.Annotation;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.atomic.AtomicBoolean;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import org.springframework.beans.BeansException;
import org.springframework.beans.CachedIntrospectionResults;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.NoSuchBeanDefinitionException;
import org.springframework.beans.factory.config.AutowireCapableBeanFactory;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.support.ResourceEditorRegistrar;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.context.ApplicationEventPublisherAware;
import org.springframework.context.ApplicationListener;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.EmbeddedValueResolverAware;
import org.springframework.context.EnvironmentAware;
import org.springframework.context.HierarchicalMessageSource;
import org.springframework.context.LifecycleProcessor;
import org.springframework.context.MessageSource;
import org.springframework.context.MessageSourceAware;
import org.springframework.context.MessageSourceResolvable;
import org.springframework.context.NoSuchMessageException;
import org.springframework.context.PayloadApplicationEvent;
import org.springframework.context.ResourceLoaderAware;
import org.springframework.context.event.ApplicationEventMulticaster;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.context.event.ContextStartedEvent;
import org.springframework.context.event.ContextStoppedEvent;
import org.springframework.context.event.SimpleApplicationEventMulticaster;
import org.springframework.context.expression.StandardBeanExpressionResolver;
import org.springframework.context.weaving.LoadTimeWeaverAware;
import org.springframework.context.weaving.LoadTimeWeaverAwareProcessor;
import org.springframework.core.ResolvableType;
import org.springframework.core.convert.ConversionService;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.Environment;
import org.springframework.core.env.StandardEnvironment;
import org.springframework.core.io.DefaultResourceLoader;
import org.springframework.core.io.Resource;
import org.springframework.core.io.ResourceLoader;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.core.io.support.ResourcePatternResolver;
import org.springframework.util.Assert;
import org.springframework.util.ObjectUtils;
import org.springframework.util.ReflectionUtils;
import org.springframework.util.StringValueResolver;

/**
 * Abstract implementation of the {@link org.springframework.context.ApplicationContext}
 * interface. Doesn't mandate the type of storage used for configuration; simply
 * implements common context functionality. Uses the Template Method design pattern,
 * requiring concrete subclasses to implement abstract methods.
 *AbstractApplicationContext为应用上下文的抽象实现。不不托管配置的存储类型；仅仅实现的一般上下文的功能性。
 *使用了模板方法设计模式，需要具体的子类实现抽象方法。
 * <p>In contrast to a plain BeanFactory, an ApplicationContext is supposed
 * to detect special beans defined in its internal bean factory:
 * Therefore, this class automatically registers
 * {@link org.springframework.beans.factory.config.BeanFactoryPostProcessor BeanFactoryPostProcessors},
 * {@link org.springframework.beans.factory.config.BeanPostProcessor BeanPostProcessors}
 * and {@link org.springframework.context.ApplicationListener ApplicationListeners}
 * which are defined as beans in the context.
 * 相对于一个空白的bean工厂，一个应用上下文应该探测bean工厂内部特殊bean的定义：
 * 因此，子类将自动探测上下文中的bean工厂后处理器，bean后处理器，以及应用监听器定义bean。
 *
 * <p>A {@link org.springframework.context.MessageSource} may also be supplied
 * as a bean in the context, with the name "messageSource"; otherwise, message
 * resolution is delegated to the parent context. Furthermore, a multicaster
 * for application events can be supplied as "applicationEventMulticaster" bean
 * of type {@link org.springframework.context.event.ApplicationEventMulticaster}
 * in the context; otherwise, a default multicaster of type
 * {@link org.springframework.context.event.SimpleApplicationEventMulticaster} will be used.
 * 消息源应该在上下文中提供一个名字为messageSource的bean；否则将使用父上下文的消息源代理。进一步说，
 * 在上下文配置，应该提供一个类型为ApplicationEventMulticaster的应用事件多播bean定义。否则，
 * 默认的多播器SimpleApplicationEventMulticaster将会被使用。
 *
 * <p>Implements resource loading through extending
 * {@link org.springframework.core.io.DefaultResourceLoader}.
 * Consequently treats non-URL resource paths as class path resources
 * (supporting full class path resource names that include the package path,
 * e.g. "mypackage/myresource.dat"), unless the {@link #getResourceByPath}
 * method is overwritten in a subclass.
 * 抽象应用上下文通过继承DefaultResourceLoader实现资源的加载。因此除非子类重写{@link #getResourceByPath}方法，
 * 否则将一个非URL资源路径对待为一个类路径资源（支持包括包路径的全类路径资源name）。
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @author Mark Fisher
 * @author Stephane Nicoll
 * @since January 21, 2001
 * @see #refreshBeanFactory
 * @see #getBeanFactory
 * @see org.springframework.beans.factory.config.BeanFactoryPostProcessor
 * @see org.springframework.beans.factory.config.BeanPostProcessor
 * @see org.springframework.context.event.ApplicationEventMulticaster
 * @see org.springframework.context.ApplicationListener
 * @see org.springframework.context.MessageSource
 */
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

	/**
	 * Name of the MessageSource bean in the factory.
	 * If none is supplied, message resolution is delegated to the parent.
	 * 工厂中的消息源bean的name。
	 * 如果没有，则使用父类的消息解决器
	 * @see MessageSource
	 */
	public static final String MESSAGE_SOURCE_BEAN_NAME = "messageSource";

	/**
	 * Name of the LifecycleProcessor bean in the factory.
	 * If none is supplied, a DefaultLifecycleProcessor is used.
	 * 工厂内的声明周期处理器bean的name。没有默认使用DefaultLifecycleProcessor。
	 * @see org.springframework.context.LifecycleProcessor
	 * @see org.springframework.context.support.DefaultLifecycleProcessor
	 */
	public static final String LIFECYCLE_PROCESSOR_BEAN_NAME = "lifecycleProcessor";

	/**
	 * Name of the ApplicationEventMulticaster bean in the factory.
	 * If none is supplied, a default SimpleApplicationEventMulticaster is used.
	 * 工厂内应用事件多播bean的name。如果没有，默认为SimpleApplicationEventMulticaster。
	 * @see org.springframework.context.event.ApplicationEventMulticaster
	 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
	 */
	public static final String APPLICATION_EVENT_MULTICASTER_BEAN_NAME = "applicationEventMulticaster";


	static {
		// Eagerly load the ContextClosedEvent class to avoid weird classloader issues
		// on application shutdown in WebLogic 8.1. (Reported by Dustin Woods.)
		/*
		 * 预加载上下文关闭事件ContextClosedEvent，以避免在 WebLogic 8.1中应用关闭的时候，
		 * 引起意向不到的类加载器问题。
		 */
		ContextClosedEvent.class.getName();
	}


	/** Logger used by this class. Available to subclasses. */
	protected final Log logger = LogFactory.getLog(getClass());

	/** Unique id for this context, if any 应用上下文id*/
	private String id = ObjectUtils.identityToString(this);

	/** Display name 展示名*/
	private String displayName = ObjectUtils.identityToString(this);

	/** Parent context 父上下文*/
	private ApplicationContext parent;

	/** Environment used by this context 上下文环境*/
	private ConfigurableEnvironment environment;

	/** BeanFactoryPostProcessors to apply on refresh 在刷新上下文时，应用的bean工厂后处理器*/
	private final List<BeanFactoryPostProcessor> beanFactoryPostProcessors =
			new ArrayList<BeanFactoryPostProcessor>();

	/** System time in milliseconds when this context started 上下文启动的时间*/
	private long startupDate;

	/** Flag that indicates whether this context is currently active 上下文当前是否激活*/
	private final AtomicBoolean active = new AtomicBoolean();

	/** Flag that indicates whether this context has been closed already 上下文当前是否已经关闭*/
	private final AtomicBoolean closed = new AtomicBoolean();

	/** Synchronization monitor for the "refresh" and "destroy" 上下为刷新和销毁的同步监控对象*/
	private final Object startupShutdownMonitor = new Object();

	/** Reference to the JVM shutdown hook, if registered 如果注册，则引用与虚拟机关闭Hook*/
	private Thread shutdownHook;

	/** ResourcePatternResolver used by this context 上下文资源模式解决器*/
	private ResourcePatternResolver resourcePatternResolver;

	/** LifecycleProcessor for managing the lifecycle of beans within this context
	 * 管理上下文中的bean的生命周期Lifecycle的生命周期处理器LifecycleProcesso
	 * */
	private LifecycleProcessor lifecycleProcessor;

	/** MessageSource we delegate our implementation of this interface to
	 * 消息源接口实现的代理
	 * */
	private MessageSource messageSource;

	/** Helper class used in event publishing 事件发布Helper */
	private ApplicationEventMulticaster applicationEventMulticaster;

	/** Statically specified listeners 静态的应用监听器*/
	private final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<ApplicationListener<?>>();

	/** ApplicationEvents published early 预发布的应用事件*/
	private Set<ApplicationEvent> earlyApplicationEvents;


	/**
	 * Create a new AbstractApplicationContext with no parent.
	 */
	public AbstractApplicationContext() {
		this.resourcePatternResolver = getResourcePatternResolver();
	}

	/**
	 * Create a new AbstractApplicationContext with the given parent context.
	 * @param parent the parent context
	 */
	public AbstractApplicationContext(ApplicationContext parent) {
		this();
		setParent(parent);
	}


	//---------------------------------------------------------------------
	// Implementation of ApplicationContext interface
	//---------------------------------------------------------------------

	/**
	 * Set the unique id of this application context.
	 * <p>Default is the object id of the context instance, or the name
	 * of the context bean if the context is itself defined as a bean.
	 * @param id the unique id of the context
	 */
	@Override
	public void setId(String id) {
		this.id = id;
	}

	@Override
	public String getId() {
		return this.id;
	}

	@Override
	public String getApplicationName() {
		return "";
	}

	/**
	 * Set a friendly name for this context.
	 * Typically done during initialization of concrete context implementations.
	 * <p>Default is the object id of the context instance.
	 */
	public void setDisplayName(String displayName) {
		Assert.hasLength(displayName, "Display name must not be empty");
		this.displayName = displayName;
	}

	/**
	 * Return a friendly name for this context.
	 * @return a display name for this context (never {@code null})
	 */
	@Override
	public String getDisplayName() {
		return this.displayName;
	}

	/**
	 * Return the parent context, or {@code null} if there is no parent
	 * (that is, this context is the root of the context hierarchy).
	 * 返回父上下文
	 */
	@Override
	public ApplicationContext getParent() {
		return this.parent;
	}
   ...
}
```
从上面可以看出，抽象应用上下文 *AbstractApplicationContext* 实际为一个可配置上下文 *ConfigurableApplicationContext* 和可销毁的bean（DisposableBean），同时拥有了资源加载功能（DefaultResourceLoader）。我们通过一个唯一的id标注抽象上下文，同时抽象上下文拥有一个展示名。除此身份识别属性之前，抽象应用上下文，有一个父上下文 *ApplicationContext* ，可配的环境配置 *ConfigurableEnvironment* ，bean工厂后处理器集（List<BeanFactoryPostProcessor>），资源模式解决器（ResourcePatternResolver），声明周期处理器（LifecycleProcessor),消息源 *MessageSource* ，事件发布器 *ApplicationEventMulticaster* ，应用监听器集（LinkedHashSet<ApplicationListener<?>>），预发布的应用事件集（LinkedHashSet<ApplicationEvent>）。除了上述的功能性属性外，抽象应用上下文，还有一个一些状态属性，如果启动时间，激活状态（AtomicBoolean），关闭状态（AtomicBoolean）。最后还有一个上下为刷新和销毁的同步监控对象和一虚拟机关闭hook线程。

抽象上下文，有两个构造函数：

```java
/**
 * Create a new AbstractApplicationContext with no parent.
 */
public AbstractApplicationContext() {
    this.resourcePatternResolver = getResourcePatternResolver();
}

/**
 * Create a new AbstractApplicationContext with the given parent context.
 * @param parent the parent context
 */
public AbstractApplicationContext(ApplicationContext parent) {
    this();
    setParent(parent);
}
```
一个为代类上下文的构造，一个是无参构造，在无参构造中，初始化资源模式解决器。

```java
/**
	 * Return the ResourcePatternResolver to use for resolving location patterns
	 * into Resource instances. Default is a
	 * {@link org.springframework.core.io.support.PathMatchingResourcePatternResolver},
	 * supporting Ant-style location patterns.
	 * 获取解决资源实例位置模式的资源模式解决器。默认为{@link org.springframework.core.io.support.PathMatchingResourcePatternResolver}
	 * 支持，Ant风格位置模式。
	 * <p>Can be overridden in subclasses, for extended resolution strategies,
	 * for example in a web environment.
	 * 子类可以重写，拓展解决策略，比如web环境。
	 * <p><b>Do not call this when needing to resolve a location pattern.</b>
	 * Call the context's {@code getResources} method instead, which
	 * will delegate to the ResourcePatternResolver.
	 * 当需要解决位置匹配模式时，不要调用这个方法。调用上下文的 {@code getResources}，可以代理ResourcePatternResolver。
	 * @return the ResourcePatternResolver for this context
	 * @see #getResources
	 * @see org.springframework.core.io.support.PathMatchingResourcePatternResolver
	 */
	protected ResourcePatternResolver getResourcePatternResolver() {
		return new PathMatchingResourcePatternResolver(this);
	}
```
从上可以看出，抽象上下文的资源模式解决器为PathMatchingResourcePatternResolver，下面我们来看PathMatchingResourcePatternResolver。

### PathMatchingResourcePatternResolver
源码参见：[PathMatchingResourcePatternResolver][]

[PathMatchingResourcePatternResolver]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/io/support/PathMatchingResourcePatternResolver.java "PathMatchingResourcePatternResolver"

```java
```


### DefaultLifecycleProcessor
源码参见：[DefaultLifecycleProcessor][]

[DefaultLifecycleProcessor]: "DefaultLifecycleProcessor"

```java
```



### ApplicationEventMulticaster
源码参见：[ApplicationEventMulticaster][]

[ApplicationEventMulticaster]: "ApplicationEventMulticaster"

```java
```



### SimpleApplicationEventMulticaster
源码参见：[SimpleApplicationEventMulticaster][]

[SimpleApplicationEventMulticaster]: "SimpleApplicationEventMulticaster"

```java
```




### StandardEnvironment
源码参见：[StandardEnvironment][]

[StandardEnvironment]: "StandardEnvironment"

```java
```



### DelegatingMessageSource
源码参见：[DelegatingMessageSource][]

[DelegatingMessageSource]: "DelegatingMessageSource"

```java
```



最后我们以BeanDefinition的类图结束这篇文章。
![BeanDefinition](/image/spring-context/BeanDefinition.png)

## 总结

抽象应用上下文 *AbstractApplicationContext* 实际为一个可配置上下文 *ConfigurableApplicationContext* 和可销毁的bean（DisposableBean），同时拥有了资源加载功能（DefaultResourceLoader）。我们通过一个唯一的id标注抽象上下文，同时抽象上下文拥有一个展示名。除此身份识别属性之前，抽象应用上下文，有一个父上下文 *ApplicationContext* ，可配的环境配置 *ConfigurableEnvironment* ，bean工厂后处理器集（List<BeanFactoryPostProcessor>），资源模式解决器（ResourcePatternResolver），声明周期处理器（LifecycleProcessor),消息源 *MessageSource* ，事件发布器 *ApplicationEventMulticaster* ，应用监听器集（LinkedHashSet<ApplicationListener<?>>），预发布的应用事件集（LinkedHashSet<ApplicationEvent>）。除了上述的功能性属性外，抽象应用上下文，还有一个一些状态属性，如果启动时间，激活状态（AtomicBoolean），关闭状态（AtomicBoolean）。最后还有一个上下为刷新和销毁的同步监控对象和一虚拟机关闭hook线程。
