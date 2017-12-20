---
layout: page
title: ConfigurableApplicationContext接口定义
subtitle: ConfigurableApplicationContext接口定义
date: 2017-12-20 08:32:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言
上一篇文章，我们看了[AutowireCapableBeanFactory][]接口，主要提供的创建bean实例，自动装配bean属性，应用bean配置属性，初始化bean，应用bean后处理器 *BeanPostProcessor* ，解决bean依赖和销毁bean操作。对于自动装配，主要提供了根据bean的name，类型和构造自动装配方式。一般不建议在在代码中直接使用AutowireCapableBeanFactory接口，我们可以通过应用上下文的ApplicationContext#getAutowireCapableBeanFactory()方法或者通过实现BeanFactoryAware，获取暴露的bean工厂，然后转换为AutowireCapableBeanFactory。NamedBeanHolder用于表示bean的name和实例的关系句柄。NamedBeanHolder可以用于Spring的根据bean的name自动装配和AOP相关的功能，避免产生不可靠的依赖。

![AutowireCapableBeanFactory](/image/spring-context/AutowireCapableBeanFactory.png)

[AutowireCapableBeanFactory]:https://donaldhan.github.io/spring-framework/2017/12/19/AutowireCapableBeanFactory%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "AutowireCapableBeanFactory"


# 目录
* [Lifecycle](#Lifecycle)
* [ConfigurableApplicationContext接口定义](#ConfigurableApplicationContext接口定义)
* [总结](#总结)

看完了AutowireCapableBeanFactory和ApplicationContext接口的定义，我们接着，来看 *ConfigurableApplicationContext* 接口的定义，再看之前先看一下父类接口Lifecycle和Closeable的定义：
先来看Lifecycle
## Lifecycle
具体源码参见：[Lifecycle][]

[Lifecycle]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/Lifecycle.java "Lifecycle"

```java
package org.springframework.context;

/**
*  Lifecycle接口是一个普通的接口，定义了控制声明周期的启动和停止操作，用于异步处理的情况。
*注意此接口不意味可以自动启动，如果有这方面的需求，可以考虑实现{@link SmartLifecycle}接口。
 * 此接口可以被组件或容器实现，比如典型的spring上下中bean定义和spring应用上下文ApplicationContext。
 * 容器应该传播启动和停止信道到所有子容器中的组件。比如在运行时环境下的停止和重启情况。
 *此接口可以通过JMX直接调用或管理操作。在管理操作的情况下， {@link org.springframework.jmx.export.MBeanExporter}定义
 *为{@link org.springframework.jmx.export.assembler.InterfaceBasedMBeanInfoAssembler},限制在声明周期范围内
 *的活动组件的可视性。
 *注意：声明周期接口，仅仅支持顶层的单例bean。在其他组件中，声明周期接口将不可探测，因此将会忽略。拓展{@link SmartLifecycle}接口
 *提供的继承应用上下文的启动和关闭阶段。
 * @author Juergen Hoeller
 * @since 2.0
 * @see SmartLifecycle
 * @see ConfigurableApplicationContext
 * @see org.springframework.jms.listener.AbstractMessageListenerContainer
 * @see org.springframework.scheduling.quartz.SchedulerFactoryBean
 */
public interface Lifecycle {

	/**
	 * 启动当前组件。如果组件已将在运行，不应该抛出异常。在容器环境下，将传播启动信号到应用的所有组件。
	 * @see SmartLifecycle#isAutoStartup()
	 */
	void start();

	/**
	 * 停止当前组件，在同步环境下，在方法返回后，组件完全停止。当异步停止行为需要的时候，可以考虑实现 {@link SmartLifecycle}接口的
	 * {@code stop(Runnable)}方法。需要注意的是：不保证停止通知发生在析构之前：在正常的关闭操作下，{@code Lifecycle} bean将会
	 * 在一般的析构回调之前，将会接受一个停止通知；然而在上下文生命周期内的热刷新或刷新尝试中断，仅仅销毁方法将会调用。如果组件还没有启动，
	 * 则不应该抛出异常。在容器环境下，应该传播停止信号到所有的组件。
	 * @see SmartLifecycle#stop(Runnable)
	 * @see org.springframework.beans.factory.DisposableBean#destroy()
	 */
	void stop();

	/**
	 * 判断当前组件是否运行。在容器中，如果所有应用的组件当前都在运行，则返回true
	 * @return whether the component is currently running
	 */
	boolean isRunning();

}
```
从上可以看出，Lifecycle接口提供了启动和关闭操作，以及判断当前组件是否运行操作。需要注意的是启动和停止操作，将会传播给容器中的所有子容器中的组件。对于停止操作，不保证停止通知发生在析构之前。对于判断当前组件是否运行操作，如果组件是容器，只有在容器中所有组件包括子容器中的组件，都在运行的情况下，才返回true。

再来看一下Closeable接口，这个时JDK的范畴。
### Closeable
```java
package java.io;

import java.io.IOException;

/**
 * A {@code Closeable} is a source or destination of data that can be closed.
 * The close method is invoked to release resources that the object is
 * holding (such as open files).
 *Closeable是一个可以关闭的数据源或目的。close方法被调用，将释放相关的资源。
 * @since 1.5
 */
public interface Closeable extends AutoCloseable {

    /**
     * Closes this stream and releases any system resources associated
     * with it. If the stream is already closed then invoking this
     * method has no effect.
     *关闭当前流，释放相关资源。如果流已关闭，则调用方法没有任何作用
     * <p> As noted in {@link AutoCloseable#close()}, cases where the
     * close may fail require careful attention. It is strongly advised
     * to relinquish the underlying resources and to internally
     * <em>mark</em> the {@code Closeable} as closed, prior to throwing
     * the {@code IOException}.
     *
     * @throws IOException if an I/O error occurs
     */
    public void close() throws IOException;
}
```
ConfigurableApplicationContext除了继承了 *Lifecycle* 和 *Closeable*，还继承了 *ApplicationContext*，关于应用上下文接口我们在前面，已将在，这里不再说，忘掉的可以去查阅。   
下面进入我们这篇文章的核心部分ConfigurableApplicationContext接口定义


## ConfigurableApplicationContext接口定义

具体源码参见：[ConfigurableApplicationContext][]

[ConfigurableApplicationContext]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/ConfigurableApplicationContext.java "ConfigurableApplicationContext"

```java

```


## 总结
Lifecycle接口提供了启动和关闭操作，以及判断当前组件是否运行操作。需要注意的是启动和停止操作，将会传播给容器中的所有子容器中的组件。对于停止操作，不保证停止通知发生在析构之前。对于判断当前组件是否运行操作，如果组件是容器，只有在容器中所有组件包括子容器中的组件，都在运行的情况下，才返回true。
