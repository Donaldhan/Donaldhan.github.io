---
layout: page
title: Dubbo框架设计源码解读一（服务和引用bean初始化）
subtitle: ServiceBean和ReferenceBean的初始化
date: 2020-11-17 23:11:00
author: valuewithTime
catalog: true
category: Dubbo
categories:
    - Dubbo
tags:
    - Dubbo
---

# 引言
随着互联网应用体量不断的增加，为了对应大量用户访问的体验效果，及提供应用的可用性，微服务应运而生，微服务使应用的业务具有更高的内聚性，同时对整体应用进行服务化的解耦；微服务之间除了通过异步的MQ进行通信，常用的还有基于同步的RPC通知机制，比如基于HTTP的GRPC, BRPC及在国内使用比较广泛的[Dubbo][]。一致在使用DUBOO，对其中的功能组件有大致的了解，本计划看一下源码的，一致没有抽出时间，最近终于忙里抽闲，一探Dubbo芳容。

[Dubbo]: https://github.com/apache/dubbo "Dubbo"


# 目录
* [dubbo的源码如何看](#dubbo的源码如何看)
* [概要框架设计](#概要框架设计)
* [源码分析](#源码分析)
    * [应用协议](#应用协议)
        * [注册器协议](#注册器协议) 
            * [服务导出Exporter](#服务导出exporter) 
            * [服务注册](#注册器协议) 
            * [订阅服务](#订阅服务)
            * [服务Invoker](#服务invoker) 
        * [Dubbo协议](#dubbo协议) 
    * [数据传输器Transport](#数据传输器transport)
        * [服务端](#服务端) 
        * [客户端](#客户端) 
    * [消息编解码](#消息编解码)
        * [消息编码器](#消息编码器)
        * [消息解码器](#消息解码器)
* [总结](#总结)
* [附](#附)

# dubbo的源码如何看
首先我们从
[incubator-dubbo-spring-boot-project github vt](https://github.com/Donaldhan/incubator-dubbo-spring-boot-project) 仓库拉取dubbo与spring boot集成的项目

再来取dubbo仓库

[dubbo github vt](https://github.com/Donaldhan/incubator-dubbo)    

从incubator-dubbo-spring-boot-project项目的dubbo-spring-boot-autoconfigure-compatible模块的
META-INF/spring.factories

可以看到对应的自动配置
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.apache.dubbo.spring.boot.autoconfigure.
DubboAutoConfiguration,\
org.apache.dubbo.spring.boot.autoconfigure.
DubboRelaxedBindingAutoConfiguration
org.springframework.context.ApplicationListener=\
org.apache.dubbo.spring.boot.context.event.
OverrideDubboConfigApplicationListener,\
org.apache.dubbo.spring.boot.context.event.
WelcomeLogoApplicationListener,\
org.apache.dubbo.spring.boot.context.event.
AwaitingNonWebApplicationListener
org.springframework.boot.env.EnvironmentPostProcessor=\
org.apache.dubbo.spring.boot.env.
DubboDefaultPropertiesEnvironmentPostProcessor
```

关键的为DubboAutoConfiguration，查看DubboAutoConfiguration，注释生命的如下两个类，这两个配置ServiceBean和ReferenceBean对应的bean后处理器ServiceAnnotationBeanPostProcessor， ReferenceAnnotationBeanPostProcessor，这是我们从dubbo源码分析dubbo框架设计入口。

### DubboAutoConfiguration

```
ServiceAnnotationBeanPostProcessor
```

#### ServiceBean
```
ReferenceAnnotationBeanPostProcessor
```
#### ReferenceBean
```
org.apache.dubbo.config.spring.ReferenceBean#getObject
```

下面从源码的角度，对dubbo框架进行一个简单的概要性的总结。

# 概要框架设计


![dubbo-framework](/image/dubbo/dubbo-framework.png)  


dubbo框架设计，主要包括消息层，传输层，协议层。消息层提供消费者调用服务请求消息、服务提供方处理结果响应消息的编解码；传输层主要建立消费者和服务者的通信通道，传输服务请求响应数据；协议层首先是基于相关协议将服务提供者，和消费者通过export暴露出去，即注册的Registry中，消费者通过Registry订阅响应的服务提供者，消费者发现有服务提供者，则与服务提供者建立连接。服务提供者接受的消费者的服务请求后，根据相关协议，调用响应的Invoker服务。


下面我们来从源码来分析Dubbo的各个组件模块。
# 源码分析
首先先看一下ServiceBean和ReferenceBean的后处理
## ServiceAnnotationBeanPostProcessor
```java
/**
 * {@link Service} Annotation
 * {@link BeanDefinitionRegistryPostProcessor Bean Definition Registry Post Processor}
 *
 * @since 2.5.8
 */
public class ServiceAnnotationBeanPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware {
@Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        //解决环境变量的包扫描路径，针对包扫描参数中有占位符的场景
        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
            //注册service bean
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }

    }


    /**
     * Registers Beans whose classes was annotated {@link Service}
     * 注册注解了Service的Bean
     * @param packagesToScan The base packages to scan
     * @param registry       {@link BeanDefinitionRegistry}
     */
    private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);

        scanner.setBeanNameGenerator(beanNameGenerator);
        //添加service注解过滤器
        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));

        for (String packageToScan : packagesToScan) {

            // Registers @Service Bean first， 包扫描
            scanner.scan(packageToScan);

            // Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not. 获取Service注解bean
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {

                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    //注册service bean
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }

                if (logger.isInfoEnabled()) {
                    logger.info(beanDefinitionHolders.size() + " annotated Dubbo's @Service Components { " +
                            beanDefinitionHolders +
                            " } were scanned under package[" + packageToScan + "]");
                }

            } else {

                if (logger.isWarnEnabled()) {
                    logger.warn("No Spring Bean annotating Dubbo's @Service was found under package["
                            + packageToScan + "]");
                }

            }

        }

    }
    /**
     * Registers {@link ServiceBean} from new annotated {@link Service} {@link BeanDefinition}
     * 注册Service bean
     * @param beanDefinitionHolder
     * @param registry
     * @param scanner
     * @see ServiceBean
     * @see BeanDefinition
     */
    private void registerServiceBean(BeanDefinitionHolder beanDefinitionHolder, BeanDefinitionRegistry registry,
                                     DubboClassPathBeanDefinitionScanner scanner) {

        Class<?> beanClass = resolveClass(beanDefinitionHolder);
        //获取service注解
        Service service = findAnnotation(beanClass, Service.class);

        Class<?> interfaceClass = resolveServiceInterfaceClass(beanClass, service);

        String annotatedServiceBeanName = beanDefinitionHolder.getBeanName();
        //构造service bean
        AbstractBeanDefinition serviceBeanDefinition =
                buildServiceBeanDefinition(service, interfaceClass, annotatedServiceBeanName);

        // ServiceBean Bean name
        String beanName = generateServiceBeanName(service, interfaceClass, annotatedServiceBeanName);

        if (scanner.checkCandidate(beanName, serviceBeanDefinition)) { // check duplicated candidate bean
            //注册bean
            registry.registerBeanDefinition(beanName, serviceBeanDefinition);

            if (logger.isWarnEnabled()) {
                logger.warn("The BeanDefinition[" + serviceBeanDefinition +
                        "] of ServiceBean has been registered with name : " + beanName);
            }

        } else {

            if (logger.isWarnEnabled()) {
                logger.warn("The Duplicated BeanDefinition[" + serviceBeanDefinition +
                        "] of ServiceBean[ bean name : " + beanName +
                        "] was be found , Did @DubboComponentScan scan to same package in many times?");
            }

        }

    }
    /**
     * 构造服务bean定义
     * @param service
     * @param interfaceClass
     * @param annotatedServiceBeanName
     * @return
     */
    private AbstractBeanDefinition buildServiceBeanDefinition(Service service, Class<?> interfaceClass,
                                                              String annotatedServiceBeanName) {

        BeanDefinitionBuilder builder = rootBeanDefinition(ServiceBean.class);

        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();

        MutablePropertyValues propertyValues = beanDefinition.getPropertyValues();

        String[] ignoreAttributeNames = of("provider", "monitor", "application", "module", "registry", "protocol", "interface", "interfaceName");

        propertyValues.addPropertyValues(new AnnotationPropertyValuesAdapter(service, environment, ignoreAttributeNames));

        // References "ref" property to annotated-@Service Bean， 设置service config
        addPropertyReference(builder, "ref", annotatedServiceBeanName);
        // Set interface
        builder.addPropertyValue("interface", interfaceClass.getName());

        /**
         * Add {@link org.apache.dubbo.config.ProviderConfig} Bean reference
         */
        String providerConfigBeanName = service.provider();
        if (StringUtils.hasText(providerConfigBeanName)) {
            addPropertyReference(builder, "provider", providerConfigBeanName);
        }

        /**
         * Add {@link org.apache.dubbo.config.MonitorConfig} Bean reference
         */
        String monitorConfigBeanName = service.monitor();
        if (StringUtils.hasText(monitorConfigBeanName)) {
            addPropertyReference(builder, "monitor", monitorConfigBeanName);
        }

        /**
         * Add {@link org.apache.dubbo.config.ApplicationConfig} Bean reference
         */
        String applicationConfigBeanName = service.application();
        if (StringUtils.hasText(applicationConfigBeanName)) {
            addPropertyReference(builder, "application", applicationConfigBeanName);
        }

        /**
         * Add {@link org.apache.dubbo.config.ModuleConfig} Bean reference
         */
        String moduleConfigBeanName = service.module();
        if (StringUtils.hasText(moduleConfigBeanName)) {
            addPropertyReference(builder, "module", moduleConfigBeanName);
        }


        /**
         * Add {@link org.apache.dubbo.config.RegistryConfig} Bean reference
         */
        String[] registryConfigBeanNames = service.registry();

        List<RuntimeBeanReference> registryRuntimeBeanReferences = toRuntimeBeanReferences(registryConfigBeanNames);

        if (!registryRuntimeBeanReferences.isEmpty()) {
            builder.addPropertyValue("registries", registryRuntimeBeanReferences);
        }

        /**
         * Add {@link org.apache.dubbo.config.ProtocolConfig} Bean reference
         */
        String[] protocolConfigBeanNames = service.protocol();

        List<RuntimeBeanReference> protocolRuntimeBeanReferences = toRuntimeBeanReferences(protocolConfigBeanNames);

        if (!protocolRuntimeBeanReferences.isEmpty()) {
            builder.addPropertyValue("protocols", protocolRuntimeBeanReferences);
        }

        return builder.getBeanDefinition();

    }
...
        }
```

从上面可以看出，ServiceAnnotationBeanPostProcessor主要做的事情是扫描
应用先的Service注解bean，并构造ServiceBean，注册到bean注册器中。

再来看一下ServiceBean，这里是服务导出的入口。

### ServiceBean

```java
/**
 * ServiceFactoryBean
 *
 * @export
 */
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware {
...
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (!isExported() && !isUnexported()) {
            if (logger.isInfoEnabled()) {
                logger.info("The service ready on spring started. service: " + getInterface());
            }
            //暴露服务
            export();
        }
    }
    ...
}
```

//ServiceConfig
```java
/**
 * ServiceConfig
 *
 * @export
 */
public class ServiceConfig<T> extends AbstractServiceConfig {
    
    /**
     * @see ExtensionLoader#DUBBO_DIRECTORY
     * @see ExtensionLoader#getAdaptiveExtension()
     */
    private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

    /**
     * @see ExtensionLoader#DUBBO_INTERNAL_DIRECTORY
     * @see ExtensionLoader#getAdaptiveExtension()
     */
    private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

    private static final Map<String, Integer> RANDOM_PORT_MAP = new HashMap<String, Integer>();

    /**
     *
     */
    private static final ScheduledExecutorService delayExportExecutor = Executors.newSingleThreadScheduledExecutor(new NamedThreadFactory("DubboServiceDelayExporter", true));
    /**
     *
     */
    private final List<URL> urls = new ArrayList<URL>();
    /**
     *
     */
    private final List<Exporter<?>> exporters = new ArrayList<Exporter<?>>();
    /**
     * interface type
     */
    private String interfaceName;
    /**
     *
     */
    private Class<?> interfaceClass;
    /**
     * reference to interface impl
     * 接口实现
     */
    private T ref;
    /**
     *service name, 服务名
     */
    private String path;
    /**
     * method configuration
     */
    private List<MethodConfig> methods;
    /**
     *
     */
    private ProviderConfig provider;
    private transient volatile boolean exported;

    private transient volatile boolean unexported;

    private volatile String generic;
    ...
     /**
     * 暴露服务
     */
    public synchronized void export() {
        if (provider != null) {
            if (export == null) {
                export = provider.getExport();
            }
            if (delay == null) {
                delay = provider.getDelay();
            }
        }
        if (export != null && !export) {
            return;
        }

        if (delay != null && delay > 0) {
            //延迟调度
            delayExportExecutor.schedule(this::doExport, delay, TimeUnit.MILLISECONDS);
        } else {
            doExport();
        }
    }
    /**
     * 暴露服务
     */
    protected synchronized void doExport() {
        ...
        checkApplication();
        checkRegistry();
        checkProtocol();
        appendProperties(this);
        checkStub(interfaceClass);
        checkMock(interfaceClass);
        if (path == null || path.length() == 0) {
            path = interfaceName;
        }
        //提供模型
        ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), ref, interfaceClass);
        //初始化模型提供者模型
        ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
        //暴露服务url
        doExportUrls();
    }
     /**
     * 暴露服务url
     */
    @SuppressWarnings({"unchecked", "rawtypes"})
    private void doExportUrls() {
        //加载注册器地址
        List<URL> registryURLs = loadRegistries(true);
        for (ProtocolConfig protocolConfig : protocols) {
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
    /**
     * @param protocolConfig
     * @param registryURLs
     */
    private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
        String name = protocolConfig.getName();
        if (name == null || name.length() == 0) {
            name = "dubbo";
        }
    ...
     String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
        Integer port = this.findConfigedPorts(protocolConfig, name, map);
        //dubbo
        URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

        if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .hasExtension(url.getProtocol())) {
            url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
        }

        String scope = url.getParameter(Constants.SCOPE_KEY);
        // don't export when none is configured
        if (!Constants.SCOPE_NONE.equalsIgnoreCase(scope)) {

            // export to local if the config is not remote (export to remote only when config is remote)
            if (!Constants.SCOPE_REMOTE.equalsIgnoreCase(scope)) {
                //暴露本地服务
                exportLocal(url);
            }
            // export to remote if the config is not local (export to local only when config is local)
            //暴露服务到远程注册器
            if (!Constants.SCOPE_LOCAL.equalsIgnoreCase(scope)) {
                if (logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }
                if (registryURLs != null && !registryURLs.isEmpty()) {
                    for (URL registryURL : registryURLs) {
                        url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                        URL monitorUrl = loadMonitor(registryURL);
                        if (monitorUrl != null) {
                            url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                        }
                        if (logger.isInfoEnabled()) {
                            logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                        }

                        // For providers, this is used to enable custom proxy to generate invoker
                        String proxy = url.getParameter(Constants.PROXY_KEY);
                        if (StringUtils.isNotEmpty(proxy)) {
                            registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
                        }
                        //创建代理Invoke JavassistProxyFactory
                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
                        //ProtocolFilterWrapper ,ZookeeperRegistry ,RestProtocol，RmiProtocol，DubboProtocol
                        Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
                    }
                } else {
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            }
        }
        this.urls.add(url);
    }

    /**
     * 暴露本地注册器地址
     * @param url
     */
    @SuppressWarnings({"unchecked", "rawtypes"})
    private void exportLocal(URL url) {
        if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
            URL local = URL.valueOf(url.toFullString())
                    .setProtocol(Constants.LOCAL_PROTOCOL)
                    .setHost(LOCALHOST)
                    .setPort(0);
            Exporter<?> exporter = protocol.export(
                    proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
            exporters.add(exporter);
            logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
        }
    }
}
```
关键在下面几句

```java
// For providers, this is used to enable custom proxy to 
String proxy = url.getParamete(Constants.PROXY_KEY);
if (StringUtils.isNotEmpty(proxy)) {
    registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
}
//创建代理Invoke JavassistProxyFactory
Invoker<?> invoker = proxyFactorygetInvoker(ref, (Class) interfaceClass,registryURL.addParameterAndEncode(Constants.EXPORT_KEY, url.toFullString());
DelegateProviderMetaDataInvokerwrapperInvoker = newDelegateProviderMetaDataInvoker(invoker,this);
//ProtocolFilterWrapper ZookeeperRegistry ,RestProtocolRmiProtocol，DubboProtocol，RegistryProtocol
Exporter<?> exporter = protocol.expor(wrapperInvoker);
exporters.add(exporter);
```
从上面导出服务实际委托给相应的协议RegistryProtocol。RegistryProtocol我们在后面的再说。



我们再来看一下ReferenceBean的处理

## ReferenceAnnotationBeanPostProcessor

```java
/**
 * {@link org.springframework.beans.factory.config.BeanPostProcessor} implementation
 * that Consumer service {@link Reference} annotated fields
 *
 * @since 2.5.7
 */
public class ReferenceAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter
        implements MergedBeanDefinitionPostProcessor, PriorityOrdered, ApplicationContextAware, BeanClassLoaderAware,
        DisposableBean {
            ...
            @Override
    public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {

        InjectionMetadata metadata = findReferenceMetadata(beanName, bean.getClass(), pvs);
        try {
            metadata.inject(bean, beanName, pvs);
        } catch (BeanCreationException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Injection of @Reference dependencies failed", ex);
        }
        return pvs;
    }
     @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        if (beanType != null) {
            InjectionMetadata metadata = findReferenceMetadata(beanName, beanType, null);
            metadata.checkConfigMembers(beanDefinition);
        }
    }
    /**
     * {@link Reference} {@link InjectionMetadata} implementation
     * 引用注入元数据
     * @since 2.5.11
     */
    private static class ReferenceInjectionMetadata extends InjectionMetadata {

        /**
         * 字段级引用元素集
         */
        private final Collection<ReferenceFieldElement> fieldElements;

        /**
         * 方法级引用元素集
         */
        private final Collection<ReferenceMethodElement> methodElements;


        public ReferenceInjectionMetadata(Class<?> targetClass, Collection<ReferenceFieldElement> fieldElements,
                                          Collection<ReferenceMethodElement> methodElements) {
            super(targetClass, combine(fieldElements, methodElements));
            this.fieldElements = fieldElements;
            this.methodElements = methodElements;
        }

        private static <T> Collection<T> combine(Collection<? extends T>... elements) {
            List<T> allElements = new ArrayList<T>();
            for (Collection<? extends T> e : elements) {
                allElements.addAll(e);
            }
            return allElements;
        }

        public Collection<ReferenceFieldElement> getFieldElements() {
            return fieldElements;
        }

        public Collection<ReferenceMethodElement> getMethodElements() {
            return methodElements;
        }
    }
    /**
     * {@link Reference} {@link Method} {@link InjectionMetadata.InjectedElement}
     */
    private class ReferenceMethodElement extends InjectionMetadata.InjectedElement {

        private final Method method;

        private final Reference reference;

        private volatile ReferenceBean<?> referenceBean;

        protected ReferenceMethodElement(Method method, PropertyDescriptor pd, Reference reference) {
            super(method, pd);
            this.method = method;
            this.reference = reference;
        }

        @Override
        protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {

            Class<?> referenceClass = pd.getPropertyType();

            referenceBean = buildReferenceBean(reference, referenceClass);

            ReflectionUtils.makeAccessible(method);

            method.invoke(bean, referenceBean.getObject());

        }

    }
    /**
     * {@link Reference} {@link Field} {@link InjectionMetadata.InjectedElement}
     */
    private class ReferenceFieldElement extends InjectionMetadata.InjectedElement {

        /**
         *
         */
        private final Field field;

        /**
         *
         */
        private final Reference reference;

        private volatile ReferenceBean<?> referenceBean;

        protected ReferenceFieldElement(Field field, Reference reference) {
            super(field, null);
            this.field = field;
            this.reference = reference;
        }

        @Override
        protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {

            Class<?> referenceClass = field.getType();

            referenceBean = buildReferenceBean(reference, referenceClass);

            ReflectionUtils.makeAccessible(field);

            field.set(bean, referenceBean.getObject());

        }

    }

    /**
     * 构建引用bean
     * @param reference
     * @param referenceClass
     * @return
     * @throws Exception
     */
    private ReferenceBean<?> buildReferenceBean(Reference reference, Class<?> referenceClass) throws Exception {

        String referenceBeanCacheKey = generateReferenceBeanCacheKey(reference, referenceClass);

        ReferenceBean<?> referenceBean = referenceBeansCache.get(referenceBeanCacheKey);

        if (referenceBean == null) {

            ReferenceBeanBuilder beanBuilder = ReferenceBeanBuilder
                    .create(reference, classLoader, applicationContext)
                    .interfaceClass(referenceClass);

            referenceBean = beanBuilder.build();

            referenceBeansCache.putIfAbsent(referenceBeanCacheKey, referenceBean);

        }

        return referenceBean;

    }


}
```

//ReferenceBean
```java
**
 * ReferenceFactoryBean
 */
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean {
    ...
     @Override
    public Object getObject() {
        return get();
    }
}
```

//ReferenceConfig
```java
public class ReferenceConfig<T> extends AbstractReferenceConfig {

    private static final long serialVersionUID = -5864351140409987595L;

    /**
     *
     */
    private static final Protocol refprotocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

    private static final Cluster cluster = ExtensionLoader.getExtensionLoader(Cluster.class).getAdaptiveExtension();

    private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
    private final List<URL> urls = new ArrayList<URL>();
    // interface name
    private String interfaceName;
    private Class<?> interfaceClass;
    private Class<?> asyncInterfaceClass;
    // client type
    private String client;
    // url for peer-to-peer invocation
    private String url;
    // method configs
    private List<MethodConfig> methods;
    // default config
    private ConsumerConfig consumer;
    private String protocol;
    // interface proxy reference
    private transient volatile T ref;
    private transient volatile Invoker<?> invoker;
    private transient volatile boolean initialized;
    private transient volatile boolean destroyed;
    ...
    public synchronized T get() {
        if (destroyed) {
            throw new IllegalStateException("Already destroyed!");
        }
        if (ref == null) {
            init();
        }
        return ref;
    }
     /**
     *
     */
    private void init() {
        if (initialized) {
            return;
        }
        initialized = true;
        ...
         //获取注册中心地址
        String hostToRegistry = ConfigUtils.getSystemProperty(Constants.DUBBO_IP_TO_REGISTRY);
        if (hostToRegistry == null || hostToRegistry.length() == 0) {
            //为空，则获取本地host
            hostToRegistry = NetUtils.getLocalHost();
        } else if (isInvalidLocalHost(hostToRegistry)) {
            throw new IllegalArgumentException("Specified invalid registry ip from property:" + Constants.DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
        }
        map.put(Constants.REGISTER_IP_KEY, hostToRegistry);

        ref = createProxy(map);
        //构建消费值模型
        ConsumerModel consumerModel = new ConsumerModel(getUniqueServiceName(), interfaceClass, ref, interfaceClass.getMethods(), attributes);
        //初始化消费者模型
        ApplicationModel.initConsumerModel(getUniqueServiceName(), consumerModel);
    }
    /**
     * @param map
     * @return
     */
    @SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
    private T createProxy(Map<String, String> map) {
        ...
         if (urls.size() == 1) {
                //创建Invoker代理 netty客户端
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                    invokers.add(refprotocol.refer(interfaceClass, url));
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // use last registry url
                    }
                }
                if (registryURL != null) { // registry url is available
                    // use AvailableCluster only when register's cluster is available
                    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                    invoker = cluster.join(new StaticDirectory(u, invokers));
                } else { // not a registry url
                    invoker = cluster.join(new StaticDirectory(invokers));
                }
            }
        ...
    }
}
```
从上面可以看出ReferenceBean后处理主要扫描Reference注解的bean，并构造ReferenceBean，ReferenceBean通过Invoker去调用服务提供者的服务，Invoker为服务的包装类。实际通过相应的协议创建。

今天先理到这，其他的我们在后面的章节中再讲。


# 总结

dubbo框架主要包括消息层，传输层，协议层。消息层提供消费者调用服务请求消息、服务提供方处理
结果响应消息的编解码；传输层主要建立消费者和服务者的通信通道，传输服务请求响应数据；协议层首先是基于相关协议将服
提供者，和消费者通过export暴露出去，即注册的Registry中，消费者通过Registry订阅响应的服务提供者，消费者发现有服务
提供者，则与服务提供者建立连接。服务提供者接受的消费者的服务请求后，根据相关协议，调用响应的Invoker服务。


ServiceAnnotationBeanPostProcessor主要做的事情是扫描 应用先的Service注解bean，并构造ServiceBean，注册到bean注册器中。导出服务实际委托给相应的协议RegistryProtocol


ReferenceBean后处理主要扫描Reference注解的bean，并构造ReferenceBean，ReferenceBean通过Invoker去调用服务提供者的服务，Invoker为服务的包装类。实际通过相应的协议创建。



# 附

[dubbo offical site](https://dubbo.apache.org/zh-cn/index.html)    
[dubbo github](https://github.com/apache/dubbo)   
[dubbo github vt](https://github.com/Donaldhan/incubator-dubbo)  
[incubator-dubbo-spring-boot-project github vt](https://github.com/Donaldhan/incubator-dubbo-spring-boot-project)  
