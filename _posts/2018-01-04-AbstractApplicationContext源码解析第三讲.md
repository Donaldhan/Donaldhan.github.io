---
layout: page
title: AbstractApplicationContext源码解析第三讲
subtitle: AbstractApplicationContext内部变量和构造，及默认的PathMatchingResourcePatternResolver，DefaultLifecycleProcessor
date: 2018-01-04 09:18:10
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
package org.springframework.core.io.support;

import java.io.File;
import java.io.IOException;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.net.JarURLConnection;
import java.net.MalformedURLException;
import java.net.URISyntaxException;
import java.net.URL;
import java.net.URLClassLoader;
import java.net.URLConnection;
import java.util.Arrays;
import java.util.Collections;
import java.util.Enumeration;
import java.util.LinkedHashSet;
import java.util.Set;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;
import java.util.zip.ZipException;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import org.springframework.core.io.DefaultResourceLoader;
import org.springframework.core.io.FileSystemResource;
import org.springframework.core.io.Resource;
import org.springframework.core.io.ResourceLoader;
import org.springframework.core.io.UrlResource;
import org.springframework.core.io.VfsResource;
import org.springframework.util.AntPathMatcher;
import org.springframework.util.Assert;
import org.springframework.util.ClassUtils;
import org.springframework.util.PathMatcher;
import org.springframework.util.ReflectionUtils;
import org.springframework.util.ResourceUtils;
import org.springframework.util.StringUtils;

/**
 * A {@link ResourcePatternResolver} implementation that is able to resolve a
 * specified resource location path into one or more matching Resources.
 * The source path may be a simple path which has a one-to-one mapping to a
 * target {@link org.springframework.core.io.Resource}, or alternatively
 * may contain the special "{@code classpath*:}" prefix and/or
 * internal Ant-style regular expressions (matched using Spring's
 * {@link org.springframework.util.AntPathMatcher} utility).
 * Both of the latter are effectively wildcards.
 * 路径匹配资源模式解决器PathMatchingResourcePatternResolver，可以解决特殊资源位置路径为一个或
 * 多个匹配的资源。源路径也许是一个简单的一对一的资源映射文件，或者包含特殊前缀（"{@code classpath*:}"），
 * 或Ant风格的表达式（可以使用{@link org.springframework.util.AntPathMatcher}匹配）。后面两种都是有效的
 * 通配符资源路径模式。
 * <p><b>No Wildcards:</b>
 *无通配符
 * <p>In the simple case, if the specified location path does not start with the
 * {@code "classpath*:}" prefix, and does not contain a PathMatcher pattern,
 * this resolver will simply return a single resource via a
 * {@code getResource()} call on the underlying {@code ResourceLoader}.
 * Examples are real URLs such as "{@code file:C:/context.xml}", pseudo-URLs
 * such as "{@code classpath:/context.xml}", and simple unprefixed paths
 * such as "{@code /WEB-INF/context.xml}". The latter will resolve in a
 * fashion specific to the underlying {@code ResourceLoader} (e.g.
 * {@code ServletContextResource} for a {@code WebApplicationContext}).
 *在一些简单的场景中，如果指定的位置路径不以{@code "classpath*:}"开始，同时不包含路径匹配模式，此方法将会使用底层的资源
 *加载器{@code ResourceLoader}调用{@code getResource()}方法，加载单个资源。实际的URL例子如，"{@code file:C:/context.xml}"，
 *伪URL"{@code classpath:/context.xml}"，简单的非前缀路径"{@code /WEB-INF/context.xml}"。最后以中情况将会
 *底层资源加载器（{@code ServletContextResource}），加载一个{@code WebApplicationContext}资源。
 * <p><b>Ant-style Patterns:</b>
 *
 * <p>When the path location contains an Ant-style pattern, e.g.:
 * 当路径位置包含Ant风格模式：
 * <pre class="code">
 * /WEB-INF/*-context.xml
 * com/mycompany/**&#47;applicationContext.xml
 * file:C:/some/path/*-context.xml
 * classpath:com/mycompany/**&#47;applicationContext.xml</pre>
 * 上述格式为Ant风格的模式路径位置。
 * the resolver follows a more complex but defined procedure to try to resolve
 * the wildcard. It produces a {@code Resource} for the path up to the last
 * non-wildcard segment and obtains a {@code URL} from it. If this URL is
 * not a "{@code jar:}" URL or container-specific variant (e.g.
 * "{@code zip:}" in WebLogic, "{@code wsjar}" in WebSphere", etc.),
 * then a {@code java.io.File} is obtained from it, and used to resolve the
 * wildcard by walking the filesystem. In the case of a jar URL, the resolver
 * either gets a {@code java.net.JarURLConnection} from it, or manually parses
 * the jar URL, and then traverses the contents of the jar file, to resolve the
 * wildcards.
 * 此解决器使用一个复杂但明确的程序，尝试处理通配符资源。将会通配符路径位置，加载路径位置中的非通配符路径下的资源URL。
 * 也就是获取通配符位置模式中的非通配符路径下的资源。如果URL是非"{@code jar:}" URL，或者包含WebLogic的"{@code zip:}"的
 * URL，或者WebSphere的"{@code wsjar}"形式，则将路径位置加载一个{@code java.io.File}URL，并解决文件系统中匹配通配符
 * 的文件资源。在jar包形式的URL中，解决器将获取一个 {@code java.net.JarURLConnection}，或手动解析Jar的URL，并检索jar包文件的
 * 内容，进一步加载匹配模式的资源。
 * <p><b>Implications on portability:</b>
 * 便利性
 * <p>If the specified path is already a file URL (either explicitly, or
 * implicitly because the base {@code ResourceLoader} is a filesystem one,
 * then wildcarding is guaranteed to work in a completely portable fashion.
 *如果给定的路径已经为一个文件URL（因为资源加载器是基于文件系统，显式或隐式），匹配过程可以保证完全的轻量级地工作。
 * <p>If the specified path is a classpath location, then the resolver must
 * obtain the last non-wildcard path segment URL via a
 * {@code Classloader.getResource()} call. Since this is just a
 * node of the path (not the file at the end) it is actually undefined
 * (in the ClassLoader Javadocs) exactly what sort of a URL is returned in
 * this case. In practice, it is usually a {@code java.io.File} representing
 * the directory, where the classpath resource resolves to a filesystem
 * location, or a jar URL of some sort, where the classpath resource resolves
 * to a jar location. Still, there is a portability concern on this operation.
 *如果给定的路径是类路径位置，将调用{@code Classloader.getResource()}方法，获取类路径位置的非通配符部分
 *URL对应的资源。实际上，返回的URL对应的文件表示一个目录，将会解决类路径资源为一个文件系统位置，或者一个Jar包URL。
 * <p>If a jar URL is obtained for the last non-wildcard segment, the resolver
 * must be able to get a {@code java.net.JarURLConnection} from it, or
 * manually parse the jar URL, to be able to walk the contents of the jar,
 * and resolve the wildcard. This will work in most environments, but will
 * fail in others, and it is strongly recommended that the wildcard
 * resolution of resources coming from jars be thoroughly tested in your
 * specific environment before you rely on it.
 *如果获取类路径位置的非通配符部分对应为jar URL, 解决器必须可以从URL获取{@code java.net.JarURLConnection}，
 *或手动解析jar 的URL，以便可以遍历jar的内容，解决匹配资源。在大多数的环境下，将会起效，但在一些特殊的情况下，
 *可能失败，强烈建议，在环境中依赖的jar包的通配符资源在使用前，要充分的测试。
 * <p><b>{@code classpath*:} Prefix:</b>
 *{@code classpath*:}前缀：
 * <p>There is special support for retrieving multiple class path resources with
 * the same name, via the "{@code classpath*:}" prefix. For example,
 * "{@code classpath*:META-INF/beans.xml}" will find all "beans.xml"
 * files in the class path, be it in "classes" directories or in JAR files.
 * This is particularly useful for autodetecting config files of the same name
 * at the same location within each jar file. Internally, this happens via a
 * {@code ClassLoader.getResources()} call, and is completely portable.
 *此解决器通过"{@code classpath*:}"前缀，可以支持检索类路径资源下的相同name对应的资源。比如：
 *"{@code classpath*:META-INF/beans.xml}"，将会查找所有类路径下的"beans.xml"文件，可以为
 * "classes"目录或者 jar文件。此种通配符配置模式对自动探测每个jar包中的相同name对应配置文件，非常有效。
 * 内部将会使用{@code ClassLoader.getResources()}方法调用，加载资源，非常便利。
 * <p>The "classpath*:" prefix can also be combined with a PathMatcher pattern in
 * the rest of the location path, for example "classpath*:META-INF/*-beans.xml".
 * In this case, the resolution strategy is fairly simple: a
 * {@code ClassLoader.getResources()} call is used on the last non-wildcard
 * path segment to get all the matching resources in the class loader hierarchy,
 * and then off each resource the same PathMatcher resolution strategy described
 * above is used for the wildcard subpath.
 * "classpath*:"前缀可以在位置路径的余下部分结合路径匹配模式使用，比如："classpath*:META-INF/*-beans.xml"。
 * 在这种情况，解决策略是简单公平的：{@code ClassLoader.getResources()}方法将会加载类路径位置的非通配符部分位置下的所有
 * 匹配模式的资源文件，包括子目录。
 * <p><b>Other notes:</b>
 *其他说明：
 * <p><b>WARNING:</b> Note that "{@code classpath*:}" when combined with
 * Ant-style patterns will only work reliably with at least one root directory
 * before the pattern starts, unless the actual target files reside in the file
 * system. This means that a pattern like "{@code classpath*:*.xml}" will
 * <i>not</i> retrieve files from the root of jar files but rather only from the
 * root of expanded directories. This originates from a limitation in the JDK's
 * {@code ClassLoader.getResources()} method which only returns file system
 * locations for a passed-in empty String (indicating potential roots to search).
 * This {@code ResourcePatternResolver} implementation is trying to mitigate the
 * jar root lookup limitation through {@link URLClassLoader} introspection and
 * "java.class.path" manifest evaluation; however, without portability guarantees.
 *需要注意的是：当"{@code classpath*:}"与Ant风格的模式一起使用时，在模式开始前，必须有至少一个根目录，除非实际的
 *目标文件存在文件系统中。这者意味着"{@code classpath*:*.xml}"将不会检索jar包的根目录，而是拓展目录的根目录。
 *这是源于JDK{@code ClassLoader.getResources()}只能为空字符串返回文件系统位置（预示者潜在的更目录搜索）。
 *当前ResourcePatternResolver的实现，尝试通过{@link URLClassLoader}检查和"java.class.path"的显示评估，
 *减轻jar包根目录的寻找限制，然而没有便利性保证。
 * <p><b>WARNING:</b> Ant-style patterns with "classpath:" resources are not
 * guaranteed to find matching resources if the root package to search is available
 * in multiple class path locations. This is because a resource such as
 * 注意："classpath:"形式的Ant风格资源，如果在多类路径位置下，没有根包可以搜索使用，不能保证发现可利用的资源。
 * 比如如下资源位置路径。
 * <pre class="code">
 *     com/mycompany/package1/service-context.xml
 * </pre>
 * may be in only one location, but when a path such as
 * 也许只有一个位置，但是当路径为如下时：
 * <pre class="code">
 *     classpath:com/mycompany/**&#47;service-context.xml
 * </pre>
 * is used to try to resolve it, the resolver will work off the (first) URL
 * returned by {@code getResource("com/mycompany");}. If this base package node
 * exists in multiple classloader locations, the actual end resource may not be
 * underneath. Therefore, preferably, use "{@code classpath*:}" with the same
 * Ant-style pattern in such a case, which will search <i>all</i> class path
 * locations that contain the root package.
 * 第一种资源URL将通过{@code getResource("com/mycompany");}来解决。如果在多个类加载器位置中存在基础包节点，
 * 实际的资源也许不再其下面。因此，最好使用"{@code classpath*:}"形式的Ant风格路径模式位置，在这种情况下，可以搜索
 * 所有包含根包的类型京位置。
 *
 * @author Juergen Hoeller
 * @author Colin Sampaleanu
 * @author Marius Bogoevici
 * @author Costin Leau
 * @author Phil Webb
 * @since 1.0.2
 * @see #CLASSPATH_ALL_URL_PREFIX
 * @see org.springframework.util.AntPathMatcher
 * @see org.springframework.core.io.ResourceLoader#getResource(String)
 * @see ClassLoader#getResources(String)
 */
public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {

	private static final Log logger = LogFactory.getLog(PathMatchingResourcePatternResolver.class);

	private static Method equinoxResolveMethod;

	static {
		try {
			// Detect Equinox OSGi (e.g. on WebSphere 6.1)
			//加载文件类型org.eclipse.core.runtime.FileLocator
			Class<?> fileLocatorClass = ClassUtils.forName("org.eclipse.core.runtime.FileLocator",
					PathMatchingResourcePatternResolver.class.getClassLoader());
			//获取org.eclipse.core.runtime.FileLocator的参数为URL的resolve方法。
			equinoxResolveMethod = fileLocatorClass.getMethod("resolve", URL.class);
			logger.debug("Found Equinox FileLocator for OSGi bundle URL resolution");
		}
		catch (Throwable ex) {
			equinoxResolveMethod = null;
		}
	}


	private final ResourceLoader resourceLoader;//内部资源加载器

	private PathMatcher pathMatcher = new AntPathMatcher();

	/**
	 * Create a new PathMatchingResourcePatternResolver with a DefaultResourceLoader.
	 * <p>ClassLoader access will happen via the thread context class loader.
	 * 根据默认的资源类型创建一个新的路径匹配资源模式解决器。
	 * @see org.springframework.core.io.DefaultResourceLoader
	 */
	public PathMatchingResourcePatternResolver() {
		this.resourceLoader = new DefaultResourceLoader();
	}

	/**
	 * Create a new PathMatchingResourcePatternResolver.
	 * <p>ClassLoader access will happen via the thread context class loader.
	 * 根据给定的资源加载器创建一个新的路径匹配资源模式解决器。
	 * @param resourceLoader the ResourceLoader to load root directories and
	 * actual resources with
	 */
	public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {
		Assert.notNull(resourceLoader, "ResourceLoader must not be null");
		this.resourceLoader = resourceLoader;
	}

	/**
	 * Create a new PathMatchingResourcePatternResolver with a DefaultResourceLoader.
	 * 根据给定的类加载器，创建一个新的路径匹配资源模式解决器。
	 * @param classLoader the ClassLoader to load classpath resources with,
	 * or {@code null} for using the thread context class loader
	 * at the time of actual resource access
	 * @see org.springframework.core.io.DefaultResourceLoader
	 */
	public PathMatchingResourcePatternResolver(ClassLoader classLoader) {
		this.resourceLoader = new DefaultResourceLoader(classLoader);
	}
    @Override
	public Resource getResource(String location) {
		return getResourceLoader().getResource(location);
	}

	@Override
	public Resource[] getResources(String locationPattern) throws IOException {
		Assert.notNull(locationPattern, "Location pattern must not be null");
		//"classpath*:"前缀路径位置模式
		if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
			// a class path resource (multiple resources for same name possible)
			//"classpath*:"前缀，Ant风格的类路径资源
			if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
				// a class path resource pattern
				//加载位置模式下的类路径资源"classpath*:"
				return findPathMatchingResources(locationPattern);
			}
			else {
				// all class path resources with the given name
				//所有给定name的类路径资源
				return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
			}
		}
		else {
			// Generally only look for a pattern after a prefix here,
			// and on Tomcat only after the "*/" separator for its "war:" protocol.
			//一般情况下，加载路径模式剔除前缀后的模式路径，在tomcat中，对于"war:"协议，为"*/" 后
		    //的路径模式。
			int prefixEnd = (locationPattern.startsWith("war:") ? locationPattern.indexOf("*/") + 1 :
					locationPattern.indexOf(":") + 1);
			if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
				// a file pattern
				//加载文件系统下文件资源
				return findPathMatchingResources(locationPattern);
			}
			else {
				// a single resource with the given name
				//加载给定资源name对应的单个资源
				return new Resource[] {getResourceLoader().getResource(locationPattern)};
			}
		}
	}
 }
 ...
}
```
从上面可以看出，路径匹配资源模式解决器PathMatchingResourcePatternResolver内部有一个Ant路径匹配器 *AntPathMatcher*，和一个资源类加载器，资源加载器可以
使用所属上下文中的资源加载器，也可以为给定类加载器的DefaultResourceLoader。路径匹配资源模式解决器主要提供了加载给定路径位置的资源方法，此方法可以解决无通配符的路径位置模式
（{@code file:C:/context.xml}，{@code classpath:/context.xml}，{@code /WEB-INF/context.xml}"），也可以解决包含Ant风格的通配符路径位置模式资源（{@code classpath*:META-INF/beans.xml}），主要以classpath*为前缀的路径位置模式，资源加载器将会查找类路径下所有相同name对应的资源文件，包括子目录和jar包。如果明确的加载资源，
可以使用{@code classpath:/context.xml}形式路径模式，如果想要探测类路径下的所有name对应的资源文件，可以使用形式路径模式。


为了便于理解抽象应用上下文，我们再来看一下DefaultLifecycleProcessor

### DefaultLifecycleProcessor
源码参见：[DefaultLifecycleProcessor][]

[DefaultLifecycleProcessor]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/DefaultLifecycleProcessor.java "DefaultLifecycleProcessor"

在看DefaultLifecycleProcessor之前，为了便于理解，我们来看器父接口，及关联接口的定义：
#### BeanFactoryAware
源码参见：[BeanFactoryAware][]

[BeanFactoryAware]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/BeanFactoryAware.java "BeanFactoryAware"
```java
package org.springframework.beans.factory;

import org.springframework.beans.BeansException;

/**
 *BeanFactoryAware接口的实现bean，可以获取其所属的bean工厂。
 *比如，bean可以通过bean工厂搜索协作的bean（依赖搜索）。注意大多数的bean将会选择bean属性or构造形式的
 *依赖注入协作bean引用形式。
 * 所有bean的声明周期方法，见@link BeanFactory BeanFactory javadocs}。
 * @author Rod Johnson
 * @author Chris Beams
 * @since 11.03.2003
 * @see BeanNameAware
 * @see BeanClassLoaderAware
 * @see InitializingBean
 * @see org.springframework.context.ApplicationContextAware
 */
public interface BeanFactoryAware extends Aware {

	/**
	 * Callback that supplies the owning factory to a bean instance.
	 *提供所属bean工厂对应的bean实例回调。
	 * <p>Invoked after the population of normal bean properties
	 * but before an initialization callback such as
	 * {@link InitializingBean#afterPropertiesSet()} or a custom init-method.
	 * 在正常的bean属性初始化后，但在{@link InitializingBean#afterPropertiesSet()}初始化回调或者
	 * 一般的初始化方法之前。
	 * @param beanFactory owning BeanFactory (never {@code null}).
	 * The bean can immediately call methods on the factory.
	 * @throws BeansException in case of initialization errors
	 * @see BeanInitializationException
	 */
	void setBeanFactory(BeanFactory beanFactory) throws BeansException;

}
```
从上面可以看出，BeanFactoryAware接口主要提供设置bean工厂操作。
#### LifecycleProcessor
源码参见：[LifecycleProcessor][]

[LifecycleProcessor]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/LifecycleProcessor.java "LifecycleProcessor"

```java
package org.springframework.context;

/**
 * Strategy interface for processing Lifecycle beans within the ApplicationContext.
 * LifecycleProcessor用于处理应用上下文中的生命周期bean
 *
 * @author Mark Fisher
 * @author Juergen Hoeller
 * @since 3.0
 */
public interface LifecycleProcessor extends Lifecycle {

	/**
	 * Notification of context refresh, e.g. for auto-starting components.
	 * 通知上下文刷新，比如自动启动组件
	 */
	void onRefresh();
	/**
	 * Notification of context close phase, e.g. for auto-stopping components.
	 * 通知上下文关闭阶段，比如自动停止组件
	 */
	void onClose();

}
```

从上面可以看，LifecycleProcessor接口主要提供了通知上下文刷新和关闭的操作。


#### SmartLifecycle
源码参见：[SmartLifecycle][]

[SmartLifecycle]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/SmartLifecycle.java "SmartLifecycle"

```java
package org.springframework.context;

/**
 *SmartLifecycle是生命周期接口的拓展，用于需要在应用上下文刷新和关闭时，以特地的顺序启动的对象。
 *{@link #isAutoStartup()}方法，返回值预示着对象是否应该在上下文刷新的时候启动。回调{@link #stop(Runnable)}方法，
 *用于需要异步关闭的对象。此方法的任何实现，在完全关闭时，必须调用回调线程的run方法，以避免上下文关闭时，不必要的延时。
 * 此接口拓展了{@link Phased}接口，{@link #getPhase()}方法返回的值，预示着生命周期组件应该启动还是停止状态过程的阶段值。
 * 启动过程以一个低的阶段值开始，并以一个高的阶段值结束（Integer.MIN_VALUE是最低的，Integer.MAX_VALUE为最高）。
 * 关闭过程则相反。所有拥有相同阶段值的组件应该在相同阶段强制排序。
 *
 *比如：如果组件B依赖组件A，组件A已经启动，组件A拥有比组件B小的阶段值。在关闭过程中，组件B将会在组件A之前关闭。
 * 任何显示的依赖关系将会优先考虑启动阶段顺序，依赖bean将会在被依赖的bean启动后，启动；在被依赖的bean关闭前，
 * 关闭。
 *
 *任何没有实现SmartLifecycle接口的上下文中的生命周期组件，阶段值将会以0对待。实现SmartLifecycle的组件如果阶段值为负，
 *也许将会在生命周期组件之前启动，也许在拥有正阶段值的组件后启动。
 *
 * 需要注意的是：由于SmartLifecycle支持自动启动，一个SmartLifecycle bean实例无论如何将会在上下文的启动的过程中
 * 初始化。因此bean定义的懒加载表示将会限制SmartLifecycle bean的实际效果。
 *
 * @author Mark Fisher
 * @since 3.0
 * @see LifecycleProcessor
 * @see ConfigurableApplicationContext
 */
public interface SmartLifecycle extends Lifecycle, Phased {

	/**
	 * 在应用上下文容器刷新时，如果容器中的生命周期组件自动启动，则此方法返回true
	 * 返回false，预示者组件需要显示调用{@link #start()} 方法启动，类似于空白的生命周期实现。
	 * @see #start()
	 * @see #getPhase()
	 * @see LifecycleProcessor#onRefresh()
	 * @see ConfigurableApplicationContext#refresh()
	 */
	boolean isAutoStartup();

	/**
	 * 如果组件当前正在运行，调用此方法表示生命周期组件必须停止。
	 * 此回调用于支持生命周期处理器{@link LifecycleProcessor}顺序，潜在并发，以一般的顺序值关闭所有组件。
	 * 此回调必须在{@code SmartLifecycle}组件实际停止后执行。
	 * 生命周期处理器仅仅回调用此方法的变体，比如{@link Lifecycle#stop()} 不会调用{@code SmartLifecycle}的实现，
	 * 除非显示地代理此方法的内部实现。
	 * @see #stop()
	 * @see #getPhase()
	 */
	void stop(Runnable callback);

}
```
从上面来看，SmartLifecycle接口主要提供关闭回调操作，在组件停止后，调用回调接口。并提供了判断组件在容器上下文刷新时，组件是否自动刷新的操作。

#### Phased
源码参见：[Phased][]

[Phased]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/Phased.java "Phased"
```java
package org.springframework.context;

/**
 *Phased用于表示可以参加阶段处理过程的对象，比如生命周期管理。
 * @author Mark Fisher
 * @since 3.0
 * @see SmartLifecycle
 */
public interface Phased {
	/**
	 * 返回对象所处的阶段值
	 */
	int getPhase();

}
```
从上面可以看出，Phased主要提供了获取组件阶段值操作。

现在我们回到DefaultLifecycleProcessor


```java
package org.springframework.context.support;

import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.BeanFactoryUtils;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.context.ApplicationContextException;
import org.springframework.context.Lifecycle;
import org.springframework.context.LifecycleProcessor;
import org.springframework.context.Phased;
import org.springframework.context.SmartLifecycle;

/**
 * Default implementation of the {@link LifecycleProcessor} strategy.
 *LifecycleProcessor策略的默认实现。
 * @author Mark Fisher
 * @author Juergen Hoeller
 * @since 3.0
 */
public class DefaultLifecycleProcessor implements LifecycleProcessor, BeanFactoryAware {

	private final Log logger = LogFactory.getLog(getClass());

	private volatile long timeoutPerShutdownPhase = 30000;//每次关闭的超时时间

	private volatile boolean running;//是否正在运行

	private volatile ConfigurableListableBeanFactory beanFactory;//所属bean工厂


	/**
	 * Specify the maximum time allotted in milliseconds for the shutdown of
	 * any phase (group of SmartLifecycle beans with the same 'phase' value).
	 * The default value is 30 seconds.
	 */
	public void setTimeoutPerShutdownPhase(long timeoutPerShutdownPhase) {
		this.timeoutPerShutdownPhase = timeoutPerShutdownPhase;
	}
    /**
     * 设置bean工厂
     */
	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		if (!(beanFactory instanceof ConfigurableListableBeanFactory)) {
			throw new IllegalArgumentException(
					"DefaultLifecycleProcessor requires a ConfigurableListableBeanFactory: " + beanFactory);
		}
		this.beanFactory = (ConfigurableListableBeanFactory) beanFactory;
	}

    /**
     * Lifecycle接口实现
     */
	// Lifecycle implementation

	/**
	 * Start all registered beans that implement Lifecycle and are
	 * <i>not</i> already running. Any bean that implements SmartLifecycle
	 * will be started within its 'phase', and all phases will be ordered
	 * from lowest to highest value. All beans that do not implement
	 * SmartLifecycle will be started in the default phase 0. A bean
	 * declared as a dependency of another bean will be started before
	 * the dependent bean regardless of the declared phase.
	 * 启动所有实现生命周期接口的还未运行的注册bean。任何实现SmartLifecycle接口的bean，
	 * 将会以其阶段值'phase'启动，所有阶段性生命周期组件将会从低到高顺序启动。所有没有实现
	 * SmartLifecycle接口的bean，将会以默认阶段值0启动。bean依赖的其他bean，将会在宿主bean
	 * 启动前，启动，并忽略掉被依赖bean的声明阶段值。
	 * 包括Lifecycle类型bean和自动启动的SmartLifecycle类型bean。
	 */
	@Override
	public void start() {
		startBeans(false);
		this.running = true;
	}

	/**
	 * Stop all registered beans that implement Lifecycle and <i>are</i>
	 * currently running. Any bean that implements SmartLifecycle
	 * will be stopped within its 'phase', and all phases will be ordered
	 * from highest to lowest value. All beans that do not implement
	 * SmartLifecycle will be stopped in the default phase 0. A bean
	 * declared as dependent on another bean will be stopped before
	 * the dependency bean regardless of the declared phase.
	 * 停止所有实现Lifecycle接口的正在运行的注册bean。任何SmartLifecycle类型的bean，将在其
	 * 阶段值内停止，所有阶段值从高到底。所有没有实现SmartLifecycle的bean将会在默认的0阶段，
	 * 停止。bean所有依赖的bean，将会在宿主bean之前，停止，并忽略被依赖的bean的阶段值。
	 */
	@Override
	public void stop() {
		stopBeans();
		this.running = false;
	}
    /**
     * 刷新容器上下文，仅启动自动启动的SmartLifecycle类型bean
     */
	@Override
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	}
	/**
	 * 关闭生命周期bean实例
	 */
	@Override
	public void onClose() {
		stopBeans();
		this.running = false;
	}

	@Override
	public boolean isRunning() {
		return this.running;
	}
    ...
}
```
从上面可以看出，默认生命周期处理器DefaultLifecycleProcessor，内部主要有3个成员变量，一个是运行状态标识，一个是生命周期bean关闭超时时间，还有一个是所属的bean工厂。
实现生命周期接口和生命周期处理器接口方法的关闭实现方法为 *startBeans，stopBeans*。下面我们来看这个两个方法。



```java
// internal helpers

	/**
	 * 将生命周期bean，按阶段值分组，并从阶段值从小到大，启动生命周期bean分组中bean。
	 * @param autoStartupOnly
	 * 是否包括非自动启动，如果为false，则包括Lifecycle非SmartLifecycle类型bean，为true只包括自动启动
	 * 的SmartLifecycle类型bean
	 */
	private void startBeans(boolean autoStartupOnly) {
		//获取容器中所有已经创建的单例Lifecycle类型bean，和SmartLifecycle类型bean
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<Integer, LifecycleGroup>();
		//遍历生命周期bean实例，按bean的阶段值分组生命周期bean
		for (Map.Entry<String, ? extends Lifecycle> entry : lifecycleBeans.entrySet()) {
			Lifecycle bean = entry.getValue();//获取生命周期bean
			//如果为非自动启动，或为SmartLifecycle类型bean，且自动启动
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);//获取bean的阶段值
				LifecycleGroup group = phases.get(phase);//获取阶段值对应的生命周期分组
				//如果分组为空，则创建对应的分组，并将bean添加到分组中
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(entry.getKey(), bean);
			}
		}
		if (!phases.isEmpty()) {
			//如果生命周期bean分组不为空，则排序生命周期bean分组
			List<Integer> keys = new ArrayList<Integer>(phases.keySet());
			Collections.sort(keys);
			for (Integer key : keys) {
				//按从阶段值从小到大启动生命周期bean分组中bean
				phases.get(key).start();
			}
		}
	}

    /**
	 * Retrieve all applicable Lifecycle beans: all singletons that have already been created,
	 * as well as all SmartLifecycle beans (even if they are marked as lazy-init).
	 * 检索所有应用生命周期类型bean：所有已经创建的单例声明周期bean，以及SmartLifecycle类型的bean，包括懒加载
	 * SmartLifecycle类型的bean。
	 * @return the Map of applicable beans, with bean names as keys and bean instances as values
	 */
	protected Map<String, Lifecycle> getLifecycleBeans() {
		Map<String, Lifecycle> beans = new LinkedHashMap<String, Lifecycle>();
		//获取bean工厂中所有生命周期bean的name
		String[] beanNames = this.beanFactory.getBeanNamesForType(Lifecycle.class, false, false);
		for (String beanName : beanNames) {
			//获取实际注册bean的name
			String beanNameToRegister = BeanFactoryUtils.transformedBeanName(beanName);
			//判断bean name对应的bean是否为工厂bean
			boolean isFactoryBean = this.beanFactory.isFactoryBean(beanNameToRegister);
			//完善bean的name
			String beanNameToCheck = (isFactoryBean ? BeanFactory.FACTORY_BEAN_PREFIX + beanName : beanName);
			if ((this.beanFactory.containsSingleton(beanNameToRegister) &&
					(!isFactoryBean || Lifecycle.class.isAssignableFrom(this.beanFactory.getType(beanNameToCheck)))) ||
					SmartLifecycle.class.isAssignableFrom(this.beanFactory.getType(beanNameToCheck))) {
				//如果bean工厂包含name对应的单例bean，且为生命周期类型的非工厂bean，或为SmartLifecycle类型的bean，
				//则获取对应的bean实例
				Lifecycle bean = this.beanFactory.getBean(beanNameToCheck, Lifecycle.class);
				if (bean != this) {
					//添加到bean name与bean 实例的映射集
					beans.put(beanNameToRegister, bean);
				}
			}
		}
		return beans;
	}

    /**
	 * Helper class for maintaining a group of Lifecycle beans that should be started
	 * and stopped together based on their 'phase' value (or the default value of 0).
	 */
	private class LifecycleGroup {
        //生命周期组成员
		private final List<LifecycleGroupMember> members = new ArrayList<LifecycleGroupMember>();

		private final int phase;//生命周期阶段值

		private final long timeout;

		private final Map<String, ? extends Lifecycle> lifecycleBeans;//生命周期bean实例集

		private final boolean autoStartupOnly;//是否自动启动

		private volatile int smartMemberCount;//SmartLifecycle成员bean数量

		public LifecycleGroup(int phase, long timeout, Map<String, ? extends Lifecycle> lifecycleBeans, boolean autoStartupOnly) {
			this.phase = phase;
			this.timeout = timeout;
			this.lifecycleBeans = lifecycleBeans;
			this.autoStartupOnly = autoStartupOnly;
		}

		/**
		 * 添加生命周期bean
		 * @param name
		 * @param bean
		 */
		public void add(String name, Lifecycle bean) {
			if (bean instanceof SmartLifecycle) {
				this.smartMemberCount++;
			}
			this.members.add(new LifecycleGroupMember(name, bean));
		}

		/**
		 *
		 */
		public void start() {
			if (this.members.isEmpty()) {
				return;
			}
			if (logger.isInfoEnabled()) {
				logger.info("Starting beans in phase " + this.phase);
			}
			//排序生命周期分组内的声明周期bean，阶段值，从小到大
			Collections.sort(this.members);
			//遍历生命周期组成员
			for (LifecycleGroupMember member : this.members) {
				if (this.lifecycleBeans.containsKey(member.name)) {
					//如果生命周期bean实例集中包含对应的bean，则启动生命周期bean。
					doStart(this.lifecycleBeans, member.name, this.autoStartupOnly);
				}
			}
		}

		public void stop() {
			if (this.members.isEmpty()) {
				return;
			}
			if (logger.isInfoEnabled()) {
				logger.info("Stopping beans in phase " + this.phase);
			}
			//排序生命周期分组内的声明周期bean，阶段值，从大到小
			Collections.sort(this.members, Collections.reverseOrder());
			CountDownLatch latch = new CountDownLatch(this.smartMemberCount);
			Set<String> countDownBeanNames = Collections.synchronizedSet(new LinkedHashSet<String>());
			//遍历组成员
			for (LifecycleGroupMember member : this.members) {
				//生命周期bean实例集中存在对应的bean，则关闭bean
				if (this.lifecycleBeans.containsKey(member.name)) {
					doStop(this.lifecycleBeans, member.name, latch, countDownBeanNames);
				}
				else if (member.bean instanceof SmartLifecycle) {
					// already removed, must have been a dependent
					latch.countDown();
				}
			}
			try {
				//超时等到所有的SmartLifecycle关闭
				latch.await(this.timeout, TimeUnit.MILLISECONDS);
				if (latch.getCount() > 0 && !countDownBeanNames.isEmpty() && logger.isWarnEnabled()) {
					logger.warn("Failed to shut down " + countDownBeanNames.size() + " bean" +
							(countDownBeanNames.size() > 1 ? "s" : "") + " with phase value " +
							this.phase + " within timeout of " + this.timeout + ": " + countDownBeanNames);
				}
			}
			catch (InterruptedException ex) {
				Thread.currentThread().interrupt();
			}
		}
	}


	/**
	 * Adapts the Comparable interface onto the lifecycle phase model.
	 * 生命周期阶段模型比较接口
	 */
	private class LifecycleGroupMember implements Comparable<LifecycleGroupMember> {

		private final String name;//生命周期组成员name

		private final Lifecycle bean;//生命周期组成员bean

		LifecycleGroupMember(String name, Lifecycle bean) {
			this.name = name;
			this.bean = bean;
		}
		/**
		 * 比较声明周期成员阶段值，相等为0，小于为-1，大于为1
		 */
		@Override
		public int compareTo(LifecycleGroupMember other) {
			int thisOrder = getPhase(this.bean);
			int otherOrder = getPhase(other.bean);
			return (thisOrder == otherOrder ? 0 : (thisOrder < otherOrder) ? -1 : 1);
		}
	}
    /**
	 * Determine the lifecycle phase of the given bean.
	 * 确定给定bean的生命周期阶段值。
	 * <p>The default implementation checks for the {@link Phased} interface.
	 * Can be overridden to apply other/further policies.
	 * 默认实现检查{@link Phased}接口的阶段值。可以重写，以实现进一步的策略。
	 * @param bean the bean to introspect
	 * @return the phase an integer value. The suggested default is 0.
	 * 如果bean为非Phased类型，则默认为0
	 * @see Phased
	 * @see SmartLifecycle
	 */
	protected int getPhase(Lifecycle bean) {
		return (bean instanceof Phased ? ((Phased) bean).getPhase() : 0);
	}

```

再来看生命周期分组启动操作
```java
/**
	 * Start the specified bean as part of the given set of Lifecycle beans,
	 * making sure that any beans that it depends on are started first.
	 * 启动给定生命周期bean集合中的特殊bean，并确保所有依赖的bean先启动。
	 * @param lifecycleBeans Map with bean name as key and Lifecycle instance as value
	 * @param beanName the name of the bean to start
	 */
	private void doStart(Map<String, ? extends Lifecycle> lifecycleBeans, String beanName, boolean autoStartupOnly) {
		//从声明周期bean实例集中移除对应的bean
		Lifecycle bean = lifecycleBeans.remove(beanName);
		if (bean != null && !this.equals(bean)) {
			//获取bean的所有依赖bean
			String[] dependenciesForBean = this.beanFactory.getDependenciesForBean(beanName);
			for (String dependency : dependenciesForBean) {
				doStart(lifecycleBeans, dependency, autoStartupOnly);
			}
			if (!bean.isRunning() &&
					(!autoStartupOnly || !(bean instanceof SmartLifecycle) || ((SmartLifecycle) bean).isAutoStartup())) {
				//如果当前bean不在运行，同时非自动启动，非SmartLifecycle类型bean或SmartLifecycle bean为非自动启动
				if (logger.isDebugEnabled()) {
					logger.debug("Starting bean '" + beanName + "' of type [" + bean.getClass() + "]");
				}
				try {
					//启动生命周期bean
					bean.start();
				}
				catch (Throwable ex) {
					throw new ApplicationContextException("Failed to start bean '" + beanName + "'", ex);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Successfully started bean '" + beanName + "'");
				}
			}
		}
	}
```
从上面可以看出，默认生命周期处理器，启动生命周期bean的过程为，将生命周期bean，按阶段值分组，并从阶段值从小到大，启动生命周期bean分组中bean。


在俩看停止生命周期bean：

```java
/**
	 * 将生命周期bean，按阶段值分组，并从阶段值从大到小，关闭生命周期bean分组中bean。
	 */
	private void stopBeans() {
		//获取容器中所有已经创建的单例Lifecycle类型bean，和SmartLifecycle类型bean
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<Integer, LifecycleGroup>();
		for (Map.Entry<String, Lifecycle> entry : lifecycleBeans.entrySet()) {
			Lifecycle bean = entry.getValue();
			int shutdownOrder = getPhase(bean);//获取bean的阶段值
			LifecycleGroup group = phases.get(shutdownOrder);//获取阶段值对应的生命周期分组
			//如果分组为空，则创建对应的分组，并将bean添加到分组中
			if (group == null) {
				group = new LifecycleGroup(shutdownOrder, this.timeoutPerShutdownPhase, lifecycleBeans, false);
				phases.put(shutdownOrder, group);
			}
			group.add(entry.getKey(), bean);
		}
		//如果生命周期bean分组不为空，则排序生命周期bean分组
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<Integer>(phases.keySet());
			//按从阶段值从大到小关闭生命周期bean分组中bean
			Collections.sort(keys, Collections.reverseOrder());
			for (Integer key : keys) {
				phases.get(key).stop();
			}
		}
	}
    /**
	 * Stop the specified bean as part of the given set of Lifecycle beans,
	 * making sure that any beans that depends on it are stopped first.
	 * 关闭生命周期bean实例集中的生命周期bean，并确保所有依赖的bean先关闭。
	 * @param lifecycleBeans Map with bean name as key and Lifecycle instance as value
	 * @param beanName the name of the bean to stop
	 */
	private void doStop(Map<String, ? extends Lifecycle> lifecycleBeans, final String beanName,
			final CountDownLatch latch, final Set<String> countDownBeanNames) {
        //从生命周期bean实例集中移除对应生命周期bean
		Lifecycle bean = lifecycleBeans.remove(beanName);
		if (bean != null) {
			//获取所有依赖的bean
			String[] dependentBeans = this.beanFactory.getDependentBeans(beanName);
			//遍历所有依赖的bean，并关闭
			for (String dependentBean : dependentBeans) {
				doStop(lifecycleBeans, dependentBean, latch, countDownBeanNames);
			}
			try {
				if (bean.isRunning()) {
					if (bean instanceof SmartLifecycle) {//如果bean为SmartLifecycle类型，且在运行
						if (logger.isDebugEnabled()) {
							logger.debug("Asking bean '" + beanName + "' of type [" + bean.getClass() + "] to stop");
						}
						//添加bean name到闭锁bean name集
						countDownBeanNames.add(beanName);
						((SmartLifecycle) bean).stop(new Runnable() {
							@Override
							public void run() {
								//释放闭锁，并从闭锁bean name集，移除对应的bean的name
								latch.countDown();
								countDownBeanNames.remove(beanName);
								if (logger.isDebugEnabled()) {
									logger.debug("Bean '" + beanName + "' completed its stop procedure");
								}
							}
						});
					}
					else {
						if (logger.isDebugEnabled()) {
							logger.debug("Stopping bean '" + beanName + "' of type [" + bean.getClass() + "]");
						}
						//如果为非SmartLifecycle类型的生命周期bean，直接关闭
						bean.stop();
						if (logger.isDebugEnabled()) {
							logger.debug("Successfully stopped bean '" + beanName + "'");
						}
					}
				}
				else if (bean instanceof SmartLifecycle) {
					// don't wait for beans that aren't running，SmartLifecycle类型bean不在运行
					latch.countDown();
				}
			}
			catch (Throwable ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Failed to stop bean '" + beanName + "'", ex);
				}
			}
		}
	}
```
从上面可看出，默认生命周期处理器，关闭生命周期bean的过程为，将生命周期bean，按阶段值分组，并从阶段值从大到小，关闭生命周期bean分组中bean。关闭生命周期bean的顺序与启动顺序
正好相反。需要注意的是无论是启动还是关闭，生命周期bean所依赖的bean都是在其之前启动或关闭，忽略掉被依赖bean的Phase阶段值。对于非生命周期bean，其阶段值默认为0。

我们来小节一下：

默认生命周期处理器DefaultLifecycleProcessor，内部主要有3个成员变量，一个是运行状态标识，一个是生命周期bean关闭超时时间，还有一个是所属的bean工厂。默认生命周期处理器，启动生命周期bean的过程为，将生命周期bean，按阶段值分组，并从阶段值从小到大，启动生命周期bean分组中bean。默认生命周期处理器，关闭生命周期bean的过程为，将生命周期bean，按阶段值分组，并从阶段值从大到小，关闭生命周期bean分组中bean。关闭生命周期bean的顺序与启动顺序正好相反。需要注意的是无论是启动还是关闭，生命周期bean所依赖的bean都是在其之前启动或关闭，忽略掉被依赖bean的Phase阶段值。对于非生命周期bean，其阶段值默认为0。
***
由于应用上下文，也是一个生命周期bean，那么应用上下文生命周期bean是否被默认的生命周期处理器，处理呢？这个问题，后面我们慢慢来解开。


由于篇幅有限，下面这些章节我们放在后面再讲。

### ApplicationEventMulticaster


### SimpleApplicationEventMulticaster



### StandardEnvironment


### DelegatingMessageSource




最后我们以DefaultLifecycleProcessor的类图结束这篇文章。
![DefaultLifecycleProcessor](/image/spring-context/DefaultLifecycleProcessor.png)

## 总结

抽象应用上下文 *AbstractApplicationContext* 实际为一个可配置上下文 *ConfigurableApplicationContext* 和可销毁的bean（DisposableBean），同时拥有了资源加载功能（DefaultResourceLoader）。我们通过一个唯一的id标注抽象上下文，同时抽象上下文拥有一个展示名。除此身份识别属性之前，抽象应用上下文，有一个父上下文 *ApplicationContext* ，可配的环境配置 *ConfigurableEnvironment* ，bean工厂后处理器集（List<BeanFactoryPostProcessor>），资源模式解决器（ResourcePatternResolver），声明周期处理器（LifecycleProcessor),消息源 *MessageSource* ，事件发布器 *ApplicationEventMulticaster* ，应用监听器集（LinkedHashSet<ApplicationListener<?>>），预发布的应用事件集（LinkedHashSet<ApplicationEvent>）。除了上述的功能性属性外，抽象应用上下文，还有一个一些状态属性，如果启动时间，激活状态（AtomicBoolean），关闭状态（AtomicBoolean）。最后还有一个上下为刷新和销毁的同步监控对象和一虚拟机关闭hook线程。


路径匹配资源模式解决器PathMatchingResourcePatternResolver内部有一个Ant路径匹配器 *AntPathMatcher*，和一个资源类加载器，资源加载器可以
使用所属上下文中的资源加载器，也可以为给定类加载器的DefaultResourceLoader。路径匹配资源模式解决器主要提供了加载给定路径位置的资源方法，此方法可以解决无通配符的路径位置模式（{@code file:C:/context.xml}，{@code classpath:/context.xml}，{@code /WEB-INF/context.xml}"），也可以解决包含Ant风格的通配符路径位置模式资源（{@code classpath*:META-INF/beans.xml}），主要以classpath*为前缀的路径位置模式，资源加载器将会查找类路径下所有相同name对应的资源文件，包括子目录和jar包。如果明确的加载资源，可以使用{@code classpath:/context.xml}形式路径模式，如果想要探测类路径下的所有name对应的资源文件，可以使用形式路径模式。

BeanFactoryAware接口主要提供设置bean工厂操作。

LifecycleProcessor接口主要提供了通知上下文刷新和关闭的操作。

Phased主要提供了获取组件阶段值操作。

SmartLifecycle接口主要提供关闭回调操作，在组件停止后，调用回调接口。并提供了判断组件在容器上下文刷新时，组件是否自动刷新的操作。

默认生命周期处理器DefaultLifecycleProcessor，内部主要有3个成员变量，一个是运行状态标识，一个是生命周期bean关闭超时时间，还有一个是所属的bean工厂。默认生命周期处理器，启动生命周期bean的过程为，将生命周期bean，按阶段值分组，并从阶段值从小到大，启动生命周期bean分组中bean。默认生命周期处理器，关闭生命周期bean的过程为，将生命周期bean，按阶段值分组，并从阶段值从大到小，关闭生命周期bean分组中bean。关闭生命周期bean的顺序与启动顺序正好相反。需要注意的是无论是启动还是关闭，生命周期bean所依赖的bean都是在其之前启动或关闭，忽略掉被依赖bean的Phase阶段值。对于非生命周期bean，其阶段值默认为0。
