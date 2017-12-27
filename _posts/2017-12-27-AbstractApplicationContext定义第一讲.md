---
layout: page
title: AbstractApplicationContext源码解析第一讲
subtitle: AbstractApplicationContext父类接口DisposableBean，DefaultResourceLoader，ContextResource，AbstractResource，AbstractFileResolvingResource声明定义
date:  2017-12-27 10:53:30
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---
# 引言

[BeanDefinition接口][]用于描述一个bean实例的属性及构造参数等元数据；主要提供了父beanname，bean类型名，作用域，懒加载，
bean依赖，自动注入候选bean，自动注入候选主要bean熟悉的设置与获取操作。同时提供了判断bean是否为单例、原型模式、抽象bean的操作，及获取bean的描述，资源描述，属性源，构造参数，原始bean定义等操作。

![BeanDefinition](/image/spring-context/BeanDefinition.png)

[BeanDefinition接口]:https://donaldhan.github.io/spring-framework/2017/12/26/BeanDefinition%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "BeanDefinition接口"

上一篇文章我们看了，BeanDefinition接口的定义，截止到上一篇文章我们将应用上下文和可配置应用上下文已看完，从这篇文章开始，我们将进入应用上下文的实现。


# 目录
* [AbstractApplicationContext定义](abstractapplicationcontext定义)
    * [DisposableBean](#disposablebean)
    * [DefaultResourceLoader](#defaultresourceloader)
    * [ContextResource](#contextresource)
    * [AbstractResource](#abstractresource)
    * [AbstractFileResolvingResource](#abstractfileresolvingresource)
    * [ClassPathResource](#classpathresource)
    * [ClassPathContextResource](#classpathcontextresource)
    * [UrlResource](#urlresource)
* [总结](#总结)
* [附](#附)


## AbstractApplicationContext定义
我们先来看一下，DisposableBean接口和默认的资源加载器DefaultResourceLoader

### DisposableBean
源码参见：[DisposableBean][]

[DisposableBean]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/DisposableBean.java "DisposableBean"

```java
package org.springframework.beans.factory;

/**
 *DisposableBean接口的实现用于在析构时，释放资源。如果bean工厂销毁一个缓存单例bean，应该调用#destroy方法。
 *应用上下文在关闭时，应该销毁所有的单例bean。
 *DisposableBean的一种可选实现为，在基于XML的bean定义中，配置bean的destroy-method。更多关于所有的bean的
 *生命周期方法，见BeanFactory的javadocs。
 *
 * @author Juergen Hoeller
 * @since 12.08.2003
 * @see org.springframework.beans.factory.support.RootBeanDefinition#getDestroyMethodName
 * @see org.springframework.context.ConfigurableApplicationContext#close
 */
public interface DisposableBean {

	/**
	 * bean工厂在析构单例bean的时候调用此方法。
	 * @throws Exception in case of shutdown errors.
	 * Exceptions will get logged but not rethrown to allow
	 * other beans to release their resources too.
	 * 在关闭错误的情况下，异常将被log输出，而不是重新抛出以允许其他bean释放资源。
	 */
	void destroy() throws Exception;

}

```
从上面可以看出，DisposableBean主要提供的销毁操作，一般用于在bean析构单例bean的时候调用，以释放bean关联的资源。


### DefaultResourceLoader
源码参见：[DefaultResourceLoader][]

[DefaultResourceLoader]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/io/DefaultResourceLoader.java "DefaultResourceLoader"

```java
package org.springframework.core.io;

import java.net.MalformedURLException;
import java.net.URL;
import java.util.Collection;
import java.util.LinkedHashSet;
import java.util.Set;

import org.springframework.util.Assert;
import org.springframework.util.ClassUtils;
import org.springframework.util.StringUtils;

/**
 *默认资源加载器DefaultResourceLoader为资源加载器接口的默认使用，可以通过资源编辑器使用，作为
 * {@link org.springframework.context.support.AbstractApplicationContext}的基类，
 * 也可以单独使用。
 * <p>Will return a {@link UrlResource} if the location value is a URL,
 * and a {@link ClassPathResource} if it is a non-URL path or a
 * "classpath:" pseudo-URL.
 *如果资源为值为URL则返回URL资源
 * @author Juergen Hoeller
 * @since 10.03.2004
 * @see FileSystemResourceLoader
 * @see org.springframework.context.support.ClassPathXmlApplicationContext
 */
public class DefaultResourceLoader implements ResourceLoader {

	private ClassLoader classLoader;

	private final Set<ProtocolResolver> protocolResolvers = new LinkedHashSet<ProtocolResolver>(4);


	/**
	 * Create a new DefaultResourceLoader.
	 * <p>ClassLoader access will happen using the thread context class loader
	 * at the time of this ResourceLoader's initialization.
	 * 创建一默认的资源加载器。在资源加载器初始化的时候，线程类上下文加载器将会访问类加载器。
	 * @see java.lang.Thread#getContextClassLoader()
	 */
	public DefaultResourceLoader() {
		this.classLoader = ClassUtils.getDefaultClassLoader();
	}

	/**
	 * Create a new DefaultResourceLoader.
	 * @param classLoader the ClassLoader to load class path resources with, or {@code null}
	 * for using the thread context class loader at the time of actual resource access
	 */
	public DefaultResourceLoader(ClassLoader classLoader) {
		this.classLoader = classLoader;
	}


	/**
	 * Specify the ClassLoader to load class path resources with, or {@code null}
	 * for using the thread context class loader at the time of actual resource access.
	 * <p>The default is that ClassLoader access will happen using the thread context
	 * class loader at the time of this ResourceLoader's initialization.
	 */
	public void setClassLoader(ClassLoader classLoader) {
		this.classLoader = classLoader;
	}

	/**
	 * Return the ClassLoader to load class path resources with.
	 * <p>Will get passed to ClassPathResource's constructor for all
	 * ClassPathResource objects created by this resource loader.
	 * @see ClassPathResource
	 */
	@Override
	public ClassLoader getClassLoader() {
		return (this.classLoader != null ? this.classLoader : ClassUtils.getDefaultClassLoader());
	}

	/**
	 * Register the given resolver with this resource loader, allowing for
	 * additional protocols to be handled.
	 * 注册跟定的资源协议解决器到资源加载器，以允许额外的协议被处理。
	 * <p>Any such resolver will be invoked ahead of this loader's standard
	 * resolution rules. It may therefore also override any default rules.
	 * 任何协议解决器将会加载器的标准解决规则前被调用。因此有可能重写默认的规则。
	 * @since 4.3
	 * @see #getProtocolResolvers()
	 */
	public void addProtocolResolver(ProtocolResolver resolver) {
		Assert.notNull(resolver, "ProtocolResolver must not be null");
		this.protocolResolvers.add(resolver);
	}

	/**
	 * Return the collection of currently registered protocol resolvers,
	 * allowing for introspection as well as modification.
	 * 返回当前注册到资源加载器的协议解决器集，允许监视和修改。
	 * @since 4.3
	 */
	public Collection<ProtocolResolver> getProtocolResolvers() {
		return this.protocolResolvers;
	}
}
```
从上面可以，默认资源加载器DefaultResourceLoader内部有两个变量，一个为类加载器 *classLoader（ClassLoader）*，一个为协议解决器集合 *protocolResolvers（LinkedHashSet<ProtocolResolver>(4)）* ，协议解决器集合初始size为4。默认资源加载器提供了类加载器属性的set与get方法，提供了协议解决器集添加和获取方法。

在我们在来看一下默认资源加载器的无参构造中，默认的类加载器。

```java
public DefaultResourceLoader() {
    this.classLoader = ClassUtils.getDefaultClassLoader();
}
```

再来看ClassUtils的获取默认类加载器方法。

```java
public abstract class ClassUtils {
/**
	 * Return the default ClassLoader to use: typically the thread context
	 * ClassLoader, if available; the ClassLoader that loaded the ClassUtils
	 * class will be used as fallback.
	 * 返回默认的类型加载器：如果可用的话，返回当前线程上下文加载器，否则返回ClassUtils的类的类加载。
	 * <p>Call this method if you intend to use the thread context ClassLoader
	 * in a scenario where you clearly prefer a non-null ClassLoader reference:
	 * for example, for class path resource loading (but not necessarily for
	 * {@code Class.forName}, which accepts a {@code null} ClassLoader
	 * reference as well).
	 *
	 * @return the default ClassLoader (only {@code null} if even the system
	 * ClassLoader isn't accessible)
	 * 如果系统类加载器不可访问你，则默认的类加载器为null。
	 * @see Thread#getContextClassLoader()
	 * @see ClassLoader#getSystemClassLoader()
	 */
	public static ClassLoader getDefaultClassLoader() {
		ClassLoader cl = null;
		try {
			//获取当前线程上下文类加载器
			cl = Thread.currentThread().getContextClassLoader();
		}
		catch (Throwable ex) {
			// Cannot access thread context ClassLoader - falling back...
		}
		if (cl == null) {
			// No thread context class loader -> use class loader of this class.
			//如果当前线程上下文类加载器为空，则获取ClassUtils类的类加载
			cl = ClassUtils.class.getClassLoader();
			if (cl == null) {
				// getClassLoader() returning null indicates the bootstrap ClassLoader
				try {
					//如果ClassUtils类的类加载为空，则获取系统类加载器
					cl = ClassLoader.getSystemClassLoader();
				}
				catch (Throwable ex) {
					// Cannot access system ClassLoader - oh well, maybe the caller can live with null...
				}
			}
		}
		return cl;
	}
}
```
从上面可以看出，默认资源加载器的默认类型加载器为当前线程上下文类加载器，如果当前线程上下文类加载器为空，则获取 *ClassUtils* 类的类加载，如果*ClassUtils*类的类加载为空，则获取系统类加载器。

再来看默认资源加载器的获取给定位置资源的方法：

```java
@Override
	public Resource getResource(String location) {
		Assert.notNull(location, "Location must not be null");
        //遍历协议解决器集，如果可以解决，则返回位置相应的资源
		for (ProtocolResolver protocolResolver : this.protocolResolvers) {
			Resource resource = protocolResolver.resolve(location, this);
			if (resource != null) {
				return resource;
			}
		}
        //如果资源位置以"/"开头，则获取路径资源
		if (location.startsWith("/")) {
			return getResourceByPath(location);
		}
		else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
			//如果资源位置以"classpath:"开头，创建路径位置的的类路径资源ClassPathResource
			return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
		}
		else {
			try {
				//否则创建URL资源
				// Try to parse the location as a URL...
				URL url = new URL(location);
				return new UrlResource(url);
			}
			catch (MalformedURLException ex) {
				// No URL -> resolve as resource path.
				return getResourceByPath(location);
			}
		}
	}

	/**
	 * Return a Resource handle for the resource at the given path.
	 * 返回给定路径位置的资源Handle。
	 * <p>The default implementation supports class path locations. This should
	 * be appropriate for standalone implementations but can be overridden,
	 * e.g. for implementations targeted at a Servlet container.
	 * 默认实现支持类路径位置。这个应该使用与独立的版本实现，但是可以被重写。比如针对Servlet容器的实现。
	 * @param path the path to the resource
	 * @return the corresponding Resource handle
	 * @see ClassPathResource
	 * @see org.springframework.context.support.FileSystemXmlApplicationContext#getResourceByPath
	 * @see org.springframework.web.context.support.XmlWebApplicationContext#getResourceByPath
	 */
	protected Resource getResourceByPath(String path) {
		return new ClassPathContextResource(path, getClassLoader());
	}
```
从上面可以看出，获取给定位置的资源方法，首先遍历协议解决器集，如果可以解决，则返回位置相应的资源，否则，如果资源位置以"/"开头，则获取路径资源 *ClassPathContextResource*
否则，如果资源位置以 *"classpath:"* 开头，创建路径位置的的类路径资源 *ClassPathResource* 否则返回给定位置的URL资源 *UrlResource* 。

再来看一下默认资源加载器的静态内部类 *ClassPathContextResource* 的声明定义。

```java
protected static class ClassPathContextResource extends ClassPathResource implements ContextResource {
}
public interface ContextResource extends Resource {
}
```
再来看另外一个分支：
```java
public class ClassPathResource extends AbstractFileResolvingResource {
}
public abstract class AbstractFileResolvingResource extends AbstractResource {

}
public abstract class AbstractResource implements Resource {
}
```

URL资源声明：
```java
public class UrlResource extends AbstractFileResolvingResource {
}
```
有了上面分析我们对 *ClassPathContextResource* 有一个概念性的了解，下面，我们将从 *ContextResource->AbstractResource->AbstractFileResolvingResource->ClassPathResource/UrlResource->ClassPathContextResource* 来分析 *ClassPathContextResource*


### ContextResource

源码参见：[ContextResource][]

[ContextResource]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/io/ContextResource.java "ContextResource"

```java
package org.springframework.core.io;

/**
 * Extended interface for a resource that is loaded from an enclosing
 * 'context', e.g. from a {@link javax.servlet.ServletContext} or a
 * {@link javax.portlet.PortletContext} but also from plain classpath paths
 * or relative file system paths (specified without an explicit prefix,
 * hence applying relative to the local {@link ResourceLoader}'s context).
 *上下文资源接口ContextResource，是一个从封闭上下文加载的拓展资源接口。
 *比如Servlet上下文{@link javax.servlet.ServletContext}及Portlet上下文，
 *类路径，文件系统的相对路径（没有明确的前缀，因此为一个本地的资源加载器上下文）。
 *
 *
 * @author Juergen Hoeller
 * @since 2.5
 * @see org.springframework.web.context.support.ServletContextResource
 * @see org.springframework.web.portlet.context.PortletContextResource
 */
public interface ContextResource extends Resource {

	/**
	 * Return the path within the enclosing 'context'.
	 * 返回上下文中的资源路径。
	 * <p>This is typically path relative to a context-specific root directory,
	 * 典型的是相对于上下文根目录的路径的路径，比如Servlet上下文Context
	 * e.g. a ServletContext root or a PortletContext root.
	 */
	String getPathWithinContext();

}
```
从上面可以看出，ContextResource表示一个封闭上下文中的资源，提供了相对于上下文根目录的相对路径操作。

### AbstractResource

源码参见：[AbstractResource][]

[AbstractResource]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/io/AbstractResource.java "AbstractResource"

```java
package org.springframework.core.io;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.net.URI;
import java.net.URISyntaxException;
import java.net.URL;

import org.springframework.core.NestedIOException;
import org.springframework.util.Assert;
import org.springframework.util.ResourceUtils;

/**
 * Convenience base class for {@link Resource} implementations,
 * pre-implementing typical behavior.
 *AbstractResource资源实现的基础类，与实现了典型的行为。
 * <p>The "exists" method will check whether a File or InputStream can
 * be opened; "isOpen" will always return false; "getURL" and "getFile"
 * throw an exception; and "toString" will return the description.
 * 判断资源是否存在方法，将会检查文件或输入流是否可以打开。isOpen方法总是返回false，
 * getURL和getFile方法，将抛出异常，toString将会资源的描述。
 * @author Juergen Hoeller
 * @since 28.12.2003
 */
public abstract class AbstractResource implements Resource {

	/**
	 * This implementation checks whether a File can be opened,
	 * falling back to whether an InputStream can be opened.
	 * This will cover both directories and content resources.
	 * 当前检查文件是否存在的实现为，检查文件是否能打开，不能则查看
	 * 输入流是否能够打开。此方法将覆盖文件目录和内容资源。
	 */
	@Override
	public boolean exists() {
		// Try file existence: can we find the file in the file system?
		try {
			return getFile().exists();
		}
		catch (IOException ex) {
			// Fall back to stream existence: can we open the stream?
			try {
				InputStream is = getInputStream();
				is.close();
				return true;
			}
			catch (Throwable isEx) {
				return false;
			}
		}
	}

	/**
	 * This implementation always returns {@code true}.
	 * 可读性总是返回true
	 */
	@Override
	public boolean isReadable() {
		return true;
	}

	/**
	 * This implementation always returns {@code false}.
	 * 可打开性总是返回false
	 */
	@Override
	public boolean isOpen() {
		return false;
	}

	/**
	 * This implementation throws a FileNotFoundException, assuming
	 * that the resource cannot be resolved to a URL.
	 * 不支持获取URL操作
	 */
	@Override
	public URL getURL() throws IOException {
		throw new FileNotFoundException(getDescription() + " cannot be resolved to URL");
	}

	/**
	 * This implementation builds a URI based on the URL returned
	 * by {@link #getURL()}.
	 * 获取URI,从URL中获取URI
	 */
	@Override
	public URI getURI() throws IOException {
		URL url = getURL();
		try {
			return ResourceUtils.toURI(url);
		}
		catch (URISyntaxException ex) {
			throw new NestedIOException("Invalid URI [" + url + "]", ex);
		}
	}

	/**
	 * This implementation throws a FileNotFoundException, assuming
	 * that the resource cannot be resolved to an absolute file path.
	 * 获取文件不支持
	 */
	@Override
	public File getFile() throws IOException {
		throw new FileNotFoundException(getDescription() + " cannot be resolved to absolute file path");
	}

	/**
	 * This implementation reads the entire InputStream to calculate the
	 * content length. Subclasses will almost always be able to provide
	 * a more optimal version of this, e.g. checking a File length.
	 * 获取整个资源输入流的可读内容长度，子类可以提供一个更优的方式检查文件可读内容长度。
	 * @see #getInputStream()
	 */
	@Override
	public long contentLength() throws IOException {
		InputStream is = getInputStream();
		Assert.state(is != null, "Resource InputStream must not be null");
		try {
			long size = 0;
			byte[] buf = new byte[255];
			int read;
			while ((read = is.read(buf)) != -1) {
				size += read;
			}
			return size;
		}
		finally {
			try {
				is.close();
			}
			catch (IOException ex) {
			}
		}
	}

	/**
	 * This implementation checks the timestamp of the underlying File,
	 * if available.
	 * 如果可用，检查底层文件的时间戳
	 * @see #getFileForLastModifiedCheck()
	 */
	@Override
	public long lastModified() throws IOException {
		//获取文件的上次修改的时间戳
		long lastModified = getFileForLastModifiedCheck().lastModified();
		if (lastModified == 0L) {
			throw new FileNotFoundException(getDescription() +
					" cannot be resolved in the file system for resolving its last-modified timestamp");
		}
		return lastModified;
	}

	/**
	 * Determine the File to use for timestamp checking.
	 * 获取文件时间戳检查的文件
	 * <p>The default implementation delegates to {@link #getFile()}.
	 * 默认的时间委托给{@link #getFile()}方法。
	 * @return the File to use for timestamp checking (never {@code null})
	 * @throws FileNotFoundException if the resource cannot be resolved as
	 * an absolute file path, i.e. is not available in a file system
	 * @throws IOException in case of general resolution/reading failures
	 */
	protected File getFileForLastModifiedCheck() throws IOException {
		return getFile();
	}

	/**
	 * This implementation throws a FileNotFoundException, assuming
	 * that relative resources cannot be created for this resource.
	 */
	@Override
	public Resource createRelative(String relativePath) throws IOException {
		throw new FileNotFoundException("Cannot create a relative resource for " + getDescription());
	}

	/**
	 * This implementation always returns {@code null},
	 * assuming that this resource type does not have a filename.
	 * 文件文件名，默认为空
	 */
	@Override
	public String getFilename() {
		return null;
	}


	/**
	 * This implementation returns the description of this resource.
	 * @see #getDescription()
	 */
	@Override
	public String toString() {
		return getDescription();
	}

	/**
	 * This implementation compares description strings.
	 * 根据资源描述判断两个资源对象是否相等
	 * @see #getDescription()
	 */
	@Override
	public boolean equals(Object obj) {
		return (obj == this ||
			(obj instanceof Resource && ((Resource) obj).getDescription().equals(getDescription())));
	}

	/**
	 * This implementation returns the description's hash code.
	 * 返回描述的的哈希值
	 * @see #getDescription()
	 */
	@Override
	public int hashCode() {
		return getDescription().hashCode();
	}

}
```
我们简单来看一下 获取URI操作：
```java
/**
 * This implementation builds a URI based on the URL returned
 * by {@link #getURL()}.
 * 获取URI,从URL中获取URI
 */
@Override
public URI getURI() throws IOException {
    URL url = getURL();
    try {
        return ResourceUtils.toURI(url);
    }
    catch (URISyntaxException ex) {
        throw new NestedIOException("Invalid URI [" + url + "]", ex);
    }
}
```
再来看ResourceUtils的toURI方法
```java
public abstract class ResourceUtils {
/**
	 * Create a URI instance for the given URL,
	 * replacing spaces with "%20" URI encoding first.
	 * 从给定的URL，创建一个URI实例，并使用"%20"，替代空格符。
	 * @param url the URL to convert into a URI instance
	 * @return the URI instance
	 * @throws URISyntaxException if the URL wasn't a valid URI
	 * @see java.net.URL#toURI()
	 */
	public static URI toURI(URL url) throws URISyntaxException {
		return toURI(url.toString());
	}

	/**
	 * Create a URI instance for the given location String,
	 * replacing spaces with "%20" URI encoding first.
	 * 根据给定的位置，建一个URI实例，并使用"%20"，替代空格符。
	 * @param location the location String to convert into a URI instance
	 * @return the URI instance
	 * @throws URISyntaxException if the location wasn't a valid URI
	 */
	public static URI toURI(String location) throws URISyntaxException {
		return new URI(StringUtils.replace(location, " ", "%20"));
	}
}
```

从上面可以看出，AbstractResource资源实现了资源的典型行为操作，判断资源是否存在操作，获取资源URI，获取资源内容大小，获取资源上次修改时间。
判断资源是否存在方法，将会先检查文件是否存在，如果文件不可打开，再检查输入流是否可以打开。获取资源URI方法，委托给 *ResourceUtils* 将资源的URL，
转化为URI。获取资源内容大小操作，即读取文件字节内容直到不可读，子类的提供更优的实现。获取资源上次修改时间，实际上是获取资源文件的上次修改时间。
由于AbstractResource描述的是一个抽象资源，牵涉到底层资源的方法isOpen、getURL、getFile，要么是不支持，要么false，要么为空，这些待具体的资源实现。


来看抽象文件资源AbstractFileResolvingResource

### AbstractFileResolvingResource


源码参见：[AbstractFileResolvingResource][]

[AbstractFileResolvingResource]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-core/src/main/java/org/springframework/core/io/AbstractFileResolvingResource.java "AbstractFileResolvingResource"

我们先来看一下获取文件方法

```java
package org.springframework.core.io;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.URI;
import java.net.URL;
import java.net.URLConnection;

import org.springframework.util.ResourceUtils;

/**
 * Abstract base class for resources which resolve URLs into File references,
 * such as {@link UrlResource} or {@link ClassPathResource}.
 *AbstractFileResolvingResource为资源抽象基类，可以解决URL到文件的引用，比如UrlResource和ClassPathResource。
 * <p>Detects the "file" protocol as well as the JBoss "vfs" protocol in URLs,
 * resolving file system references accordingly.
 *
 * @author Juergen Hoeller
 * @since 3.0
 */
public abstract class AbstractFileResolvingResource extends AbstractResource {

	/**
	 * This implementation returns a File reference for the underlying class path
	 * resource, provided that it refers to a file in the file system.
	 * 返回底层类路径资源的文件引用，参考于文件系统中文件。
	 * @see org.springframework.util.ResourceUtils#getFile(java.net.URL, String)
	 */
	@Override
	public File getFile() throws IOException {
		URL url = getURL();//获取资源URL
		if (url.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
			return VfsResourceDelegate.getResource(url).getFile();
		}
		//根据资源URL和描述获取底层资源文件
		return ResourceUtils.getFile(url, getDescription());
	}
}
```

我们先来看VFS文件获取，然后再看文件系统资源获取

VfsResourceDelegate

```java
/**
	 * Inner delegate class, avoiding a hard JBoss VFS API dependency at runtime.
	 * 内部代理类，避免Jboss VFS API 运行时环境下的硬依赖。
	 */
	private static class VfsResourceDelegate {

		public static Resource getResource(URL url) throws IOException {
			return new VfsResource(VfsUtils.getRoot(url));
		}

		public static Resource getResource(URI uri) throws IOException {
			return new VfsResource(VfsUtils.getRoot(uri));
		}
	}
```

再来看VfsResource

```java
package org.springframework.core.io;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.net.URI;
import java.net.URL;

import org.springframework.core.NestedIOException;
import org.springframework.util.Assert;

/**
 * JBoss VFS based {@link Resource} implementation.
 * JBoss VFS 文件资源实现
 * <p>As of Spring 4.0, this class supports VFS 3.x on JBoss AS 6+ (package
 * {@code org.jboss.vfs}) and is in particular compatible with JBoss AS 7 and
 * WildFly 8.
 * 从spring4.0开始，此类支持VFS 3.x on JBoss AS 6+ (package
 * {@code org.jboss.vfs})，在一些特性上兼容JBoss AS 7 and
 * WildFly 8.
 *
 *VfsResource的打开文件获取URL，获取输入流等方法，实际为通过VfsUtils调用org.jboss.vfs.VirtualFile的方法。
 * @author Juergen Hoeller
 * @author Costin Leau
 * @author Sam Brannen
 * @since 3.0
 * @see org.jboss.vfs.VirtualFile
 */
public class VfsResource extends AbstractResource {

	private final Object resource;//资源对象


	public VfsResource(Object resource) {
		Assert.notNull(resource, "VirtualFile must not be null");
		this.resource = resource;
	}
    /**
     *
     * 实际委托给VfsUtils打开文件返回输入流，VfsUtils则是调用org.jboss.vfs.VirtualFile的getPhysicalFile
     */
 	@Override
 	public File getFile() throws IOException {
 		return VfsUtils.getFile(this.resource);
 	}

    /**
     * 实际委托给VfsUtils打开文件返回输入流，VfsUtils则是调用org.jboss.vfs.VirtualFile的openStream
     */
	@Override
	public InputStream getInputStream() throws IOException {
		return VfsUtils.getInputStream(this.resource);
	}
}
```
往下走，再看VfsUtils


```java
import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.net.URI;
import java.net.URL;

import org.springframework.util.ReflectionUtils;

/**
 * Utility for detecting and accessing JBoss VFS in the classpath.
 *
 * <p>As of Spring 4.0, this class supports VFS 3.x on JBoss AS 6+ (package
 * {@code org.jboss.vfs}) and is in particular compatible with JBoss AS 7 and
 * WildFly 8.
 *
 * <p>Thanks go to Marius Bogoevici for the initial patch.
 * <b>Note:</b> This is an internal class and should not be used outside the framework.
 *
 * @author Costin Leau
 * @author Juergen Hoeller
 * @since 3.0.3
 */
public abstract class VfsUtils {

	private static final String VFS3_PKG = "org.jboss.vfs.";//Jboos，vfs包名
	private static final String VFS_NAME = "VFS";

	private static Method VFS_METHOD_GET_ROOT_URL;
	private static Method VFS_METHOD_GET_ROOT_URI;

	private static Method VIRTUAL_FILE_METHOD_EXISTS;
	private static Method VIRTUAL_FILE_METHOD_GET_INPUT_STREAM;//VirtualFile的打开输入流方法
	private static Method VIRTUAL_FILE_METHOD_GET_SIZE;//获取文件size方法
	private static Method VIRTUAL_FILE_METHOD_GET_LAST_MODIFIED;//获取文件上次修改时间戳方法
	private static Method VIRTUAL_FILE_METHOD_TO_URL;//转换资源为URL方法
	private static Method VIRTUAL_FILE_METHOD_TO_URI;//转换资源为URI方法
	private static Method VIRTUAL_FILE_METHOD_GET_NAME;//获取文件名方法
	private static Method VIRTUAL_FILE_METHOD_GET_PATH_NAME;//获取文件路径方法
	private static Method VIRTUAL_FILE_METHOD_GET_CHILD;//获取资源的孩子文件

	protected static Class<?> VIRTUAL_FILE_VISITOR_INTERFACE;
	protected static Method VIRTUAL_FILE_METHOD_VISIT;

	private static Field VISITOR_ATTRIBUTES_FIELD_RECURSE;
	private static Method GET_PHYSICAL_FILE;

	static {
		//获取VfsUtils的类加载器
		ClassLoader loader = VfsUtils.class.getClassLoader();
		try {
			Class<?> vfsClass = loader.loadClass(VFS3_PKG + VFS_NAME);
			VFS_METHOD_GET_ROOT_URL = ReflectionUtils.findMethod(vfsClass, "getChild", URL.class);
			VFS_METHOD_GET_ROOT_URI = ReflectionUtils.findMethod(vfsClass, "getChild", URI.class);
           //加载Jboss的VirtualFile类
			Class<?> virtualFile = loader.loadClass(VFS3_PKG + "VirtualFile");
			//初始化VirtualFile类的相关方法
			VIRTUAL_FILE_METHOD_EXISTS = ReflectionUtils.findMethod(virtualFile, "exists");
			VIRTUAL_FILE_METHOD_GET_INPUT_STREAM = ReflectionUtils.findMethod(virtualFile, "openStream");
			VIRTUAL_FILE_METHOD_GET_SIZE = ReflectionUtils.findMethod(virtualFile, "getSize");
			VIRTUAL_FILE_METHOD_GET_LAST_MODIFIED = ReflectionUtils.findMethod(virtualFile, "getLastModified");
			VIRTUAL_FILE_METHOD_TO_URI = ReflectionUtils.findMethod(virtualFile, "toURI");
			VIRTUAL_FILE_METHOD_TO_URL = ReflectionUtils.findMethod(virtualFile, "toURL");
			VIRTUAL_FILE_METHOD_GET_NAME = ReflectionUtils.findMethod(virtualFile, "getName");
			VIRTUAL_FILE_METHOD_GET_PATH_NAME = ReflectionUtils.findMethod(virtualFile, "getPathName");
			GET_PHYSICAL_FILE = ReflectionUtils.findMethod(virtualFile, "getPhysicalFile");
			VIRTUAL_FILE_METHOD_GET_CHILD = ReflectionUtils.findMethod(virtualFile, "getChild", String.class);

			VIRTUAL_FILE_VISITOR_INTERFACE = loader.loadClass(VFS3_PKG + "VirtualFileVisitor");
			VIRTUAL_FILE_METHOD_VISIT = ReflectionUtils.findMethod(virtualFile, "visit", VIRTUAL_FILE_VISITOR_INTERFACE);

			Class<?> visitorAttributesClass = loader.loadClass(VFS3_PKG + "VisitorAttributes");
			VISITOR_ATTRIBUTES_FIELD_RECURSE = ReflectionUtils.findField(visitorAttributesClass, "RECURSE");
		}
		catch (Throwable ex) {
			throw new IllegalStateException("Could not detect JBoss VFS infrastructure", ex);
		}
	}
    /**
	 * 获取文件，调用VirtualFile的getPhysicalFile
	 * @param vfsResource
	 * @return
	 * @throws IOException
	 */
	static File getFile(Object vfsResource) throws IOException {
		return (File) invokeVfsMethod(GET_PHYSICAL_FILE, vfsResource);
	}

	/**
	 * 获取URI
	 * @param url
	 * @return
	 * @throws IOException
	 */
	static Object getRoot(URI url) throws IOException {
		return invokeVfsMethod(VFS_METHOD_GET_ROOT_URI, null, url);
	}

	// protected methods used by the support sub-package

	/**
	 * 获取URL
	 * @param url
	 * @return
	 * @throws IOException
	 */
	protected static Object getRoot(URL url) throws IOException {
		return invokeVfsMethod(VFS_METHOD_GET_ROOT_URL, null, url);
	}
    /**
	 * 获取文件输入流，调用VirtualFile的openStream
	 * @param vfsResource
	 * @return
	 * @throws IOException
	 */
	static InputStream getInputStream(Object vfsResource) throws IOException {
		return (InputStream) invokeVfsMethod(VIRTUAL_FILE_METHOD_GET_INPUT_STREAM, vfsResource);
	}
    ...
}
```
 从VfsUtils的源码可以看出，VfsUtils实际是通过反射调用org.jboss.vfs.VirtualFile的相应方法。

 VFS文件获取实际是委托给VfsResourceDelegate，VfsResourceDelegate代理的是JBoos的VFS资源VfsResource，
 VfsResource内部关联一个底层资源对象，VfsResource的所有关于Resource的操作实际上委托给VfsUtils，VfsUtils实际是通过反射调用org.jboss.vfs.VirtualFile的相应方法。

 再看获取文件系统中的资源ResourceUtils
 ```java
 /**
 	 * Resolve the given resource URL to a {@code java.io.File},
 	 * i.e. to a file in the file system.
 	 * 解决给定资源的URL为 {@code java.io.File}。
 	 * @param resourceUrl the resource URL to resolve
 	 * @param description a description of the original resource that
 	 * the URL was created for (for example, a class path location)
 	 * @return a corresponding File object
 	 * @throws FileNotFoundException if the URL cannot be resolved to
 	 * a file in the file system
 	 */
 	public static File getFile(URL resourceUrl, String description) throws FileNotFoundException {
 		Assert.notNull(resourceUrl, "Resource URL must not be null");
 		//如果为非文件系统，则抛出异常
 		if (!URL_PROTOCOL_FILE.equals(resourceUrl.getProtocol())) {
 			throw new FileNotFoundException(
 					description + " cannot be resolved to absolute file path " +
 					"because it does not reside in the file system: " + resourceUrl);
 		}
 		try {
 			//否则，根据资源的URI创建文件
 			return new File(toURI(resourceUrl).getSchemeSpecificPart());
 		}
 		catch (URISyntaxException ex) {
 			// Fallback for URLs that are not valid URIs (should hardly ever happen).
 			//如果URI资源语义异常，根据URL的文件路径创建文件
 			return new File(resourceUrl.getFile());
 		}
}
 ```

从上面可以看出，获取文件操作，首先检查文件是否为JBOSS的VFS文件，如果是则VFS文件获取委托给VfsResourceDelegate，VfsResourceDelegate代理的是JBoos的VFS资源VfsResource，
VfsResource内部关联一个底层资源对象，VfsResource的所有关于Resource的操作实际上委托给VfsUtils，VfsUtils实际是通过反射调用org.jboss.vfs.VirtualFile的相应方法。
否则委托给ResourceUtils获取文件系统中的文件资源。

再来看根据URI获取文件资源
```java
/**
 * This implementation returns a File reference for the given URI-identified
 * resource, provided that it refers to a file in the file system.
 * 返回给定URI资源关联的文件应用，参考与文件系统中的文件。
 * @see org.springframework.util.ResourceUtils#getFile(java.net.URI, String)
 */
protected File getFile(URI uri) throws IOException {
    //如果是Jboos文件，则委托给VfsResourceDelegate
    if (uri.getScheme().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
        return VfsResourceDelegate.getResource(uri).getFile();
    }
    //否则而根据URI和资源描述，获取文件
    return ResourceUtils.getFile(uri, getDescription());
}
```
从上可以看出，根据URI获取文件资源思路与获取文件相同。

再来看获取上次修改文件方法
```java
/**
	 * This implementation determines the underlying File
	 * (or jar file, in case of a resource in a jar/zip).
	 */
	@Override
	protected File getFileForLastModifiedCheck() throws IOException {
		URL url = getURL();//获取资源URL
		if (ResourceUtils.isJarURL(url)) {//如果资源URL是Jar形式
			//从存档文件中抽取存档在文件系统中URL
			URL actualUrl = ResourceUtils.extractArchiveURL(url);
			//如果资源为Jboss的VFS文件资源，委托给VfsResourceDelegate代理获取资源文件
			if (actualUrl.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
				return VfsResourceDelegate.getResource(actualUrl).getFile();
			}
			//根据资源URL和描述获取底层资源文件
			return ResourceUtils.getFile(actualUrl, "Jar URL");
		}
		else {
			//否则直接委托给getFile
			return getFile();
		}
	}
```

再来看ResourceUtils的URL类型判断和从存档文件中抽取存档在文件系统中URL
```java
/**
	 * Determine whether the given URL points to a resource in a jar file.
	 * i.e. has protocol "jar", "war, ""zip", "vfszip" or "wsjar".
	 * 判断给定的URL指定的资源是否在jar包文件中
	 * @param url the URL to check
	 * @return whether the URL has been identified as a JAR URL
	 */
	public static boolean isJarURL(URL url) {
		String protocol = url.getProtocol();
		return (URL_PROTOCOL_JAR.equals(protocol) || URL_PROTOCOL_WAR.equals(protocol) ||
				URL_PROTOCOL_ZIP.equals(protocol) || URL_PROTOCOL_VFSZIP.equals(protocol) ||
				URL_PROTOCOL_WSJAR.equals(protocol));
	}
    /**
	 * Extract the URL for the outermost archive from the given jar/war URL
	 * (which may point to a resource in a jar file or to a jar file itself).
	 * 从给定的最外层的Jar或War URL 中抽取出，抽取内部的文件系统URL（可以指向jar包中的文件，或者一个jar文件）。
	 * <p>In the case of a jar file nested within a war file, this will return
	 * a URL to the war file since that is the one resolvable in the file system.
	 * 在jar嵌入war包中的情况，由于war在文件系统中是一种解决的URL，所以方法返回的为War的URL
	 * @param jarUrl the original URL
	 * @return the URL for the actual jar file
	 * @throws MalformedURLException if no valid jar file URL could be extracted
	 * @since 4.1.8
	 * @see #extractJarFileURL(URL)
	 */
	public static URL extractArchiveURL(URL jarUrl) throws MalformedURLException {
		String urlFile = jarUrl.getFile();

		int endIndex = urlFile.indexOf(WAR_URL_SEPARATOR);//如果是war资源文件
		if (endIndex != -1) {
			// Tomcat's "war:file:...mywar.war*/WEB-INF/lib/myjar.jar!/myentry.txt"
			String warFile = urlFile.substring(0, endIndex);//war:file:...mywar.war
			//如果是war包
			if (URL_PROTOCOL_WAR.equals(jarUrl.getProtocol())) {
				return new URL(warFile);
			}
			int startIndex = warFile.indexOf(WAR_URL_PREFIX);
			if (startIndex != -1) {//file:...mywar.war
				return new URL(warFile.substring(startIndex + WAR_URL_PREFIX.length()));
			}
		}
        //否则从jar中抽出资源文件
		// Regular "jar:file:...myjar.jar!/myentry.txt"
		return extractJarFileURL(jarUrl);
	}
    /**
	 * Extract the URL for the actual jar file from the given URL
	 * (which may point to a resource in a jar file or to a jar file itself).
	 * 从给定的URL中抽取实际的jar文件的URL
	 * @param jarUrl the original URL
	 * @return the URL for the actual jar file
	 * @throws MalformedURLException if no valid jar file URL could be extracted
	 */
	public static URL extractJarFileURL(URL jarUrl) throws MalformedURLException {
		// Regular "jar:file:...myjar.jar!/myentry.txt"
		String urlFile = jarUrl.getFile();
		int separatorIndex = urlFile.indexOf(JAR_URL_SEPARATOR);
		if (separatorIndex != -1) {
			String jarFile = urlFile.substring(0, separatorIndex);//jar:file:...myjar.jar
			try {
				return new URL(jarFile);
			}
			catch (MalformedURLException ex) {
				// Probably no protocol in original jar URL, like "jar:C:/mypath/myjar.jar".
				// This usually indicates that the jar file resides in the file system.
				//有可能在原始Jar包URL中不存在协议，如："jar:C:/mypath/myjar.jar".
				//则通常意味着jar包文件存在于文件系统中。
				if (!jarFile.startsWith("/")) {
					jarFile = "/" + jarFile;
				}
				return new URL(FILE_URL_PREFIX + jarFile);
			}
		}
		else {
			return jarUrl;
		}
	}
```
从上可以看出获取上次修改文件方法，如果是jar包文件，则委托给ResourceUtils，从给定的最外层的Jar或War URL 中抽取出，抽取内部的文件系统URL（可以指向jar包中的文件，或者一个jar文件）。
如果URL资源为VFS，则委托给VfsResourceDelegate，否则委托给ResourceUtils的获取文件方法。



再来看文件是否可读
```java
@Override
	public boolean isReadable() {
		try {
			URL url = getURL();
			//如果文件系统资源文件，如果文件可读且不是目录，则返回true
			if (ResourceUtils.isFileURL(url)) {
				// Proceed with file system resolution
				File file = getFile();
				return (file.canRead() && !file.isDirectory());
			}
			else {
				return true;
			}
		}
		catch (IOException ex) {
			return false;
		}
	}
//ResourceUtils
    /**
    	 * Determine whether the given URL points to a resource in the file system,
    	 * i.e. has protocol "file", "vfsfile" or "vfs".
    	 * 判断给定URL是否指向的文件系统资源，比如协议"file", "vfsfile" or "vfs"
    	 * @param url the URL to check
    	 * @return whether the URL has been identified as a file system URL
    	 */
    	public static boolean isFileURL(URL url) {
    		String protocol = url.getProtocol();
    		return (URL_PROTOCOL_FILE.equals(protocol) || URL_PROTOCOL_VFSFILE.equals(protocol) ||
    				URL_PROTOCOL_VFS.equals(protocol));
    	}
```
从上可以看出，判断文件是否可读，主要是判断文件是否存在，同时为非目录。

再来看文件是否存在方法
```java
@Override
	public boolean exists() {
		try {
			URL url = getURL();
			if (ResourceUtils.isFileURL(url)) {
				// Proceed with file system resolution
				//如果是文件系统，则委托给文件的exists的方法。
				return getFile().exists();
			}
			else {
				// Try a URL connection content-length header
				//打开URL连接
				URLConnection con = url.openConnection();
				customizeConnection(con);//设置请求方法，及是否使用缓存
				HttpURLConnection httpCon =
						(con instanceof HttpURLConnection ? (HttpURLConnection) con : null);
				if (httpCon != null) {
					int code = httpCon.getResponseCode();
					if (code == HttpURLConnection.HTTP_OK) {
						return true;
					}
					else if (code == HttpURLConnection.HTTP_NOT_FOUND) {
						return false;
					}
				}
				//则检查内容大小
				if (con.getContentLength() >= 0) {
					return true;
				}
				if (httpCon != null) {
					// no HTTP OK status, and no content-length header: give up
					//没有HTTP OK状态，且没有内容长度头，则放弃
					httpCon.disconnect();
					return false;
				}
				else {
					// Fall back to stream existence: can we open the stream?
					//最后降级为输入流是否可以打开
					InputStream is = getInputStream();
					is.close();
					return true;
				}
			}
		}
		catch (IOException ex) {
			return false;
		}
	}
    /**
    	 * Customize the given {@link URLConnection}, obtained in the course of an
    	 * {@link #exists()}, {@link #contentLength()} or {@link #lastModified()} call.
    	 * <p>Calls {@link ResourceUtils#useCachesIfNecessary(URLConnection)} and
    	 * delegates to {@link #customizeConnection(HttpURLConnection)} if possible.
    	 * 根据给定的HttpURLConnection，获取 {@link #exists()}, {@link #contentLength()} 或 {@link #lastModified()}
    	 * 方法的调用。同时调用{@link ResourceUtils#useCachesIfNecessary(URLConnection)}方法设置是否使用缓存，
    	 * 设置头部方法。
    	 * Can be overridden in subclasses.
    	 * @param con the URLConnection to customize
    	 * @throws IOException if thrown from URLConnection methods
    	 */
    	protected void customizeConnection(URLConnection con) throws IOException {
    		ResourceUtils.useCachesIfNecessary(con);
    		if (con instanceof HttpURLConnection) {//如果资源为网络资源，则设置请求方法
    			customizeConnection((HttpURLConnection) con);
    		}
    	}

    	/**
    	 * Customize the given {@link HttpURLConnection}, obtained in the course of an
    	 * {@link #exists()}, {@link #contentLength()} or {@link #lastModified()} call.
    	 * <p>Sets request method "HEAD" by default. Can be overridden in subclasses.
    	 * 根据给定的HttpURLConnection，获取 {@link #exists()}, {@link #contentLength()} 或 {@link #lastModified()}
    	 * 方法的调用，默认请求方法为HEAD，子类可以重写。
    	 * @param con the HttpURLConnection to customize
    	 * @throws IOException if thrown from HttpURLConnection methods
    	 */
    	protected void customizeConnection(HttpURLConnection con) throws IOException {
    		con.setRequestMethod("HEAD");
    	}
```

从上面可以看出，检查文件是否存在，如果是文件系统，则委托给文件的exists的方法，否则URLConnection，如果URLConnection为HttpURLConnection，
获取HttpURLConnection的转台，OK则存在，否则false。


再来其他方法
```java
@Override
	public long contentLength() throws IOException {
		URL url = getURL();
		//如果是文件系统资源文件，则获取文件内容长度
		if (ResourceUtils.isFileURL(url)) {
			// Proceed with file system resolution
			return getFile().length();
		}
		else {
			// Try a URL connection content-length header
			//否则获取URL连接的内容长度头
			URLConnection con = url.openConnection();
			customizeConnection(con);
			return con.getContentLength();
		}
	}

	@Override
	public long lastModified() throws IOException {
		URL url = getURL();
		//如果文件系统资源文件或者jar包文件，则资源文件的上次修改时间戳
		if (ResourceUtils.isFileURL(url) || ResourceUtils.isJarURL(url)) {
			// Proceed with file system resolution
			try {
				return super.lastModified();
			}
			catch (FileNotFoundException ex) {
				// Defensively fall back to URL connection check instead
			}
		}
		// Try a URL connection last-modified header
		//否则返回URL资源的上次修改时间
		URLConnection con = url.openConnection();
		customizeConnection(con);
		return con.getLastModified();
	}
```
获取资源内容长度和上次修改时间与判断文件是否存在的思想基本相同。

先来小节一下：

AbstractFileResolvingResource获取文件操作，首先检查文件是否为JBOSS的VFS文件，如果是则VFS文件获取委托给VfsResourceDelegate，VfsResourceDelegate代理的是JBoos的VFS资源VfsResource，VfsResource内部关联一个底层资源对象，VfsResource的所有关于Resource的操作实际上委托给VfsUtils，VfsUtils实际是通过反射调用org.jboss.vfs.VirtualFile的相应方法。
否则委托给ResourceUtils获取文件系统中的文件资源。根据URI获取文件资源思路与获取文件相同。获取上次修改文件方法，如果是jar包文件，则委托给ResourceUtils，从给定的最外层的Jar或War URL 中抽取出，抽取内部的文件系统URL（可以指向jar包中的文件，或者一个jar文件）。如果URL资源为VFS，则委托给VfsResourceDelegate，否则委托给ResourceUtils的获取文件方法。判断文件是否可读，主要是判断文件是否存在，同时为非目录。检查文件是否存在，如果是文件系统，则委托给文件的exists的方法，否则URLConnection，如果URLConnection为HttpURLConnection，
获取HttpURLConnection的转台，OK则存在，否则false。获取资源内容长度和上次修改时间与判断文件是否存在的思想基本相同。

下面这3小节我们放在下一篇在讲。

### ClassPathResource

### ClassPathContextResource

### UrlResource

最后我们以AbstractFileResolvingResource的类图结束这篇文章。
![AbstractFileResolvingResource](/image/spring-context/AbstractFileResolvingResource.png)


## 总结

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

## 附
```java
public abstract class ResourceUtils {

	/** Pseudo URL prefix for loading from the class path: "classpath:" 从类路径加载的伪URL前缀*/
	public static final String CLASSPATH_URL_PREFIX = "classpath:";

	/** URL prefix for loading from the file system: "file:" 从文件系统加载的URL前缀*/
	public static final String FILE_URL_PREFIX = "file:";

	/** URL prefix for loading from a jar file: "jar:" 从jar包加载的URL前缀*/
	public static final String JAR_URL_PREFIX = "jar:";

	/** URL prefix for loading from a war file on Tomcat: "war:" 从Tomcat war包加载的URL前缀*/
	public static final String WAR_URL_PREFIX = "war:";

	/** URL protocol for a file in the file system: "file" 文件系统中的文件URL协议*/
	public static final String URL_PROTOCOL_FILE = "file";

	/** URL protocol for an entry from a jar file: "jar" jar包文件的URL协议*/
	public static final String URL_PROTOCOL_JAR = "jar";

	/** URL protocol for an entry from a war file: "war" war包文件的URL协议*/
	public static final String URL_PROTOCOL_WAR = "war";

	/** URL protocol for an entry from a zip file: "zip" 压缩文件zip的URL协议*/
	public static final String URL_PROTOCOL_ZIP = "zip";

	/** URL protocol for an entry from a WebSphere jar file: "wsjar" WebSphere jar包文件的URL协议*/
	public static final String URL_PROTOCOL_WSJAR = "wsjar";

	/** URL protocol for an entry from a JBoss jar file: "vfszip" JBoss jar包文件的URL协议*/
	public static final String URL_PROTOCOL_VFSZIP = "vfszip";

	/** URL protocol for a JBoss file system resource: "vfsfile" JBoss 文件系统资源的URL协议*/
	public static final String URL_PROTOCOL_VFSFILE = "vfsfile";

	/** URL protocol for a general JBoss VFS resource: "vfs" JBoss VFS资源的URL协议*/
	public static final String URL_PROTOCOL_VFS = "vfs";

	/** File extension for a regular jar file: ".jar" jar包文件拓展名*/
	public static final String JAR_FILE_EXTENSION = ".jar";

	/** Separator between JAR URL and file path within the JAR: "!/"  在jar包中的jar URL与文件路径的分割符*/
	public static final String JAR_URL_SEPARATOR = "!/";

	/** Special separator between WAR URL and jar part on Tomcat 在Tomcat中，war资源URL与jar的分割符*/
	public static final String WAR_URL_SEPARATOR = "*/";


	/**
	 * Return whether the given resource location is a URL:
	 * either a special "classpath" pseudo URL or a standard URL.
	 * 判断给定资源位置是否为一个URL，一个标准的RUL或者 "classpath"伪URL
	 * @param resourceLocation the location String to check
	 * @return whether the location qualifies as a URL
	 * @see #CLASSPATH_URL_PREFIX
	 * @see java.net.URL
	 */
	public static boolean isUrl(String resourceLocation) {
		if (resourceLocation == null) {
			return false;
		}
		if (resourceLocation.startsWith(CLASSPATH_URL_PREFIX)) {
			return true;
		}
		try {
			new URL(resourceLocation);
			return true;
		}
		catch (MalformedURLException ex) {
			return false;
		}
	}
}
```
