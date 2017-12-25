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
[ConfigurableConversionService接口][]主要是用于加强 *ConversionService*
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
package org.springframework.core.env;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import org.springframework.util.Assert;
import org.springframework.util.ObjectUtils;

/**
 * Abstract base class representing a source of name/value property pairs. The underlying
 * {@linkplain #getSource() source object} may be of any type {@code T} that encapsulates
 * properties. Examples include {@link java.util.Properties} objects, {@link java.util.Map}
 * objects, {@code ServletContext} and {@code ServletConfig} objects (for access to init
 * parameters). Explore the {@code PropertySource} type hierarchy to see provided
 * implementations.
 *抽象基础类属性源PropertySource是一个表示属性值对的源。通过 {@linkplain #getSource() source object}获取的属性源，
 *可以是任何底层封装了属性的属性源。比如{@link java.util.Properties}，{@link java.util.Map}，{@code ServletContext}，
 *{@code ServletConfig}（访问初始化参数）对象。更多，查看属性源的实现。
 * <p>{@code PropertySource} objects are not typically used in isolation, but rather
 * through a {@link PropertySources} object, which aggregates property sources and in
 * conjunction with a {@link PropertyResolver} implementation that can perform
 * precedence-based searches across the set of {@code PropertySources}.
 *属性对象PropertySource，一般不单独使用，而在通过PropertySources，属性源集PropertySources，可以聚合属性，
 *同时使用PropertyResolver实现，执行跨属性源的基于优先级的搜索。
 * <p>{@code PropertySource} identity is determined not based on the content of
 * encapsulated properties, but rather based on the {@link #getName() name} of the
 * {@code PropertySource} alone. This is useful for manipulating {@code PropertySource}
 * objects when in collection contexts. See operations in {@link MutablePropertySources}
 * as well as the {@link #named(String)} and {@link #toString()} methods for details.
 *PropertySource的特质，决定了不能基于内容封装属性，但是可以基于name{@link #getName() name}。
 *在集合上下文中，这种方式非常有用。具体操作参见{@link MutablePropertySources}，及{@link #named(String)} and {@link #toString()}
 *方法。
 * <p>Note that when working with @{@link
 * org.springframework.context.annotation.Configuration Configuration} classes that
 * the @{@link org.springframework.context.annotation.PropertySource PropertySource}
 * annotation provides a convenient and declarative way of adding property sources to the
 * enclosing {@code Environment}.
 *注意：当使用注解配置@{@linkorg.springframework.context.annotation.Configuration Configuration}时，
 * @{@link org.springframework.context.annotation.PropertySource PropertySource}注解为
 * 声明和添加属性源到环境配置Environment，提供了便利。
 *
 * @author Chris Beams
 * @since 3.1
 * @see PropertySources
 * @see PropertyResolver
 * @see PropertySourcesPropertyResolver
 * @see MutablePropertySources
 * @see org.springframework.context.annotation.PropertySource
 */
public abstract class PropertySource<T> {

	protected final Log logger = LogFactory.getLog(getClass());

	protected final String name;

	protected final T source;


	/**
	 * Create a new {@code PropertySource} with the given name and source object.
	 */
	public PropertySource(String name, T source) {
		Assert.hasText(name, "Property source name must contain at least one character");
		Assert.notNull(source, "Property source must not be null");
		this.name = name;
		this.source = source;
	}

	/**
	 * Create a new {@code PropertySource} with the given name and with a new
	 * {@code Object} instance as the underlying source.
	 * <p>Often useful in testing scenarios when creating anonymous implementations
	 * that never query an actual source but rather return hard-coded values.
	 */
	@SuppressWarnings("unchecked")
	public PropertySource(String name) {
		this(name, (T) new Object());
	}


	/**
	 * Return the name of this {@code PropertySource}
	 * 获取属性源的name
	 */
	public String getName() {
		return this.name;
	}

	/**
	 * Return the underlying source object for this {@code PropertySource}.
	 * 返回底层属性源对象
	 */
	public T getSource() {
		return this.source;
	}

	/**
	 * Return whether this {@code PropertySource} contains the given name.
	 * <p>This implementation simply checks for a {@code null} return value
	 * from {@link #getProperty(String)}. Subclasses may wish to implement
	 * a more efficient algorithm if possible.
	 * 判断当前属性源是否包含给定name对应的属性。当前实现根据检查{@link #getProperty(String)}方法获取的属性，
	 * 是否为空。子类可以重写这个方法。
	 *
	 * @param name the property name to find
	 */
	public boolean containsProperty(String name) {
		return (getProperty(name) != null);
	}

	/**
	 * Return the value associated with the given name,
	 * or {@code null} if not found.
	 * 返回当前属性源是否包含给定name对应的属性，没有返回null
	 * @param name the property to find
	 * @see PropertyResolver#getRequiredProperty(String)
	 */
	public abstract Object getProperty(String name);


	/**
	 * This {@code PropertySource} object is equal to the given object if:
	 * <ul>
	 * <li>they are the same instance
	 * <li>the {@code name} properties for both objects are equal
	 * </ul>
	 * <p>No properties other than {@code name} are evaluated.
	 */
	@Override
	public boolean equals(Object obj) {
		return (this == obj || (obj instanceof PropertySource &&
				ObjectUtils.nullSafeEquals(this.name, ((PropertySource<?>) obj).name)));
	}

	/**
	 * Return a hash code derived from the {@code name} property
	 * of this {@code PropertySource} object.
	 */
	@Override
	public int hashCode() {
		return ObjectUtils.nullSafeHashCode(this.name);
	}

	/**
	 * Produce concise output (type and name) if the current log level does not include
	 * debug. If debug is enabled, produce verbose output including the hash code of the
	 * PropertySource instance and every name/value property pair.
	 * <p>This variable verbosity is useful as a property source such as system properties
	 * or environment variables may contain an arbitrary number of property pairs,
	 * potentially leading to difficult to read exception and log messages.
	 * @see Log#isDebugEnabled()
	 */
	@Override
	public String toString() {
		if (logger.isDebugEnabled()) {
			return getClass().getSimpleName() + "@" + System.identityHashCode(this) +
					" {name='" + this.name + "', properties=" + this.source + "}";
		}
		else {
			return getClass().getSimpleName() + " {name='" + this.name + "'}";
		}
	}


	/**
	 * Return a {@code PropertySource} implementation intended for collection comparison purposes only.
	 * 返回一个用于集合比较的属性源实现ComparisonPropertySource。
	 * <p>Primarily for internal use, but given a collection of {@code PropertySource} objects, may be
	 * used as follows:
	 * 主要是内部使用，但是给定的属性源集合对象，可能为如下：
	 * <pre class="code">
	 * {@code List<PropertySource<?>> sources = new ArrayList<PropertySource<?>>();
	 * sources.add(new MapPropertySource("sourceA", mapA));
	 * sources.add(new MapPropertySource("sourceB", mapB));
	 * assert sources.contains(PropertySource.named("sourceA"));
	 * assert sources.contains(PropertySource.named("sourceB"));
	 * assert !sources.contains(PropertySource.named("sourceC"));
	 * }</pre>
	 * The returned {@code PropertySource} will throw {@code UnsupportedOperationException}
	 * if any methods other than {@code equals(Object)}, {@code hashCode()}, and {@code toString()}
	 * are called.
	 * 如果调用{@code equals(Object)}, {@code hashCode()}, and {@code toString()之外的任何方法，将会抛出
	 * UnsupportedOperationException异常
	 * @param name the name of the comparison {@code PropertySource} to be created and returned.
	 */
	public static PropertySource<?> named(String name) {
		return new ComparisonPropertySource(name);
	}


	/**
	 * {@code PropertySource} to be used as a placeholder in cases where an actual
	 * property source cannot be eagerly initialized at application context
	 * creation time.  For example, a {@code ServletContext}-based property source
	 * must wait until the {@code ServletContext} object is available to its enclosing
	 * {@code ApplicationContext}.  In such cases, a stub should be used to hold the
	 * intended default position/order of the property source, then be replaced
	 * during context refresh.
	 * 在应用上下文创建时，属性源不能够初始化的情况下，StubPropertySource属性源可以作为占位符使用。
	 * 比如基于{@code ServletContext}的属性源必须等待应用上下文内部封装的{@code ServletContext}对象可用。
	 * 在这种情况下，存根属性源StubPropertySource，用于保持属性的位置顺序，当上下文刷新时，再替换。
	 * @see org.springframework.context.support.AbstractApplicationContext#initPropertySources()
	 * @see org.springframework.web.context.support.StandardServletEnvironment
	 * @see org.springframework.web.context.support.ServletContextPropertySource
	 */
	public static class StubPropertySource extends PropertySource<Object> {

		public StubPropertySource(String name) {
			super(name, new Object());
		}

		/**
		 * Always returns {@code null}.
		 */
		@Override
		public String getProperty(String name) {
			return null;
		}
	}


	/**
	 * @see PropertySource#named(String)
	 */
	static class ComparisonPropertySource extends StubPropertySource {

		private static final String USAGE_ERROR =
				"ComparisonPropertySource instances are for use with collection comparison only";

		public ComparisonPropertySource(String name) {
			super(name);
		}

		@Override
		public Object getSource() {
			throw new UnsupportedOperationException(USAGE_ERROR);
		}

		@Override
		public boolean containsProperty(String name) {
			throw new UnsupportedOperationException(USAGE_ERROR);
		}

		@Override
		public String getProperty(String name) {
			throw new UnsupportedOperationException(USAGE_ERROR);
		}
	}

}

```

从上面可以看出，属性抽象类PropertySource，可以理解底层属性源source的封装类，提供了获取属性值，及判断属性值是否存在的操作，同时提供了包装属性源为不可读属性源ComparisonPropertySource操作。PropertySource属性不能单独使用，要配合属性源集PropertySources（Iterable）使用。

最后以MutablePropertySources的类图结束这篇文章。

![MutablePropertySources](/image/spring-context/MutablePropertySources.png)

## 总结

PropertySources（Iterable）是一个包含一个或多个属性源PropertySource对象的Holder，提供了判断是否存在给定name对应的属性源和获取给定name对应的属性源的操作。

MutablePropertySources为属性源Holder PropertySource的具体实现，内部通过一个属性源集合（CopyOnWriteArrayList）来管理内部的属性源，
主要提供添加、移除、替换、是否包含属性源操作，这些操作实际上通过 *CopyOnWriteArrayList* 的相应操作完成。

属性抽象类PropertySource，可以理解底层属性源source的封装类，提供了获取属性值，及判断属性值是否存在的操作，同时提供了包装属性源为不可读属性源ComparisonPropertySource操作。PropertySource属性不能单独使用，要配合属性源集PropertySources（Iterable）使用。
