---
layout: page
title: Dubbo框架设计源码解读三（Dubbo协议，服务导出，引用）
subtitle: dubbo-protocol
date: 2020-11-19 23:17:00
author: valuewithTime
catalog: true
category: Dubbo
categories:
    - Dubbo
tags:
    - Dubbo
---

# 引言
注册协议，导出服务主要有注册服务和订阅服务。注册服务，实际是将基于Dubbo协议的服务URL写到ZK上，如何在注册的过程中，由于Dubbo自身机制导致的注册失败，将加入的失败注册集，并有定时钟，进行重试注册。订阅服务，监听服务提供者的节点路径。消费者注册到ZK上的订阅服务节点上，具体的订阅委托给目录服务。

注册目录服务依赖于注册器，消费者从注册器获取服务提供者，实际为从注册目录服务获取服务列表（zk注册器为，服务节点下的提供者），并根据路由策略，选择可用的服务提供者Invoker。注册器目录处理提供服务路由，同时监听服务的变化。如果注册器节点信息存在变化，则重新刷新服务，建立服务Invoker索引。

这是上一篇注册协议的内容，今天我们来看一下dubbo协议。

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


dubbo框架主要包括序列化，消息层，传输层，协议层。序列化主要是请求消息和响应消息的序列化，比如基于Javad的ObjectOut/InputStream序列化、基于JSON的序列化。消息层提供消费者调用服务请求消息、服务提供方处理
结果响应消息的编解码；传输层主要建立消费者和服务者的通信通道，传输服务请求响应数据，比如基于Netty和Mina的，默认为Netty；协议层首先是基于相关协议将服提供者，和消费者通过export暴露出去，即注册器Registry中，消费者通过Registry订阅响应的服务提供者，消费者发现有服务
提供者，则与服务提供者建立连接，注册协议有基于Zookeeper，Redis等，在注册协议中还有一个注册器目录服务，用于提供消费者和服务者列表，及根据负载均衡策略选择服务者。服务提供者接受的消费者的服务请求后，根据相关协议，调用相应的Invoker服务。 消费者和服务者的RPC调用协议，实际在DubboProtocol中，协议首先导出服务，消费者发送RPC请求，调用Exporter服务容器中的Invoker。


下面我们来从源码来分析Dubbo的各个组件模块。
# 源码分析

[Dubbo框架设计源码解读一（服务和引用bean初始化）](https://donaldhan.github.io/dubbo/2020/11/17/dubbo-framework-service-reference-bean-init.html)  



## 应用协议


### 注册器协议

[Dubbo框架设计源码解读二（注册器，服务注册，订阅）](https://donaldhan.github.io/dubbo/2020/11/19/dubbo-framework-application-registry-protocol.html)


服务导出Exporter、服务Invoker、服务注册、订阅服务；

### Dubbo协议
Dubbo协议是消费者和服务者通信的基础，包括服务的调用。

在上一篇注册协议和注册器目录中，引用的一个Protocol，这个协议根据SPI模式为DubboProtocol，以我们一般使用的都是这个协议。

注册器协议中，有如下一个功能，导出服务到本地， 实际委托的相应的协议，比如Dubbo协议DubboProtocol的export操作。

//RegistryProtocol
```java
/**
     * Reexport the invoker of the modified url
     *
     * @param originInvoker
     * @param newInvokerUrl
     */
    @SuppressWarnings("unchecked")
    private <T> void doChangeLocalExport(final Invoker<T> originInvoker, URL newInvokerUrl) {
        String key = getCacheKey(originInvoker);
        final ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
        if (exporter == null) {
            logger.warn(new IllegalStateException("error state, exporter should not be null"));
        } else {
            final Invoker<T> invokerDelegete = new InvokerDelegete<T>(originInvoker, newInvokerUrl);
            exporter.setExporter(protocol.export(invokerDelegete));
        }
    }
```

另外一个在注册目录中，当前监听注册器节点变化是，重新索引服务，在转换URL为Invoker，实际委托的相应的协议，比如Dubbo协议DubboProtocol的
refer操作。

//RegistryDirectory
```java
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
```

下面我们分别来看这两个关键部分
#### 导出服务

```java
/**
 * dubbo protocol support.
 */
public class DubboProtocol extends AbstractProtocol {

    public static final String NAME = "dubbo";

    public static final int DEFAULT_PORT = 20880;
    /**
     * 服务回调
     */
    private static final String IS_CALLBACK_SERVICE_INVOKE = "_isCallBackServiceInvoke";
    /**
     *
     */
    private static DubboProtocol INSTANCE;
    /**
     *
     */
    private final Map<String, ExchangeServer> serverMap = new ConcurrentHashMap<String, ExchangeServer>(); // <host:port,Exchanger>
    /**
     *
     */
    private final Map<String, ReferenceCountExchangeClient> referenceClientMap = new ConcurrentHashMap<String, ReferenceCountExchangeClient>(); // <host:port,Exchanger>
    /**
     *
     */
    private final ConcurrentMap<String, LazyConnectExchangeClient> ghostClientMap = new ConcurrentHashMap<String, LazyConnectExchangeClient>();
    /**
     *
     */
    private final ConcurrentMap<String, Object> locks = new ConcurrentHashMap<String, Object>();
    /**
     *
     */
    private final Set<String> optimizers = new ConcurrentHashSet<String>();
    /**
     * consumer side export a stub service for dispatching event
     * servicekey-stubmethods
     */
    private final ConcurrentMap<String, String> stubServiceMethodsMap = new ConcurrentHashMap<String, String>();
    ...
    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        URL url = invoker.getUrl();

        // export service. 暴露的服务key
        String key = serviceKey(url);
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);

        //export an stub service for dispatching event
        Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice) {
            String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                if (logger.isWarnEnabled()) {
                    logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }
            } else {
                stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
            }
        }

        openServer(url);
        optimizeSerialization(url);
        return exporter;
    }
/**
     *
     * @param url
     */
    private void openServer(URL url) {
        // find server.
        String key = url.getAddress();
        //client can export a service which's only for server to invoke
        boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
        if (isServer) {
            ExchangeServer server = serverMap.get(key);
            if (server == null) {
                synchronized (this) {
                    server = serverMap.get(key);
                    if (server == null) {
                        //
                        serverMap.put(key, createServer(url));
                    }
                }
            } else {
                // server supports reset, use together with override
                server.reset(url);
            }
        }
    }
    /**
     * @param url
     * @return
     */
    private ExchangeServer createServer(URL url) {
        // send readonly event when server closes, it's enabled by default
        url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
        // enable heartbeat by default
        url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
        //默认为netty
        String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);
        //保证Transporter存在具体的实现
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported server type: " + str + ", url: " + url);
        }
        //添加协议编码器
        url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
        ExchangeServer server;
        try {
            server = Exchangers.bind(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
        }
        //netty
        str = url.getParameter(Constants.CLIENT_KEY);
        if (str != null && str.length() > 0) {
            Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
            if (!supportedTypes.contains(str)) {
                throw new RpcException("Unsupported client type: " + str);
            }
        }
        return server;
    }
}
```

//Exchangers
```java
/**
 * Exchanger facade. (API, Static, ThreadSafe)
 */
public class Exchangers {

    static {
        // check duplicate jar package
        Version.checkDuplicate(Exchangers.class);
    }

    private Exchangers() {
    }
    ...
     /**
     * @param url
     * @param handler
     * @return
     * @throws RemotingException
     */
    public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        return getExchanger(url).bind(url, handler);
    }
    /**
     * @param url
     * @return
     */
    public static Exchanger getExchanger(URL url) {
        String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
        return getExchanger(type);
    }

    /**
     * @param type
     * @return
     */
    public static Exchanger getExchanger(String type) {
        return ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type);
    }
    ...
}
```

根据SPI机制，Exchanger为HeaderExchanger

```java
/**
 * DefaultMessenger
 *
 *
 */
public class HeaderExchanger implements Exchanger {

    public static final String NAME = "header";

    @Override
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }

    @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

}
```

//Transporters
```java
/**
 * Transporter facade. (API, Static, ThreadSafe)
 */
public class Transporters {

    static {
        // check duplicate jar package
        Version.checkDuplicate(Transporters.class);
        Version.checkDuplicate(RemotingException.class);
    }

    private Transporters() {
    }
...
 /**
     * @param url
     * @param handlers
     * @return
     * @throws RemotingException
     */
    public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handlers == null || handlers.length == 0) {
            throw new IllegalArgumentException("handlers == null");
        }
        ChannelHandler handler;
        if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            handler = new ChannelHandlerDispatcher(handlers);
        }
        //绑定url
        return getTransporter().bind(url, handler);
    }
     /**
     * @return
     */
    public static Transporter getTransporter() {
        return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }
}
```

Transporter根据我们配置Dubbo协议的客户端和服务端类型，可以为netty，mina的，默认SPI为netty。

//NettyTransporter
```java
public class NettyTransporter implements Transporter {

    public static final String NAME = "netty3";

    @Override
    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }

    @Override
    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyClient(url, listener);
    }

}
```

//MinaTransporter
```java
public class MinaTransporter implements Transporter {

    public static final String NAME = "mina";

    @Override
    public Server bind(URL url, ChannelHandler handler) throws RemotingException {
        return new MinaServer(url, handler);
    }

    @Override
    public Client connect(URL url, ChannelHandler handler) throws RemotingException {
        return new MinaClient(url, handler);
    }

}
```

从上面可以看出， dubbo协议的导出服务，实际上创建一个服务Server，根据dubbo协议配置，可为NettyServer，或MinaServer。默认为NettyServer。

再来看转换服务。

#### 转换服务


//DubboProtocol
```java
@Override
    public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
        optimizeSerialization(url);
        // create rpc invoker.
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        invokers.add(invoker);
        return invoker;
    }
     /**
     * @param url
     * @return
     */
    private ExchangeClient[] getClients(URL url) {
        // whether to share connection
        boolean service_share_connect = false;
        int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
        // if not configured, connection is shared, otherwise, one connection for one service
        if (connections == 0) {
            service_share_connect = true;
            connections = 1;
        }

        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            if (service_share_connect) {
                clients[i] = getSharedClient(url);
            } else {
                clients[i] = initClient(url);
            }
        }
        return clients;
    }
     /**
     * Create new connection
     */
    private ExchangeClient initClient(URL url) {

        // client type setting.
        String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));

        url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
        // enable heartbeat by default
        url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));

        // BIO is not allowed since it has severe performance issue.
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported client type: " + str + "," +
                    " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
        }

        ExchangeClient client;
        try {
            // connection should be lazy
            if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
                client = new LazyConnectExchangeClient(url, requestHandler);
            } else {
                client = Exchangers.connect(url, requestHandler);
            }
        } catch (RemotingException e) {
            throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
        }
        return client;
    }
```


//Exchangers
```java
/**
     * @param url
     * @param handler
     * @return
     * @throws RemotingException
     */
    public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        return getExchanger(url).connect(url, handler);
    }
```


//Transporters
```java
public static Client connect(String url, ChannelHandler... handler) throws RemotingException {
        return connect(URL.valueOf(url), handler);
    }

    public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        ChannelHandler handler;
        if (handlers == null || handlers.length == 0) {
            handler = new ChannelHandlerAdapter();
        } else if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            handler = new ChannelHandlerDispatcher(handlers);
        }
        return getTransporter().connect(url, handler);
    }
```

//NettyTransporter
```java
public class NettyTransporter implements Transporter {

    public static final String NAME = "netty3";

    @Override
    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }

    @Override
    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyClient(url, listener);
    }

}
```

//MinaTransporter
```java
public class MinaTransporter implements Transporter {

    public static final String NAME = "mina";

    @Override
    public Server bind(URL url, ChannelHandler handler) throws RemotingException {
        return new MinaServer(url, handler);
    }

    @Override
    public Client connect(URL url, ChannelHandler handler) throws RemotingException {
        return new MinaClient(url, handler);
    }

}
```

从上面可以看出， dubbo协议的转换服务Refer，实际上创建一个消费Client，根据dubbo协议配置，可为NettyClient，或MinaClient。默认为NettyClient。


# 总结

Dubbo协议是消费者和服务者通信的基础，包括服务的调用。注册器协议中，有如下一个功能，导出服务到本地， 实际委托的相应的协议，比如Dubbo协议DubboProtocol的export操作。注册器目录，当前监听注册器节点变化是，重新索引服务，在转换URL为Invoker，实际委托的相应的协议，比如Dubbo协议DubboProtocol的
refer操作。 dubbo协议的导出服务，实际上创建一个服务Server，根据dubbo协议配置，可为NettyServer，或MinaServer。默认为NettyServer。


# 附

[dubbo offical site](https://dubbo.apache.org/zh-cn/index.html)    
[dubbo github](https://github.com/apache/dubbo)   
[dubbo github vt](https://github.com/Donaldhan/incubator-dubbo)  
[incubator-dubbo-spring-boot-project github vt](https://github.com/Donaldhan/incubator-dubbo-spring-boot-project)  
