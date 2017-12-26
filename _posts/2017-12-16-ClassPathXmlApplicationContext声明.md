---
layout: page
title: ClassPathXmlApplicationContext声明
subtitle: Spring基于xml的类型路径应用上下文的声明
date: 2017-12-16 13:34:00
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
* [ClassPathXmlApplicationContext声明](#classpathxmlapplicationcontext声明)
* [ClassPathXmlApplicationContext类图](#classpathxmlapplicationcontext类图)
* [总结](#总结)
* [附：UML工具类](#附：uml工具类)


## ClassPathXmlApplicationContext声明
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
## ClassPathXmlApplicationContext类图
上述，从源码描述上，追溯了 *ClassPathXmlApplicationContext* 的声明，是不是有点头晕，我也是，所以用StartUML画了一个
的ClassPathXmlApplicationContext类图，如下：

![ClassPathXmlApplicationContext](/image/spring-context/ClassPathXmlApplicationContext.png)

从ClassPathXmlApplicationContext的类图中，可以看出ClassPathXmlApplicationContext直接或间接地实现了 *EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,MessageSource, ApplicationEventPublisher, ResourceLoader，Lifecycle，Closeable，BeanNameAware，InitializingBean，DisposableBean* ，现在我们还不能完全理解这些接口的含义，我们将在接下来的文章中说明这些接口的作用。

## 总结
ClassPathXmlApplicationContext直接或间接地实现了
*EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,MessageSource, ApplicationEventPublisher, ResourceLoader，Lifecycle，Closeable，BeanNameAware，InitializingBean，DisposableBean*

## 附：UML工具类

eclipse UML 插件:[AmaterasUML][] , [download][AmaterasUML download],在安装AmaterasUML之前要安装[GEF][]和[JDT][]
还有 *Eclipse UML Generators* 这些插件安装是要靠运气的，祝你好运。

[AmaterasUML]:http://amateras.osdn.jp/cgi-bin/fswiki_en/wiki.cgi?page=AmaterasUML "AmaterasUML"
[AmaterasUML download]:https://zh.osdn.net/projects/amateras/releases/p4435
[GEF]:http://www.eclipse.org/gef/
[JDT]:http://www.eclipse.org/jdt/

当然其他UML设计软件比如IBM的Rational Rose，Sybase的PowerDesigner，除此之外还有[enterprise architect][],不过这些软件都是要注册的，祝你破解成功，当然我们还是支持正版的。开源UML软件有 *ArgoUML* ,可惜的是不兼容win10系统，都是泪呀！最后权衡利弊，选择了[StarUML][]，作为UML设计软件。
### StarUML使用方法
1. 单机左侧工具栏 *Toolbox* 中的类或接口，在画图框中点击，即出现相应的类或接口，双击可以，添加访问属性，子类，父类，父接口；对于接口可以添加父类接口或子类接口或具体实现。
2. 当画图的时候，如果想要将两个独立的类或接口添加关系，首先单击继承，实现，组合，聚合等关系，然后在画图板中，单击一个类，出现圆圈，单击过后，不要送，拉倒另外一个类或接口，出现圆圈即可；如果想一次性添加多个类或接口之前的关闭，我们首先要双击工具中的继承，实现，组合，聚合等关系，当出现锁时，表示可添加关系，直接连接独立的类或接口即可。
3. 我们可以使用shift键或 *Ctrt+A* 选择所有，添加对齐方式或者作色，具体在右下角编辑器 *Editors* 中。
4. 在画完图以后，我们可以将类图导出为图片或pdf，具体见 *File->Export Diagrams As* 。
5. 如果想要拷贝一个文件的Diagrams到另外一个文件，可以使用 *Modle Explorer* 中的模型，只要将model中个某个类或接口复制，粘贴到另一个文件的model，再讲类拖到画图板中，即可，如果拷贝的模型在当前画图板中拥有继承和实现关系，相应的也会自动加上。
6. 在view菜单中，我们可以显示或隐藏，侧边栏，工具箱，工具栏，导航栏，编辑器及状态栏；在工具栏中可以查看类图的小地图，在地图上任意点击，即进入地图上的相应区域。
7. 添加静态内部类，首先在 *Modle Explorer* 对应外部类上右击，选择 *add class* ，然后在外部类的内部出现一个class类型，然后将class拉倒画图中即可，再添加关系，重命名。

[enterprise architect]:http://www.sparxsystems.cn/
[staruml]:http://staruml.io/download
