---
layout: page
title: Dubbo框架设计源码解读二（注册器，服务注册，订阅）
subtitle: RegistryProtocol和ZookpeerRegistry, RegistryDirectory
date: 2020-11-19 22:23:00
author: valuewithTime
catalog: true
category: Dubbo
categories:
    - Dubbo
tags:
    - Dubbo
---

# 引言


dubbo框架主要包括消息层，传输层，协议层。消息层提供消费者调用服务请求消息、服务提供方处理
结果响应消息的编解码；传输层主要建立消费者和服务者的通信通道，传输服务请求响应数据；协议层首先是基于相关协议将服
提供者，和消费者通过export暴露出去，即注册的Registry中，消费者通过Registry订阅响应的服务提供者，消费者发现有服务
提供者，则与服务提供者建立连接。服务提供者接受的消费者的服务请求后，根据相关协议，调用响应的Invoker服务。


ServiceAnnotationBeanPostProcessor主要做的事情是扫描 应用先的Service注解bean，并构造ServiceBean，注册到bean注册器中。导出服务实际委托给相应的协议RegistryProtocol


ReferenceBean后处理主要扫描Reference注解的bean，并构造ReferenceBean，ReferenceBean通过Invoker去调用服务提供者的服务，Invoker为服务的包装类。实际通过相应的协议创建。

这是上一篇的内容，今天我们来看一下应用协议。


# 目录
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


# 概要框架设计


![dubbo-framework](/image/dubbo/dubbo-framework.png)  


dubbo框架设计，主要包括消息层，传输层，协议层。消息层提供消费者调用服务请求消息、服务提供方处理结果响应消息的编解码；传输层主要建立消费者和服务者的通信通道，传输服务请求响应数据；协议层首先是基于相关协议将服务提供者，和消费者通过export暴露出去，即注册的Registry中，消费者通过Registry订阅响应的服务提供者，消费者发现有服务提供者，则与服务提供者建立连接。服务提供者接受的消费者的服务请求后，根据相关协议，调用响应的Invoker服务。


下面我们来从源码来分析Dubbo的各个组件模块。
# 源码分析

[Dubbo框架设计源码解读第一篇（服务和引用bean初始化）](https://donaldhan.github.io/dubbo/2020/11/17/dubbo-framework-service-reference-bean-init.html)  


## 应用协议


### 注册器协议
我们从ServiceConfig导出本地服务来看服务的导出
//SerivceBean
//ServiceConfig
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

服务的导出，依赖于注册协议RegistryProtocol

//RegistryProtocol
```java
/**
 * RegistryProtocol
 *
 */
public class RegistryProtocol implements Protocol {

    private final static Logger logger = LoggerFactory.getLogger(RegistryProtocol.class);
    private static RegistryProtocol INSTANCE;
    private final Map<URL, NotifyListener> overrideListeners = new ConcurrentHashMap<URL, NotifyListener>();
    /**
     *To solve the problem of RMI repeated exposure port conflicts, the services that have been exposed are no longer exposed.
     * providerurl <--> exporter
     */
    private final Map<String, ExporterChangeableWrapper<?>> bounds = new ConcurrentHashMap<String, ExporterChangeableWrapper<?>>();
    private Cluster cluster;
    /**
     * 服务协议
     */
    private Protocol protocol;
    /**
     * 注册器工厂
     */
    private RegistryFactory registryFactory;
    /**
     *
     */
    private ProxyFactory proxyFactory;

    public RegistryProtocol() {
        INSTANCE = this;
    }

    public static RegistryProtocol getRegistryProtocol() {
        if (INSTANCE == null) {
            ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(Constants.REGISTRY_PROTOCOL); // load
        }
        return INSTANCE;
    }

    //Filter the parameters that do not need to be output in url(Starting with .)
    private static String[] getFilteredKeys(URL url) {
        Map<String, String> params = url.getParameters();
        if (params != null && !params.isEmpty()) {
            List<String> filteredKeys = new ArrayList<String>();
            for (Map.Entry<String, String> entry : params.entrySet()) {
                if (entry != null && entry.getKey() != null && entry.getKey().startsWith(Constants.HIDE_KEY_PREFIX)) {
                    filteredKeys.add(entry.getKey());
                }
            }
            return filteredKeys.toArray(new String[filteredKeys.size()]);
        } else {
            return new String[]{};
        }
    }
    ...
    **
     * 导出服务，暴露的值服务者还是消费者
     * @param originInvoker
     * @param <T>
     * @return
     * @throws RpcException
     */
    @Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        //export invoker
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
        //获取服务提供者的注册器URL
        URL registryUrl = getRegistryUrl(originInvoker);

        //registry provider
        final Registry registry = getRegistry(originInvoker);
        //获取服务提供者URL
        final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);

        //to judge to delay publish whether or not
        boolean register = registeredProviderUrl.getParameter("register", true);
        //注册到服务消费者注册Table
        ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registeredProviderUrl);

        if (register) {
            //设置为已注册
            register(registryUrl, registeredProviderUrl);
            ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
        }

        // Subscribe the override data
        // FIXME
        // When the provider subscribes, it will affect the scene :
        // a certain JVM exposes the service and call the same service.
        // Because the subscribed is cached key with the name of the service, it causes the subscription information to cover.
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
        //org.apache.dubbo.registry.zookeeper.ZookeeperRegistry.doSubscribe
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
        //Ensure that a new exporter instance is returned every time export
        return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registeredProviderUrl);
    }
}
```


导出服务关键有3个分别如下；


1. 获取注册器
```java
//registry provider
final Registry registry = getRegistry(originInvoker);
```

2. 注册服务
```java
//to judge to delay publish whether or not
boolean register = registeredProviderUrl.getParamete("register", true);
//注册到服务消费者注册Table
ProviderConsumerRegTable.registerProvider(originInvoker,registryUrl, registeredProviderUrl);
if (register) {
    //设置为已注册
    register(registryUrl, registeredProviderUrl);
    ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
}
```

3. 订阅服务
```java
//org.apache.dubbo.registry.zookeeper.ZookeeperRegistry.doSubscribe
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
```



下面我们分别来看这3点


### 获取注册器
```java
//registry provider
final Registry registry = getRegistry(originInvoker);
```

//RegistryProtocol
```java
/**
     * Get an instance of registry based on the address of invoker
     *
     * @param originInvoker
     * @return
     */
    private Registry getRegistry(final Invoker<?> originInvoker) {
        URL registryUrl = getRegistryUrl(originInvoker);
        return registryFactory.getRegistry(registryUrl);
    }
    /**
     * Boot
     * //        @Bean
     *     public RegistryConfig dubboRegistry() {
     *         RegistryConfig registry = new RegistryConfig();
     * //        registry.setAddress(environment.getProperty("zookeeper_server"));
     *         registry.setAddress("zookeeper://127.0.0.1:2181");
     *         return registry;
     *     }
     *
     * <dubbo:registry address="zookeeper://10.20.153.10:2181?backup=10.20.153.11:2181,10.20.153.12:2181" />
     * @param originInvoker
     * @return
     */
    private URL getRegistryUrl(Invoker<?> originInvoker) {
        URL registryUrl = originInvoker.getUrl();
        if (Constants.REGISTRY_PROTOCOL.equals(registryUrl.getProtocol())) {
            String protocol = registryUrl.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_DIRECTORY);
            registryUrl = registryUrl.setProtocol(protocol).removeParameter(Constants.REGISTRY_KEY);
        }
        return registryUrl;
    }
```


//ZookeeperRegistryFactory
```java
/**
 * ZookeeperRegistryFactory.
 *
 */
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {

    /**
     *
     */
    private ZookeeperTransporter zookeeperTransporter;

    public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
        this.zookeeperTransporter = zookeeperTransporter;
    }

    @Override
    public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }

}
```

注册器，比较常用的是zookeeper注册器，下面我们要将的注册协议，也是以zookeeper注册协议为主。



### 服务注册
```java
//to judge to delay publish whether or not
boolean register = registeredProviderUrl.getParamete("register", true);
//注册到服务消费者注册Table
ProviderConsumerRegTable.registerProvider(originInvoker,registryUrl, registeredProviderUrl);
if (register) {
    //设置为已注册
    register(registryUrl, registeredProviderUrl);
    ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
}
```


//RegistryProtocol
```java
  /**
     * 注册服务
     * @param registryUrl
     * @param registedProviderUrl
     */
    public void register(URL registryUrl, URL registedProviderUrl) {
        Registry registry = registryFactory.getRegistry(registryUrl);
        //org.apache.dubbo.registry.zookeeper.ZookeeperRegistry.doRegister
        registry.register(registedProviderUrl);
    }

```


```java
/**
 * FailbackRegistry. (SPI, Prototype, ThreadSafe)
 */
public abstract class FailbackRegistry extends AbstractRegistry {

    /*  retry task map */

    private final ConcurrentMap<URL, FailedRegisteredTask> failedRegistered = new ConcurrentHashMap<URL, FailedRegisteredTask>();

    private final ConcurrentMap<URL, FailedUnregisteredTask> failedUnregistered = new ConcurrentHashMap<URL, FailedUnregisteredTask>();

    private final ConcurrentMap<Holder, FailedSubscribedTask> failedSubscribed = new ConcurrentHashMap<Holder, FailedSubscribedTask>();

    private final ConcurrentMap<Holder, FailedUnsubscribedTask> failedUnsubscribed = new ConcurrentHashMap<Holder, FailedUnsubscribedTask>();

    private final ConcurrentMap<Holder, FailedNotifiedTask> failedNotified = new ConcurrentHashMap<Holder, FailedNotifiedTask>();

    /**
     * The time in milliseconds the retryExecutor will wait
     */
    private final int retryPeriod;

    // Timer for failure retry, regular check if there is a request for failure, and if there is, an unlimited retry
    private final HashedWheelTimer retryTimer;

    public FailbackRegistry(URL url) {
        super(url);
        this.retryPeriod = url.getParameter(Constants.REGISTRY_RETRY_PERIOD_KEY, Constants.DEFAULT_REGISTRY_RETRY_PERIOD);

        // since the retry task will not be very much. 128 ticks is enough.
        retryTimer = new HashedWheelTimer(new NamedThreadFactory("DubboRegistryRetryTimer", true), retryPeriod, TimeUnit.MILLISECONDS, 128);
    }
    ...
@Override
    public void register(URL url) {
        super.register(url);
        removeFailedRegistered(url);
        removeFailedUnregistered(url);
        try {
            // Sending a registration request to the server side,  具体的注册实现，有具体的注册器实现
            doRegister(url);
        } catch (Exception e) {
            Throwable t = e;

            // If the startup detection is opened, the Exception is thrown directly.
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true)
                    && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }

            // Record a failed registration request to a failed list, retry regularly
            addFailedRegistered(url);
        }
    }
    ...
}
```
来看FailbackRegistry的父类AbstractRegistry的注册实现

//AbstractRegistry
```java
/**
 * AbstractRegistry. (SPI, Prototype, ThreadSafe)
 *
 */
public abstract class AbstractRegistry implements Registry {

    // URL address separator, used in file cache, service provider URL separation
    private static final char URL_SEPARATOR = ' ';
    // URL address separated regular expression for parsing the service provider URL list in the file cache
    private static final String URL_SPLIT = "\\s+";
    // Log output
    protected final Logger logger = LoggerFactory.getLogger(getClass());
    // Local disk cache, where the special key value.registies records the list of registry centers, and the others are the list of notified service providers
    private final Properties properties = new Properties();
    // File cache timing writing
    private final ExecutorService registryCacheExecutor = Executors.newFixedThreadPool(1, new NamedThreadFactory("DubboSaveRegistryCache", true));
    // Is it synchronized to save the file
    private final boolean syncSaveFile;
    private final AtomicLong lastCacheChanged = new AtomicLong();
    /**
     * 服务注册
     */
    private final Set<URL> registered = new ConcurrentHashSet<URL>();
    /**
     * 订阅者
     */
    private final ConcurrentMap<URL, Set<NotifyListener>> subscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
    /**
     * 订阅者通知URL
     */
    private final ConcurrentMap<URL, Map<String, List<URL>>> notified = new ConcurrentHashMap<URL, Map<String, List<URL>>>();
    private URL registryUrl;
    // Local disk cache file, 本地缓存文件
    private File file;

    public AbstractRegistry(URL url) {
        setUrl(url);
        // Start file save timer
        syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
        String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getParameter(Constants.APPLICATION_KEY) + "-" + url.getAddress() + ".cache");
        File file = null;
        if (ConfigUtils.isNotEmpty(filename)) {
            file = new File(filename);
            if (!file.exists() && file.getParentFile() != null && !file.getParentFile().exists()) {
                if (!file.getParentFile().mkdirs()) {
                    throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
                }
            }
        }
        this.file = file;
        loadProperties();
        notify(url.getBackupUrls());
    }
...
 @Override
    public void register(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("register url == null");
        }
        if (logger.isInfoEnabled()) {
            logger.info("Register: " + url);
        }
        registered.add(url);
    }
    ...
}
```

实际为将注册URL添加到相应的注册器ConcurrentHashSet中。

再来看实际注册器的实现


//ZookeeperRegistry
```java
/**
 * ZookeeperRegistry
 *
 */
public class ZookeeperRegistry extends FailbackRegistry {

    private final static Logger logger = LoggerFactory.getLogger(ZookeeperRegistry.class);

    private final static int DEFAULT_ZOOKEEPER_PORT = 2181;

    private final static String DEFAULT_ROOT = "dubbo";

    private final String root;

    private final Set<String> anyServices = new ConcurrentHashSet<String>();

    /**
     * 再zk节点监控器
     */
    private final ConcurrentMap<URL, ConcurrentMap<NotifyListener, ChildListener>> zkListeners = new ConcurrentHashMap<URL, ConcurrentMap<NotifyListener, ChildListener>>();

    private final ZookeeperClient zkClient;
  public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
        super(url);
        if (url.isAnyHost()) {
            throw new IllegalStateException("registry address == null");
        }
        String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
        if (!group.startsWith(Constants.PATH_SEPARATOR)) {
            group = Constants.PATH_SEPARATOR + group;
        }
        this.root = group;
        zkClient = zookeeperTransporter.connect(url);
        zkClient.addStateListener(new StateListener() {
            @Override
            public void stateChanged(int state) {
                if (state == RECONNECTED) {
                    try {
                        //重连恢复
                        recover();
                    } catch (Exception e) {
                        logger.error(e.getMessage(), e);
                    }
                }
            }
        });
    }
    ...
    @Override
    public void doRegister(URL url) {
        try {
            //创建持计划服务路径
            zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
}
```

从上面可以看出，注册服务，实际是将基于Dubbo协议的服务URL写到ZK上，如何在注册的过程中，由于Dubbo自身机制导致的注册失败，将加入的失败注册集，并有定时钟，进行重试注册。


我们再看订阅服务

### 订阅服务
```java
//org.apache.dubbo.registry.zookeeper.ZookeeperRegistry.doSubscribe
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
```


//FailbackRegistry

```java
 @Override
    public void subscribe(URL url, NotifyListener listener) {
        super.subscribe(url, listener);
        removeFailedSubscribed(url, listener);
        try {
            // Sending a subscription request to the server side
            doSubscribe(url, listener);
        } catch (Exception e) {
            Throwable t = e;

            List<URL> urls = getCacheUrls(url);
            if (urls != null && !urls.isEmpty()) {
                notify(url, listener, urls);
                logger.error("Failed to subscribe " + url + ", Using cached list: " + urls + " from cache file: " + getUrl().getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/dubbo-registry-" + url.getHost() + ".cache") + ", cause: " + t.getMessage(), t);
            } else {
                // If the startup detection is opened, the Exception is thrown directly.
                boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                        && url.getParameter(Constants.CHECK_KEY, true);
                boolean skipFailback = t instanceof SkipFailbackWrapperException;
                if (check || skipFailback) {
                    if (skipFailback) {
                        t = t.getCause();
                    }
                    throw new IllegalStateException("Failed to subscribe " + url + ", cause: " + t.getMessage(), t);
                } else {
                    logger.error("Failed to subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
                }
            }

            // Record a failed registration request to a failed list, retry regularly
            addFailedSubscribed(url, listener);
        }
    }
```

//AbstractRegistry
```java
  @Override
    public void subscribe(URL url, NotifyListener listener) {
        if (url == null) {
            throw new IllegalArgumentException("subscribe url == null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("subscribe listener == null");
        }
        if (logger.isInfoEnabled()) {
            logger.info("Subscribe: " + url);
        }
        Set<NotifyListener> listeners = subscribed.get(url);
        if (listeners == null) {
            subscribed.putIfAbsent(url, new ConcurrentHashSet<NotifyListener>());
            listeners = subscribed.get(url);
        }
        listeners.add(listener);
    }
```


//ZookeeperRegistry
```java
@Override
    public void doSubscribe(final URL url, final NotifyListener listener) {
        try {
            if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
                String root = toRootPath();
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                if (listeners == null) {
                    zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                    listeners = zkListeners.get(url);
                }
                ChildListener zkListener = listeners.get(listener);
                if (zkListener == null) {
                    listeners.putIfAbsent(listener, new ChildListener() {
                        @Override
                        public void childChanged(String parentPath, List<String> currentChilds) {
                            for (String child : currentChilds) {
                                child = URL.decode(child);
                                if (!anyServices.contains(child)) {
                                    anyServices.add(child);
                                    subscribe(url.setPath(child).addParameters(Constants.INTERFACE_KEY, child,
                                            Constants.CHECK_KEY, String.valueOf(false)), listener);
                                }
                            }
                        }
                    });
                    zkListener = listeners.get(listener);
                }
                zkClient.create(root, false);
                List<String> services = zkClient.addChildListener(root, zkListener);
                if (services != null && !services.isEmpty()) {
                    for (String service : services) {
                        service = URL.decode(service);
                        anyServices.add(service);
                        subscribe(url.setPath(service).addParameters(Constants.INTERFACE_KEY, service,
                                Constants.CHECK_KEY, String.valueOf(false)), listener);
                    }
                }
            } else {
                List<URL> urls = new ArrayList<URL>();
                for (String path : toCategoriesPath(url)) {
                    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                    if (listeners == null) {
                        zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                        listeners = zkListeners.get(url);
                    }
                    ChildListener zkListener = listeners.get(listener);
                    if (zkListener == null) {
                        listeners.putIfAbsent(listener, new ChildListener() {
                            @Override
                            public void childChanged(String parentPath, List<String> currentChilds) {
                                //节点服务节点有变化，则通知服务订阅值
                                ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                            }
                        });
                        zkListener = listeners.get(listener);
                    }
                    zkClient.create(path, false);
                    List<String> children = zkClient.addChildListener(path, zkListener);
                    if (children != null) {
                        urls.addAll(toUrlsWithEmpty(url, path, children));
                    }
                }
                notify(url, listener, urls);
            }
        } catch (Throwable e) {
            throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
从上面可以，订阅服务，监听服务提供者的节点路径。

这是服务注册和订阅，我们再来看一下消费者的注册


//RegistryProtocol
```java
 /**
     * @param type Service class
     * @param url  URL address for the remote service
     * @param <T>
     * @return
     * @throws RpcException
     */
    @Override
    @SuppressWarnings("unchecked")
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
        Registry registry = registryFactory.getRegistry(url);
        if (RegistryService.class.equals(type)) {
            return proxyFactory.getInvoker((T) registry, type, url);
        }

        // group="a,b" or group="*"
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
        String group = qs.get(Constants.GROUP_KEY);
        if (group != null && group.length() > 0) {
            if ((Constants.COMMA_SPLIT_PATTERN.split(group)).length > 1
                    || "*".equals(group)) {
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
        return doRefer(cluster, registry, type, url);
    }
    /**
     * @param cluster
     * @param registry
     * @param type
     * @param url
     * @param <T>
     * @return
     */
    private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        //org.apache.dubbo.registry.integration.RegistryDirectory.toInvokers
        //org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol.refer
        directory.setProtocol(protocol);
        // all attributes of REFER_KEY
        Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
        URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);
        if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
                && url.getParameter(Constants.REGISTER_KEY, true)) {
            //注册消费者
            registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                    Constants.CHECK_KEY, String.valueOf(false)));
        }
        directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
                Constants.PROVIDERS_CATEGORY
                        + "," + Constants.CONFIGURATORS_CATEGORY
                        + "," + Constants.ROUTERS_CATEGORY));
        //合并服务
        Invoker invoker = cluster.join(directory);
        //注册消费者
        ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
        return invoker;
    }
```

从上可以看出，消费者注册到ZK上的订阅服务节点上，具体的订阅委托给目录服务。


我们来看一下注册目录服务

```java
/**
 * RegistryDirectory
 */
public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener {

    private static final Logger logger = LoggerFactory.getLogger(RegistryDirectory.class);

    /**
     *
     */
    private static final Cluster cluster = ExtensionLoader.getExtensionLoader(Cluster.class).getAdaptiveExtension();

    private static final RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getAdaptiveExtension();

    private static final ConfiguratorFactory configuratorFactory = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class).getAdaptiveExtension();
    /**
     *  Initialization at construction time, assertion not null
     */
    private final String serviceKey;
    /**
     * Initialization at construction time, assertion not null
     */
    private final Class<T> serviceType;
    /**
     *  Initialization at construction time, assertion not null
     */
    private final Map<String, String> queryMap;
    /**
     * Initialization at construction time, assertion not null, and always assign non null value
     */
    private final URL directoryUrl;
    private final String[] serviceMethods;
    private final boolean multiGroup;
    /**
     * Initialization at the time of injection, the assertion is not null
     */
    private Protocol protocol;
    /**
     * Initialization at the time of injection, the assertion is not null
     */
    private Registry registry;
    private volatile boolean forbidden = false;

    /**
     *  Initialization at construction time, assertion not null, and always assign non null value
     */
    private volatile URL overrideDirectoryUrl;

    /**
     * override rules
     * Priority: override>-D>consumer>provider
     * Rule one: for a certain provider <ip:port,timeout=100>
     * Rule two: for all providers <* ,timeout=5000>
     *     The initial value is null and the midway may be assigned to null, please use the local variable reference
     */
    private volatile List<Configurator> configurators;

    /**
     * Map<url, Invoker> cache service url to invoker mapping.
     * The initial value is null and the midway may be assigned to null, please use the local variable reference
     */
    private volatile Map<String, Invoker<T>> urlInvokerMap;

    /**
     * Map<methodName, Invoker> cache service method to invokers mapping.
     * The initial value is null and the midway may be assigned to null, please use the local variable reference
     */
    private volatile Map<String, List<Invoker<T>>> methodInvokerMap;
    /**
     * Set<invokerUrls> cache invokeUrls to invokers mapping.
     * 缓存服务URL
     * The initial value is null and the midway may be assigned to null, please use the local variable reference
     */
    private volatile Set<URL> cachedInvokerUrls;
    public RegistryDirectory(Class<T> serviceType, URL url) {
        super(url);
        if (serviceType == null) {
            throw new IllegalArgumentException("service type is null.");
        }
        if (url.getServiceKey() == null || url.getServiceKey().length() == 0) {
            throw new IllegalArgumentException("registry serviceKey is null.");
        }
        this.serviceType = serviceType;
        this.serviceKey = url.getServiceKey();
        this.queryMap = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
        this.overrideDirectoryUrl = this.directoryUrl = url.setPath(url.getServiceInterface()).clearParameters().addParameters(queryMap).removeParameter(Constants.MONITOR_KEY);
        String group = directoryUrl.getParameter(Constants.GROUP_KEY, "");
        this.multiGroup = group != null && ("*".equals(group) || group.contains(","));
        String methods = queryMap.get(Constants.METHODS_KEY);
        this.serviceMethods = methods == null ? null : Constants.COMMA_SPLIT_PATTERN.split(methods);
    }
    ...
    /**
     * 订阅服务 key point
     * @param url
     */
    public void subscribe(URL url) {
        setConsumerUrl(url);
        registry.subscribe(url, this);
    }
    ...
}

```

在消费者注册到注册器时，有关键的一句
//RegistryProtocol
```java
//合并服务
        Invoker invoker = cluster.join(directory);
```
//AvailableCluster
```java
/**
 * AvailableCluster
 * 可利用族
 */
public class AvailableCluster implements Cluster {

    public static final String NAME = "available";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {

        return new AbstractClusterInvoker<T>(directory) {
            @Override
            public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
                for (Invoker<T> invoker : invokers) {
                    if (invoker.isAvailable()) {
                        return invoker.invoke(invocation);
                    }
                }
                throw new RpcException("No provider available in " + invokers);
            }
        };

    }
```

//AbstractClusterInvoker
```java
**
 * AbstractClusterInvoker
 */
public abstract class AbstractClusterInvoker<T> implements Invoker<T> {

    private static final Logger logger = LoggerFactory.getLogger(AbstractClusterInvoker.class);

    /**
     *
     */
    protected final Directory<T> directory;

    /**
     *
     */
    protected final boolean availablecheck;

    private AtomicBoolean destroyed = new AtomicBoolean(false);
    ...
      @Override
    public Result invoke(final Invocation invocation) throws RpcException {
        checkWhetherDestroyed();

        // binding attachments into invocation.
        Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
        if (contextAttachments != null && contextAttachments.size() != 0) {
            ((RpcInvocation) invocation).addAttachments(contextAttachments);
        }

        List<Invoker<T>> invokers = list(invocation);
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        return doInvoke(invocation, invokers, loadbalance);
    }
 /**
     * @param invocation
     * @return
     * @throws RpcException
     */
    protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
        return directory.list(invocation);
    }
}
```

//AbstractDirectory
```java
/**
 * Abstract implementation of Directory: Invoker list returned from this Directory's list method have been filtered by Routers
 *
 */
public abstract class AbstractDirectory<T> implements Directory<T> {

    // logger
    private static final Logger logger = LoggerFactory.getLogger(AbstractDirectory.class);

    /**
     *
     */
    private final URL url;

    private volatile boolean destroyed = false;

    /**
     * 消费者uRl
     */
    private volatile URL consumerUrl;

    /**
     * 服务路由器
     */
    private volatile List<Router> routers;
     public AbstractDirectory(URL url) {
        this(url, null);
    }

    public AbstractDirectory(URL url, List<Router> routers) {
        this(url, url, routers);
    }

    public AbstractDirectory(URL url, URL consumerUrl, List<Router> routers) {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }

        if (url.getProtocol().equals(Constants.REGISTRY_PROTOCOL)) {
            Map<String, String> queryMap = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
            this.url = url.clearParameters().addParameters(queryMap).removeParameter(Constants.MONITOR_KEY);
        } else {
            this.url = url;
        }

        this.consumerUrl = consumerUrl;
        setRouters(routers);
    }
    @Override
    public List<Invoker<T>> list(Invocation invocation) throws RpcException {
        if (destroyed) {
            throw new RpcException("Directory already destroyed .url: " + getUrl());
        }
        List<Invoker<T>> invokers = doList(invocation);
        // local reference， 本地路由
        List<Router> localRouters = this.routers;
        if (localRouters != null && !localRouters.isEmpty()) {
            for (Router router : localRouters) {
                try {
                    if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, false)) {
                        invokers = router.route(invokers, getConsumerUrl(), invocation);
                    }
                } catch (Throwable t) {
                    logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
                }
            }
        }
        return invokers;
    }
    ...
}
```

//RegistryDirectory
```java
@Override
    public List<Invoker<T>> doList(Invocation invocation) {
        if (forbidden) {
            // 1. No service provider 2. Service providers are disabled
            throw new RpcException(RpcException.FORBIDDEN_EXCEPTION,
                    "No provider available from registry " + getUrl().getAddress() + " for service " + getConsumerUrl().getServiceKey() + " on consumer " + NetUtils.getLocalHost()
                            + " use dubbo version " + Version.getVersion() + ", please check status of providers(disabled, not registered or in blacklist).");
        }
        List<Invoker<T>> invokers = null;
        // local reference
        Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap;
        if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
            String methodName = RpcUtils.getMethodName(invocation);
            Object[] args = RpcUtils.getArguments(invocation);
            if (args != null && args.length > 0 && args[0] != null
                    && (args[0] instanceof String || args[0].getClass().isEnum())) {
                // The routing can be enumerated according to the first parameter
                invokers = localMethodInvokerMap.get(methodName + "." + args[0]);
            }
            if (invokers == null) {
                invokers = localMethodInvokerMap.get(methodName);
            }
            if (invokers == null) {
                invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
            }
        }
        return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
    }
```



注册目录服务依赖于注册器，消费者从注册器获取服务提供者，实际为从注册目录服务获取服务列表（zk注册器为，服务节点下的提供者），并根据路由策略，选择可用的服务提供者Invoker。

再来看一下注册服务目录的监听机制
//RegistryDirectory
```java
 @Override
    public synchronized void notify(List<URL> urls) {
        List<URL> invokerUrls = new ArrayList<URL>();
        List<URL> routerUrls = new ArrayList<URL>();
        List<URL> configuratorUrls = new ArrayList<URL>();
        for (URL url : urls) {
            String protocol = url.getProtocol();
            String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
            if (Constants.ROUTERS_CATEGORY.equals(category)
                    || Constants.ROUTE_PROTOCOL.equals(protocol)) {
                //服务路由
                routerUrls.add(url);
            } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
                    || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
                //配置，重载协议
                configuratorUrls.add(url);
            } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
                //提供者
                invokerUrls.add(url);
            } else {
                logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
            }
        }
        // configurators
        if (configuratorUrls != null && !configuratorUrls.isEmpty()) {
            this.configurators = toConfigurators(configuratorUrls);
        }
        // routers
        if (routerUrls != null && !routerUrls.isEmpty()) {
            List<Router> routers = toRouters(routerUrls);
            if (routers != null) { // null - do nothing
                setRouters(routers);
            }
        }
        List<Configurator> localConfigurators = this.configurators; // local reference
        // merge override parameters
        this.overrideDirectoryUrl = directoryUrl;
        if (localConfigurators != null && !localConfigurators.isEmpty()) {
            for (Configurator configurator : localConfigurators) {
                this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
            }
        }
        // providers ， 刷新服务
        refreshInvoker(invokerUrls);
    }
    /* Convert the invokerURL list to the Invoker Map. The rules of the conversion are as follows:
     * 转换服务提供URL为MAP，规则如下：
     * 1.If URL has been converted to invoker, it is no longer re-referenced and obtained directly from the cache,
     * and notice that any parameter changes in the URL will be re-referenced.
     * 如果URL已经转换为Invoker，不在重新索引，注解从缓存中获取，任务URL的参数改变，将会通知重新索引
     * 2.If the incoming invoker list is not empty, it means that it is the latest invoker list
     * 如果invokerUrls，不为空，则为最近的服务List
     * 3.If the list of incoming invokerUrl is empty, It means that the rule is only a override rule or a route rule,
     * which needs to be re-contrasted to decide whether to re-reference.
     * 如果服务提供者URL为空，则意味着，规则或路由规则重写，需要重新确定是否需要重新索引服务提供者
     * 2017/8/31 FIXME The thread pool should be used to refresh the address, otherwise the task may be accumulated.
     * @param invokerUrls this parameter can't be null
     */
    private void refreshInvoker(List<URL> invokerUrls) {
        if (invokerUrls != null && invokerUrls.size() == 1 && invokerUrls.get(0) != null
                && Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
            //空服务列表
            // Forbid to access
            this.forbidden = true;
            // Set the method invoker map to null
            this.methodInvokerMap = null;
            // Close all invokers
            destroyAllInvokers();
        } else {
            this.forbidden = false; // Allow to access
            Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
            if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
                invokerUrls.addAll(this.cachedInvokerUrls);
            } else {
                this.cachedInvokerUrls = new HashSet<URL>();
                //Cached invoker urls, convenient for comparison， 重新缓存
                this.cachedInvokerUrls.addAll(invokerUrls);
            }
            if (invokerUrls.isEmpty()) {
                return;
            }
            // Translate url list to Invoker map
            Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);
            // Change method name to map Invoker Map
            Map<String, List<Invoker<T>>> newMethodInvokerMap = toMethodInvokers(newUrlInvokerMap);
            // state change
            // If the calculation is wrong, it is not processed.
            if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
                logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls.toString()));
                return;
            }
            this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
            this.urlInvokerMap = newUrlInvokerMap;
            try {
                destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
            } catch (Exception e) {
                logger.warn("destroyUnusedInvokers error. ", e);
            }
        }
    }
    /**
     * Turn urls into invokers, and if url has been refer, will not re-reference.
     * 转为URL为服务，如果已经索引，不在重新索引
     * @param urls
     * @return invokers
     */
    private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
        Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<String, Invoker<T>>();
        if (urls == null || urls.isEmpty()) {
            return newUrlInvokerMap;
        }
        Set<String> keys = new HashSet<String>();
        String queryProtocols = this.queryMap.get(Constants.PROTOCOL_KEY);
        for (URL providerUrl : urls) {
            // If protocol is configured at the reference side, only the matching protocol is selected
            //选择匹配reference端的协议
            if (queryProtocols != null && queryProtocols.length() > 0) {
                boolean accept = false;
                String[] acceptProtocols = queryProtocols.split(",");
                for (String acceptProtocol : acceptProtocols) {
                    if (providerUrl.getProtocol().equals(acceptProtocol)) {
                        accept = true;
                        break;
                    }
                }
                if (!accept) {
                    continue;
                }
            }
            if (Constants.EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
                continue;
            }
            if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
                logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() + " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost()
                        + ", supported protocol: " + ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
                continue;
            }
            //合并url采纳数
            URL url = mergeUrl(providerUrl);
    // The parameter urls are sorted
            String key = url.toFullString();
            // Repeated url
            if (keys.contains(key)) {
                continue;
            }
            keys.add(key);
            // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters,
            // if the server url changes, then refer again
            Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap;
            Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
            if (invoker == null) {
                // Not in the cache, refer again
                try {
                    boolean enabled = true;
                    if (url.hasParameter(Constants.DISABLED_KEY)) {
                        enabled = !url.getParameter(Constants.DISABLED_KEY, false);
                    } else {
                        enabled = url.getParameter(Constants.ENABLED_KEY, true);
                    }
                    if (enabled) {
                        //服务代理，key point, org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol.refer
                        invoker = new InvokerDelegate<T>(protocol.refer(serviceType, url), url, providerUrl);
                    }
                } catch (Throwable t) {
                    logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
                }
                if (invoker != null) { // Put new invoker in cache
                    newUrlInvokerMap.put(key, invoker);
                }
            } else {
                newUrlInvokerMap.put(key, invoker);
            }
        }
        keys.clear();
        return newUrlInvokerMap;
    }
     /**
     * Transform the invokers list into a mapping relationship with a method
     *
     * @param invokersMap Invoker Map
     * @return Mapping relation between Invoker and method
     */
    private Map<String, List<Invoker<T>>> toMethodInvokers(Map<String, Invoker<T>> invokersMap) {
        Map<String, List<Invoker<T>>> newMethodInvokerMap = new HashMap<String, List<Invoker<T>>>();
        // According to the methods classification declared by the provider URL, the methods is compatible with the registry to execute the filtered methods
        List<Invoker<T>> invokersList = new ArrayList<Invoker<T>>();
        if (invokersMap != null && invokersMap.size() > 0) {
            for (Invoker<T> invoker : invokersMap.values()) {
                String parameter = invoker.getUrl().getParameter(Constants.METHODS_KEY);
                if (parameter != null && parameter.length() > 0) {
                    String[] methods = Constants.COMMA_SPLIT_PATTERN.split(parameter);
                    if (methods != null && methods.length > 0) {
                        for (String method : methods) {
                            if (method != null && method.length() > 0
                                    && !Constants.ANY_VALUE.equals(method)) {
                                List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
                                if (methodInvokers == null) {
                                    methodInvokers = new ArrayList<Invoker<T>>();
                                    newMethodInvokerMap.put(method, methodInvokers);
                                }
                                methodInvokers.add(invoker);
                            }
                        }
                    }
                }
                invokersList.add(invoker);
            }
        }
        List<Invoker<T>> newInvokersList = route(invokersList, null);
        newMethodInvokerMap.put(Constants.ANY_VALUE, newInvokersList);
        if (serviceMethods != null && serviceMethods.length > 0) {
            for (String method : serviceMethods) {
                List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
                if (methodInvokers == null || methodInvokers.isEmpty()) {
                    methodInvokers = newInvokersList;
                }
                newMethodInvokerMap.put(method, route(methodInvokers, method));
            }
        }
        // sort and unmodifiable
        for (String method : new HashSet<String>(newMethodInvokerMap.keySet())) {
            List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
            Collections.sort(methodInvokers, InvokerComparator.getComparator());
            newMethodInvokerMap.put(method, Collections.unmodifiableList(methodInvokers));
        }
        return Collections.unmodifiableMap(newMethodInvokerMap);
    }
```


注册器目录处理提供服务路由，同时监听服务的变化。如果注册器节点信息存在变化，则重新刷新服务，建立服务Invoker索引。


# 总结

注册协议，导出服务主要有注册服务和订阅服务。注册服务，实际是将基于Dubbo协议的服务URL写到ZK上，如何在注册的过程中，由于Dubbo自身机制导致的注册失败，将加入的失败注册集，并有定时钟，进行重试注册。订阅服务，监听服务提供者的节点路径。消费者注册到ZK上的订阅服务节点上，具体的订阅委托给目录服务。

注册目录服务依赖于注册器，消费者从注册器获取服务提供者，实际为从注册目录服务获取服务列表（zk注册器为，服务节点下的提供者），并根据路由策略，选择可用的服务提供者Invoker。注册器目录处理提供服务路由，同时监听服务的变化。如果注册器节点信息存在变化，则重新刷新服务，建立服务Invoker索引。


# 附

[dubbo offical site](https://dubbo.apache.org/zh-cn/index.html)    
[dubbo github](https://github.com/apache/dubbo)   
[dubbo github vt](https://github.com/Donaldhan/incubator-dubbo)  
[incubator-dubbo-spring-boot-project github vt](https://github.com/Donaldhan/incubator-dubbo-spring-boot-project)  
