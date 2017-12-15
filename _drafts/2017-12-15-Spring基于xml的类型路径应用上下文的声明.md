---
layout: page
title: my blog
subtitle: sub title
date: 2017-12-14 21:00:00
author: donaldhan
catalog: true
category: spring-framework
categories:
    -  spring-framework
tags:
    - spring-context
---
[Spring源码阅读引导篇][]

[Spring源码阅读引导篇]: https://donaldhan.github.io/spring-framework/2017/12/14/Spring%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E5%BC%95%E5%AF%BC%E7%AF%87.html "Spring源码阅读引导篇"

# 引言
上一篇文章[Spring源码阅读引导篇][]，我们从spring bean容器，事务及mvc3个方面，展示了spring这3个特性的用法，以及spring3
和spring4，在3方面配置的不同点。 这3特性，我们先从spring核心bean容器开始。对于spring核心bean容器，我们先看 *ClassPathXmlApplicationContext* ,
然后在分析ContextLoaderListener。

## 目录
* [](#)
* [](#)
* [](#)

先从基于xml的类路径应用上下文 *ClassPathXmlApplicationContext* 的声明定义开始：

```java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
}
```

再来看其父类抽象xml应用上下文 *AbstractXmlApplicationContext* 的声明
```java
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext {
}
```
再来看抽象可刷新配置应用上下文 *AbstractRefreshableConfigApplicationContext* 的声明
```java
public abstract class AbstractRefreshableConfigApplicationContext extends AbstractRefreshableApplicationContext
		implements BeanNameAware, InitializingBean {
}
```
从AbstractRefreshableConfigApplicationContext的声明来看继承了抽象可刷新应用上下文 *AbstractRefreshableApplicationContext*，同时实现了两个接口一个是 *BeanNameAware*,另一个为 *InitializingBean*，我们先来看 *AbstractRefreshableApplicationContext* 的声明定义，这个分支追溯完之后，我们在回来看  *BeanNameAware* 和 *InitializingBean* 接口的声明定义。

```java
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
}
```
再来看抽象应用上下文的定义 *AbstractApplicationContext*

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {
}
```
从抽象应用上下文的定义来看，其继承了默认资源加载器 *DefaultResourceLoader* ，同时实现了
可配置应用上下文接口 *ConfigurableApplicationContext* 和 *DisposableBean* 接口。

***

先来看 *ConfigurableApplicationContext*，回来再看 *DefaultResourceLoader* 和  *DisposableBean* 接口的声明定义。

```java
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {
}
```
从可配置上下文接口的声明，总于可以看到点眉目，可配置上下文继承了应用上下文 *ApplicationContext*, 同时实现了 *Lifecycle*
和 *Closeable* 两个类。 再来应用上下文 *ApplicationContext* 的声明定义：

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
}

```

是不是又是吓的一头汗，还有没尽头了呢,大致看了一下，*EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
MessageSource, ApplicationEventPublisher, ResourcePatternResolver* 这些都是接口。

我们依次来看这些接口的定义

```java
public interface EnvironmentCapable {
}
```

```java
public interface ListableBeanFactory extends BeanFactory {
}
```

```java
public interface BeanFactory {
}
```

```java
public interface HierarchicalBeanFactory extends BeanFactory {
}
```

```java
public interface MessageSource {
}
```

```java
public interface ApplicationEventPublisher {
}
```

```java
public interface ResourcePatternResolver extends ResourceLoader {
}
```


```java
public interface ResourceLoader {
}
```


退回到 *ConfigurableApplicationContext*，来看 *DefaultResourceLoader* 和  *DisposableBean* 接口的声明定义。

```java
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {
}
```

```java
public interface Lifecycle {}

```

```java
/**
 * A {@code Closeable} is a source or destination of data that can be closed.
 * The close method is invoked to release resources that the object is
 * holding (such as open files).
 *
 * @since 1.5
 */
public interface Closeable extends AutoCloseable {
}

```

回到抽象应用上下文的定义 *AbstractApplicationContext*

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {
}
```
来看接口 *DisposableBean* 和 默认资源加载器 *DefaultResourceLoader* 的声明定义：

```java
public class DefaultResourceLoader implements ResourceLoader {
}
```

```java
public interface DisposableBean {}

```

回到抽象可刷新配置应用上下文 *AbstractRefreshableConfigApplicationContext* 的声明
```java
public abstract class AbstractRefreshableConfigApplicationContext extends AbstractRefreshableApplicationContext
		implements BeanNameAware, InitializingBean {
}
```
在看来 *BeanNameAware* 和 *InitializingBean* 接口声明

```java
public interface BeanNameAware extends Aware {
}

```

```java
public interface Aware {

}
```
```java
public interface InitializingBean {
}
```




MDT

eclipse UML 插件[AmaterasUML][]
[AmaterasUML]:http://amateras.osdn.jp/cgi-bin/fswiki_en/wiki.cgi?page=AmaterasUML "AmaterasUML"
[AmaterasUML download]:https://zh.osdn.net/projects/amateras/releases/p4435
[GEF]:http://www.eclipse.org/gef/
[JDT]:http://www.eclipse.org/jdt/
[enterprise architect]:http://www.sparxsystems.cn/
[staruml]:http://staruml.io/download
[Eclipse UML Generators ]:
*-->* 表示继承 *--* 表示接口实现  
ClassPathXmlApplicationContext  
    -->AbstractXmlApplicationContext  
        -->AbstractRefreshableConfigApplicationContext  
            -->AbstractRefreshableApplicationContext  
                -->AbstractApplicationContext   
            --BeanNameAware  
            --InitializingBean
