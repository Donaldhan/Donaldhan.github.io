---
layout: page
title: AbstractApplicationContext源码解析第五讲
subtitle:  AbstractApplicationContext源码解析第五讲
date: 2018-01-25 09:56:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言

我们可以通过setEnvironment配置上下文环境，通过getEnvironment获取上下文位置，如果没有则上下文的环境默认为StandardEnvironment。
需要注意的是在修改应用上下文环境操作，应该在刷新上下文refresh操作之前。

获取自动装配bean工厂AutowireCapableBeanFactory，实际委托给获取bean工厂方法getBeanFactory，getBeanFactory方法待子类扩展。


抽象应用上下文发布应用事件实际委托给内部的应用事件多播器，应用事件多播器在上下文刷新的时候初始化，如果配置有应用事件多播器
，则使用配置的应用事件多播器发布应用事件，否则默认使用SimpleApplicationEventMulticaster。在发布应用事件的时候，如果预发布事件集不为空，
则发布事件集中的应用事件，如果当上下文的父上下文不为空，则调用父上下文的事件发布操作，将事件传播给父上下文中的应用监听器。

在设置上下文的父应用上文时，如果父上下文的环境配置为可配置环境，则合并父上下文环境到当前上下文环境。

添加bean工厂后处理器，实际为将bean工厂后处理器添加到上下文的bean工厂后处理器集中beanFactoryPostProcessors（ArrayList<BeanFactoryPostProcessor>）。添加应用监听器，实际上，是将监听器添加到上下文的监听器集合applicationListeners（LinkedHashSet<ApplicationListener<?>>）中。

应用上下文刷新的过程为：
1. 准备上下文刷新操作；
准备上下文刷新操作主要初始化应用上下文环境中的占位符属性源，验证所有需要可解决的标注属性，创建预发布应用事件集earlyApplicationEvents（LinkedHashSet<ApplicationEvent>）。
初始化属性源方法 *#initPropertySources* 待子类实现。

2. 告诉子类刷新内部bean工厂；
通知子类刷新内部bean工厂实际操作在 *#refreshBeanFactory* 中，刷新bean工厂操作待子类扩展，在刷新完bean工厂之后，返回当前上下文的bean工厂，
返回当前上下文的bean工厂 *#getBeanFactory* 待子类实现。

3. 准备上下文使用的bean工厂；
预备bean工厂畅主要的是设置工厂类加载器，Spring EL表达式解决器 *StandardBeanExpressionResolver*，资源编辑器 *ResourceEditorRegistrar*；
添加bean后处理器 *ApplicationContextAwareProcessor* 处理与应用上下文关联的Awarebean实例，比如 *EnvironmentAware，EmbeddedValueResolverAware，ResourceLoaderAware
ApplicationEventPublisherAware，MessageSourceAware，ApplicationContextAware*，并忽略掉这些依赖。同时注册 *BeanFactory，ResourceLoader，ApplicationEventPublisher，
ApplicationContext* 类到工厂可解决类。添加应用监听器探测器，探测应用监听器bean定义，并添加到应用上下文中。如果bean工厂中包含加载时间织入器，则设置加载时间织入器后处理器
*LoadTimeWeaverAwareProcessor* ， 并配置类型匹配的临时类加载器。最后如果需要，注册环境，系统属性，系统变量单例bean到bean工厂。

4. 在上下文子类中，允许后处理bean工厂；
在标准初始化完成后，修改应用上下文内部的bean工厂。所有的bean定义已经加载，但是还没bean已完成初始化。
允许在确定的应用上下文实现中注册特殊的bean后处理器。方法postProcessBeanFactory待子类扩展。

5. 调用注册到上下文中的工厂后处理器；
这一步所有的做的事情为调用bean工厂后处理器，处理bean工厂，具体过程为：如果为bean工厂为bean定义注册器实例，
则调用上下文中的bean定义注册器后处理器,处理工厂，然后调用实现了PriorityOrdered，Ordered接口和剩余的bean定义注册器后处理器，处理bean工厂；
如果bean工厂不是bean定义注册器实例，则使用上下文中的bean工厂后处理器，处理bean工厂。从bean工厂内获取所有bean工厂处理器实例，
按实现了PriorityOrdered，Ordered接口和剩余的bean工厂后处理器顺序，处理bean工厂；最后清空bean工厂的元数据缓存。
如果bean工厂的临时加载器为空，且包含加载时间织入器，则设置bean工厂的加载时间织入器后处理器，设置bean工厂的临时类加载器为[ContextTypeMatchClassLoader][]。

6. 注册拦截bean创建的bean后处理器；
注册bean工厂的bean后处理操作，主要委托给[PostProcessorRegistrationDelegate][],注册过程为：
注册实现PriorityOrdered，Ordered和剩余的bean后处理器到bean工厂，然后注册内部bean后处理器[MergedBeanDefinitionPostProcessor][];
最后添加应用监听器探测器[ApplicationListenerDetector][]。

7. 初始化上下文的消息源；
如果bean工厂包含消息源，则配置为上下文消息源，如果当前上下文父上下文不为空且消息源为层级消息源HierarchicalMessageSource，
配置当前消息源的父消息源为当前上下文内部父消息源。否则创建上下文内部的父消息源的代理DelegatingMessageSource，并注册消息源到bean工厂。

8. 始化上下文的事件多播器；
初始化上下文的事件多播器过程为，如果bean工厂包含应用事件多播器，则配置应用上下文的应用事件多播器为bean工厂内的应用事件多播器；
否则使用SimpleApplicationEventMulticaster，并注册到bean工厂。

9. 初始化上下文子类中的其他特殊的bean；
初始化上下文子类中的其他特殊的bean方法onRefresh，待子类扩展使用。

10. 检查监听器bean，并注册；
注册监听器过程，主要是注册应用上下文中的应用监听器到应用事件多播器，注册bean工厂内部的应用监听器
到应用事件多播器，如果存在预发布应用事件，则多播应用事件。

11. 初始化所有遗留的非懒加载单例bean；
完成上下文bean工厂的初始化，初始化所有遗留的单例bean，如果bean工厂内存在转换服务ConversionService，则配置bean工厂的转换服务；
如果没有bean后处理器（PropertyPlaceholderConfigurer）注册，则注册一个默认的嵌入式值解决器StringValueResolver，主要解决注解属性的值；
初始化先前加载时间织入器Aware bean，允许注册他们的变换器；停止使用临时类加载器用于类型匹配；冻结bean工厂配置；最后初始化所有剩余的单例bean。

12. 完成上下文刷新；
完成上下文刷新主要所有的工作为，初始化上下文的生命周期处理器，默认为 [DefaultLifecycleProcessor][],生命周期处理器主要管理
bean工厂内部的生命周期bean。然后启动bean工厂内的生命周期bean。然后发布上下文刷新事件给应用监听器，最后注册上下文到[LiveBeansView][] MBean，
以便监控上下文中bean工厂内部的bean的定义及依赖。
如果以上过程出现异常则执行13,14步：

13. 销毁已经创建的单例bean，避免资源空占；
销毁bean操作，主要委托给应用上文的内部bean工厂，bean工厂完成销毁所有的单例bean。

14. 重置上下文激活状态标志；
最后执行15步，

15. 由于我们不在需要单例bean的元数据，重置Spring核心的一般内省缓存。
主要清除，类的方法与属性缓存，可解决类型缓存，及类缓存。

![AbstractApplicationContext](/image/spring-context/AbstractApplicationContext.png)

这是我们上一篇文章[AbstractApplicationContext源码解析第四讲][]所讲的内容，主要的是应用上下文的刷新操作，今天我们来看抽象应用上下文的剩余操作，主要是关闭上下文，bean工厂，消息源，及资源相关的操作。

[AbstractApplicationContext源码解析第四讲]:https://donaldhan.github.io/spring-framework/2018/01/24/AbstractApplicationContext%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%AC%AC%E5%9B%9B%E8%AE%B2.html "AbstractApplicationContext源码解析第四讲"




# 目录
* [AbstractApplicationContext定义](abstractapplicationcontext定义)
    * [](#)
    * [](#)
* [总结](#总结)
* [附](#附)

## AbstractApplicationContext定义
源码参见：[AbstractApplicationContext][]

[AbstractApplicationContext]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java "AbstractApplicationContext"

```java
```


###
源码参见：[][]

[]: ""

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

## 附
