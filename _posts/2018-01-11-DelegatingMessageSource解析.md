---
layout: page
title: DelegatingMessageSource解析
subtitle: DelegatingMessageSource解析
date: 2018-01-11 15:17:19
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
package org.springframework.context.support;

import java.text.MessageFormat;
import java.util.HashMap;
import java.util.Locale;
import java.util.Map;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import org.springframework.util.ObjectUtils;

/**
 * MessageSourceSupport为消息源的基础实现类，提供了消除格式化处理的基础支撑。但是没有实现spring上下文中
 * 消息源的具体方法。
 *{@link AbstractMessageSource}继承了基类，并提供了{@code getMessage}的具体实现，可以代理
 *消息编码解决的中心模板方法。
 * @author Juergen Hoeller
 * @since 2.5.5
 */
public abstract class MessageSourceSupport {
	private static final MessageFormat INVALID_MESSAGE_FORMAT = new MessageFormat("");
	private boolean alwaysUseMessageFormat = false;//是否应用消息格式规则，解析没有参数的消息
	/**
	 * 缓存已经产生消息的消息格式
	 */
	private final Map<String, Map<Locale, MessageFormat>> messageFormatsPerMessage =
			new HashMap<String, Map<Locale, MessageFormat>>();

	/**
	 * 渲染给定的默认消息。
	 * 默认的实现，将会使用传入的参数，解决消息中的占位符。子类可以重写此方法，用于定制消息的处理过程
	 * @param defaultMessage the passed-in default message String
	 * @param args array of arguments that will be filled in for params within
	 * the message, or {@code null} if none.
	 * @param locale the Locale used for formatting
	 * @return the rendered default message (with resolved arguments)
	 * @see #formatMessage(String, Object[], java.util.Locale)
	 */
	protected String renderDefaultMessage(String defaultMessage, Object[] args, Locale locale) {
		return formatMessage(defaultMessage, args, locale);
	}

	/**
	 * 使用缓存的消息格式，格式化给定消息字符串。
	 * 默认使用参数解决消息中的占位符
	 * @param msg the message to format 需要格式化的消息
	 * @param args array of arguments that will be filled in for params within
	 * the message, or {@code null} if none
	 * 将会填充到消息中占位符的参数数组
	 * @param locale the Locale used for formatting 格式本地化
	 * @return the formatted message (with resolved arguments)
	 */
	protected String formatMessage(String msg, Object[] args, Locale locale) {
		if (msg == null || (!isAlwaysUseMessageFormat() && ObjectUtils.isEmpty(args))) {
			return msg;
		}
		MessageFormat messageFormat = null;
		synchronized (this.messageFormatsPerMessage) {
		    //获取消息本地化消息格式映射
			Map<Locale, MessageFormat> messageFormatsPerLocale = this.messageFormatsPerMessage.get(msg);
			if (messageFormatsPerLocale != null) {
				messageFormat = messageFormatsPerLocale.get(locale);
			}
			else {
				messageFormatsPerLocale = new HashMap<Locale, MessageFormat>();
				this.messageFormatsPerMessage.put(msg, messageFormatsPerLocale);
			}
			if (messageFormat == null) {
				try {
					//根据消息与本地化创建消息格式
					messageFormat = createMessageFormat(msg, locale);
				}
				catch (IllegalArgumentException ex) {
					// Invalid message format - probably not intended for formatting,
					// rather using a message structure with no arguments involved...
					if (isAlwaysUseMessageFormat()) {
						throw ex;
					}
					// Silently proceed with raw message if format not enforced...
					messageFormat = INVALID_MESSAGE_FORMAT;
				}
				messageFormatsPerLocale.put(locale, messageFormat);
			}
		}
		if (messageFormat == INVALID_MESSAGE_FORMAT) {
			return msg;
		}
		synchronized (messageFormat) {
			//根据参数的本地化信息，格式化消息
			return messageFormat.format(resolveArguments(args, locale));
		}
	}

	/**
	 * 根基给定的消息和本地化创建消息格式
	 * @param msg the message to create a MessageFormat for
	 * @param locale the Locale to create a MessageFormat for
	 * @return the MessageFormat instance
	 */
	protected MessageFormat createMessageFormat(String msg, Locale locale) {
		return new MessageFormat((msg != null ? msg : ""), locale);
	}
	/**
	 * 将参数解析为本地化参数
	 * 默认实现，简单返回给定的参数数组，为了解决特殊的参数类型，子类可以重写此方。
	 * @param args the original argument array
	 * @param locale the Locale to resolve against
	 * @return the resolved argument array
	 */
	protected Object[] resolveArguments(Object[] args, Locale locale) {
		return args;
	}
}
```
从上面可以看出，MessageSourceSupport内部主要成员为消息格式缓存messageFormatsPerLocale（HashMap<String, Map<Locale, MessageFormat>>()）用于存储消息的消息格式，
以及是否应用消息格式规则，解析没有参数的消息标志alwaysUseMessageFormat。此消息源实现基础类主要提供了根据给定的消息，消息参数和本地化渲染消息操作，即使用参数替代默认消息
中的占位符。

我们再来看代理消息源：

```java
package org.springframework.context.support;

import java.util.Locale;

import org.springframework.context.HierarchicalMessageSource;
import org.springframework.context.MessageSource;
import org.springframework.context.MessageSourceResolvable;
import org.springframework.context.NoSuchMessageException;

/**
 * Empty {@link MessageSource} that delegates all calls to the parent MessageSource.
 * If no parent is available, it simply won't resolve any message.
 *父消息源的代理类。如果没有父消息可用，不会解决任何消息。
 * <p>Used as placeholder by AbstractApplicationContext, if the context doesn't
 * define its own MessageSource. Not intended for direct use in applications.
 *如果上下文没有定义消息源，AbstractApplicationContext使用此代理类作为消息源的占位符。不能直接被应用
 *使用。
 * @author Juergen Hoeller
 * @since 1.1.5
 * @see AbstractApplicationContext
 */
public class DelegatingMessageSource extends MessageSourceSupport implements HierarchicalMessageSource {

	private MessageSource parentMessageSource;


	@Override
	public void setParentMessageSource(MessageSource parent) {
		this.parentMessageSource = parent;
	}

	@Override
	public MessageSource getParentMessageSource() {
		return this.parentMessageSource;
	}


	@Override
	public String getMessage(String code, Object[] args, String defaultMessage, Locale locale) {
		if (this.parentMessageSource != null) {
			return this.parentMessageSource.getMessage(code, args, defaultMessage, locale);
		}
		else {
			return renderDefaultMessage(defaultMessage, args, locale);
		}
	}

	@Override
	public String getMessage(String code, Object[] args, Locale locale) throws NoSuchMessageException {
		if (this.parentMessageSource != null) {
			return this.parentMessageSource.getMessage(code, args, locale);
		}
		else {
			throw new NoSuchMessageException(code, locale);
		}
	}

	@Override
	public String getMessage(MessageSourceResolvable resolvable, Locale locale) throws NoSuchMessageException {
		if (this.parentMessageSource != null) {
			return this.parentMessageSource.getMessage(resolvable, locale);
		}
		else {
			if (resolvable.getDefaultMessage() != null) {
				return renderDefaultMessage(resolvable.getDefaultMessage(), resolvable.getArguments(), locale);
			}
			String[] codes = resolvable.getCodes();
			String code = (codes != null && codes.length > 0 ? codes[0] : null);
			throw new NoSuchMessageException(code, locale);
		}
	}

}
```
从上面可以看出， DelegatingMessageSource内部有一父消息源，获取编码消息操作直接委托给父类数据源，如果没有父消息源，同时有默认消息，则使用消息参数
渲染默认消息，并返回。如果没有父消息可用，不会解决任何消息，抛出异常。代理消息源用于，上下文没有定义消息源情况下，AbstractApplicationContext
使用 DelegatingMessageSource为消息源的占位符，此代理消息源不能被应用直接使用。

最后我们以DelegatingMessageSource的类图结束这篇文章。
![DelegatingMessageSource](/image/spring-context/DelegatingMessageSource.png)

## 总结
HierarchicalMessageSource主要提供了设置父消息的操作。此接口用于，当消息源不能解决给定消息时，尝试使用父消息源解决消息，即层级解决消息。

MessageSourceSupport内部主要成员为消息格式缓存messageFormatsPerLocale（HashMap<String, Map<Locale, MessageFormat>>()）用于存储消息的消息格式，
以及是否应用消息格式规则，解析没有参数的消息标志alwaysUseMessageFormat。此消息源实现基础类主要提供了根据给定的消息，消息参数和本地化渲染消息操作，即使用参数替代默认消息
中的占位符。

DelegatingMessageSource内部有一父消息源，获取编码消息操作直接委托给父类数据源，如果没有父消息源，同时有默认消息，则使用消息参数
渲染默认消息，并返回。如果没有父消息可用，不会解决任何消息，抛出异常。代理消息源用于，上下文没有定义消息源情况下，AbstractApplicationContext
使用 DelegatingMessageSource为消息源的占位符，此代理消息源不能被应用直接使用。
