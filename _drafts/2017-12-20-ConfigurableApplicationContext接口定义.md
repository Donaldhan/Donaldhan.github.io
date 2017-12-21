---
layout: page
title: ConfigurableApplicationContext接口定义
subtitle: ConfigurableApplicationContext接口定义
date: 2017-12-20 08:32:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言
上一篇文章，我们看了[AutowireCapableBeanFactory][]接口，主要提供的创建bean实例，自动装配bean属性，应用bean配置属性，初始化bean，应用bean后处理器 *BeanPostProcessor* ，解决bean依赖和销毁bean操作。对于自动装配，主要提供了根据bean的name，类型和构造自动装配方式。一般不建议在在代码中直接使用AutowireCapableBeanFactory接口，我们可以通过应用上下文的ApplicationContext#getAutowireCapableBeanFactory()方法或者通过实现BeanFactoryAware，获取暴露的bean工厂，然后转换为AutowireCapableBeanFactory。NamedBeanHolder用于表示bean的name和实例的关系句柄。NamedBeanHolder可以用于Spring的根据bean的name自动装配和AOP相关的功能，避免产生不可靠的依赖。

![AutowireCapableBeanFactory](/image/spring-context/AutowireCapableBeanFactory.png)

[AutowireCapableBeanFactory]:https://donaldhan.github.io/spring-framework/2017/12/19/AutowireCapableBeanFactory%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "AutowireCapableBeanFactory"


# 目录
* [Lifecycle](#Lifecycle)
* [ConfigurableApplicationContext接口定义](#ConfigurableApplicationContext接口定义)
    * [BeanFactoryPostProcessor](#BeanFactoryPostProcessor)
    * [ProtocolResolver](#ProtocolResolver)
    * [ConfigurableEnvironment](#ConfigurableEnvironment)
    * [ConfigurableListableBeanFactory](#ConfigurableListableBeanFactory)
* [总结](#总结)
* [附](#附)

看完了AutowireCapableBeanFactory和ApplicationContext接口的定义，我们接着，来看 *ConfigurableApplicationContext* 接口的定义，再看之前先看一下父类接口Lifecycle和Closeable的定义：
先来看Lifecycle
## Lifecycle
具体源码参见：[Lifecycle][]

[Lifecycle]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/Lifecycle.java "Lifecycle"

```java
package org.springframework.context;

/**
*  Lifecycle接口是一个普通的接口，定义了控制声明周期的启动和停止操作，用于异步处理的情况。
*注意此接口不意味可以自动启动，如果有这方面的需求，可以考虑实现{@link SmartLifecycle}接口。
 * 此接口可以被组件或容器实现，比如典型的spring上下中bean定义和spring应用上下文ApplicationContext。
 * 容器应该传播启动和停止信道到所有子容器中的组件。比如在运行时环境下的停止和重启情况。
 *此接口可以通过JMX直接调用或管理操作。在管理操作的情况下， {@link org.springframework.jmx.export.MBeanExporter}定义
 *为{@link org.springframework.jmx.export.assembler.InterfaceBasedMBeanInfoAssembler},限制在声明周期范围内
 *的活动组件的可视性。
 *注意：声明周期接口，仅仅支持顶层的单例bean。在其他组件中，声明周期接口将不可探测，因此将会忽略。拓展{@link SmartLifecycle}接口
 *提供的继承应用上下文的启动和关闭阶段。
 * @author Juergen Hoeller
 * @since 2.0
 * @see SmartLifecycle
 * @see ConfigurableApplicationContext
 * @see org.springframework.jms.listener.AbstractMessageListenerContainer
 * @see org.springframework.scheduling.quartz.SchedulerFactoryBean
 */
public interface Lifecycle {

	/**
	 * 启动当前组件。如果组件已将在运行，不应该抛出异常。在容器环境下，将传播启动信号到应用的所有组件。
	 * @see SmartLifecycle#isAutoStartup()
	 */
	void start();

	/**
	 * 停止当前组件，在同步环境下，在方法返回后，组件完全停止。当异步停止行为需要的时候，可以考虑实现 {@link SmartLifecycle}接口的
	 * {@code stop(Runnable)}方法。需要注意的是：不保证停止通知发生在析构之前：在正常的关闭操作下，{@code Lifecycle} bean将会
	 * 在一般的析构回调之前，将会接受一个停止通知；然而在上下文生命周期内的热刷新或刷新尝试中断，仅仅销毁方法将会调用。如果组件还没有启动，
	 * 则不应该抛出异常。在容器环境下，应该传播停止信号到所有的组件。
	 * @see SmartLifecycle#stop(Runnable)
	 * @see org.springframework.beans.factory.DisposableBean#destroy()
	 */
	void stop();

	/**
	 * 判断当前组件是否运行。在容器中，如果所有应用的组件当前都在运行，则返回true
	 * @return whether the component is currently running
	 */
	boolean isRunning();

}
```
从上可以看出，Lifecycle接口提供了启动和关闭操作，以及判断当前组件是否运行操作。需要注意的是启动和停止操作，将会传播给容器中的所有子容器中的组件。对于停止操作，不保证停止通知发生在析构之前。对于判断当前组件是否运行操作，如果组件是容器，只有在容器中所有组件包括子容器中的组件，都在运行的情况下，才返回true。

再来看一下Closeable接口，这个时JDK的范畴。
### Closeable
```java
package java.io;

import java.io.IOException;

/**
 * A {@code Closeable} is a source or destination of data that can be closed.
 * The close method is invoked to release resources that the object is
 * holding (such as open files).
 *Closeable是一个可以关闭的数据源或目的。close方法被调用，将释放相关的资源。
 * @since 1.5
 */
public interface Closeable extends AutoCloseable {

    /**
     * Closes this stream and releases any system resources associated
     * with it. If the stream is already closed then invoking this
     * method has no effect.
     *关闭当前流，释放相关资源。如果流已关闭，则调用方法没有任何作用
     * <p> As noted in {@link AutoCloseable#close()}, cases where the
     * close may fail require careful attention. It is strongly advised
     * to relinquish the underlying resources and to internally
     * <em>mark</em> the {@code Closeable} as closed, prior to throwing
     * the {@code IOException}.
     *
     * @throws IOException if an I/O error occurs
     */
    public void close() throws IOException;
}
```
ConfigurableApplicationContext除了继承了 *Lifecycle* 和 *Closeable*，还继承了 *ApplicationContext*，关于应用上下文接口我们在前面，已将在，这里不再说，忘掉的可以去查阅。   
下面进入我们这篇文章的核心部分ConfigurableApplicationContext接口定义


## ConfigurableApplicationContext接口定义

具体源码参见：[ConfigurableApplicationContext][]

[ConfigurableApplicationContext]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/ConfigurableApplicationContext.java "ConfigurableApplicationContext"

```java
package org.springframework.context;

import java.io.Closeable;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.Environment;
import org.springframework.core.io.ProtocolResolver;

/**
 *ConfigurableApplicationContext接口可以被大多数的应用上下文实现。为配置应用上下文提供便利，
 *另外，可以作为应用上下文的客户端。
 *相关配配置和生命周期方法被封装在此接口，以避免应用上下文客户端看见。接口当前方法啊，应该在启动和关闭代码间调用。
 * @author Juergen Hoeller
 * @author Chris Beams
 * @since 03.11.2003
 */
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {

	/**
	 * 配置文件路径分割符
	 * @see org.springframework.context.support.AbstractXmlApplicationContext#setConfigLocation
	 * @see org.springframework.web.context.ContextLoader#CONFIG_LOCATION_PARAM
	 * @see org.springframework.web.servlet.FrameworkServlet#setContextConfigLocation
	 */
	String CONFIG_LOCATION_DELIMITERS = ",; \t\n";

	/**
	 * bean工厂内类型转化ConversionService bean的name，没有则为默认值。
	 * @see org.springframework.core.convert.ConversionService
	 * @since 3.0
	 */
	String CONVERSION_SERVICE_BEAN_NAME = "conversionService";

	/**
	 * bean工厂中LoadTimeWeaver bean的name，如果存在LoadTimeWeaver这样的bean，为了允许LoadTimeWeaver可以处理
	 * 实际的bean类型，则上下文将使用匹配类型的临时类型加载器。
	 * @since 2.5
	 * @see org.springframework.instrument.classloading.LoadTimeWeaver
	 */
	String LOAD_TIME_WEAVER_BEAN_NAME = "loadTimeWeaver";

	/**
	 * Name of the {@link Environment} bean in the factory.
	 * bean工厂中环境的name
	 * @since 3.1
	 */
	String ENVIRONMENT_BEAN_NAME = "environment";

	/**
	 * Name of the System properties bean in the factory.
	 * bean工厂中系统属性bean的name
	 * @see java.lang.System#getProperties()
	 */
	String SYSTEM_PROPERTIES_BEAN_NAME = "systemProperties";

	/**
	 * Name of the System environment bean in the factory.
	 * bean工厂中系统bean的name
	 * @see java.lang.System#getenv()
	 */
	String SYSTEM_ENVIRONMENT_BEAN_NAME = "systemEnvironment";


	/**
	 * Set the unique id of this application context.
	 * 设置应用上下文的唯一id
	 * @since 3.0
	 */
	void setId(String id);

	/**
	 * 设置应用上下文的父上下文。主要父上下文不可以改变：当类型的对象创建时，如果对象不可用，对象应在构造范围之外。
	 * 比如，Web上下文WebApplicationContext的配置
	 * @param parent the parent context
	 * @see org.springframework.web.context.ConfigurableWebApplicationContext
	 */
	void setParent(ApplicationContext parent);

	/**
	 * Set the {@code Environment} for this application context.
	 * 设置应用上下文的环境
	 * @param environment the new environment
	 * @since 3.1
	 */
	void setEnvironment(ConfigurableEnvironment environment);

	/**
	 * Return the {@code Environment} for this application context in configurable
	 * form, allowing for further customization.
	 * 获取应用上下文的可配置环境。
	 * @since 3.1
	 */
	@Override
	ConfigurableEnvironment getEnvironment();

	/**
	 * 添加bean工厂后处理器BeanFactoryPostProcessor，在任何bean的定义被评估之前，应用上下文刷新时，
	 * 将会应用bean工厂后处理器到内部的bean工厂。在上下文配置的过程中，调用。
	 * @param postProcessor the factory processor to register
	 */
	void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);

	/**
	 * 添加应用监听器，当上下文事件发生时，将会被通知，比如上下文刷新，上下文关闭事件。
	 * 需要注意是，如果上下文还没有激活，或者上下文事件已经激活，当前事件正在多播，当刷新的时候，已经注册到上下文的
	 * 监听器将会被通知。
	 * @param listener the ApplicationListener to register
	 * @see org.springframework.context.event.ContextRefreshedEvent
	 * @see org.springframework.context.event.ContextClosedEvent
	 */
	void addApplicationListener(ApplicationListener<?> listener);

	/**
	 * 注册当前上下文的给定协议解决器，允许额外的资源协议被处理。
	 * @since 4.3
	 */
	void addProtocolResolver(ProtocolResolver resolver);

	/**
	 * 加载或刷新配置的持久化表示层，可以是一个XML文件，属性文件，或者相关数据schema。如果startup方法失败，
	 * 应该销毁所有已经创建的单例bean，以便资源的空置。换句话，在调用启动方法后，所有的bean要么初始化，要么没有初始化。
	 * @throws BeansException if the bean factory could not be initialized
	 * 如果bean工厂不能够初始化，则抛出BeansException异常
	 * @throws IllegalStateException if already initialized and multiple refresh
	 * attempts are not supported
	 * 如果已经初始化且多次尝试刷新不支持，则抛出IllegalStateException异常。
	 */
	void refresh() throws BeansException, IllegalStateException;

	/**
	 * 注意一个JVM运行时关闭hook，在虚拟机关闭时，关闭当前上下文，除非上下文已经关闭。
	 * 此方法可以被调用多次。每个上下文实例，最多注意一个关闭hook。
	 * @see java.lang.Runtime#addShutdownHook
	 * @see #close()
	 */
	void registerShutdownHook();

	/**
	 * 关闭当前应用上下文，释放上下文关联的资源和持有的锁。包括销毁所有缓存的单例bean。
	 * 需要注意的是：不会调用父上下文的关闭方法，因为父上下文有自己独立的声明周期。
	 * 此方法可以调用多次，如果上下文已经关闭，则忽略。
	 */
	@Override
	void close();

	/**
	 * 判断当前上下文是否激活，也就是上下文是否至少刷新一次并且没有归案必。
	 * @return whether the context is still active
	 * @see #refresh()
	 * @see #close()
	 * @see #getBeanFactory()
	 */
	boolean isActive();

	/**
	 * 返回上下文内部bean工厂。可以用于访问底层工厂的相关功能。
	 * 需要注意的是：不要使用bean工厂的后处理器；因为单实例bean已经初始化。在bean可用之前，可以使用bean工厂后处理器，
	 * 拦截bean工厂的设置过程。
	 * 一般情况下，当上下文处于激活状态，内部的bean工厂是可以访问的，也就是在{@link #refresh()} 和 {@link #close()}.
	 * 方法之前的情况，{@link #isActive()可用于检查上下文是否处于合适的状态。
	 * @return the underlying bean factory
	 * 底层的bean工厂
	 * @throws IllegalStateException if the context does not hold an internal
	 * bean factory (usually if {@link #refresh()} hasn't been called yet or
	 * if {@link #close()} has already been called)
	 * 如果上下文还没有持有内部bean工厂，则抛出IllegalStateException异常。即{@link #refresh()}还
	 * 没有调用，或{@link #close()}方法已经调用。
	 * @see #isActive()
	 * @see #refresh()
	 * @see #close()
	 * @see #addBeanFactoryPostProcessor
	 */
	ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;

}

```
从上可以看出，ConfigurableApplicationContext具备应用上下文 *ApplicationContex* 相关操作以外，同时具有了生命周期和流属性。除此之外，
提供了设置应用id，设置父类上下文，设置环境 *ConfigurableEnvironment*，添加应用监听器，添加bean工厂后处理器 *BeanFactoryPostProcessor*，添加协议解决器 *ProtocolResolver*，刷新应用上下文，关闭应用上下文，判断上下文状态，以及注册虚拟机关闭Hook等操作，同时重写了获取环境操作，此操作返回的为可配置环境 *ConfigurableEnvironment*。最关键的是提供了获取内部bean工厂的访问操作，
方法返回为 *ConfigurableListableBeanFactory*。需要注意的是，调用关闭操作，并不关闭父类的应用上下文，应用上下文与父类的上下文生命周期，相互独立。  
从ConfigurableApplicationContext的方法，而已看出ConfigurableApplicationContext主要实现的是set相关的操作，而ApplicationContext是get相关的操在，我们可以简单的理解为读写操作分离。   
下面我们依次来看一下ConfigurableApplicationContext接口关联的接口，*BeanFactoryPostProcessor*，*ProtocolResolver*，*ConfigurableEnvironment*，*ConfigurableListableBeanFactory*。

我们先来看BeanFactoryPostProcessor接口。
### BeanFactoryPostProcessor

源码参见：[BeanFactoryPostProcessor][]

[BeanFactoryPostProcessor]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/config/BeanFactoryPostProcessor.java "BeanFactoryPostProcessor"

```java
/*
 * Copyright 2002-2012 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;

/**
 *BeanFactoryPostProcessor后处理器，允许对应用上下文的bean定义进行一般的修改，调整底层bean工厂
 *上下文中的bean属性。
 *应用上下文可以自动探测，在上下文中的bean工厂后处理器BeanFactoryPostProcessor bean的定义，
 *在其他任何bean在创建前，应用bean工厂后处理器。
 *bean工厂后处理器在系统管理级的配置文件非常有用，用于重写应用上下文中的配置属性。
 *具体查看属性资源配置及其具体的实现，是一种开箱即用的加强配置的解决方案。
 * bean工厂后处理器，修改bean定义或与之交互，而不是bean的实例。这样也许引起bean的过早初始化，
 * 违背的bean容器的原则，进而带来意想不到的影响。如果需要与bean的实例进行交互，
 * 可以使用bean后处理器替代BeanPostProcessor。
 *
 * @author Juergen Hoeller
 * @since 06.07.2003
 * @see BeanPostProcessor
 * @see PropertyResourceConfigurer
 */
public interface BeanFactoryPostProcessor {

	/**
	 * 在上下文标准初始化完毕后，修改应用上下文的内部bean工厂。所有的bean定义都将加载，
	 * 但是没有bean已经被初始化。允许重写或添加属性到将要初始化的bean。
	 * @param beanFactory the bean factory used by the application context
	 * 上下文bean工厂正在使用的bean工厂。
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}

```
从上面可以看出，bean工厂后处理器BeanFactoryPostProcessor，主要提供了修改上下文的内部bean工厂操作，可以修改bean的定义，
而不能修改bean的实例属性。如果要与bean的实例交互，可以实现bean后处理器BeanPostProcessor。bean工厂后处理器一般用系统级的配置，比如 *PropertyResourceConfigurer*，并重写bean定义的属性。

### ProtocolResolver

源码参见：[ProtocolResolver][]

[ProtocolResolver]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/io/ProtocolResolver.java  "ProtocolResolver"

```java
package org.springframework.core.io;

/**
 *ProtocolResolver接口是一个特殊协议资源的解决策略处理器。
 *默认资源加载器的系统包接口SPI，在没有子类资源加载器和应用上下文的实现的情况下，
 *运行处理一般的资源协议。
 * @author Juergen Hoeller
 * @since 4.3
 * @see DefaultResourceLoader#addProtocolResolver
 */
public interface ProtocolResolver {

	/**
	 * 如果协议的实现匹配，则根据资源加载器，解决给定位置的资源。
	 * @param location the user-specified resource location
	 * @param resourceLoader the associated resource loader
	 * @return a corresponding {@code Resource} handle if the given location
	 * matches this resolver's protocol, or {@code null} otherwise
	 */
	Resource resolve(String location, ResourceLoader resourceLoader);

}

```
从上面可以看出，ProtocolResolver接口，主要提供了根据资源位置加载相应资源的操作，在没有子类资源加载器和应用上下文的实现的情况下，默认资源加载器 *DefaultResourceLoader* 的系统包接口SPI，允许处理一般的资源协议。

### ConfigurableEnvironment

源码参见：[ConfigurableEnvironment][]

[ConfigurableEnvironment]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/env/ConfigurableEnvironment.java "ConfigurableEnvironment"

```java
package org.springframework.core.env;

import java.util.Map;

/**
 *配置环境接口ConfigurableEnvironment是大多数环境类型需要实现的配置接口。为设置激活和默认配置，及操纵底层属性源提供了
 *便利。允许客户端通过定制转换服务设置和校验需要的属性，更多的通过{@link ConfigurablePropertyResolver}。
 * 操纵属性源。
 * 属性源可以被移除，重排序或替换；另外属性源可以通过 {@link #getPropertySources()}方法返回的{@link MutablePropertySources}
 * 添加到环境中。下面是一个可配置环境的标准实现{@link StandardEnvironment}，尽管一些特殊的默认属性源不同，但一般情况下，适合所有实现。
 *
 * <h4>Example: adding a new property source with highest search priority</h4>
 * 添加一个最高优先级的属性源
 * <pre class="code">
 * ConfigurableEnvironment environment = new StandardEnvironment();
 * MutablePropertySources propertySources = environment.getPropertySources();
 * Map<String, String> myMap = new HashMap<String, String>();
 * myMap.put("xyz", "myValue");
 * propertySources.addFirst(new MapPropertySource("MY_MAP", myMap));
 * </pre>
 *
 * <h4>Example: removing the default system properties property source</h4>
 * 移除默认系统属性源。
 * <pre class="code">
 * MutablePropertySources propertySources = environment.getPropertySources();
 * propertySources.remove(StandardEnvironment.SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME)
 * </pre>
 *
 * <h4>Example: mocking the system environment for testing purposes</h4>
 * mock系统环境
 * <pre class="code">
 * MutablePropertySources propertySources = environment.getPropertySources();
 * MockPropertySource mockEnvVars = new MockPropertySource().withProperty("xyz", "myValue");
 * propertySources.replace(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, mockEnvVars);
 * </pre>
 *
 * 当一个环境被应用上下文使用时，比较重要的是，{@code PropertySource}的所有操作必须在{@link
 * org.springframework.context.support.AbstractApplicationContext#refresh() refresh()}
 * 调用之前。这可以确保，在容器启动的过程中，所有的属性源都可用，包括{@linkplain
 * org.springframework.context.support.PropertySourcesPlaceholderConfigurer property
 * placeholder configurers}.
 *
 * @author Chris Beams
 * @since 3.1
 * @see StandardEnvironment
 * @see org.springframework.context.ConfigurableApplicationContext#getEnvironment
 */
public interface ConfigurableEnvironment extends Environment, ConfigurablePropertyResolver {

	/**
	 * 设置环境的激活配置集。在容器启动的过程中，可以根据配置来决定是否将bean定义注册到容器中。
	 * <p>Any existing active profiles will be replaced with the given arguments; call
	 * 任何已经存在的激活配置，将会被参数指定的配置集替代；当参数为0时，则清除当前激活的配置。如果先要保护已经激活的
	 * 配置集，可以使用{@link #addActiveProfile}方法。
	 * @see #addActiveProfile
	 * @see #setDefaultProfiles
	 * @see org.springframework.context.annotation.Profile
	 * @see AbstractEnvironment#ACTIVE_PROFILES_PROPERTY_NAME
	 * @throws IllegalArgumentException if any profile is null, empty or whitespace-only
	 */
	void setActiveProfiles(String... profiles);

	/**
	 * 添加配置到当前激活的配置集。
	 * @see #setActiveProfiles
	 * @throws IllegalArgumentException if the profile is null, empty or whitespace-only
	 */
	void addActiveProfile(String profile);

	/**
	 * 设置默认的配置集
	 * 如果没有任何配置显示地通过{@link #setActiveProfiles}设置配置，则使用默认的配置集。
	 * @see AbstractEnvironment#DEFAULT_PROFILES_PROPERTY_NAME
	 * @throws IllegalArgumentException if any profile is null, empty or whitespace-only
	 */
	void setDefaultProfiles(String... profiles);

	/**
	 * 返回当前环境的mutable形式属性源{@link PropertySources}，当根据环境对象解决属性的时候，可以使用
	 * 属性源集合。
	 * 多样属性源{@link MutablePropertySources}的相关方法，如下，
	 * {@link MutablePropertySources#addFirst addFirst},
	 * {@link MutablePropertySources#addLast addLast},
	 * {@link MutablePropertySources#addBefore addBefore} and
	 * {@link MutablePropertySources#addAfter addAfter}，
	 * 如果需要的话，我们可以控制这些属性源的顺序。这种 策略非常有用，比如，确保用户定义的属性源由系统属性集或者系统环境
	 * 变量集。
	 * @see AbstractEnvironment#customizePropertySources
	 */
	MutablePropertySources getPropertySources();

	/**
	 * Return the value of {@link System#getenv()} if allowed by the current
	 * {@link SecurityManager}, otherwise return a map implementation that will attempt
	 * to access individual keys using calls to {@link System#getenv(String)}.
	 * 如果当前安全管理器允许，返回系统环境变量{@link System#getenv()}的值，否则将尝试使用{@link System#getenv(String)}，
	 * 方法获取每个key的属性值，再放入到Map集合中。
	 * 需要注意的是，大多数的{@link Environment}的实现，将会包括系统环境变量Map作为一个可以搜索的属性源PropertySource。
	 * 因此强烈建议，除非有明确的需要添加其他属性源，否则此方法不建议直接调用。
	 * 调用{@link Map#get(Object)}方法，不会返回一个非法访问异常；比如当安全管理禁止访问属性值，null将会返回，
	 * 同时一个INFO级的日志信息将会通知这个异常。
	 */
	Map<String, Object> getSystemEnvironment();

	/**
	 * 如果当前安全管理器允许，将返回系统属性{@link System#getProperties()}的值，否则将调用{@link System#getProperty(String)}.
	 * 方法获取每个key的值，添加的结果集中。
	 * 需要注意的是，大多数的{@link Environment}的实现，将会包括系统属性Map作为一个可以搜索的属性源PropertySource。
	 * 因此强烈建议，除非有明确的需要添加其他属性源，否则此方法不建议直接调用。
	 * 调用{@link Map#get(Object)}方法，不会返回一个非法访问异常；比如当安全管理禁止访问属性值，null将会返回，
	 * 同时一个INFO级的日志信息将会通知这个异常。
	 */
	Map<String, Object> getSystemProperties();

	/**
	 * 添加给定父类环境激活配置，默认配置和属性源到当前环境（child）的各自的集合中。
	 * 在父类和子类中，如果任何相同命名的属性源实例已经存在，则子类的属性源实例将会保留，父类的实例将会被丢弃。
	 * 通过这种方法，运行子类重新父类的属性源，可以避免通过一般属性源类型冗余的搜索。比如系统环境变量和系统属性。
	 * 激活和默认的配置名也将过滤，以避免多余副本的存在，引起冲突。
	 * 在任何情况下父类的环境是不可修改的。注意，在调用{@code merge}方法后，任何父类环境修改的发生，将不会影响其子类的环境。
	 * 因此，在调用{@code merge}方法前，我们应该优先配置父类的属性源和配置信息。
	 * @param parent the environment to merge with
	 * @since 3.1.2
	 * @see org.springframework.context.support.AbstractApplicationContext#setParent
	 */
	void merge(ConfigurableEnvironment parent);

}
```
从上面可以看出，ConfigurableEnvironment接口提供设置、添加环境配置，设置默认配置，获取属性源 *MutablePropertySources*，获取系统环境变量，获取系统属性，及合并环境。注意设置环境配置，会覆盖先前的配置，如果要添加配置到当前配置集，要使用添加环境配置方法。另外需要注意的是，父类的环境是不可修改，在调用{@code merge}方法前，我们应该优先配置父类的属性源和配置信息，同时子类的环境配置中的属性源将会时父类中的同名数据源失效。

关于 *MutablePropertySources* ，如果篇幅够的话，我们会在这篇讲，否则将放在后续文章中。

在来看一下ConfigurableEnvironment的父类接口ConfigurablePropertyResolver

#### ConfigurablePropertyResolver

源码参见：[ConfigurablePropertyResolver][]

[ConfigurablePropertyResolver]: "ConfigurablePropertyResolver"

```java

```


### ConfigurableListableBeanFactory

源码参见：[ConfigurableListableBeanFactory][]

[ConfigurableListableBeanFactory]: "ConfigurableListableBeanFactory"

```java
```





## 总结
Lifecycle接口提供了启动和关闭操作，以及判断当前组件是否运行操作。需要注意的是启动和停止操作，将会传播给容器中的所有子容器中的组件。对于停止操作，不保证停止通知发生在析构之前。对于判断当前组件是否运行操作，如果组件是容器，只有在容器中所有组件包括子容器中的组件，都在运行的情况下，才返回true。

ConfigurableApplicationContext具备应用上下文 *ApplicationContex* 相关操作以外，同时具有了生命周期和流属性。除此之外，
提供了设置应用id，设置父类上下文，设置环境 *ConfigurableEnvironment*，添加应用监听器，添加bean工厂后处理器 *BeanFactoryPostProcessor*，添加协议解决器 *ProtocolResolver*，刷新应用上下文，关闭应用上下文，判断上下文状态，以及注册虚拟机关闭Hook等操作，同时重写了获取环境操作，此操作返回的为可配置环境 *ConfigurableEnvironment*。最关键的是提供了获取内部bean工厂的访问操作，
方法返回为 *ConfigurableListableBeanFactory*。需要注意的是，调用关闭操作，并不关闭父类的应用上下文，应用上下文与父类的上下文生命周期，相互独立。  

从ConfigurableApplicationContext的方法，而已看出ConfigurableApplicationContext主要实现的是set相关的操作，而ApplicationContext是get相关的操在，我们可以简单的理解为读写操作分离。

bean工厂后处理器BeanFactoryPostProcessor，主要提供了修改上下文的内部bean工厂操作，可以修改bean的定义，
而不能修改bean的实例属性。如果要与bean的实例交互，可以实现bean后处理器BeanPostProcessor。bean工厂后处理器一般用系统级的配置，比如 *PropertyResourceConfigurer*，并重写bean定义的属性。

ProtocolResolver接口，主要提供了根据资源位置加载相应资源的操作，在没有子类资源加载器和应用上下文的实现的情况下，默认资源加载器 *DefaultResourceLoader* 的系统包接口SPI，允许处理一般的资源协议。

ConfigurableEnvironment接口提供设置、添加环境配置，设置默认配置，获取属性源 *MutablePropertySources*，获取系统环境变量，获取系统属性，及合并环境。注意设置环境配置，会覆盖先前的配置，如果要添加配置到当前配置集，要使用添加环境配置方法。另外需要注意的是，父类的环境是不可修改，在调用{@code merge}方法前，我们应该优先配置父类的属性源和配置信息，同时子类的环境配置中的属性源将会时父类中的同名数据源失效。

# 附
应用上下文相关事件：
## ContextRefreshedEvent
```java
package org.springframework.context.event;

import org.springframework.context.ApplicationContext;

/**
 * Event raised when an {@code ApplicationContext} gets initialized or refreshed.
 *
 * @author Juergen Hoeller
 * @since 04.03.2003
 * @see ContextClosedEvent
 */
@SuppressWarnings("serial")
public class ContextRefreshedEvent extends ApplicationContextEvent {

	/**
	 * Create a new ContextRefreshedEvent.
	 * @param source the {@code ApplicationContext} that has been initialized
	 * or refreshed (must not be {@code null})
	 */
	public ContextRefreshedEvent(ApplicationContext source) {
		super(source);
	}
}
```

## ContextClosedEvent
```java
package org.springframework.context.event;

import org.springframework.context.ApplicationContext;

/**
 * Event raised when an {@code ApplicationContext} gets closed.
 * @author Juergen Hoeller
 * @since 12.08.2003
 * @see ContextRefreshedEvent
 */
@SuppressWarnings("serial")
public class ContextClosedEvent extends ApplicationContextEvent {

	/**
	 * Creates a new ContextClosedEvent.
	 * @param source the {@code ApplicationContext} that has been closed
	 * (must not be {@code null})
	 */
	public ContextClosedEvent(ApplicationContext source) {
		super(source);
	}
}
```
## ApplicationContextEvent

```java
package org.springframework.context.event;

import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationEvent;

/**
 * Base class for events raised for an {@code ApplicationContext}.
 *
 * @author Juergen Hoeller
 * @since 2.5
 */
@SuppressWarnings("serial")
public abstract class ApplicationContextEvent extends ApplicationEvent {

	/**
	 * Create a new ContextStartedEvent.
	 * @param source the {@code ApplicationContext} that the event is raised for
	 * (must not be {@code null})
	 */
	public ApplicationContextEvent(ApplicationContext source) {
		super(source);
	}

	/**
	 * Get the {@code ApplicationContext} that the event was raised for.
	 */
	public final ApplicationContext getApplicationContext() {
		return (ApplicationContext) getSource();
	}

}
```
