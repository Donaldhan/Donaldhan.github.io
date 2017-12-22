---
layout: page
title: AutowireCapableBeanFactory接口定义
subtitle: AutowireCapableBeanFactory接口定义
date: 2017-12-19 22:47:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

[ApplicationContext接口定义][]

[ApplicationContext接口定义]:  "ApplicationContext接口定义"

# 引言

上一篇文章我们看了[ApplicationContext接口定义][]接口及其父类接口的定义，先来回顾一下：   
ApplicationContext接口主要提供了获取父上下文，自动装配bean工厂 *AutowireCapableBeanFactory*，应用上下文name，展示name，启动时间戳及应用id的操作。应用上下文继承了 *EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,MessageSource, ApplicationEventPublisher, ResourcePatternResolver* ，具有了访问bean容器中组件，配置环境，加载文件或类路径资源，发布应用事件到监听器，已经解决国际化消息的功能。另外需要注意的是，应用上下文具有，父上下文的继承性（HierarchicalBeanFactory）。定义在子孙上下文中的bean定义将会有限考虑。这意味着，一个单独的父上下文可以被整个web应用上下文所使用。这一点体现在，当我们使用spring的核心容器特性和spring mvc时，在web.xml中，我们有两个配置一个是上下文监听器（org.springframework.web.context.ContextLoaderListener），同时需要配置应用上下文bean的定义配置，一般是ApplicationContext.xml，另一个是Servlet分发器（org.springframework.web.servlet.DispatcherServlet），同时需要配置WebMVC相关配置，一般是springmvc.xml。应用一般运行的在Web容器中，Web容器可以访问应用上下文，同时Web容器的Servlet也可以访问应用上下文，然而每个servlet有自己的上下文，独立于其他servlet。

![ApplicationContext](/image/spring-context/ApplicationContext.png)

今天我么来看一下*AutowireCapableBeanFactory* 接口的定义

# 目录
* [AutowireCapableBeanFactory接口定义](#autowirecapablebeanfactory接口定义)
    * [BeanPostProcessor](#beanpostprocessor)
    * [NamedBeanHolder](#namedbeanholder)
* [总结](#总结)




## AutowireCapableBeanFactory接口定义
具体源码，可以参见[AutowireCapableBeanFactory][]

[AutowireCapableBeanFactory]:
https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/config/AutowireCapableBeanFactory.java "AutowireCapableBeanFactory"

```java
/*
 * Copyright 2002-2016 the original author or authors.
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

import java.util.Set;

import org.springframework.beans.BeansException;
import org.springframework.beans.TypeConverter;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.NoSuchBeanDefinitionException;
import org.springframework.beans.factory.NoUniqueBeanDefinitionException;

/**
 * Extension of the {@link org.springframework.beans.factory.BeanFactory}
 * interface to be implemented by bean factories that are capable of
 * autowiring, provided that they want to expose this functionality for
 * existing bean instances.
 *AutowireCapableBeanFactory接口拓展了{@link org.springframework.beans.factory.BeanFactory}接口，
 *具体的bean工厂的实现可以自动装配，提供已经存在bean实例的功能性暴露。
 * <p>This subinterface of BeanFactory is not meant to be used in normal
 * application code: stick to {@link org.springframework.beans.factory.BeanFactory}
 * or {@link org.springframework.beans.factory.ListableBeanFactory} for
 * typical use cases.
 *此接口不意味者，可以用在正常的应用代码中。
 * <p>Integration code for other frameworks can leverage this interface to
 * wire and populate existing bean instances that Spring does not control
 * the lifecycle of. This is particularly useful for WebWork Actions and
 * Tapestry Page objects, for example.
 * 继承代码到其他框架，可以利用这个接口自动装配那些不在spring控制范围内的bean实例。
 * 对应WebWork的Actions特别有用。
 * <p>Note that this interface is not implemented by
 * {@link org.springframework.context.ApplicationContext} facades,
 * as it is hardly ever used by application code. That said, it is available
 * from an application context too, accessible through ApplicationContext's
 * {@link org.springframework.context.ApplicationContext#getAutowireCapableBeanFactory()}
 * method.
 * 需要注意的是，这个接口不是应用上下文门面的实现，在能在程序中硬编码使用。也就是说，
 * 我们可以通过应用上下的{@link org.springframework.context.ApplicationContext#getAutowireCapableBeanFactory()}
 * 的方法，进行访问。
 *
 * <p>You may also implement the {@link org.springframework.beans.factory.BeanFactoryAware}
 * interface, which exposes the internal BeanFactory even when running in an
 * ApplicationContext, to get access to an AutowireCapableBeanFactory:
 * simply cast the passed-in BeanFactory to AutowireCapableBeanFactory.
 *
 * 你也可以实现{@link org.springframework.beans.factory.BeanFactoryAware}接口，可以暴露内部的bean工厂；
 * 当运行在运行上下文中时，可以访问AutowireCapableBeanFactory：只需要将BeanFactory转化为AutowireCapableBeanFactory
 * 即可。
 *
 * @author Juergen Hoeller
 * @since 04.12.2003
 * @see org.springframework.beans.factory.BeanFactoryAware
 * @see org.springframework.beans.factory.config.ConfigurableListableBeanFactory
 * @see org.springframework.context.ApplicationContext#getAutowireCapableBeanFactory()
 */
public interface AutowireCapableBeanFactory extends BeanFactory {

	/**
	 * Constant that indicates no externally defined autowiring. Note that
	 * BeanFactoryAware etc and annotation-driven injection will still be applied.
	 * 常量，表示内部没有定义自动装配。主要BeanFactoryAware等和注解驱动依赖注入，依然启作用。
	 * @see #createBean
	 * @see #autowire
	 * @see #autowireBeanProperties
	 */
	int AUTOWIRE_NO = 0;

	/**
	 * Constant that indicates autowiring bean properties by name
	 * (applying to all bean property setters).
	 * 表示根据name自动装配bean属性（应用与bean所有setter属性）
	 * @see #createBean
	 * @see #autowire
	 * @see #autowireBeanProperties
	 */
	int AUTOWIRE_BY_NAME = 1;

	/**
	 * Constant that indicates autowiring bean properties by type
	 * (applying to all bean property setters).
	 * 表示依赖于类型自动装配bean属性（应用与bean所有setter属性）
	 * @see #createBean
	 * @see #autowire
	 * @see #autowireBeanProperties
	 */
	int AUTOWIRE_BY_TYPE = 2;

	/**
	 * Constant that indicates autowiring the greediest constructor that
	 * can be satisfied (involves resolving the appropriate constructor).
	 * 表示依赖于构造自动装配（解决构造相关的属性）
	 * @see #createBean
	 * @see #autowire
	 */
	int AUTOWIRE_CONSTRUCTOR = 3;

	/**
	 * Constant that indicates determining an appropriate autowire strategy
	 * through introspection of the bean class.
	 * 从spring3.o已经废弃，通过bean类的内省机制，确定一个合适的自动装配策略。
	 * @see #createBean
	 * @see #autowire
	 * @deprecated as of Spring 3.0: If you are using mixed autowiring strategies,
	 * prefer annotation-based autowiring for clearer demarcation of autowiring needs.
	 */
	@Deprecated
	int AUTOWIRE_AUTODETECT = 4;


	//-------------------------------------------------------------------------
	// Typical methods for creating and populating external bean instances
	//-------------------------------------------------------------------------

	/**
	 * Fully create a new bean instance of the given class.
	 * 完全创建一个新的给定class的bean实例。
	 * <p>Performs full initialization of the bean, including all applicable
	 * {@link BeanPostProcessor BeanPostProcessors}.
	 * 执行所有bean的初始化操作，包括所有应用层的bean处理器BeanPostProcessors。
	 * <p>Note: This is intended for creating a fresh instance, populating annotated
	 * fields and methods as well as applying all standard bean initialization callbacks.
	 * It does <i>not</> imply traditional by-name or by-type autowiring of properties;
	 * use {@link #createBean(Class, int, boolean)} for those purposes.
	 * 注意，这个方法，创建一个新的bean实例，并向所有bean初始化回调一样，处理注解fields和方法。但是，不意味着，
	 * 依赖name或类型，自动装配属性，为了这个目的可以使用{@link #createBean(Class, int, boolean)}方法。
	 * @param beanClass the class of the bean to create
	 * 创建bean的类型
	 * @return the new bean instance 返回bean的实例
	 * @throws BeansException if instantiation or wiring failed
	 * 如果自动装配或初始化失败，则抛出BeansException异常。
	 */
	<T> T createBean(Class<T> beanClass) throws BeansException;

	/**
	 * Populate the given bean instance through applying after-instantiation callbacks
	 * and bean property post-processing (e.g. for annotation-driven injection).
	 * 在初始化回调处理完后，装配给定的bean实例，比如注解自动装配。
	 * <p>Note: This is essentially intended for (re-)populating annotated fields and
	 * methods, either for new instances or for deserialized instances. It does
	 * <i>not</i> imply traditional by-name or by-type autowiring of properties;
	 * use {@link #autowireBeanProperties} for those purposes.
	 * 注意，方法用于处理注解驱动的fields的方法，比如新的实例，或反序列化实例。但是，不意味着，
	 * 依赖name或类型，自动装配属性，为了这个目的可以使用 {@link #autowireBeanProperties}方法
	 * @param existingBean the existing bean instance
	 * 已经完成标准初始化回调的bean实例
	 * @throws BeansException if wiring failed
	 * 如果自动装配失败，则抛出BeansException异常。
	 */
	void autowireBean(Object existingBean) throws BeansException;

	/**
	 * Configure the given raw bean: autowiring bean properties, applying
	 * bean property values, applying factory callbacks such as {@code setBeanName}
	 * and {@code setBeanFactory}, and also applying all bean post processors
	 * (including ones which might wrap the given raw bean).
	 * 配置给定原始bean：自动装配bean属性，bean属性值，应用工厂回调，比如{@code setBeanName}，{@code setBeanFactory}，
	 * 同时，应用所有bean后处理器，（包括，bean实例包装的原始bean）
	 * <p>This is effectively a superset of what {@link #initializeBean} provides,
	 * fully applying the configuration specified by the corresponding bean definition.
	 * <b>Note: This method requires a bean definition for the given name!</b>
	 * 此方法相当与{@link #initializeBean}与依赖bean定义全配置bean。
	 * 注意：此方法需要一个的bean定义的name。
	 * @param existingBean the existing bean instance
	 * 已经存在的bean实例
	 * @param beanName the name of the bean, to be passed to it if necessary
	 * (a bean definition of that name has to be available)
	 * 如果需要，则传入一个bean定义的name
	 * @return the bean instance to use, either the original or a wrapped one
	 *返回bean实例，或是原始实例，或包装后的实例。
	 * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
	 * if there is no bean definition with the given name
	 * 如果没有给定的name对应的bean定义，则抛出NoSuchBeanDefinitionException异常。
	 * @throws BeansException if the initialization failed
	 * 如果初始化，则抛出BeansException异常。
	 * @see #initializeBean
	 */
	Object configureBean(Object existingBean, String beanName) throws BeansException;


	//-------------------------------------------------------------------------
	// Specialized methods for fine-grained control over the bean lifecycle
	//-------------------------------------------------------------------------

	/**
	 * 此方法与上述createBean(Class<T> beanClass)方法，不同的是，控制自动装配的策略，是依赖name还是类型，还是构造。
	 * Fully create a new bean instance of the given class with the specified
	 * autowire strategy. All constants defined in this interface are supported here.
	 * <p>Performs full initialization of the bean, including all applicable
	 * {@link BeanPostProcessor BeanPostProcessors}. This is effectively a superset
	 * of what {@link #autowire} provides, adding {@link #initializeBean} behavior.
	 * @param beanClass the class of the bean to create
	 * @param autowireMode by name or type, using the constants in this interface
	 * 依赖于类型还是name，还是构造进行自动装配
	 * @param dependencyCheck whether to perform a dependency check for objects
	 * (not applicable to autowiring a constructor, thus ignored there)
	 * 是否执行依赖检查
	 * @return the new bean instance
	 * @throws BeansException if instantiation or wiring failed
	 * @see #AUTOWIRE_NO
	 * @see #AUTOWIRE_BY_NAME
	 * @see #AUTOWIRE_BY_TYPE
	 * @see #AUTOWIRE_CONSTRUCTOR
	 */
	Object createBean(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException;

	/**
	 * 此方法与上述autowireBean(Object existingBean)方法，不同的是，控制自动装配的策略，是依赖name还是类型，还是构造。
	 * Instantiate a new bean instance of the given class with the specified autowire
	 * strategy. All constants defined in this interface are supported here.
	 * Can also be invoked with {@code AUTOWIRE_NO} in order to just apply
	 * before-instantiation callbacks (e.g. for annotation-driven injection).
	 * 在初始化回调以前，比如注解驱动注入，为了调整应用，可以传入{@code AUTOWIRE_NO}。
	 * <p>Does <i>not</i> apply standard {@link BeanPostProcessor BeanPostProcessors}
	 * callbacks or perform any further initialization of the bean.
	 * 需要注意的是，此方法不应用bean后处理器回调和进一步的bean初始化。
	 * This interface offers distinct, fine-grained operations for those purposes, for example
	 * {@link #initializeBean}. However, {@link InstantiationAwareBeanPostProcessor}
	 * callbacks are applied, if applicable to the construction of the instance.
	 * 此方法与#initializeBean方法不同。然而如果使用是构造实例模式，将会调用{@link InstantiationAwareBeanPostProcessor}回调。
	 * @param beanClass the class of the bean to instantiate
	 * @param autowireMode by name or type, using the constants in this interface
	 * @param dependencyCheck whether to perform a dependency check for object
	 * references in the bean instance (not applicable to autowiring a constructor,
	 * thus ignored there)
	 * @return the new bean instance
	 * @throws BeansException if instantiation or wiring failed
	 * @see #AUTOWIRE_NO
	 * @see #AUTOWIRE_BY_NAME
	 * @see #AUTOWIRE_BY_TYPE
	 * @see #AUTOWIRE_CONSTRUCTOR
	 * @see #AUTOWIRE_AUTODETECT
	 * @see #initializeBean
	 * @see #applyBeanPostProcessorsBeforeInitialization
	 * @see #applyBeanPostProcessorsAfterInitialization
	 */
	Object autowire(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException;

	/**
	 * 此方主要是自动装配bean的属性。
	 * Autowire the bean properties of the given bean instance by name or type.
	 * Can also be invoked with {@code AUTOWIRE_NO} in order to just apply
	 * after-instantiation callbacks (e.g. for annotation-driven injection).
	 * 依赖于name和类型自动装配给定bean的属性。 在初始化回调以前，比如注解驱动注入，为了调整应用，可以传入{@code AUTOWIRE_NO}。
	 * <p>Does <i>not</i> apply standard {@link BeanPostProcessor BeanPostProcessors}
	 * callbacks or perform any further initialization of the bean. This interface
	 * offers distinct, fine-grained operations for those purposes, for example
	 * {@link #initializeBean}. However, {@link InstantiationAwareBeanPostProcessor}
	 * callbacks are applied, if applicable to the configuration of the instance.
	 * @param existingBean the existing bean instance
	 * @param autowireMode by name or type, using the constants in this interface
	 * @param dependencyCheck whether to perform a dependency check for object
	 * references in the bean instance
	 * @throws BeansException if wiring failed
	 * @see #AUTOWIRE_BY_NAME
	 * @see #AUTOWIRE_BY_TYPE
	 * @see #AUTOWIRE_NO
	 */
	void autowireBeanProperties(Object existingBean, int autowireMode, boolean dependencyCheck)
			throws BeansException;

	/**
	 * Apply the property values of the bean definition with the given name to
	 * the given bean instance. The bean definition can either define a fully
	 * self-contained bean, reusing its property values, or just property values
	 * meant to be used for existing bean instances.
	 * 应用给定name的bean的定义的属性给指定bean实例。bean定义可以是一个完全自包含的bean，重用他的属性，或
	 * 调整属性，意味着用于已经存在的bean实例。
	 * <p>This method does <i>not</i> autowire bean properties; it just applies
	 * explicitly defined property values. Use the {@link #autowireBeanProperties}
	 * method to autowire an existing bean instance.
	 * 此方法不会自动装配bean属性，仅仅使用显示定义的bean的属性值。使用 {@link #autowireBeanProperties}方法自动注入
	 * 已经存在的bean实例。
	 * <b>Note: This method requires a bean definition for the given name!</b>
	 * 注意：此方法需要bean定义的给定name。
	 * <p>Does <i>not</i> apply standard {@link BeanPostProcessor BeanPostProcessors}
	 * callbacks or perform any further initialization of the bean. This interface
	 * offers distinct, fine-grained operations for those purposes, for example
	 * {@link #initializeBean}. However, {@link InstantiationAwareBeanPostProcessor}
	 * callbacks are applied, if applicable to the configuration of the instance.
	 * 需要注意的是，此方法不应用bean后处理器回调和进一步的bean初始化。此方法与#initializeBean方法不同。
	 * 然而如果应用到配置实例，将会调用{@link InstantiationAwareBeanPostProcessor}回调。
	 * @param existingBean the existing bean instance
	 * @param beanName the name of the bean definition in the bean factory
	 * (a bean definition of that name has to be available)
	 * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
	 * if there is no bean definition with the given name
	 * @throws BeansException if applying the property values failed
	 * @see #autowireBeanProperties
	 */
	void applyBeanPropertyValues(Object existingBean, String beanName) throws BeansException;

	/**
	 * Initialize the given raw bean, applying factory callbacks
	 * such as {@code setBeanName} and {@code setBeanFactory},
	 * also applying all bean post processors (including ones which
	 * might wrap the given raw bean).
	 * 初始化给定的原始bean，应用工厂调用，比如{@code setBeanName} 和 {@code setBeanFactory},
	 * 同时应有所有bean后处理器，包括包装的指定原始bean。
	 * <p>Note that no bean definition of the given name has to exist
	 * in the bean factory. The passed-in bean name will simply be used
	 * for callbacks but not checked against the registered bean definitions.
	 * 注意：如果在bean工厂中，必须有给定的name的bean的定义。bean的name仅仅用于回调，并不检查注册bean的定义。
	 * @param existingBean the existing bean instance
	 * @param beanName the name of the bean, to be passed to it if necessary
	 * (only passed to {@link BeanPostProcessor BeanPostProcessors})
	 * @return the bean instance to use, either the original or a wrapped one
	 * @throws BeansException if the initialization failed
	 */
	Object initializeBean(Object existingBean, String beanName) throws BeansException;

	/**
	 * Apply {@link BeanPostProcessor BeanPostProcessors} to the given existing bean
	 * instance, invoking their {@code postProcessBeforeInitialization} methods.
	 * The returned bean instance may be a wrapper around the original.
	 * 应用{@link BeanPostProcessor BeanPostProcessors}到给定存在的bean实例，并调用{@code postProcessBeforeInitialization}
	 * 方法，返回的bean实例，也许是一个原始的包装bean。
	 * @param existingBean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one
	 * @throws BeansException if any post-processing failed
	 * @see BeanPostProcessor#postProcessBeforeInitialization
	 */
	Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException;

	/**
	 * Apply {@link BeanPostProcessor BeanPostProcessors} to the given existing bean
	 * instance, invoking their {@code postProcessAfterInitialization} methods.
	 * The returned bean instance may be a wrapper around the original.
	 * 此方法与上面方法不同是调用bean后处理的{@code postProcessAfterInitialization}。
	 * @param existingBean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one
	 * @throws BeansException if any post-processing failed
	 * @see BeanPostProcessor#postProcessAfterInitialization
	 */
	Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException;

	/**
	 * Destroy the given bean instance (typically coming from {@link #createBean}),
	 * applying the {@link org.springframework.beans.factory.DisposableBean} contract as well as
	 * registered {@link DestructionAwareBeanPostProcessor DestructionAwareBeanPostProcessors}.
	 * <p>Any exception that arises during destruction should be caught
	 * and logged instead of propagated to the caller of this method.
	 * 销毁给定bean的实例，比如来自于{@link #createBean})方法创建的bean实例，
	 * 同时调用{@link org.springframework.beans.factory.DisposableBean}，以及注册的
	 * {@link DestructionAwareBeanPostProcessor DestructionAwareBeanPostProcessors}.
	 * 在析构的构成中国，任何异常的抛出，将会传播到方法的调用者。
	 * @param existingBean the bean instance to destroy
	 */
	void destroyBean(Object existingBean);


	//-------------------------------------------------------------------------
	// Delegate methods for resolving injection points
	//-------------------------------------------------------------------------

	/**
	 * Resolve the bean instance that uniquely matches the given object type, if any,
	 * including its bean name.
	 * 若果存在，返回唯一匹配自定类型的bean实例，包括bean的name。
	 * <p>This is effectively a variant of {@link #getBean(Class)} which preserves the
	 * bean name of the matching instance.
	 * 此方法等同于{@link #getBean(Class)}方法，并保存bean的name。
	 * @param requiredType type the bean must match; can be an interface or superclass.
	 * {@code null} is disallowed.
	 * 需要匹配的类型，可以是接口或类，但不能为null。
	 * @return the bean name plus bean instance
	 * @throws NoSuchBeanDefinitionException if no matching bean was found
	 * @throws NoUniqueBeanDefinitionException if more than one matching bean was found
	 * 如果存在多个匹配的bean，则抛出NoUniqueBeanDefinitionException异常
	 * @throws BeansException if the bean could not be created
	 * @since 4.3.3
	 * @see #getBean(Class)
	 */
	<T> NamedBeanHolder<T> resolveNamedBean(Class<T> requiredType) throws BeansException;

	/**
	 * Resolve the specified dependency against the beans defined in this factory.
	 * 在工厂根据bean的定义，解决特殊的依赖。
	 * @param descriptor the descriptor for the dependency (field/method/constructor)
	 * @param requestingBeanName the name of the bean which declares the given dependency
	 * @return the resolved object, or {@code null} if none found
	 * @throws NoSuchBeanDefinitionException if no matching bean was found
	 * @throws NoUniqueBeanDefinitionException if more than one matching bean was found
	 * @throws BeansException if dependency resolution failed for any other reason
	 * @since 2.5
	 * @see #resolveDependency(DependencyDescriptor, String, Set, TypeConverter)
	 */
	Object resolveDependency(DependencyDescriptor descriptor, String requestingBeanName) throws BeansException;

	/**
	 * Resolve the specified dependency against the beans defined in this factory.
	 * 此方法与上面方法的不同是，多了一个类型转换器
	 * @param descriptor the descriptor for the dependency (field/method/constructor)
	 * @param requestingBeanName the name of the bean which declares the given dependency
	 * @param autowiredBeanNames a Set that all names of autowired beans (used for
	 * resolving the given dependency) are supposed to be added to
	 * @param typeConverter the TypeConverter to use for populating arrays and collections
	 * 用于数组与集合类的转换。
	 * @return the resolved object, or {@code null} if none found
	 * @throws NoSuchBeanDefinitionException if no matching bean was found
	 * @throws NoUniqueBeanDefinitionException if more than one matching bean was found
	 * @throws BeansException if dependency resolution failed for any other reason
	 * @since 2.5
	 * @see DependencyDescriptor
	 */
	Object resolveDependency(DependencyDescriptor descriptor, String requestingBeanName,
			Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException;

}
```
从上面可看出，AutowireCapableBeanFactory接口，主要提供的创建bean实例，自动装配bean属性，应用bean配置属性，初始化bean，应用bean后处理器 *BeanPostProcessor* ，解决bean依赖和销毁bean操作。对于自动装配，主要提供了根据bean的name，类型和构造自动装配方式。一般不建议在
在代码中直接使用AutowireCapableBeanFactory接口，我们可以通过应用上下文的ApplicationContext#getAutowireCapableBeanFactory()方法或者通过实现BeanFactoryAware，获取暴露的bean工厂，然后转换为AutowireCapableBeanFactory。

AutowireCapableBeanFactory应用bean后处理器的操作中，涉及到一个接口为 *BeanPostProcessor*，在AutowireCapableBeanFactory获取指定类型的唯一bean实例操作中，有一个 *NamedBeanHolder*，我们再来看一下NamedBeanHolder的定义。

我们先来看一下bean后处理器BeanPostProcessor接口
### BeanPostProcessor


具体源码参见：[BeanPostProcessor][]

[BeanPostProcessor]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/config/BeanPostProcessor.java "BeanPostProcessor"

```java
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;

/**
 *bean后处理器BeanPostProcessor是一个运行修改bean实例的工厂Hook。
 *应用上下文ApplicationContexts，可以在bean的定义中，自动探测bean后处理，并应用它到后续创建的bean。
 *空白的bean工厂允许编程上注册bean后处理器，应用到所有工厂创建的bean。
 *典型应用为，通过实现{@link #postProcessBeforeInitialization}方法标记接口，
 *通过实现{@link #postProcessAfterInitialization}方法，使用代理包装初始化后的bean实例。
 * @author Juergen Hoeller
 * @since 10.10.2003，
 * @see InstantiationAwareBeanPostProcessor
 * @see DestructionAwareBeanPostProcessor
 * @see ConfigurableBeanFactory#addBeanPostProcessor
 * @see BeanFactoryPostProcessor
 */
public interface BeanPostProcessor {

	/**
	 * 在任何bean初始化回调（比如初始beanInitializingBean的{@code afterPropertiesSet}方法，和
	 * 一般的初始化方法）之前，应用bean后处理器到给定的bean实例。bean将会配置属性值。
	 * 返回的bean实例可能是一个原始bean的包装。
	 * @param bean the new bean instance
	 * 新创建的bean实例
	 * @param beanName the name of the bean
	 * bean的name
	 * @return the bean instance to use, either the original or a wrapped one;
	 * if {@code null}, no subsequent BeanPostProcessors will be invoked
	 * 返回bean的实例，有可能是原始类，也有可能是原始类包装。
	 * @throws org.springframework.beans.BeansException in case of errors
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 */
	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

	/**
	 *  在任何bean初始化回调（比如初始beanInitializingBean的{@code afterPropertiesSet}方法，和
	 * 一般的初始化方法）之后，应用bean后处理器到给定的bean实例。bean将会配置属性值。
	 * 返回的bean实例可能是一个原始bean的包装。
	 * 在工厂bean的情况下，从spring2.0开始，工厂bean创建对象和工厂bean初始化的时候，
	 * 都会调用此回调。bean后处理器，通过相关{@code bean instanceof FactoryBean}，即bean是否为
	 * 工厂bean的检查，来决定是否应用到工厂bean或创建的对象，或两者都会调用此回调。
	 * 与其他bean形成鲜明对比的是，{@link InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation}
	 * 方法触发以后，回调也会触发。
	 * @param bean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one;
	 * if {@code null}, no subsequent BeanPostProcessors will be invoked
	 * @throws org.springframework.beans.BeansException in case of errors
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.beans.factory.FactoryBean
	 */
	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}

```
从上面可以看出，bean后处理器BeanPostProcessor，提供了初始化bean的操作，一个是在 [InitializingBean][] 的{@code afterPropertiesSet}方法之前，一个是在之后。初始化之后的操作，对于bean为工厂bean的情况，通过判断bean是否为
工厂bean的检查，来决定是否应用到工厂bean或创建的对象，或两者都会调用此回调，与其他bean形成鲜明对比的是，{@link [InstantiationAwareBeanPostProcessor][]#postProcessBeforeInstantiation}方法触发以后，回调也会触发。

[InitializingBean]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/InitializingBean.java "InitializingBean"   
[InstantiationAwareBeanPostProcessor]:
https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/config/InstantiationAwareBeanPostProcessor.java "InstantiationAwareBeanPostProcessor"

在AutowireCapableBeanFactory获取指定类型的唯一bean实例操作中，有一个NamedBeanHolder，我们再来看一下NamedBeanHolder的定义。

```java
<T> NamedBeanHolder<T> resolveNamedBean(Class<T> requiredType) throws BeansException;
```

### NamedBeanHolder
具体源码参见：[NamedBeanHolder][]

[NamedBeanHolder]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/config/NamedBeanHolder.java "NamedBeanHolder"

```java
package org.springframework.beans.factory.config;

import org.springframework.beans.factory.NamedBean;
import org.springframework.util.Assert;

/**
 * A simple holder for a given bean name plus bean instance.
 *bean的name和实例的句柄
 * @author Juergen Hoeller
 * @since 4.3.3
 * @see AutowireCapableBeanFactory#resolveNamedBean(Class)
 */
public class NamedBeanHolder<T> implements NamedBean {

	private final String beanName;//bean name

	private final T beanInstance;//bean 实例


	/**
	 * Create a new holder for the given bean name plus instance.
	 * 根据bean的name和实例创建bean句柄
	 * @param beanName the name of the bean
	 * @param beanInstance the corresponding bean instance
	 */
	public NamedBeanHolder(String beanName, T beanInstance) {
		Assert.notNull(beanName, "Bean name must not be null");
		this.beanName = beanName;
		this.beanInstance = beanInstance;
	}


	/**
	 * Return the name of the bean (never {@code null}).
	 */
	@Override
	public String getBeanName() {
		return this.beanName;
	}

	/**
	 * Return the corresponding bean instance (can be {@code null}).
	 */
	public T getBeanInstance() {
		return this.beanInstance;
	}

}
```
从上面可以看出，NamedBeanHolder用于表示bean的name和实例的关系句柄。再来看一下NamedBean接口的定义。
### NamedBean接口
具体源码参见：[NamedBean][]

[NamedBean]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/NamedBean.java "NamedBean"

```java
package org.springframework.beans.factory;

/**
 * Counterpart of {@link BeanNameAware}. Returns the bean name of an object.
 * NamedBean接口可以理解为BeanNameAware的副本。返回对象bean的name。
 * <p>This interface can be introduced to avoid a brittle dependence on
 * bean name in objects used with Spring IoC and Spring AOP.
 * 此接口可用于Spring依赖注入和AOP中，解决bean name的依赖，避免产生不可靠的依赖。
 * @author Rod Johnson
 * @since 2.0
 * @see BeanNameAware
 */
public interface NamedBean {

	/**
	 * Return the name of this bean in a Spring bean factory, if known.
	 * 如果知道，则返回spring bean工厂中的bean的name。
	 */
	String getBeanName();

}
```
从上面可以看出，NamedBean接口主要提供了获取bean的name的操作。

今天我们用AutowireCapableBeanFactory类图结束本篇文章。

![AutowireCapableBeanFactory](/image/spring-context/AutowireCapableBeanFactory.png)

## 总结

AutowireCapableBeanFactory接口，主要提供的创建bean实例，自动装配bean属性，应用bean配置属性，初始化bean，应用bean后处理器 *BeanPostProcessor* ，解决bean依赖和销毁bean操作。对于自动装配，主要提供了根据bean的name，类型和构造自动装配方式。一般不建议在
在代码中直接使用AutowireCapableBeanFactory接口，我们可以通过应用上下文的ApplicationContext#getAutowireCapableBeanFactory()方法或者通过实现BeanFactoryAware，获取暴露的bean工厂，然后转换为AutowireCapableBeanFactory。  
bean后处理器BeanPostProcessor，提供了初始化bean的操作，一个是在 [InitializingBean][] 的{@code afterPropertiesSet}方法之前，一个是在之后。初始化之后的操作，对于bean为工厂bean的情况，通过判断bean是否为
工厂bean的检查，来决定是否应用到工厂bean或创建的对象，或两者都会调用此回调，与其他bean形成鲜明对比的是，{@link [InstantiationAwareBeanPostProcessor][]#postProcessBeforeInstantiation}方法触发以后，回调也会触发。  
NamedBeanHolder用于表示bean的name和实例的关系句柄。NamedBeanHolder可以用于Spring的根据bean的name自动装配和AOP相关的功能，避免产生不可靠的依赖。
