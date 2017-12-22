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

[ConfigurableBeanFactory]: "ConfigurableBeanFactory"

```java

```




### ConfigurableBeanFactory
源码参见：[ConfigurableBeanFactory][]

[ConfigurableBeanFactory]: "ConfigurableBeanFactory"

```java

```

HierarchicalBeanFactory, SingletonBeanRegistry

SingletonBeanRegistry




## 总结
ConfigurableListableBeanFactory接口主要提供了，注册给定自动注入值的依赖类型，决定给定name的对应bean，是否可以作为其他bean中声明匹配的自动依赖注入类型的候选，此操作检查祖先工厂。获取给定name的bean的定义，忽略给定类型或接口的依赖自动注入，获取工厂中的bean的name集操作，同时提供了清除不被考虑的bean的元数据缓存，
冻结bean工厂的bean的定义，判断bean工厂的bean定义是否冻结，以及确保所有非懒加载单例bean被初始化，包括工厂bean相关操作。需要注意的是，bean工厂冻结后，注册的bean定义不能在修改，及进一步的后处理；如果确保所有非懒加载单例bean被初始化失败，记得调用{@link #destroySingletons()}方法，清除已经初始化的单例bean。
