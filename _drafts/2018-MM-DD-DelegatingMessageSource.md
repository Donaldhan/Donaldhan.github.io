---
layout: page
title: my blog
subtitle: sub title
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

AbstractEnvironment主要的成员变量为激活配置集activeProfiles（LinkedHashSet<String>）,默认配置解defaultProfiles（ LinkedHashSet<String>）
，属性源管理器propertySources（MutablePropertySources），还有一属性源解决器propertyResolver（[PropertySourcesPropertyResolver][]）。
激活配置与默认配置的相关操作实际为相关配置集集合操作。整合环境操作，主要是整合属性源，激活配置与默认配置。
ConfigurablePropertyResolver和PropertyResolver接口的实现实际委托个内部的属性源解决器propertyResolver。
StandardEnvironment的默认属性源集有系统属性源和环境变量属性源。

AbstractPropertyResolver抽象属性解决器主要，主要是从属性源中加载相关的属性，替代给定文中的占位符。

抽象属性解决器成员为转换器服务conversionService（ConfigurableConversionService），默认为[DefaultConversionService][]。
抽象属性解决器主要所做的工作为从属性源中加载相关的属性，替代给定文中的占位符。

PropertySourcesPropertyResolver内部有一个属性源集propertySources（PropertySources），为在AbstractEnvironment的
变量属性源解决器propertyResolver（[PropertySourcesPropertyResolver][]）声明中，定义的实际为MutablePropertySources，即环境的属性源。
在获取属性的过程中，如果需要类型转换，则委托给内部转换器服务，默认为[DefaultConversionService][]。

GenericConversionService主要使用Converters来管理类型转换器，Converters的内部主要有两个集合来存放转换器，
一个是条件转换器集globalConverters（ LinkedHashSet<GenericConverter>），另一位为源类型与目标类型对ConvertiblePair的转换器ConvertersForPair映射集
converters（LinkedHashMap<ConvertiblePair, ConvertersForPair>）。为了快速地找到类型转化器，GenericConversionService内部使用一个转换器缓存
converterCache（ConcurrentReferenceHashMap<ConverterCacheKey, GenericConverter>），来存方已知的转换器，当转化器添加或移除时，需要清除缓存。

DefaultConversionService内部有一个懒加载的共享实例，DefaultConversionService内部添加大多数环境需要使用的
转化器，如原始类型，及集合类转换器。

![StandardEnvironment](/image/spring-context/StandardEnvironment.png)

上述为我们上一篇文章[StandardEnvironment源码解析][]所讲的内容。


[StandardEnvironment源码解析]:https://donaldhan.github.io/spring-framework/2018/01/10/StandardEnvironment%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.html "StandardEnvironment源码解析"

截止到上一篇文章我们将抽象应用上下文的标准环境配置看完，今天我们来看另外一个模块消息源 *MessageSource*。


# 目录
* [DelegatingMessageSource解析](delegatingmessagesource解析)
    * [HierarchicalMessageSource](#HierarchicalMessageSource)
    * [MessageSourceSupport](#MessageSourceSupport)
* [总结](#总结)

## DelegatingMessageSource解析
源码参见：[DelegatingMessageSource][]

[DelegatingMessageSource]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/DelegatingMessageSource.java "DelegatingMessageSource"

```java
```
再看DelegatingMessageSource之前，先来看起父类MessageSourceSupport和父接口HierarchicalMessageSource

### HierarchicalMessageSource
源码参见：[HierarchicalMessageSource][]

[HierarchicalMessageSource]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/HierarchicalMessageSource.java "HierarchicalMessageSource"

```java
package org.springframework.context;

/**
 *HierarchicalMessageSource为消息源的子接口，实现的对象可以层级解决消息。
 * @author Rod Johnson
 * @author Juergen Hoeller
 */
public interface HierarchicalMessageSource extends MessageSource {

	/**
	 * 设置当前消息源的父消息源，当此消息源不能解决给定消息时，尝试使用父消息源
	 * 解决消息。
	 * @param parent the parent MessageSource that will be used to
	 * resolve messages that this object can't resolve.
	 * May be {@code null}, in which case no further resolution is possible.
	 */
	void setParentMessageSource(MessageSource parent);
	/**
	 * 返回消息源的父消息源，没有则为null
	 */
	MessageSource getParentMessageSource();

}

```
从上面可看出，HierarchicalMessageSource主要提供了设置父消息的操作。此接口用于，当消息源不能解决给定消息时，尝试使用父消息源解决消息，即层级解决消息。

### MessageSourceSupport
源码参见：[MessageSourceSupport][]

[MessageSourceSupport]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/MessageSourceSupport.java "MessageSourceSupport"

```java
```


最后我们以BeanDefinition的类图结束这篇文章。
![DelegatingMessageSource](/image/spring-context/DelegatingMessageSource.png)

## 总结
HierarchicalMessageSource主要提供了设置父消息的操作。此接口用于，当消息源不能解决给定消息时，尝试使用父消息源解决消息，即层级解决消息。
