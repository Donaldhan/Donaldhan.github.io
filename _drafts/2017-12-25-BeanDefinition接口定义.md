---
layout: page
title: BeanDefinition接口定义
subtitle: BeanDefinition接口及父类接口定义
date: 2017-12-25 12:46:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---
# 引言
先来回顾一下，上一篇文章[MutablePropertySources定义][],MutablePropertySources为属性源Holder PropertySource的具体实现，内部通过一个属性源集合（CopyOnWriteArrayList）来管理内部的属性源，主要提供添加、移除、替换、是否包含属性源操作，这些操作实际上通过 *CopyOnWriteArrayList* 的相应操作完成。

![MutablePropertySources](/image/spring-context/MutablePropertySources.png)

在[ConfigurableListableBeanFactory接口定义][]中，提供了一个操作为获取给定name的bean的定义 *BeanDefinition*。由于篇幅问题，我们没有将 *BeanDefinition*，
今天我们就来*BeanDefinition*的接口定义。

[MutablePropertySources定义]:https://donaldhan.github.io/spring-framework/2017/12/25/MutablePropertySources%E5%AE%9A%E4%B9%89.html "MutablePropertySources定义"

[ConfigurableListableBeanFactory接口定义]:https://donaldhan.github.io/spring-framework/2017/12/22/ConfigurableListableBeanFactory%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "ConfigurableListableBeanFactory接口定义"

# 目录
* [BeanDefinition接口定义](#beandefinition接口定义)
    * [AttributeAccessor](#attributeaccessor)
    * [BeanMetadataElement](#beanmetadataelement)
* [总结](#总结)
## BeanDefinition接口定义
源码参见：[BeanDefinition][]

[BeanDefinition]: "BeanDefinition"

```java
```

### AttributeAccessor
源码参见：[AttributeAccessor][]

[AttributeAccessor]: "AttributeAccessor"

```java
```


### BeanMetadataElement
源码参见：[BeanMetadataElement][]

[BeanMetadataElement]: "BeanMetadataElement"

```java
```



## 总结
