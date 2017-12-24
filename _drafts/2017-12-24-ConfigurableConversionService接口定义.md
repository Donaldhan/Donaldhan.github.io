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
* [总结](#总结)
* [附](#附)
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
[ConversionService]: "ConversionService"

```java
```

### ConverterRegistry
源码参见：[ConverterRegistry][]

[ConverterRegistry]: "ConverterRegistry"
```java
```

## 总结
ConfigurableConversionService接口主要是用于加强 *ConversionService*
暴露的可读操作，为添加和移除转换器Converter提供便利，而没有提供除 *ConversionService* 和 *ConverterRegistry* 之外的操作。


## 附
