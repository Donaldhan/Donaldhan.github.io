---
layout: page
title: my blog
subtitle: sub title
date: 2017-12-27 10:53:30
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言

DisposableBean主要提供的销毁操作，一般用于在bean析构单例bean的时候调用，以释放bean关联的资源。

默认资源加载器DefaultResourceLoader内部有两个变量，一个为类加载器 *classLoader（ClassLoader）*，一个为协议解决器集合 *protocolResolvers（LinkedHashSet<ProtocolResolver>(4)）* ，协议解决器集合初始size为4。默认资源加载器提供了类加载器属性的set与get方法，提供了协议解决器集添加和获取方法。

默认资源加载器的默认类型加载器为当前线程上下文类加载器，如果当前线程上下文类加载器为空，则获取 *ClassUtils* 类的类加载，如果*ClassUtils*类的类加载为空，则获取系统类加载器。

获取给定位置的资源方法，首先遍历协议解决器集，如果可以解决，则返回位置相应的资源，否则，如果资源位置以"/"开头，则获取路径资源 *ClassPathContextResource*
否则，如果资源位置以 *"classpath:"* 开头，创建路径位置的的类路径资源 *ClassPathResource* 否则返回给定位置的URL资源 *UrlResource* 。

ContextResource表示一个封闭上下文中的资源，提供了相对于上下文根目录的相对路径操作。

AbstractResource资源实现了资源的典型行为操作，判断资源是否存在操作，获取资源URI，获取资源内容大小，获取资源上次修改时间。
判断资源是否存在方法，将会先检查文件是否存在，如果文件不可打开，再检查输入流是否可以打开。获取资源URI方法，委托给 *ResourceUtils* 将资源的URL，
转化为URI。获取资源内容大小操作，即读取文件字节内容直到不可读，子类的提供更优的实现。获取资源上次修改时间，实际上是获取资源文件的上次修改时间。
由于AbstractResource描述的是一个抽象资源，牵涉到底层资源的方法isOpen、getURL、getFile，要么是不支持，要么false，要么为空，这些待具体的资源实现。


AbstractFileResolvingResource获取文件操作，首先检查文件是否为JBOSS的VFS文件，如果是则VFS文件获取委托给VfsResourceDelegate，VfsResourceDelegate代理的是JBoos的VFS资源VfsResource，VfsResource内部关联一个底层资源对象，VfsResource的所有关于Resource的操作实际上委托给VfsUtils，VfsUtils实际是通过反射调用org.jboss.vfs.VirtualFile的相应方法。
否则委托给ResourceUtils获取文件系统中的文件资源。根据URI获取文件资源思路与获取文件相同。获取上次修改文件方法，如果是jar包文件，则委托给ResourceUtils，从给定的最外层的Jar或War URL 中抽取出，抽取内部的文件系统URL（可以指向jar包中的文件，或者一个jar文件）。如果URL资源为VFS，则委托给VfsResourceDelegate，否则委托给ResourceUtils的获取文件方法。判断文件是否可读，主要是判断文件是否存在，同时为非目录。检查文件是否存在，如果是文件系统，则委托给文件的exists的方法，否则URLConnection，如果URLConnection为HttpURLConnection，
获取HttpURLConnection的转台，OK则存在，否则false。获取资源内容长度和上次修改时间与判断文件是否存在的思想基本相同。

这是我们上一篇[AbstractApplicationContext源码解析第一讲][]的内容，下面我们来接着往下看 *ClassPathResource->ClassPathContextResource->UrlResource* 。


![AbstractFileResolvingResource](/image/spring-context/AbstractFileResolvingResource.png)

[AbstractApplicationContext源码解析第一讲]:https://donaldhan.github.io/spring-framework/2017/12/27/AbstractApplicationContext%E5%AE%9A%E4%B9%89%E7%AC%AC%E4%B8%80%E8%AE%B2.html "AbstractApplicationContext源码解析第一讲"



# 目录
* [AbstractApplicationContext定义](abstractapplicationcontext定义)
    * [ClassPathResource](#classpathresource)
    * [ClassPathContextResource](#classpathcontextresource)
    * [UrlResource](#urlresource)
* [总结](#总结)
* [附](#附)

## AbstractApplicationContext定义

上一讲讲到ClassPathResource的父类抽象文件资源AbstractFileResolvingResource。下面我们来看资源的具体实现类ClassPathResource。

### ClassPathResource

源码参见：[ClassPathResource][]

[ClassPathResource]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/io/ClassPathResource.java "ClassPathResource"

```java
```

### ClassPathContextResource

源码参见：[ClassPathContextResource][]

[ClassPathContextResource]: "ClassPathContextResource"

```java
```
### UrlResource

源码参见：[UrlResource][]

[UrlResource]: "UrlResource"

```java
```

源码参见：[AbstractApplicationContext][]

[AbstractApplicationContext]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java "AbstractApplicationContext"

```java
```


最后我们以BeanDefinition的类图结束这篇文章。
![BeanDefinition](/image/spring-context/BeanDefinition.png)



## 总结




## 附
