---
layout: page
title: AbstractApplicationContext总结
subtitle: AbstractApplicationContext总结
date: 2018-01-26 16:37:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言

到目前为止，我们已经把AbstractApplicationContext相关的操作与涉及的概念已经讲述完，今天来总结一下AbstractApplicationContext的相关操及设计到的概念。相关文章如下：
* [AbstractApplicationContext源码解析第一讲][]
* [AbstractApplicationContext源码解析第二讲][]
* [AbstractApplicationContext源码解析第三讲][]
* [AbstractApplicationContext源码解析第四讲][]
* [AbstractApplicationContext源码解析第五讲][]
* [SimpleApplicationEventMulticaster解析][]
* [StandardEnvironment源码解析][]
* [DelegatingMessageSource解析][]

我们可以结合AbstractApplicationContext类图及涉及的类及接口，重新认识抽象应用上下文的作用：
![AbstractApplicationContext](/image/spring-context/AbstractApplicationContext.png)

# 目录
* [AbstractApplicationContext源码解析第一讲](abstractapplicationcontext源码解析第一讲)
* [AbstractApplicationContext源码解析第二讲](#AbstractApplicationContext源码解析第二讲)
* [AbstractApplicationContext源码解析第三讲](#AbstractApplicationContext源码解析第三讲)
* [SimpleApplicationEventMulticaster解析](#SimpleApplicationEventMulticaster解析)
* [StandardEnvironment源码解析](#StandardEnvironment源码解析)
* [DelegatingMessageSource解析](#DelegatingMessageSource解析)
* [AbstractApplicationContext源码解析第四讲](#AbstractApplicationContext源码解析第四讲)
* [AbstractApplicationContext源码解析第五讲](#AbstractApplicationContext源码解析第五讲)

## AbstractApplicationContext源码解析第一讲

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
获取HttpURLConnection的转台，OK则存在，否则false。获取资源内容长度和上次修改时间与判断文件是否存在的思想基本相同


## AbstractApplicationContext源码解析第二讲

ClassPathResource内部有3变量，一个为类资源路径path（String），一个类机载器classLoader（ClassLoader），一个为资源类clazz（Class<?> ），
同时提供根据3个内部变量构成类路径资源的构造。获取类路径资源URL，如果资源类不为空，从资源类的类加载器获取资源，否则从从类加载器加载资源，如果还不能加载资源，则从从系统类加载器加载资源。针对类的加载器不存在的情况，则获取系统类加载器加载资源，如果系统类加载器为空，则使用Bootstrap类加载器加载资源。打开类路径资源输入流的思路和获取文件URL的方法类似，如果资源类不为空，从资源类的类加载器打开输入流，否则从类加载器打开输入流，如果类加载器为空，则从系统类加载器加载资源，打开输入流。打开类路径资源输入流，先获取类路径资源URL，在委托URL打开输入流。

默认资源加载器DefaultResourceLoader的根据给定位置加载资源的方法，当给定资源的位置以资源位置以"/"开头，加载的资源类型为ClassPathContextResource。
ClassPathContextResource表示一个上下文相对路径的类路径资源。

UrlResource内部有3个变量，一个为资源的URI，一个为资源URL，另外一个为干净的URL，提供提供了根据资源URL，URI和资源协议、位置、分片来构建UrlResource
资源的构造方法，获取资源输入流，及获取文件都是委托给内部的URL。

## AbstractApplicationContext源码解析第三讲


抽象应用上下文 *AbstractApplicationContext* 实际为一个可配置上下文 *ConfigurableApplicationContext* 和可销毁的bean（DisposableBean），同时拥有了资源加载功能（DefaultResourceLoader）。我们通过一个唯一的id标注抽象上下文，同时抽象上下文拥有一个展示名。除此身份识别属性之前，抽象应用上下文，有一个父上下文 *ApplicationContext* ，可配的环境配置 *ConfigurableEnvironment* ，bean工厂后处理器集（List<BeanFactoryPostProcessor>），资源模式解决器（ResourcePatternResolver），声明周期处理器（LifecycleProcessor),消息源 *MessageSource* ，事件发布器 *ApplicationEventMulticaster* ，应用监听器集（LinkedHashSet<ApplicationListener<?>>），预发布的应用事件集（LinkedHashSet<ApplicationEvent>）。除了上述的功能性属性外，抽象应用上下文，还有一个一些状态属性，如果启动时间，激活状态（AtomicBoolean），关闭状态（AtomicBoolean）。最后还有一个上下为刷新和销毁的同步监控对象和一虚拟机关闭hook线程。


路径匹配资源模式解决器PathMatchingResourcePatternResolver内部有一个Ant路径匹配器 *AntPathMatcher*，和一个资源类加载器，资源加载器可以
使用所属上下文中的资源加载器，也可以为给定类加载器的DefaultResourceLoader。路径匹配资源模式解决器主要提供了加载给定路径位置的资源方法，此方法可以解决无通配符的路径位置模式（{@code file:C:/context.xml}，{@code classpath:/context.xml}，{@code /WEB-INF/context.xml}"），也可以解决包含Ant风格的通配符路径位置模式资源（{@code classpath*:META-INF/beans.xml}），主要以classpath*为前缀的路径位置模式，资源加载器将会查找类路径下所有相同name对应的资源文件，包括子目录和jar包。如果明确的加载资源，可以使用{@code classpath:/context.xml}形式路径模式，如果想要探测类路径下的所有name对应的资源文件，可以使用形式路径模式。

BeanFactoryAware接口主要提供设置bean工厂操作。LifecycleProcessor接口主要提供了通知上下文刷新和关闭的操作。Phased主要提供了获取组件阶段值操作。
SmartLifecycle接口主要提供关闭回调操作，在组件停止后，调用回调接口。并提供了判断组件在容器上下文刷新时，组件是否自动刷新的操作。

默认生命周期处理器[DefaultLifecycleProcessor][]，内部主要有3个成员变量，一个是运行状态标识，一个是生命周期bean关闭超时时间，还有一个是所属的bean工厂。默认生命周期处理器，启动生命周期bean的过程为，将生命周期bean，按阶段值分组，并从阶段值从小到大，启动生命周期bean分组中bean。默认生命周期处理器，关闭生命周期bean的过程为，将生命周期bean，按阶段值分组，并从阶段值从大到小，关闭生命周期bean分组中bean。关闭生命周期bean的顺序与启动顺序正好相反。需要注意的是无论是启动还是关闭，生命周期bean所依赖的bean都是在其之前启动或关闭，忽略掉被依赖bean的Phase阶段值。对于非生命周期bean，其阶段值默认为0。

## SimpleApplicationEventMulticaster解析
应用事件多播器ApplicationEventMulticaster主要提供了应用事件监听器的管理操作（添加、移除），同时提供了发布应用事件到所管理的应用监听器的操作。应用事件多播器典型应用，为代理应用上下文，发布相关应用事件。BeanClassLoaderAware主要体用了设置bean类加载器的操作，主要用于框架实现类想用根据的name获取bean的应用类型的场景。

AbstractApplicationEventMulticaster内部有一个存放监听器的集合 *ListenerRetriever*，事件监听器缓存retrieverCache（*ConcurrentHashMap<ListenerCacheKey, ListenerRetriever>*）用于存放应用事件与监听器映射关系，bean类加载器 *ClassLoader*，所属bean工厂BeanFactory
用于获取监听器bean name对应的监听器。所有的监听器注册操作实际由 *ListenerRetriever* 来完成，*ListenerRetriever* 使用LinkedHashSet来管理监听器。注意在每次添加和移除监听器之后，将会清除监听器缓存。抽象应用事件多播器除了管理监听器相关的实现此外，提供了获取注册到多播器监听器的方法，实际为ListenerRetriever整合
内部监听器集和监听器bean name对应的监听器；同时还有获取给定事件类型的对应的监听器，即关注给定事件类型的监听器，这过程首先从监听器缓存
中获取事件相关的监听器，如果存在，则从监听器检索器中检索出关闭事件的监听器，并封装在监听器检索器ListenerRetriever中，然后添加到监听器缓存中。
监听器缓存键ListenerCacheKey为事件类型与事件源的封装。

简单事件多播器[][SimpleApplicationEventMulticaster]，主要实现了多播器的多播事件操作，即将应用事件传递给相应的应用监听器，非关注
此事件的监听器，将会被忽略。默认情况下，简单事件多播器在当前线程下调用监听器的事件处理器操作，当然我们也可以设置多播器的任务执行器 *Executor*，委托任务执行器
调用监听器的事件处理器操作，同时我们也可以设置异常处理器 *ErrorHandler* 用于处理调用监听器过程中异常。

## StandardEnvironment源码解析
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

## DelegatingMessageSource解析
HierarchicalMessageSource主要提供了设置父消息的操作。此接口用于，当消息源不能解决给定消息时，尝试使用父消息源解决消息，即层级解决消息。

MessageSourceSupport内部主要成员为消息格式缓存messageFormatsPerLocale（HashMap<String, Map<Locale, MessageFormat>>()）用于存储消息的消息格式，以及是否应用消息格式规则，解析没有参数的消息标志alwaysUseMessageFormat。此消息源实现基础类主要提供了根据给定的消息，消息参数和本地化渲染消息操作，即使用参数替代默认消息中的占位符。

DelegatingMessageSource内部有一父消息源，获取编码消息操作直接委托给父类数据源，如果没有父消息源，同时有默认消息，则使用消息参数渲染默认消息，并返回。如果没有父消息可用，不会解决任何消息，抛出异常。代理消息源用于，上下文没有定义消息源情况下，AbstractApplicationContext使用 DelegatingMessageSource为消息源的占位符，此代理消息源不能被应用直接使用。

## AbstractApplicationContext源码解析第四讲
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

## AbstractApplicationContext源码解析第五讲
注册虚拟机关闭Hook，委托给运行时Runtime，而hook线程完成实际工作为，从[LiveBeansView][]反注册当前应用上下文，
发布上下文关闭事件ContextClosedEvent，销毁上下文bean工厂中所有缓存的单例bean，关闭上下文状态closeBeanFactory，如果希望，
可以让子类做一些其他清除工作onClose；closeBeanFactory和onClose方法待子类扩展。

销毁应用上下文，和关闭应用上下文实际委托给doClose操作，完成的任务和虚拟机关闭Hook线程相同，不同的是，在关闭应用上下文的时候，
如果关闭Hook不为空，要从运行时环境中移除关闭Hook线程。

bean工厂接口，可配置、层级bean工厂接口的操作实现实际委托给内部bean工厂。MessageSource接口实现委托给内部的消息源。获取给定位置的资源实际委托给内部的资源模式解决器。

Lifecycle生命周期接口的启动和停止实际是调用应用上下文的生命周期处理器的启动和停止操作。



[AbstractApplicationContext源码解析第一讲]:https://donaldhan.github.io/spring-framework/2017/12/27/AbstractApplicationContext%E5%AE%9A%E4%B9%89%E7%AC%AC%E4%B8%80%E8%AE%B2.html "AbstractApplicationContext源码解析第一讲"

[AbstractApplicationContext源码解析第二讲]:https://donaldhan.github.io/spring-framework/2017/12/27/AbstractApplicationContext%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%AC%AC%E4%BA%8C%E8%AE%B2.html "AbstractApplicationContext源码解析第二讲"

[AbstractApplicationContext源码解析第三讲]:https://donaldhan.github.io/spring-framework/2018/01/04/AbstractApplicationContext%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%AC%AC%E4%B8%89%E8%AE%B2.html "AbstractApplicationContext源码解析第三讲"

[AbstractApplicationContext源码解析第四讲]:https://donaldhan.github.io/spring-framework/2018/01/24/AbstractApplicationContext%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%AC%AC%E5%9B%9B%E8%AE%B2.html "AbstractApplicationContext源码解析第四讲"


[AbstractApplicationContext源码解析第五讲]:https://donaldhan.github.io/spring-framework/2018/01/26/AbstractApplicationContext%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%AC%AC%E4%BA%94%E8%AE%B2.html "AbstractApplicationContext源码解析第五讲"

[SimpleApplicationEventMulticaster解析]:https://donaldhan.github.io/spring-framework/2018/01/06/SimpleApplicationEventMulticaster%E8%A7%A3%E6%9E%90.html "SimpleApplicationEventMulticaster解析"

[StandardEnvironment源码解析]:https://donaldhan.github.io/spring-framework/2018/01/10/StandardEnvironment%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.html "StandardEnvironment源码解析"


[DelegatingMessageSource解析]:https://donaldhan.github.io/spring-framework/2018/01/11/DelegatingMessageSource%E8%A7%A3%E6%9E%90.html "DelegatingMessageSource解析"  

[LiveBeansView]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/LiveBeansView.java "LiveBeansView"  

[PostProcessorRegistrationDelegate]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/PostProcessorRegistrationDelegate.java "PostProcessorRegistrationDelegate"  
[ContextTypeMatchClassLoader]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/ContextTypeMatchClassLoader.java "ContextTypeMatchClassLoader"


[MergedBeanDefinitionPostProcessor]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-beans/src/main/java/org/springframework/beans/factory/support/MergedBeanDefinitionPostProcessor.java "MergedBeanDefinitionPostProcessor"   

[ApplicationListenerDetector]:https://github.com/Donaldhan/spring-framework/blob/4.3.x/spring-context/src/main/java/org/springframework/context/support/ApplicationListenerDetector.java "ApplicationListenerDetector"
