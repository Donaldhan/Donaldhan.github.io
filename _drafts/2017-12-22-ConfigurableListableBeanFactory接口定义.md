---
layout: page
title: ConfigurableListableBeanFactory接口定义
subtitle: ConfigurableListableBeanFactory接口及父接口定义
date: 2017-12-22 10:35:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言

ConfigurableApplicationContext具备应用上下文 *ApplicationContex* 相关操作以外，同时具有了生命周期和流属性。除此之外，
提供了设置应用id，设置父类上下文，设置环境 *ConfigurableEnvironment*，添加应用监听器，添加bean工厂后处理器 *BeanFactoryPostProcessor*，添加协议解决器 *ProtocolResolver*，刷新应用上下文，关闭应用上下文，判断上下文状态，以及注册虚拟机关闭Hook等操作，同时重写了获取环境操作，此操作返回的为可配置环境 *ConfigurableEnvironment*。最关键的是提供了获取内部bean工厂的访问操作，
方法返回为 *ConfigurableListableBeanFactory*。需要注意的是，调用关闭操作，并不关闭父类的应用上下文，应用上下文与父类的上下文生命周期，相互独立。

![ConfigurableApplicationContext](/image/spring-context/ConfigurableApplicationContext.png)

今天我们来看一下ConfigurableListableBeanFactory接口的定义。
# 目录
* [ConfigurableApplicationContext接口定义](#configurableapplicationcontext接口定义)
    * [ConfigurableBeanFactory](#configurablebeanfactory)
    * [SingletonBeanRegistry](#singletonbeanregistry)
* [总结](#总结)
* [附](#附)



### ConfigurableListableBeanFactory接口定义

源码参见：[ConfigurableListableBeanFactory][]

[ConfigurableListableBeanFactory]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/config/ConfigurableListableBeanFactory.java "ConfigurableListableBeanFactory"

```java
package org.springframework.beans.factory.config;

import java.util.Iterator;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.ListableBeanFactory;
import org.springframework.beans.factory.NoSuchBeanDefinitionException;

/**
 * ConfigurableListableBeanFactory是大多数listable bean工厂实现的配置接口，为分析修改bean的定义
 * 或单例bean预初始化提供了便利。
 * 此接口为bean工厂的子接口，不意味着可以在正常的应用代码中使用，在典型的应用中，使用{@link org.springframework.beans.factory.BeanFactory}
 * 或者{@link org.springframework.beans.factory.ListableBeanFactory}。当需要访问bean工厂配置方法时，
 * 此接口只允许框架内部使用。
 * @author Juergen Hoeller
 * @since 03.11.2003
 * @see org.springframework.context.support.AbstractApplicationContext#getBeanFactory()
 */
public interface ConfigurableListableBeanFactory
		extends ListableBeanFactory, AutowireCapableBeanFactory, ConfigurableBeanFactory {

	/**
	 * 忽略给定的依赖类型的自动注入，比如String。默认为无。
	 * @param type the dependency type to ignore
	 */
	void ignoreDependencyType(Class<?> type);

	/**
	 * 忽略给定的依赖接口的自动注入。
	 * 应用上下文注意解决依赖的其他方法，比如通过BeanFactoryAware的BeanFactory，
	 * 通过ApplicationContextAware的ApplicationContext。
	 * 默认，仅仅BeanFactoryAware接口被忽略。如果想要更多的类型被忽略，调用此方法即可。
	 * @param ifc the dependency interface to ignore
	 * @see org.springframework.beans.factory.BeanFactoryAware
	 * @see org.springframework.context.ApplicationContextAware
	 */
	void ignoreDependencyInterface(Class<?> ifc);

	/**
	 * 注册一个与自动注入值相关的特殊依赖类型。这个方法主要用于，工厂、上下文的引用的自动注入，然而工厂和
	 * 上下文的实例bean，并不工厂中：比如应用上下文的依赖，可以解决应用上下文实例中的bean。
	 * 需要注意的是：在一个空白的bean工厂中，没有这种默认的类型注册，设置没有bean工厂接口字节。
	 * @param dependencyType
	 * 需要注册的依赖类型。典型地使用，比如一个bean工厂接口，只要给定的自动注入依赖是bean工厂的拓展即可，
	 * 比如ListableBeanFactory。
	 * @param autowiredValue
	 * 相关的自动注入的值。也许是一个对象工厂{@link org.springframework.beans.factory.ObjectFactory}的实现，
	 * 运行懒加载方式解决实际的目标值。
	 */
	void registerResolvableDependency(Class<?> dependencyType, Object autowiredValue);

	/**
	 * 决定给定name的对应bean，是否可以作为其他bean中声明匹配的自动依赖注入类型的候选。此方法检查祖先工厂。
	 * @param beanName the name of the bean to check
	 * 需要检查的bean的name
	 * @param descriptor the descriptor of the dependency to resolve
	 * 依赖描述
	 * @return whether the bean should be considered as autowire candidate
	 * 返回bean是否为自动注入的候选
	 * @throws NoSuchBeanDefinitionException if there is no bean with the given name
	 */
	boolean isAutowireCandidate(String beanName, DependencyDescriptor descriptor)
			throws NoSuchBeanDefinitionException;

	/**
	 * 返回给定bean的bean定义，运行访问bean的属性值和构造参数（可以在bean工厂后处理器处理的过程中修改）。
	 * 返回的bean定义不应该为bean定义的copy，而是原始注册到bean工厂的bean的定义。意味着，如果需要，
	 * 应该投射到一个更精确的类型。
	 * 此方法不考虑祖先工厂。即只能访问当前工厂中bean定义。
	 * @param beanName the name of the bean
	 * @return the registered BeanDefinition
	 * @throws NoSuchBeanDefinitionException if there is no bean with the given name
	 * defined in this factory
	 */
	BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

	/**
	 * 返回当前工厂中的所有bean的name的统一视图集。
	 * 包括bean的定义，及自动注册的单例bean实例，首先bean定义与bean的name一致，然后根据类型、注解检索bean的name。
	 * @return the composite iterator for the bean names view
	 * @since 4.1.2
	 * @see #containsBeanDefinition
	 * @see #registerSingleton
	 * @see #getBeanNamesForType
	 * @see #getBeanNamesForAnnotation
	 */
	Iterator<String> getBeanNamesIterator();

	/**
	 * 清除整合bean定义的缓存，移除还没有缓存所有元数据的bean。
	 * 典型的触发场景，在原始的bean定义修改之后，比如应用 {@link BeanFactoryPostProcessor}。需要注意的是，
	 * 在当前时间点，bean定义已经存在的元数据将会被保存。
	 * @since 4.2
	 * @see #getBeanDefinition
	 * @see #getMergedBeanDefinition
	 */
	void clearMetadataCache();

	/**
	 * 冻结所有bean定义，通知上下文，注册的bean定义不能在修改，及进一步的后处理。
	 * <p>This allows the factory to aggressively cache bean definition metadata.
	 */
	void freezeConfiguration();

	/**
	 * 返回bean工厂中的bean定义是否已经冻结。即不应该修改和进一步的后处理。
	 * @return {@code true} if the factory's configuration is considered frozen
	 * 如果冻结，则返回true。
	 */
	boolean isConfigurationFrozen();

	/**
	 * 确保所有非懒加载单例bean被初始化，包括工厂bean{@link org.springframework.beans.factory.FactoryBean FactoryBeans}。
	 * Typically invoked at the end of factory setup, if desired.
	 * 如果需要，在bean工厂设置后，调用此方法。
	 * @throws BeansException
	 * 如果任何一个单例bean不能够创建，将抛出BeansException。
	 * 需要注意的是：操作有可能遗留一些已经初始化的bean，可以调用{@link #destroySingletons()}完全清楚。
	 * @see #destroySingletons()
	 */
	void preInstantiateSingletons() throws BeansException;

}
```
从上面可以看出，ConfigurableListableBeanFactory接口主要提供了，注册给定自动注入值的依赖类型，决定给定name的对应bean，是否可以作为其他bean中声明匹配的自动依赖注入类型的候选，此操作检查祖先工厂。获取给定name的bean的定义，忽略给定类型或接口的依赖自动注入，获取工厂中的bean的name集操作，同时提供了清除不被考虑的bean的元数据缓存，
冻结bean工厂的bean的定义，判断bean工厂的bean定义是否冻结，以及确保所有非懒加载单例bean被初始化，包括工厂bean相关操作。需要注意的是，bean工厂冻结后，注册的bean定义不能在修改，及进一步的后处理；如果确保所有非懒加载单例bean被初始化失败，记得调用{@link #destroySingletons()}方法，清除已经初始化的单例bean。

ConfigurableListableBeanFactory接口继承了 *ListableBeanFactory, AutowireCapableBeanFactory, ConfigurableBeanFactory* 接口， *ListableBeanFactory, AutowireCapableBeanFactory* 前文中一分析过，我们再来一下可配置bean工厂 *ConfigurableBeanFactory* 接口的定义。

### ConfigurableBeanFactory
源码参见：[ConfigurableBeanFactory][]

[ConfigurableBeanFactory]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/config/ConfigurableBeanFactory.java "ConfigurableBeanFactory"

```java
package org.springframework.beans.factory.config;

import java.beans.PropertyEditor;
import java.security.AccessControlContext;

import org.springframework.beans.PropertyEditorRegistrar;
import org.springframework.beans.PropertyEditorRegistry;
import org.springframework.beans.TypeConverter;
import org.springframework.beans.factory.BeanDefinitionStoreException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.HierarchicalBeanFactory;
import org.springframework.beans.factory.NoSuchBeanDefinitionException;
import org.springframework.core.convert.ConversionService;
import org.springframework.util.StringValueResolver;

/**
 *ConfigurableBeanFactory是一个大多数bean工厂都会实现的接口。为配置bean工厂和工厂接口中客户端操作，
 *提供了便利。
 * 配置bean工厂不以为者可以在应用代码中，直接使用：应配合{@link org.springframework.beans.factory.BeanFactory}
 * 或@link org.springframework.beans.factory.ListableBeanFactory}使用。
 * 此接口的扩展接口，运行框架内部使用，用于访问bean工厂的配置方法。
 *
 * @author Juergen Hoeller
 * @since 03.11.2003
 * @see org.springframework.beans.factory.BeanFactory
 * @see org.springframework.beans.factory.ListableBeanFactory
 * @see ConfigurableListableBeanFactory
 */
public interface ConfigurableBeanFactory extends HierarchicalBeanFactory, SingletonBeanRegistry {

	/**
	 * 标准的单例作用域模式标识。一般的作用域可以通过{@code registerScope}方法注册。
	 * @see #registerScope
	 */
	String SCOPE_SINGLETON = "singleton";

	/**
	 * 标准的原型作用域模式标识。一般的作用域可以通过{@code registerScope}方法注册。
	 * @see #registerScope
	 */
	String SCOPE_PROTOTYPE = "prototype";


	/**
	 * 设置bean工厂的父bean工厂。需要注意的是，如果父类bean工厂在工厂初始化的时候，如果不可能，
	 * 应该在构造外部进行设置，这时父类不能改变。
	 * @param parentBeanFactory the parent BeanFactory
	 * @throws IllegalStateException if this factory is already associated with
	 * a parent BeanFactory
	 * @see #getParentBeanFactory()
	 */
	void setParentBeanFactory(BeanFactory parentBeanFactory) throws IllegalStateException;

	/**
	 * 设置工厂加载bean类的类加载器。默认为当前线程上下文的类加载器。
	 * 需要注意的是，如果类的定义还有对应的bean类型解决器，类加载器只能应用于bean的定义。
	 * 从spring2.0开始，一旦工厂处理bean的定义，仅仅拥有bean类型名字的bean定义才能被解决。
	 * @param beanClassLoader the class loader to use,
	 * or {@code null} to suggest the default class loader
	 */
	void setBeanClassLoader(ClassLoader beanClassLoader);

	/**
	 * 返回当前bean工厂的类加载器。
	 */
	ClassLoader getBeanClassLoader();

	/**
	 * 设置工厂的临时类加载器，一般用于类型匹配的目的。默认为无，简单地使用标准的bean类型加载器。
	 * 如果处于加载织入时间（load-time weaving），为确保实际的bean类型尽可能的懒加载，
	 * 一个临时的类加载器通常需要指定。一旦在bean工厂启动完成阶段后，临时类加载器将会被移除。
	 * @since 2.5
	 */
	void setTempClassLoader(ClassLoader tempClassLoader);

	/**
	 * 返回临时类加载器
	 * if any.
	 * @since 2.5
	 */
	ClassLoader getTempClassLoader();

	/**
	 *设置是否缓存bean的元数据，比如bean的定义，bean类型解决器。默认是缓存。
	 * 关闭缓存bean元数据，将会开启一些特殊bean定义对象的热刷新。如果关闭缓存，
	 * 任何bean实例创建时，将会重新为新创建的类查询bean类加载器。
	 */
	void setCacheBeanMetadata(boolean cacheBeanMetadata);

	/**
	 * 返回当前是否缓存bean的元数据。
	 */
	boolean isCacheBeanMetadata();

	/**
	 * 设置bean定义中的表达式值的解析器。BeanExpressionResolver
	 * 默认情况下，bean工厂中，是不支持表达式的。应用上下文将会设置一个标准的表达式策略
	 * 解析器，以统一的Spring EL 兼容形式，支持"#{...}"表达式。
	 * @since 3.0
	 */
	void setBeanExpressionResolver(BeanExpressionResolver resolver);

	/**
	 * 获取bean定义中的表达式值的解析器
	 * @since 3.0
	 */
	BeanExpressionResolver getBeanExpressionResolver();

	/**
	 * 设置用于转换bean的属性的转换服务ConversionService。可以作为java bean的属性
	 * 编辑器PropertyEditors的一种替代。
	 * @since 3.0
	 */
	void setConversionService(ConversionService conversionService);

	/**
	 * 获取类型转换服务
	 * @since 3.0
	 */
	ConversionService getConversionService();

	/**
	 * 添加一个属性编辑注册器应用到所有bean的创建过程。
	 * 属性编辑注册器创建一个属性编辑器实例，并注册到给定的注册器中，并尝试刷新每个bean的创建。
	 * 注意需要避免与定制编辑器之间的同步，因此一般情况下，最好使用addPropertyEditorRegistrar方法，
	 * 替代{@link #registerCustomEditor}方法。
	 * @param registrar the PropertyEditorRegistrar to register
	 */
	void addPropertyEditorRegistrar(PropertyEditorRegistrar registrar);

	/**
	 * 注册给定的定制属性编辑器到给定类型的所有属性。在工厂配置的过程中调用。
	 * 注意此方法注册一个共享的定制编辑器实例，可以线程安全地访问编辑器实例。
	 * 最好使用addPropertyEditorRegistrar方法， 替代{@link #registerCustomEditor}方法。
	 * 以避免定制编辑器的同步。
	 * @param requiredType type of the property
	 * @param propertyEditorClass the {@link PropertyEditor} class to register
	 */
	void registerCustomEditor(Class<?> requiredType, Class<? extends PropertyEditor> propertyEditorClass);

	/**
	 * 初始化已经注册到bean工厂的属性编辑注册器与定制编辑器的关系。
	 * @param registry the PropertyEditorRegistry to initialize
	 */
	void copyRegisteredEditorsTo(PropertyEditorRegistry registry);

	/**
	 * 谁知bean工厂用于bean属性值或构造参数值转换的类型转化器。
	 * 此方法将会重写默认属性编辑器机制，因此使任何定制编辑器或编辑注册器不相关。
	 * @see #addPropertyEditorRegistrar
	 * @see #registerCustomEditor
	 * @since 2.5
	 */
	void setTypeConverter(TypeConverter typeConverter);

	/**
	 * 获取bean工厂的类型转换器。有类型转化器是非线程安全的，每次调用，返回的可能是一个新的实例。
	 * 如果默认的属性编辑器机制激活，通过返回的类型转换器，可以了解到所有注册的属性编辑器。
	 * @since 2.5
	 */
	TypeConverter getTypeConverter();

	/**
	 * Add a String resolver for embedded values such as annotation attributes.
	 * @param valueResolver the String resolver to apply to embedded values
	 * @since 3.0
	 */
	void addEmbeddedValueResolver(StringValueResolver valueResolver);

	/**
	 * 判断是否注册嵌入值解决器到bean工厂，可以用于{@link #resolveEmbeddedValue(String)}方法。
	 * @since 4.3
	 */
	boolean hasEmbeddedValueResolver();

	/**
	 * 解决给定的嵌入值，比如注解属性
	 * @param value the value to resolve
	 * @return the resolved value (may be the original value as-is)
	 * @since 3.0
	 */
	String resolveEmbeddedValue(String value);

	/**
	 * 添加一个bean后处理器，将会用于bean工厂创建的bean。在工厂配置的工厂，将会调用。
	 * 需要注意的是，bean后处理器处理的顺序与注册的顺序有关；任何实现{@link org.springframework.core.Ordered}
	 * 接口的bean后处理器的排序语义将会被忽略。在程序中注册一个bean后处理器，将会自动探测上下文中的
	 * bean后处理器。
	 * @param beanPostProcessor the post-processor to register
	 */
	void addBeanPostProcessor(BeanPostProcessor beanPostProcessor);

	/**
	 * 获取当前注册的bean后处理器数量。时
	 */
	int getBeanPostProcessorCount();

	/**
	 * 依赖于给定作用的实现，注册给定的作用域，
	 * @param scopeName the scope identifier
	 * @param scope the backing Scope implementation
	 */
	void registerScope(String scopeName, Scope scope);

	/**
	 * 返回当前注册的作用与的name集。
     * 此方法将会返回显示地注册的作用域。
	 * 单例和原型模式作用域不会暴露。
	 * @return the array of scope names, or an empty array if none
	 * @see #registerScope
	 */
	String[] getRegisteredScopeNames();

	/**
	 * 根据给定作用域的name，返回相应的实现。此方法将会返回显示地注册的作用域。
	 * 单例和原型模式作用域不会暴露。
	 * @param scopeName the name of the scope
	 * @return the registered Scope implementation, or {@code null} if none
	 * @see #registerScope
	 */
	Scope getRegisteredScope(String scopeName);

	/**
	 * 返回当前工厂相关的安全访问控制上下文java.security.AccessControlContext。
	 * @return the applicable AccessControlContext (never {@code null})
	 * @since 3.0
	 */
	AccessControlContext getAccessControlContext();

	/**
	 * 从给定的工厂，拷贝所有相关的配置。
	 * 应包括所有标准的配置，bean后处理器，作用域，工厂内部设置。
	 * 不包括任何实际bean定义的元数据，比如bean定义对象，bean的别名。
	 * @param otherFactory the other BeanFactory to copy from
	 */
	void copyConfigurationFrom(ConfigurableBeanFactory otherFactory);

	/**
	 * 创建给定bean name的别名。这个方法支持bean的name，在XML配置中，属于非法，xml只支持ids别名。
	 * 在工厂配置，经常配置调用，但是也可以使用运行时注册别名。因此，一个工厂的实现，
	 * 应该同步别名访问。
	 * @param beanName the canonical name of the target bean
	 * @param alias the alias to be registered for the bean
	 * @throws BeanDefinitionStoreException if the alias is already in use
	 * 如果别名已经存在，则抛出BeanDefinitionStoreException。
	 */
	void registerAlias(String beanName, String alias) throws BeanDefinitionStoreException;

	/**
	 * 在解决所有目标name的别名和注册到工厂的别名，使用给定的StringValueResolver。
	 * <p>The value resolver may for example resolve placeholders
	 * in target bean names and even in alias names.
	 * 值解决的例子，比如解决给定bean的名称甚至别名中的占位符。
	 * @param valueResolver the StringValueResolver to apply
	 * @since 2.5
	 */
	void resolveAliases(StringValueResolver valueResolver);

	/**
	 * 返回给定bean的名称对应的合并bean定义，如果需要使用它的父工厂合并一个孩子的bean定义。
	 * 考虑祖先bean工厂中的bean定义。
	 * @param beanName the name of the bean to retrieve the merged definition for
	 * @return a (potentially merged) BeanDefinition for the given bean
	 * @throws NoSuchBeanDefinitionException if there is no bean definition with the given name
	 * @since 2.5
	 */
	BeanDefinition getMergedBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

	/**
	 * 检查给定name的bean是否在工厂中。
	 * @param name the name of the bean to check
	 * @return whether the bean is a FactoryBean
	 * ({@code false} means the bean exists but is not a FactoryBean)
	 * @throws NoSuchBeanDefinitionException if there is no bean with the given name
	 * @since 2.5
	 */
	boolean isFactoryBean(String name) throws NoSuchBeanDefinitionException;

	/**
	 * 显示地控制指定bean的创建状态，仅仅容器内部使用。
	 * @param beanName the name of the bean
	 * @param inCreation whether the bean is currently in creation
	 * @since 3.1
	 */
	void setCurrentlyInCreation(String beanName, boolean inCreation);

	/**
	 * 判断当前bean是否创建。
	 * @param beanName the name of the bean
	 * @return whether the bean is currently in creation
	 * @since 2.5
	 */
	boolean isCurrentlyInCreation(String beanName);

	/**
	 * 注册一个依赖bean到给定的bean，在给定bean销毁前销毁依赖的bean。
	 * @param beanName the name of the bean
	 * @param dependentBeanName the name of the dependent bean
	 * @since 2.5
	 */
	void registerDependentBean(String beanName, String dependentBeanName);

	/**
	 * 返回依赖于给定bean的所有bean的name。
	 * @param beanName the name of the bean
	 * @return the array of dependent bean names, or an empty array if none
	 * @since 2.5
	 */
	String[] getDependentBeans(String beanName);

	/**
	 * 返回给定bean的所有依赖bean的name。
	 * @param beanName the name of the bean
	 * @return the array of names of beans which the bean depends on,
	 * or an empty array if none
	 * @since 2.5
	 */
	String[] getDependenciesForBean(String beanName);

	/**
	 * 根据bean的定义，销毁给定bean的实例，通常为一个原型bean实例。
	 * 在析构的过程中，任何异常应该被捕捉，同时log，而不是传给方法的调用者。
	 * @param beanName the name of the bean definition
	 * @param beanInstance the bean instance to destroy
	 */
	void destroyBean(String beanName, Object beanInstance);

	/**
	 * 销毁当前目标作用的指定作用域bean。
	 * 在析构的过程中，任何异常应该被捕捉，同时log，而不是传给方法的调用者。
	 * @param beanName the name of the scoped bean
	 */
	void destroyScopedBean(String beanName);

	/**
	 * 销毁所有工厂中的单例bean，包括内部bean，比如已经被注册为disposable的bean。
	 * 在工厂关闭的时候，将会被调用。
	 * 在析构的过程中，任何异常应该被捕捉，同时log，而不是传给方法的调用者。
	 */
	void destroySingletons();

}

```
从上面可以看出，ConfigurableBeanFactory接口主要提供了，bean作用域，类加载器，临时类加载器，Spring EL解决器，转换服务 *ConversionService*，属性编辑器注册器，嵌入值解决器，bean后处理的配置操作，同时提供了，是否缓存bean的元数据，设置bean的创建状态，判断bean是否为工厂bean，拷贝bean工厂的配置，获取bean的定义，设置bean的别名，解决基于bean的name的依赖，获取bean的依赖bean信息和获取依赖于bean的bean的name操作，还有销毁给定作用域的bean。需要注意的是，设置bean的创建状态操作属于容器的内部操作，获取作用域时，不包括单例和原型作用域。此接口不能够在应用中直接调用，要配合{@link org.springframework.beans.factory.BeanFactory}
  或@link org.springframework.beans.factory.ListableBeanFactory}使用。此接口的扩展接口，运行框架内部使用，用于访问bean工厂的配置方法。



ConfigurableBeanFactory有两个父接口，一个为HierarchicalBeanFactory，一个为SingletonBeanRegistry，HierarchicalBeanFactory前文中已说，我们再来看一下SingletonBeanRegistry接口。关于涉及到的转换服务 *ConversionService*，我们将在后续文章中再说。

### SingletonBeanRegistry
源码参见：[SingletonBeanRegistry][]


[SingletonBeanRegistry]: "SingletonBeanRegistry"

```java

```

HierarchicalBeanFactory, SingletonBeanRegistry

SingletonBeanRegistry




## 总结
ConfigurableListableBeanFactory接口主要提供了，注册给定自动注入值的依赖类型，决定给定name的对应bean，是否可以作为其他bean中声明匹配的自动依赖注入类型的候选，此操作检查祖先工厂。获取给定name的bean的定义，忽略给定类型或接口的依赖自动注入，获取工厂中的bean的name集操作，同时提供了清除不被考虑的bean的元数据缓存，
冻结bean工厂的bean的定义，判断bean工厂的bean定义是否冻结，以及确保所有非懒加载单例bean被初始化，包括工厂bean相关操作。需要注意的是，bean工厂冻结后，注册的bean定义不能在修改，及进一步的后处理；如果确保所有非懒加载单例bean被初始化失败，记得调用{@link #destroySingletons()}方法，清除已经初始化的单例bean。

ConfigurableBeanFactory接口主要提供了，bean作用域，类加载器，临时类加载器，Spring EL解决器，转换服务 *ConversionService*，属性编辑器注册器，嵌入值解决器，bean后处理的配置操作，同时提供了，是否缓存bean的元数据，设置bean的创建状态，判断bean是否为工厂bean，拷贝bean工厂的配置，获取bean的定义，设置bean的别名，解决基于bean的name的依赖，获取bean的依赖bean信息和获取依赖于bean的bean的name操作，还有有销毁给定作用域的bean。需要注意的是，设置bean的创建状态操作属于容器的内部操作，获取作用域时，不包括单例和原型作用域。此接口不能够在应用中直接调用，要配合{@link org.springframework.beans.factory.BeanFactory}
  或@link org.springframework.beans.factory.ListableBeanFactory}使用。此接口的扩展接口，运行框架内部使用，用于访问bean工厂的配置方法。
