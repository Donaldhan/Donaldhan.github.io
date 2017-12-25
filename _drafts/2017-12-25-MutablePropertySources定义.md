---
layout: page
title: MutablePropertySources定义
subtitle: MutablePropertySources定义
date: 2017-12-25 09:41:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言
[ConfigurableConversionService][]接口主要是用于加强 *ConversionService*
暴露的可读操作，为添加和移除转换器Converter提供便利，而没有提供除 *ConversionService* 和 *ConverterRegistry* 之外的操作。

![ConfigurableConversionService](/image/spring-context/ConfigurableConversionService.png)

在[ConfigurableApplicationContext接口定义][]这篇文章中，我们有讲到配置环境接口 *ConfigurableEnvironment* ,在接口中有一个操作 *getPropertySources* 用于获取属性源 *MutablePropertySources*，今天我们就来看一下MutablePropertySources的定义。

[ConfigurableConversionService接口]:https://donaldhan.github.io/spring-framework/2017/12/24/ConfigurableConversionService%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "ConfigurableConversionService接口"

[ConfigurableApplicationContext接口定义]:https://donaldhan.github.io/spring-framework/2017/12/20/ConfigurableApplicationContext%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "ConfigurableApplicationContext接口定义"

# 目录
* [MutablePropertySources定义](#mutablepropertysources定义)
    * [PropertySources](#propertysources)
    * [PropertySource](#propertysource)
* [总结](#总结)

## MutablePropertySources定义
源码参见：[MutablePropertySources][]

[MutablePropertySources]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/env/MutablePropertySources.java "MutablePropertySources"

由于MutablePropertySources实现了PropertySources接口我们先来看一下PropertySources接口的定义。

### PropertySources

源码参见：[PropertySources][]

[PropertySources]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/env/PropertySources.java "PropertySources"

```java
package org.springframework.core.env;

/**
 * PropertySources接口为包含一个或多个属性源PropertySource对象的Holder。
 * @author Chris Beams
 * @since 3.1
 */
public interface PropertySources extends Iterable<PropertySource<?>> {

	/**
	 * 判断是否存在给定name对应的属性源
	 * @param name the {@linkplain PropertySource#getName() name of the property source} to find
	 */
	boolean contains(String name);

	/**
	 * 返回给定name对应的属性源，如果没有返回null。
	 * @param name the {@linkplain PropertySource#getName() name of the property source} to find
	 */
	PropertySource<?> get(String name);

}

```
从上面可以看出，PropertySources（Iterable）是一个包含一个或多个属性源PropertySource对象的Holder，提供了判断是否存在给定name对应的属性源和获取给定name对应的属性源的操作。


再来看[MutablePropertySources][]的定义。


```java
package org.springframework.core.env;

import java.util.Iterator;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

/**
* PropertySources接口的默认实现，允许操作包含的属性源，提供了从一个已经存在的属性源集PropertySources实例的拷贝操作。
* 使用{@link #addFirst}和 {@link #addLast}等方法，保证属性源的优先级，属性源的优先级与添加到属性源集中的顺序有关。
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 * @see PropertySourcesPropertyResolver
 */
public class MutablePropertySources implements PropertySources {

	private final Log logger;

	private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<PropertySource<?>>();


	/**
	 * Create a new {@link MutablePropertySources} object.
	 */
	public MutablePropertySources() {
		this.logger = LogFactory.getLog(getClass());
	}

	/**
	 * Create a new {@code MutablePropertySources} from the given propertySources
	 * object, preserving the original order of contained {@code PropertySource} objects.
	 */
	public MutablePropertySources(PropertySources propertySources) {
		this();
		for (PropertySource<?> propertySource : propertySources) {
			addLast(propertySource);
		}
	}

	/**
	 * Create a new {@link MutablePropertySources} object and inherit the given logger,
	 * usually from an enclosing {@link Environment}.
	 */
	MutablePropertySources(Log logger) {
		this.logger = logger;
	}


	@Override
	public boolean contains(String name) {
		return this.propertySourceList.contains(PropertySource.named(name));
	}

	@Override
	public PropertySource<?> get(String name) {
		int index = this.propertySourceList.indexOf(PropertySource.named(name));
		return (index != -1 ? this.propertySourceList.get(index) : null);
	}

	@Override
	public Iterator<PropertySource<?>> iterator() {
		return this.propertySourceList.iterator();
	}

	/**
	 * Add the given property source object with highest precedence.
	 */
	public void addFirst(PropertySource<?> propertySource) {
		if (logger.isDebugEnabled()) {
			logger.debug("Adding PropertySource '" + propertySource.getName() + "' with highest search precedence");
		}
		removeIfPresent(propertySource);
		this.propertySourceList.add(0, propertySource);
	}

	/**
	 * Add the given property source object with lowest precedence.
	 */
	public void addLast(PropertySource<?> propertySource) {
		if (logger.isDebugEnabled()) {
			logger.debug("Adding PropertySource '" + propertySource.getName() + "' with lowest search precedence");
		}
		removeIfPresent(propertySource);
		this.propertySourceList.add(propertySource);
	}

	/**
	 * Add the given property source object with precedence immediately higher
	 * than the named relative property source.
	 */
	public void addBefore(String relativePropertySourceName, PropertySource<?> propertySource) {
		if (logger.isDebugEnabled()) {
			logger.debug("Adding PropertySource '" + propertySource.getName() +
					"' with search precedence immediately higher than '" + relativePropertySourceName + "'");
		}
		assertLegalRelativeAddition(relativePropertySourceName, propertySource);
		removeIfPresent(propertySource);
		int index = assertPresentAndGetIndex(relativePropertySourceName);
		addAtIndex(index, propertySource);
	}

	/**
	 * Add the given property source object with precedence immediately lower
	 * than the named relative property source.
	 */
	public void addAfter(String relativePropertySourceName, PropertySource<?> propertySource) {
		if (logger.isDebugEnabled()) {
			logger.debug("Adding PropertySource '" + propertySource.getName() +
					"' with search precedence immediately lower than '" + relativePropertySourceName + "'");
		}
		assertLegalRelativeAddition(relativePropertySourceName, propertySource);
		removeIfPresent(propertySource);
		int index = assertPresentAndGetIndex(relativePropertySourceName);
		addAtIndex(index + 1, propertySource);
	}

	/**
	 * Return the precedence of the given property source, {@code -1} if not found.
	 */
	public int precedenceOf(PropertySource<?> propertySource) {
		return this.propertySourceList.indexOf(propertySource);
	}

	/**
	 * Remove and return the property source with the given name, {@code null} if not found.
	 * @param name the name of the property source to find and remove
	 */
	public PropertySource<?> remove(String name) {
		if (logger.isDebugEnabled()) {
			logger.debug("Removing PropertySource '" + name + "'");
		}
		int index = this.propertySourceList.indexOf(PropertySource.named(name));
		return (index != -1 ? this.propertySourceList.remove(index) : null);
	}

	/**
	 * Replace the property source with the given name with the given property source object.
	 * @param name the name of the property source to find and replace
	 * @param propertySource the replacement property source
	 * @throws IllegalArgumentException if no property source with the given name is present
	 * @see #contains
	 */
	public void replace(String name, PropertySource<?> propertySource) {
		if (logger.isDebugEnabled()) {
			logger.debug("Replacing PropertySource '" + name + "' with '" + propertySource.getName() + "'");
		}
		int index = assertPresentAndGetIndex(name);
		this.propertySourceList.set(index, propertySource);
	}

	/**
	 * Return the number of {@link PropertySource} objects contained.
	 */
	public int size() {
		return this.propertySourceList.size();
	}

	@Override
	public String toString() {
		return this.propertySourceList.toString();
	}

	/**
	 * Ensure that the given property source is not being added relative to itself.
	 */
	protected void assertLegalRelativeAddition(String relativePropertySourceName, PropertySource<?> propertySource) {
		String newPropertySourceName = propertySource.getName();
		if (relativePropertySourceName.equals(newPropertySourceName)) {
			throw new IllegalArgumentException(
					"PropertySource named '" + newPropertySourceName + "' cannot be added relative to itself");
		}
	}

	/**
	 * Remove the given property source if it is present.
	 */
	protected void removeIfPresent(PropertySource<?> propertySource) {
		this.propertySourceList.remove(propertySource);
	}

	/**
	 * Add the given property source at a particular index in the list.
	 */
	private void addAtIndex(int index, PropertySource<?> propertySource) {
		removeIfPresent(propertySource);
		this.propertySourceList.add(index, propertySource);
	}

	/**
	 * Assert that the named property source is present and return its index.
	 * @param name {@linkplain PropertySource#getName() name of the property source} to find
	 * @throws IllegalArgumentException if the named property source is not present
	 */
	private int assertPresentAndGetIndex(String name) {
		int index = this.propertySourceList.indexOf(PropertySource.named(name));
		if (index == -1) {
			throw new IllegalArgumentException("PropertySource named '" + name + "' does not exist");
		}
		return index;
	}

}

```
从上面可以看出，MutablePropertySources为属性源Holder PropertySource的具体实现，内部通过一个属性源集合（CopyOnWriteArrayList）来管理内部的属性源，
主要提供添加、移除、替换、是否包含属性源操作，这些操作实际上通过 *CopyOnWriteArrayList* 的相应操作完成。

### PropertySource

源码参见：[PropertySource][]

[PropertySource]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/env/PropertySource.java "PropertySource"

```java
```



## 总结

PropertySources（Iterable）是一个包含一个或多个属性源PropertySource对象的Holder，提供了判断是否存在给定name对应的属性源和获取给定name对应的属性源的操作。

MutablePropertySources为属性源Holder PropertySource的具体实现，内部通过一个属性源集合（CopyOnWriteArrayList）来管理内部的属性源，
主要提供添加、移除、替换、是否包含属性源操作，这些操作实际上通过 *CopyOnWriteArrayList* 的相应操作完成。
