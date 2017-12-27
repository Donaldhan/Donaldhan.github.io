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
package org.springframework.core.io;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.net.URL;

import org.springframework.util.Assert;
import org.springframework.util.ClassUtils;
import org.springframework.util.ObjectUtils;
import org.springframework.util.StringUtils;

/**
 * {@link Resource} implementation for class path resources. Uses either a
 * given {@link ClassLoader} or a given {@link Class} for loading resources.
 *ClassPathResource是资源类路径资源的实现。从给定的类加载器或类加载资源。
 * <p>Supports resolution as {@code java.io.File} if the class path
 * resource resides in the file system, but not for resources in a JAR.
 * Always supports resolution as URL.
 *如果类路径资源在文件系统中，可以转换为{@code java.io.File}，jar包中的资源不包括，总是支持URL方法的解决方法。
 * @author Juergen Hoeller
 * @author Sam Brannen
 * @since 28.12.2003
 * @see ClassLoader#getResourceAsStream(String)
 * @see Class#getResourceAsStream(String)
 */
public class ClassPathResource extends AbstractFileResolvingResource {

	private final String path;//资源路径

	private ClassLoader classLoader;//资源类加载器

	private Class<?> clazz;//资源类。


	/**
	 * Create a new {@code ClassPathResource} for {@code ClassLoader} usage.
	 * A leading slash will be removed, as the ClassLoader resource access
	 * methods will not accept it.
	 * <p>The thread context class loader will be used for
	 * loading the resource.
	 * 使用类加载器创建一个类路径资源。leading slash‘/’，将会被移除，类加载器资源访问方法不接受leading slash。
	 * 当前线程上下文类加载器将被用于加载资源。
	 * @param path the absolute path within the class path
	 * @see java.lang.ClassLoader#getResourceAsStream(String)
	 * @see org.springframework.util.ClassUtils#getDefaultClassLoader()
	 */
	public ClassPathResource(String path) {
		this(path, (ClassLoader) null);
	}

	/**
	 * Create a new {@code ClassPathResource} for {@code ClassLoader} usage.
	 * A leading slash will be removed, as the ClassLoader resource access
	 * methods will not accept it.
	 * 使用类加载器创建一个类路径资源。leading slash‘/’，将会被移除，类加载器资源访问方法不接受leading slash。
	 * 当前线程上下文类加载器将被用于加载资源。
	 * @param path the absolute path within the classpath
	 * @param classLoader the class loader to load the resource with,
	 * or {@code null} for the thread context class loader
	 * @see ClassLoader#getResourceAsStream(String)
	 */
	public ClassPathResource(String path, ClassLoader classLoader) {
		Assert.notNull(path, "Path must not be null");
		//清理路径，替换window文件夹分割符为统一文件夹分割符，剔除上次目录符‘..’和当前目录符‘.’
		String pathToUse = StringUtils.cleanPath(path);
		//剔除leading slash‘/’，绝对路径开始符
		if (pathToUse.startsWith("/")) {
			pathToUse = pathToUse.substring(1);
		}
		this.path = pathToUse;
		this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
	}

	/**
	 * Create a new {@code ClassPathResource} for {@code Class} usage.
	 * The path can be relative to the given class, or absolute within
	 * the classpath via a leading slash.
	 * 根据给定的资源路径和资源类创建类路径资源ClassPathResource
	 * @param path relative or absolute path within the class path
	 * @param clazz the class to load resources with
	 * @see java.lang.Class#getResourceAsStream
	 */
	public ClassPathResource(String path, Class<?> clazz) {
		Assert.notNull(path, "Path must not be null");
		this.path = StringUtils.cleanPath(path);
		this.clazz = clazz;
	}

	/**
	 * Create a new {@code ClassPathResource} with optional {@code ClassLoader}
	 * 根据资路径，类加载器，资源类，创建类路径资源ClassPathResource
	 * and {@code Class}. Only for internal usage.
	 * @param path relative or absolute path within the classpath
	 * @param classLoader the class loader to load the resource with, if any
	 * @param clazz the class to load resources with, if any
	 */
	protected ClassPathResource(String path, ClassLoader classLoader, Class<?> clazz) {
		this.path = StringUtils.cleanPath(path);
		this.classLoader = classLoader;
		this.clazz = clazz;
	}
}
```
从上面来看，ClassPathResource内部有3变量，一个为类资源路径path（String），一个类机载器classLoader（ClassLoader），一个为资源类clazz（Class<?> ），
同时提供根据3个内部变量构成类路径资源的构造。

再来看获取类路径资源URL
```java
/**
	 * This implementation returns a URL for the underlying class path resource,
	 * if available.
	 * 如果可用，获取底层的clas路径资源的URL
	 * @see java.lang.ClassLoader#getResource(String)
	 * @see java.lang.Class#getResource(String)
	 */
	@Override
	public URL getURL() throws IOException {
		URL url = resolveURL();
		if (url == null) {
			throw new FileNotFoundException(getDescription() + " cannot be resolved to URL because it does not exist");
		}
		return url;
	}
	/**
	 * Resolves a URL for the underlying class path resource.
	 * @return the resolved URL, or {@code null} if not resolvable
	 */
	protected URL resolveURL() {
		if (this.clazz != null) {
			//如果资源类不为空，从资源类的类加载器获取资源
			return this.clazz.getResource(this.path);
		}
		else if (this.classLoader != null) {
			//如果类加载器不为空，从类加载器加载资源
			return this.classLoader.getResource(this.path);
		}
		else {
			//否则从系统类加载器加载资源
			return ClassLoader.getSystemResource(this.path);
		}
	}
```
解决URL方法有3点要看：
1.
```java
if (this.clazz != null) {
    //如果资源类不为空，从资源类的类加载器获取资源
    return this.clazz.getResource(this.path);
}
```
2.
```java
else if (this.classLoader != null) {
    //如果类加载器不为空，从类加载器加载资源
    return this.classLoader.getResource(this.path);
}
```
3.
```java
else {
    //否则从系统类加载器加载资源
    return ClassLoader.getSystemResource(this.path);
}
```
下面我们分别来看上述三点
1.
```java
if (this.clazz != null) {
    //如果资源类不为空，从资源类的类加载器获取资源
    return this.clazz.getResource(this.path);
}
```

//Class

```java
/**
    * Finds a resource with a given name.  The rules for searching resources
    * associated with a given class are implemented by the defining
    * {@linkplain ClassLoader class loader} of the class.  This method
    * delegates to this object's class loader.  If this object was loaded by
    * the bootstrap class loader, the method delegates to {@link
    * ClassLoader#getSystemResource}.
    *
    * <p> Before delegation, an absolute resource name is constructed from the
    * given resource name using this algorithm:
    *
    * <ul>
    *
    * <li> If the {@code name} begins with a {@code '/'}
    * (<tt>'&#92;u002f'</tt>), then the absolute name of the resource is the
    * portion of the {@code name} following the {@code '/'}.
    *
    * <li> Otherwise, the absolute name is of the following form:
    *
    * <blockquote>
    *   {@code modified_package_name/name}
    * </blockquote>
    *
    * <p> Where the {@code modified_package_name} is the package name of this
    * object with {@code '/'} substituted for {@code '.'}
    * (<tt>'&#92;u002e'</tt>).
    *
    * </ul>
    *
    * @param  name name of the desired resource
    * @return      A  {@link java.net.URL} object or {@code null} if no
    *              resource with this name is found
    * @since  JDK1.1
    */
   public java.net.URL getResource(String name) {
       name = resolveName(name);
       //获取当前类加载器器
       ClassLoader cl = getClassLoader0();
       if (cl==null) {
           // A system class.
           //从系统加载器加载
           return ClassLoader.getSystemResource(name);
       }
       return cl.getResource(name);
   }
   // Package-private to allow ClassLoader access
   ClassLoader getClassLoader0() { return classLoader; }

   // Initialized in JVM not by private constructor
   // This field is filtered from reflection access, i.e. getDeclaredField
   // will throw NoSuchFieldException
   private final ClassLoader classLoader;

```

//ClassLoader
```java
/**
     * Find a resource of the specified name from the search path used to load
     * classes.  This method locates the resource through the system class
     * loader (see {@link #getSystemClassLoader()}).
     *
     * @param  name
     *         The resource name
     *
     * @return  A {@link java.net.URL <tt>URL</tt>} object for reading the
     *          resource, or <tt>null</tt> if the resource could not be found
     *
     * @since  1.1
     */
    public static URL getSystemResource(String name) {
        ClassLoader system = getSystemClassLoader();
        if (system == null) {
            return getBootstrapResource(name);
        }
        return system.getResource(name);
    }
    /**
     * Returns the system class loader for delegation.  This is the default
     * delegation parent for new <tt>ClassLoader</tt> instances, and is
     * typically the class loader used to start the application.
     *
     * <p> This method is first invoked early in the runtime's startup
     * sequence, at which point it creates the system class loader and sets it
     * as the context class loader of the invoking <tt>Thread</tt>.
     *
     * <p> The default system class loader is an implementation-dependent
     * instance of this class.
     *
     * <p> If the system property "<tt>java.system.class.loader</tt>" is defined
     * when this method is first invoked then the value of that property is
     * taken to be the name of a class that will be returned as the system
     * class loader.  The class is loaded using the default system class loader
     * and must define a public constructor that takes a single parameter of
     * type <tt>ClassLoader</tt> which is used as the delegation parent.  An
     * instance is then created using this constructor with the default system
     * class loader as the parameter.  The resulting class loader is defined
     * to be the system class loader.
     *
     * <p> If a security manager is present, and the invoker's class loader is
     * not <tt>null</tt> and the invoker's class loader is not the same as or
     * an ancestor of the system class loader, then this method invokes the
     * security manager's {@link
     * SecurityManager#checkPermission(java.security.Permission)
     * <tt>checkPermission</tt>} method with a {@link
     * RuntimePermission#RuntimePermission(String)
     * <tt>RuntimePermission("getClassLoader")</tt>} permission to verify
     * access to the system class loader.  If not, a
     * <tt>SecurityException</tt> will be thrown.  </p>
     *
     * @return  The system <tt>ClassLoader</tt> for delegation, or
     *          <tt>null</tt> if none
     *
     * @throws  SecurityException
     *          If a security manager exists and its <tt>checkPermission</tt>
     *          method doesn't allow access to the system class loader.
     *
     * @throws  IllegalStateException
     *          If invoked recursively during the construction of the class
     *          loader specified by the "<tt>java.system.class.loader</tt>"
     *          property.
     *
     * @throws  Error
     *          If the system property "<tt>java.system.class.loader</tt>"
     *          is defined but the named class could not be loaded, the
     *          provider class does not define the required constructor, or an
     *          exception is thrown by that constructor when it is invoked. The
     *          underlying cause of the error can be retrieved via the
     *          {@link Throwable#getCause()} method.
     *
     * @revised  1.4
     */
    @CallerSensitive
    public static ClassLoader getSystemClassLoader() {
        initSystemClassLoader();
        if (scl == null) {
            return null;
        }
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkClassLoaderPermission(scl, Reflection.getCallerClass());
        }
        return scl;
    }
    //初始系统类加载器
    private static synchronized void initSystemClassLoader() {
       if (!sclSet) {
           if (scl != null)
               throw new IllegalStateException("recursive invocation");
           sun.misc.Launcher l = sun.misc.Launcher.getLauncher();
           if (l != null) {
               Throwable oops = null;
               scl = l.getClassLoader();
               try {
                   scl = AccessController.doPrivileged(
                       new SystemClassLoaderAction(scl));
               } catch (PrivilegedActionException pae) {
                   oops = pae.getCause();
                   if (oops instanceof InvocationTargetException) {
                       oops = oops.getCause();
                   }
               }
               if (oops != null) {
                   if (oops instanceof Error) {
                       throw (Error) oops;
                   } else {
                       // wrap the exception
                       throw new Error(oops);
                   }
               }
           }
           sclSet = true;
       }
   }
   //系统加载类，在当前访问控制权限下，获取系统类加载器，没有，则返回父加载器，存在则设置
   //当前线程的上下文加载器
   class SystemClassLoaderAction
       implements PrivilegedExceptionAction<ClassLoader> {
       private ClassLoader parent;

       SystemClassLoaderAction(ClassLoader parent) {
           this.parent = parent;
       }

       public ClassLoader run() throws Exception {
           String cls = System.getProperty("java.system.class.loader");
           if (cls == null) {
               return parent;
           }

           Constructor<?> ctor = Class.forName(cls, true, parent)
               .getDeclaredConstructor(new Class<?>[] { ClassLoader.class });
           ClassLoader sys = (ClassLoader) ctor.newInstance(
               new Object[] { parent });
           Thread.currentThread().setContextClassLoader(sys);
           return sys;
       }
   }
```
如果系统加载器为空，则获取BootStap加载器
```java
/**
    * Find resources from the VM's built-in classloader.
    */
   private static URL getBootstrapResource(String name) {
       URLClassPath ucp = getBootstrapClassPath();
       Resource res = ucp.getResource(name);
       return res != null ? res.getURL() : null;
   }
   // Returns the URLClassPath that is used for finding system resources.
   static URLClassPath getBootstrapClassPath() {
       return sun.misc.Launcher.getBootstrapClassPath();
   }
```

//Launcher
```java
public static URLClassPath getBootstrapClassPath()
    {
        return BootClassPathHolder.bcp;
    }
    private static class BootClassPathHolder
       {

           static final URLClassPath bcp;

           static
           {
               URL aurl[];
               if(Launcher.bootClassPath != null)
                   aurl = (URL[])AccessController.doPrivileged(new PrivilegedAction() {

                       public URL[] run()
                       {
                           File afile[] = Launcher.getClassPath(Launcher.bootClassPath);
                           int i = afile.length;
                           HashSet hashset = new HashSet();
                           for(int j = 0; j < i; j++)
                           {
                               File file = afile[j];
                               if(!file.isDirectory())
                                   file = file.getParentFile();
                               if(file != null && hashset.add(file))
                                   MetaIndex.registerDirectory(file);
                           }

                           return Launcher.pathToURLs(afile);
                       }

                       public volatile Object run()
                       {
                           return run();
                       }

                   });
               else
                   aurl = new URL[0];
               bcp = new URLClassPath(aurl, Launcher.factory, null);
               bcp.initLookupCache(null);
           }

           private BootClassPathHolder()
           {
           }
       }
         private static URLStreamHandlerFactory factory = new Factory();
          private static Launcher launcher = new Launcher();
          private static String bootClassPath = System.getProperty("sun.boot.class.path");
          private ClassLoader loader;
          private static URLStreamHandler fileHandler;
```
再来看第2点
2.
```java
else if (this.classLoader != null) {
    //如果类加载器不为空，从类加载器加载资源
    return this.classLoader.getResource(this.path);
}
```
//ClassLoader
```java
public URL getResource(String name) {
        URL url;
        if (parent != null) {
            url = parent.getResource(name);
        } else {
            url = getBootstrapResource(name);
        }
        if (url == null) {
            url = findResource(name);
        }
        return url;
    }
    /**
     * Find resources from the VM's built-in classloader.
     */
    private static URL getBootstrapResource(String name) {
        URLClassPath ucp = getBootstrapClassPath();
        Resource res = ucp.getResource(name);
        return res != null ? res.getURL() : null;
    }
```
有了上面的代码线面一点应该很好理解
3.
```java
else {
    //否则从系统类加载器加载资源
    return ClassLoader.getSystemResource(this.path);
}
```

从上面可以看出获取类路径资源URL，如果资源类不为空，从资源类的类加载器获取资源，否则从从类加载器加载资源，如果还不能加载资源，则从从系统类加载器加载资源。
针对类的加载器不存在的情况，则获取系统类加载器加载资源，如果系统类加载器为空，则使用Bootstrap类加载器加载资源。


再来看获取类资源文件输入流：
```java
/**
	 * This implementation opens an InputStream for the given class path resource.
	 * 从给定的类路径资源，打开一个输入流
	 * @see java.lang.ClassLoader#getResourceAsStream(String)
	 * @see java.lang.Class#getResourceAsStream(String)
	 */
	@Override
	public InputStream getInputStream() throws IOException {
		InputStream is;
		if (this.clazz != null) {
			//如果资源类不为空，从资源类的类加载器打开输入流
			is = this.clazz.getResourceAsStream(this.path);
		}
		else if (this.classLoader != null) {
			//如果类加载器不为空，从类加载器打开输入流
			is = this.classLoader.getResourceAsStream(this.path);
		}
		else {
			//否则从系统类加载器加载资源，打开输入流
			is = ClassLoader.getSystemResourceAsStream(this.path);
		}
		if (is == null) {
			throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
		}
		return is;
	}
```
有了获取文件URL的分析，这个应该很好理解，思路基本查不到，我们看一个从资源类的类加载器打开输入流。



我们简单看一下从资源类的类加载器打开输入流
```java
if (this.clazz != null) {
    //如果资源类不为空，从资源类的类加载器打开输入流
    is = this.clazz.getResourceAsStream(this.path);
}
```
//Class
```java
public InputStream getResourceAsStream(String name) {
       name = resolveName(name);
       ClassLoader cl = getClassLoader0();
       if (cl==null) {
           // A system class.
           return ClassLoader.getSystemResourceAsStream(name);
       }
       return cl.getResourceAsStream(name);
   }
```

//ClassLoader

```java
public InputStream getResourceAsStream(String name) {
       URL url = getResource(name);
       try {
           return url != null ? url.openStream() : null;
       } catch (IOException e) {
           return null;
       }
   }
   public static InputStream getSystemResourceAsStream(String name) {
        URL url = getSystemResource(name);
        try {
            return url != null ? url.openStream() : null;
        } catch (IOException e) {
            return null;
        }
    }
```



从上面可以看出，打开类路径资源输入流的思路和获取文件URL的方法类似，如果资源类不为空，从资源类的类加载器打开输入流，否则从类加载器打开输入流，如果类加载器为空，则
从系统类加载器加载资源，打开输入流。打开类路径资源输入流，先获取类路径资源URL，在委托URL打开输入流。

再来看其他方法，其他方法就不多讲了，很好理解
```java
/**
	 * This implementation checks for the resolution of a resource URL.
	 * 检查当前URL资源是否存在
	 * @see java.lang.ClassLoader#getResource(String)
	 * @see java.lang.Class#getResource(String)
	 */
	@Override
	public boolean exists() {
		return (resolveURL() != null);
	}
    /**
	 * This implementation creates a ClassPathResource, applying the given path
	 * relative to the path of the underlying resource of this descriptor.
	 * 根据与底层资源描述关联的路径的相对路径创建一个ClassPathResource
	 * @see org.springframework.util.StringUtils#applyRelativePath(String, String)
	 */
	@Override
	public Resource createRelative(String relativePath) {
		String pathToUse = StringUtils.applyRelativePath(this.path, relativePath);
		return (this.clazz != null ? new ClassPathResource(pathToUse, this.clazz) :
				new ClassPathResource(pathToUse, this.classLoader));
	}

	/**
	 * This implementation returns the name of the file that this class path
	 * resource refers to.
	 * 获取类路径资源的文件名
	 * @see org.springframework.util.StringUtils#getFilename(String)
	 */
	@Override
	public String getFilename() {
		return StringUtils.getFilename(this.path);
	}

	/**
	 * This implementation returns a description that includes the class path location.
	 * 获取资源描述，包括类路径位置。
	 */
	@Override
	public String getDescription() {
		StringBuilder builder = new StringBuilder("class path resource [");
		String pathToUse = path;
		if (this.clazz != null && !pathToUse.startsWith("/")) {
			builder.append(ClassUtils.classPackageAsResourcePath(this.clazz));
			builder.append('/');
		}
		if (pathToUse.startsWith("/")) {
			pathToUse = pathToUse.substring(1);
		}
		builder.append(pathToUse);
		builder.append(']');
		return builder.toString();
	}


	/**
	 * This implementation compares the underlying class path locations.
	 */
	@Override
	public boolean equals(Object obj) {
		if (obj == this) {
			return true;
		}
		//如果为ClassPathResource资源
		if (obj instanceof ClassPathResource) {
			ClassPathResource otherRes = (ClassPathResource) obj;
			//当前仅当路径，类加载器，资源类相等才相等。
			return (this.path.equals(otherRes.path) &&
					ObjectUtils.nullSafeEquals(this.classLoader, otherRes.classLoader) &&
					ObjectUtils.nullSafeEquals(this.clazz, otherRes.clazz));
		}
		return false;
	}

	/**
	 * This implementation returns the hash code of the underlying
	 * class path location.
	 * 底层文件路径位置的hash值
	 */
	@Override
	public int hashCode() {
		return this.path.hashCode();
	}
```
小节一下：
ClassPathResource内部有3变量，一个为类资源路径path（String），一个类机载器classLoader（ClassLoader），一个为资源类clazz（Class<?> ），
同时提供根据3个内部变量构成类路径资源的构造。获取类路径资源URL，如果资源类不为空，从资源类的类加载器获取资源，否则从从类加载器加载资源，如果还不能加载资源，则从从系统类加载器加载资源。针对类的加载器不存在的情况，则获取系统类加载器加载资源，如果系统类加载器为空，则使用Bootstrap类加载器加载资源。打开类路径资源输入流的思路和获取文件URL的方法类似，如果资源类不为空，从资源类的类加载器打开输入流，否则从类加载器打开输入流，如果类加载器为空，则从系统类加载器加载资源，打开输入流。打开类路径资源输入流，先获取类路径资源URL，在委托URL打开输入流。

再来看ClassPathContextResource

### ClassPathContextResource

源码参见：[ClassPathContextResource][]

[ClassPathContextResource]: "ClassPathContextResource"

```java
/**
	 * ClassPathResource that explicitly expresses a context-relative path
	 * through implementing the ContextResource interface.
	 * ClassPathContextResource显示地表示一个上下文相对路径的类路径资源
	 */
	protected static class ClassPathContextResource extends ClassPathResource implements ContextResource {

		public ClassPathContextResource(String path, ClassLoader classLoader) {
			super(path, classLoader);
		}

		@Override
		public String getPathWithinContext() {
			return getPath();
		}

		@Override
		public Resource createRelative(String relativePath) {
			String pathToUse = StringUtils.applyRelativePath(getPath(), relativePath);
			return new ClassPathContextResource(pathToUse, getClassLoader());
		}
	}
```
从上面可以看出，ClassPathContextResource表示一个上下文相对路径的类路径资源。

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
ClassPathResource内部有3变量，一个为类资源路径path（String），一个类机载器classLoader（ClassLoader），一个为资源类clazz（Class<?> ），
同时提供根据3个内部变量构成类路径资源的构造。获取类路径资源URL，如果资源类不为空，从资源类的类加载器获取资源，否则从从类加载器加载资源，如果还不能加载资源，则从从系统类加载器加载资源。针对类的加载器不存在的情况，则获取系统类加载器加载资源，如果系统类加载器为空，则使用Bootstrap类加载器加载资源。打开类路径资源输入流的思路和获取文件URL的方法类似，如果资源类不为空，从资源类的类加载器打开输入流，否则从类加载器打开输入流，如果类加载器为空，则从系统类加载器加载资源，打开输入流。打开类路径资源输入流，先获取类路径资源URL，在委托URL打开输入流。




## 附
