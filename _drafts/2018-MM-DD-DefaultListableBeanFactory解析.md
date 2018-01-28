---
layout: page
title: DefaultListableBeanFactory解析
subtitle: DefaultListableBeanFactory解析
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

[BeanDefinition接口][]用于描述一个bean实例的属性及构造参数等元数据；主要提供了父beanname，bean类型名，作用域，懒加载，
bean依赖，自动注入候选bean，自动注入候选主要bean熟悉的设置与获取操作。同时提供了判断bean是否为单例、原型模式、抽象bean的操作，及获取bean的描述，资源描述，属性源，构造参数，原始bean定义等操作。

![BeanDefinition](/image/spring-context/BeanDefinition.png)

[BeanDefinition接口]:https://donaldhan.github.io/spring-framework/2017/12/26/BeanDefinition%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "BeanDefinition接口"

上一篇文章我们看了，BeanDefinition接口的定义，截止到上一篇文章我们将应用上下文和可配置应用上下文已看完，从这篇文章开始，我们将进入应用上下文的实现。


# 目录
* [DefaultListableBeanFactory定义](defaultlistablebeanfactory定义)
    * [AliasRegistry](#aliasregistry)
    * [SimpleAliasRegistry](#simplealiasregistry)
    * [SingletonBeanRegistry](#singletonbeanregistry)
    * [DefaultSingletonBeanRegistry](#defaultsingletonbeanregistry)
    * [FactoryBeanRegistrySupport](#factorybeanregistrysupport)
    * [AbstractBeanFactory](#abstractbeanfactory)
    * [AbstractAutowireCapableBeanFactory](#abstractautowirecapablebeanfactory)
* [总结](#总结)

## DefaultListableBeanFactory定义
源码参见：[DefaultListableBeanFactory][]

[DefaultListableBeanFactory]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultListableBeanFactory.java "DefaultListableBeanFactory"

### AliasRegistry
源码参见：[AliasRegistry][]

[AliasRegistry]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/AliasRegistry.java ""

```java
package org.springframework.core;

public interface AliasRegistry {
	void registerAlias(String name, String alias);
	void removeAlias(String alias);
	boolean isAlias(String name);
	String[] getAliases(String name);
}
```
从上面可以看出，别名注册器AliasRegistry接口，主要提供了bean别名的注册，移除操作，及判断别名是否存在和获取bean的别名。

再来看一个别名注册器的简单实现SimpleAliasRegistry。

### SimpleAliasRegistry
源码参见：[SimpleAliasRegistry][]

[SimpleAliasRegistry]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/SimpleAliasRegistry.java "SimpleAliasRegistry"

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import org.springframework.util.Assert;
import org.springframework.util.StringUtils;
import org.springframework.util.StringValueResolver;
public class SimpleAliasRegistry implements AliasRegistry {

	/** Map from alias to canonical name */
	private final Map<String, String> aliasMap = new ConcurrentHashMap<String, String>(16);
	@Override
	public void registerAlias(String name, String alias) {
		Assert.hasText(name, "'name' must not be empty");
		Assert.hasText(alias, "'alias' must not be empty");
		if (alias.equals(name)) {
			this.aliasMap.remove(alias);
		}
		else {
			String registeredName = this.aliasMap.get(alias);
			if (registeredName != null) {
				if (registeredName.equals(name)) {
					// An existing alias - no need to re-register
					//bean的别名已经存在，不需要注册
					return;
				}
				if (!allowAliasOverriding()) {
					throw new IllegalStateException("Cannot register alias '" + alias + "' for name '" +
							name + "': It is already registered for name '" + registeredName + "'.");
				}
			}
			checkForAliasCircle(name, alias);//检查别名是否存在循环引用
			this.aliasMap.put(alias, name);//注册bean别名
		}
	}
    /**
	 * Check whether the given name points back to the given alias as an alias
	 * in the other direction already, catching a circular reference upfront
	 * and throwing a corresponding IllegalStateException.
	 * 检查给定的name是否存在指向给定别名的循环引用
	 * @param name the candidate name
	 * @param alias the candidate alias
	 * @see #registerAlias
	 * @see #hasAlias
	 */
	protected void checkForAliasCircle(String name, String alias) {
		if (hasAlias(alias, name)) {
			throw new IllegalStateException("Cannot register alias '" + alias +
					"' for name '" + name + "': Circular reference - '" +
					name + "' is a direct or indirect alias for '" + alias + "' already");
		}
	}
    ...
}
```
从上面可以看出，别名注册器的简单实现SimpleAliasRegistry，主要通过ConcurrentHashMap<String, String>来管理bean的别名，
key为bean的别名alias，value值为bean的name。

### SingletonBeanRegistry
源码参见：[SingletonBeanRegistry][]

[SingletonBeanRegistry]: "SingletonBeanRegistry"

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

别名注册器AliasRegistry接口，主要提供了bean别名的注册，移除操作，及判断别名是否存在和获取bean的别名。
别名注册器的简单实现SimpleAliasRegistry，主要通过ConcurrentHashMap<String, String>来管理bean的别名，
key为bean的别名alias，value值为bean的name。
