---
layout: page
title: ConfigurableConversionService接口定义
subtitle: ConfigurableConversionService接口及父类接口定义
date: 2017-12-24 20:29:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---
# 引言
先来回顾一下上一篇文章[ConfigurableListableBeanFactory接口定义][]，ConfigurableListableBeanFactory接口主要提供了，注册给定自动注入值的依赖类型，决定给定name的对应bean，是否可以作为其他bean中声明匹配的自动依赖注入类型的候选，此操作检查祖先工厂。获取给定name的bean的定义，忽略给定类型或接口的依赖自动注入，获取工厂中的bean的name集操作，同时提供了清除不被考虑的bean的元数据缓存，冻结bean工厂的bean的定义，判断bean工厂的bean定义是否冻结，以及确保所有非懒加载单例bean被初始化，包括工厂bean相关操作。需要注意的是，bean工厂冻结后，注册的bean定义不能在修改，及进一步的后处理；如果确保所有非懒加载单例bean被初始化失败，记得调用{@link #destroySingletons()}方法，清除已经初始化的单例bean。

![ConfigurableListableBeanFactory](/image/spring-context/ConfigurableListableBeanFactory.png)

在[ConfigurableApplicationContext接口定义][]这篇文章中，我们有讲到配置环境接口 *ConfigurableEnvironment* 接口定义的时候，配置环境接口的父类接口 *ConfigurablePropertyResolver* 有一个设置配置转换服务 *ConfigurableConversionService* 的方法，今天我们就来看一下ConfigurableConversionService接口的作用。
[ConfigurableListableBeanFactory接口定义]:https://donaldhan.github.io/spring-framework/2017/12/22/ConfigurableListableBeanFactory%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "ConfigurableListableBeanFactory接口定义"

[ConfigurableApplicationContext接口定义]:https://donaldhan.github.io/spring-framework/2017/12/20/ConfigurableApplicationContext%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "ConfigurableApplicationContext接口定义"

# 目录
* [ConfigurableConversionService接口定义](#configurableconversionservice接口定义)
    * [ConversionService](#conversionservice)
    * [ConverterRegistry](#converterregistry)
    * [ConverterFactory](#converterfactory)
    * [Converter](#converter)
    * [GenericConverter](#genericconverter)
* [总结](#总结)

## ConfigurableConversionService接口定义

源码参见：[ConfigurableConversionService][]

[ConfigurableConversionService]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/convert/support/ConfigurableConversionService.java "ConfigurableConversionService"

```java
package org.springframework.core.convert.support;

import org.springframework.core.convert.ConversionService;
import org.springframework.core.convert.converter.ConverterRegistry;

/**
*ConfigurableConversionService是大多数转换服务ConversionService实现的接口。用于加强ConversionService
*暴露的可读操作，为添加和移除{@link org.springframework.core.convert.converter.Converter
* Converters}提供便利。在应用上下文的启动代码片段中，ConfigurableConversionService对于添加和移除
* 转换器特别有用。
 * @author Chris Beams
 * @since 3.1
 * @see org.springframework.core.env.ConfigurablePropertyResolver#getConversionService()
 * @see org.springframework.core.env.ConfigurableEnvironment
 * @see org.springframework.context.ConfigurableApplicationContext#getEnvironment()
 */
public interface ConfigurableConversionService extends ConversionService, ConverterRegistry {

}
```
从上面可以看出，ConfigurableConversionService接口主要是用于加强 *ConversionService*
暴露的可读操作，为添加和移除转换器Converter提供便利，而没有提供除 *ConversionService* 和 *ConverterRegistry* 之外的操作。

再来看ConversionService接口的定义。
### ConversionService
源码参见：[ConversionService][]
[ConversionService]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/convert/ConversionService.java "ConversionService"

```java
package org.springframework.core.convert;

/**
 *ConversionService为类型转换接口。同时为转换系统的入口点。调用此系统可以执行一个线程安全的类型转换。
 * @author Keith Donald
 * @author Phillip Webb
 * @since 3.0
 */
public interface ConversionService {

	/**
	 * 如果源类型可以转换为目标类型，则返回true。如果方法返回true表示， {@link #convert(Object, Class)}方法
	 * 可以将源类型额一个实例转换为目标类型。
	 * 需要注意集合、数组、Map这些类型：
	 * 对于集合、数组、Map这些类型之间的转化，即使底层元素不可转换，甚至转换时会抛出异常，此方法也可能返回true。
	 * 当出现异常时，调用者可以处理这些异常情况。
	 * @param sourceType the source type to convert from (may be {@code null} if source is {@code null})
	 * @param targetType the target type to convert to (required)
	 * @return {@code true} if a conversion can be performed, {@code false} if not
	 * @throws IllegalArgumentException if {@code targetType} is {@code null}
	 */
	boolean canConvert(Class<?> sourceType, Class<?> targetType);

	/**
	 * 如果源类型描述对象可以转换为目标描述对象，则返回true。类型描述TypeDescriptor提供了源类型和目标类型的
	 * 额外的上下文信息，比如对象的fields，属性位置等。
	 * is capable of converting an instance of {@code sourceType} to {@code targetType}.
	 * 如果方法返回true表示， {@link #convert(Object, TypeDescriptor, TypeDescriptor)}方法
	 * 可以将源类型额一个实例转换为目标类型。
	 * 需要注意集合、数组、Map这些类型：
	 * 对于集合、数组、Map这些类型之间的转化，即使底层元素不可转换，甚至转换时会抛出异常，此方法也可能返回true。
	 * 当出现异常时，调用者可以处理这些异常情况。
	 * @param sourceType context about the source type to convert from
	 * (may be {@code null} if source is {@code null})
	 * @param targetType context about the target type to convert to (required)
	 * @return {@code true} if a conversion can be performed between the source and target types,
	 * {@code false} if not
	 * @throws IllegalArgumentException if {@code targetType} is {@code null}
	 */
	boolean canConvert(TypeDescriptor sourceType, TypeDescriptor targetType);

	/**
	 * 转换给定源对象为目标类型。
	 * @param source the source object to convert (may be {@code null})
	 * @param targetType the target type to convert to (required)
	 * @return the converted object, an instance of targetType
	 * @throws ConversionException if a conversion exception occurred
	 * @throws IllegalArgumentException if targetType is {@code null}
	 */
	<T> T convert(Object source, Class<T> targetType);

	/**
	 * 根据给定的源对象，及源对象类型描述，将源对象转换为目标类型。
	 * 类型描述TypeDescriptor提供了源类型和目标类型的
	 * 额外的上下文信息，比如对象的fields，属性位置等。
	 * @param source the source object to convert (may be {@code null})
	 * @param sourceType context about the source type to convert from
	 * (may be {@code null} if source is {@code null})
	 * @param targetType context about the target type to convert to (required)
	 * @return the converted object, an instance of {@link TypeDescriptor#getObjectType() targetType}
	 * @throws ConversionException if a conversion exception occurred
	 * @throws IllegalArgumentException if targetType is {@code null},
	 * or {@code sourceType} is {@code null} but source is not {@code null}
	 */
	Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);

}

```
从上面可以看出，ConversionService接口提供了判断两种类型或类型描述 *TypeDescriptor* 是否可以转换的操作和将源对象转换为目标类型的操作。需要注意的是，判断是否可以转换操作返回true，并不表示可以转换成功，比如集合、数组、Map这些类型：对于集合、数组、Map这些类型之间的转化，即使底层元素不可转换，甚至转换时会抛出异常，判断方法也可能返回true。
当出现异常时，调用者可以处理这些异常情况。

再来看一下转换注册器ConverterRegistry：

### ConverterRegistry
源码参见：[ConverterRegistry][]

[ConverterRegistry]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/convert/converter/ConverterRegistry.java "ConverterRegistry"

```java
package org.springframework.core.convert.converter;

/**
 *注册转换器到类型转换系统。
 * @author Keith Donald
 * @author Juergen Hoeller
 * @since 3.0
 */
public interface ConverterRegistry {

	/**
	 * 添加一个空白的转换器到注册器。可转换源、目标类型对从转换器的参数化类型中提取。
	 * @throws IllegalArgumentException if the parameterized types could not be resolved
	 */
	void addConverter(Converter<?, ?> converter);

	/**
	 * 添加一个空白的转换器到注册器。源、目标类型对显示地添加。在不用创建转换对的情况下，可以重用转换器。
	 * @since 3.1
	 */
	<S, T> void addConverter(Class<S> sourceType, Class<T> targetType, Converter<? super S, ? extends T> converter);

	/**
	 * 添加GenericConverter到注册器
	 */
	void addConverter(GenericConverter converter);

	/**
	 * 添加转换器工厂到注册器
	 * @throws IllegalArgumentException if the parameterized types could not be resolved
	 */
	void addConverterFactory(ConverterFactory<?, ?> factory);

	/**
	 * 移除所有从源类型到目的类型的转换器。
	 * @param sourceType the source type
	 * @param targetType the target type
	 */
	void removeConvertible(Class<?> sourceType, Class<?> targetType);

}

```
从上面可以看出，ConverterRegistry接口主要提供了，添加转换器 *Converter，GenericConverter* ，添加转换器工厂 *ConverterFactory* 以及移除转换器操作。

下面我们分别来看Converter，GenericConverter，ConverterFactory，先来看一下ConverterFactory。

### ConverterFactory
源码参见：[ConverterFactory][]

[ConverterFactory]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/convert/converter/ConverterFactory.java "ConverterFactory"

```java
package org.springframework.core.convert.converter;

/**
 *ConverterFactory接口提供了源类型S到目标类型R的子类的转换器。
 *具体实现可以参见{@link ConditionalConverter}.
 * @author Keith Donald
 * @since 3.0
 * @see ConditionalConverter
 * @param <S> the source type converters created by this factory can convert from
 * @param <R> the target range (or base) type converters created by this factory can convert to;
 * for example {@link Number} for a set of number subtypes.
 */
public interface ConverterFactory<S, R> {

	/**
	 * 获取从源类型S到目标类型T的转换器，T同时是一个R的实例
	 * @param <T> the target type
	 * @param targetType the target type to convert to
	 * @return a converter from S to T
	 */
	<T extends R> Converter<S, T> getConverter(Class<T> targetType);

}
```
从上面可以看出，ConverterFactory接口主要提供了，获取源类型S到目标类型R的子类的转换器操作。

在来看Converter接口的定义。

### Converter
源码参见：[Converter][]

[Converter]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/convert/converter/Converter.java "Converter"

```java
package org.springframework.core.convert.converter;

/**
 *Converter接口用于转换源类型S对象，到目标类型T。
 *接口的实现必须是线程安全且可以共享。
 *具体实现可以同时实现{@link ConditionalConverter}接口.
 * @author Keith Donald
 * @since 3.0
 * @param <S> the source type
 * @param <T> the target type
 */
public interface Converter<S, T> {

	/**
	 * 转换源类型S对象，到目标类型T
	 * @param source the source object to convert, which must be an instance of {@code S} (never {@code null})
	 * @return the converted object, which must be an instance of {@code T} (potentially {@code null})
	 * @throws IllegalArgumentException if the source cannot be converted to the desired target type
	 */
	T convert(S source);

}
```
从上面可以看出，Converter接口主要提供了，转换源类型S对象，到目标类型T的操作，具体的实现必须是线程安全且可以共享。

### GenericConverter
源码参见：[GenericConverter][]

[GenericConverter]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/convert/converter/GenericConverter.java "GenericConverter"

```java
package org.springframework.core.convert.converter;

import java.util.Set;

import org.springframework.core.convert.TypeDescriptor;
import org.springframework.util.Assert;

/**
 *GenericConverter是一个两种或多种类型进行转换的转换器接口。
 * GenericConverter是一个灵活的系统转换器接口，但是也有复杂的。灵活的GenericConverter，可以支持多种多种源和目标类型之间的转换，
 * 可以通过{@link #getConvertibleTypes()获取，另外在类型转换的过程中，具体的实现可以访问源目标类型描述的field上下文。
 * 这个可以允许解决源和目标的field元数据，比如注解和泛型信息，并可以用于转换逻辑。
 *当{@link Converter}和{@link ConverterFactory}有效时，此接口一般应该不会用到。
 *具体实现可以同时实现{@link ConditionalConverter}接口.
 * @author Keith Donald
 * @author Juergen Hoeller
 * @since 3.0
 * @see TypeDescriptor
 * @see Converter
 * @see ConverterFactory
 * @see ConditionalConverter
 */
public interface GenericConverter {

	/**
	 * 返回转换器可以转换的类型转换对。每个转换对是一个源类型到目标类型转换pair。
	 * 对于条件转换器ConditionalConverter，此方法也许返回null，表示所有的源和目标类型之间的转换将会被考虑。
	 */
	Set<ConvertiblePair> getConvertibleTypes();

	/**
	 * 根据源对象、类型描述及目标类型描述，转换对象为目标类型
	 * @param source the source object to convert (may be {@code null})
	 * @param sourceType the type descriptor of the field we are converting from
	 * @param targetType the type descriptor of the field we are converting to
	 * @return the converted object
	 */
	Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);


	/**
	 * 源码目标类型转换对。
	 */
	final class ConvertiblePair {

		private final Class<?> sourceType;

		private final Class<?> targetType;

		/**
		 * Create a new source-to-target pair.
		 * @param sourceType the source type
		 * @param targetType the target type
		 */
		public ConvertiblePair(Class<?> sourceType, Class<?> targetType) {
			Assert.notNull(sourceType, "Source type must not be null");
			Assert.notNull(targetType, "Target type must not be null");
			this.sourceType = sourceType;
			this.targetType = targetType;
		}

		public Class<?> getSourceType() {
			return this.sourceType;
		}

		public Class<?> getTargetType() {
			return this.targetType;
		}

		@Override
		public boolean equals(Object other) {
			if (this == other) {
				return true;
			}
			if (other == null || other.getClass() != ConvertiblePair.class) {
				return false;
			}
			ConvertiblePair otherPair = (ConvertiblePair) other;
			return (this.sourceType == otherPair.sourceType && this.targetType == otherPair.targetType);
		}

		@Override
		public int hashCode() {
			return (this.sourceType.hashCode() * 31 + this.targetType.hashCode());
		}

		@Override
		public String toString() {
			return (this.sourceType.getName() + " -> " + this.targetType.getName());
		}
	}

}

```
从上面可以看出，GenericConverter接口提供了类型转换的操作，可以支持多种类型之间的转换。同时提供了类型之间转化的关系句柄ConvertiblePair。另外在类型转换的过程中，具体的实现可以访问源目标类型描述的field上下文元数据，比如注解和泛型信息。


## 总结
ConfigurableConversionService接口主要是用于加强 *ConversionService*
暴露的可读操作，为添加和移除转换器Converter提供便利，而没有提供除 *ConversionService* 和 *ConverterRegistry* 之外的操作。

ConversionService接口提供了判断两种类型或类型描述 *TypeDescriptor* 是否可以转换的操作和将源对象转换为目标类型的操作。需要注意的是，判断是否可以转换操作返回true，并不表示可以转换成功，比如集合、数组、Map这些类型：对于集合、数组、Map这些类型之间的转化，即使底层元素不可转换，甚至转换时会抛出异常，判断方法也可能返回true。
当出现异常时，调用者可以处理这些异常情况。

ConverterRegistry接口主要提供了，添加转换器 *Converter，GenericConverter* ，添加转换器工厂 *ConverterFactory* 以及移除转换器操作。

ConverterFactory接口主要提供了，获取源类型S到目标类型R的子类的转换器操作。

Converter接口主要提供了，转换源类型S对象，到目标类型T的操作，具体的实现必须是线程安全且可以共享。

从上面可以看出，GenericConverter接口提供了类型转换的操作，可以支持多种类型之间的转换。同时提供了类型之间转化的关系句柄ConvertiblePair。另外在类型转换的过程中，具体的实现可以访问源目标类型描述的field上下文元数据，比如注解和泛型信息。
