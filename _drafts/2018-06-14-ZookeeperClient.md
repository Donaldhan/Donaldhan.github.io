---
layout: page
title: ZookeeperClient
subtitle: ZookeeperClient
date: 2018-06-14 17:07:15
author: donaldhan
catalog: true
category: Zookeeper
categories:
    - Zookeeper
tags:
    - Zookeeper
---

# 引言
随着分布式应用的发展，对配置的管理变得越来越繁琐，单机配置中心，容易导致配置中心挂掉的情况下，出现整体应用瘫痪的场景。Zookeeper出现解决了单点配置的缺点，
同时我们可以很容易使用Zookeeper建立配置集群，当集群中的某台配置服务器宕机时，不会对应用造成任务影响。更重要的是ZK提供了观察者机制，可以动态监听配置的变更。
Zookeeper以文件目录的方式存储数据，使我们可以非常方便的管理配置。

这篇文章，我们不打算深入的研究Zookeeper，我们关注的是Zookeeper原生API， ZkClient和Curator，3客户端的优缺点，以后有时间我们在来探究Zookeeper的源码。这篇文章所有使用的示例代码可以参考[zookeeper-demo][]。





# 目录
* [原生API](#原生API)
* [ZkClient](#ZkClient)
* [Curator](#Curator)
* [总结](#总结)

## 原生API
 Apache 原生API,的缺点：

 1. 由于设置和获取路径节点的数据都是字节序列，所以自己去处理序列化。
 2. 同时事件注册是一次性的，如果需要持续监听一个节点，必须在监听器捕捉一个事件后，重新注册。
 3. 无法创建父路径不存在的路径。
 4. 删除操作，只能删除节点下没有子节点的路径。  
 客户端为 @see org.apache.zookeeper.ZooKeeper，观察者为 @see org.apache.zookeeper.Watcher

 ZK的节点有5种操作权限：
 CREATE、READ、WRITE、DELETE、ADMIN 也就是 增、删、改、查、管理权限，这5种权限简写为crwda(即：每个单词的首字符缩写)
 注：*这5种权限中，delete是指对子节点的删除权限，其它4种权限指对自身节点的操作权限*。

 身份的认证有4种方式：
 1. world：默认方式，相当于全世界都能访问
 2. auth：代表已经认证通过的用户(cli中可以通过addauth digest user:pwd 来添加当前上下文中的授权用户)
 3. digest：即用户名:密码这种方式认证，这也是业务系统中最常用的
 4. ip：使用Ip地址认证
一般使用digest方式。

 设置访问控制：
 方式一：（推荐）
 1. 增加一个认证用户
 ```
 addauth digest 用户名:密码明文
 eg. addauth digest user:password
  ```
2. 设置权限
 setAcl /path auth:用户名:密码明文:权限
 eg. setAcl /test auth:user:password:cdrwa
3. 查看Acl设置
 getAcl /path

 方式二：
 ```
 setAcl /path digest:用户名:密码密文:权限
 ```
 注：这里的加密规则是SHA1加密，然后base64编码。

建议使用第一种方式。
在创建Zookeeper原生API客户端的时候，我们一般使用如下方式：

```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher)
       throws IOException
{
     this(connectString, sessionTimeout, watcher, false);
}
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
           boolean canBeReadOnly)
       throws IOException
   {
      ...
      //配置默认的watcher

       watchManager.defaultWatcher = watcher;
      //解析连接主机和端口
       ConnectStringParser connectStringParser = new ConnectStringParser(
               connectString);
      //获取主机ip地址
       HostProvider hostProvider = new StaticHostProvider(
               connectStringParser.getServerAddresses());
        //创建客户端
       cnxn = new ClientCnxn(connectStringParser.getChrootPath(),
               hostProvider, sessionTimeout, this, watchManager,
               getClientCnxnSocket(), canBeReadOnly);
        //启动客户端
       cnxn.start();
   }
```

针对Zookeeper，主要有两个参数
```java
public class ZooKeeper {

    public static final String ZOOKEEPER_CLIENT_CNXN_SOCKET = "zookeeper.clientCnxnSocket";
    //与服务端通信的客户端
    protected final ClientCnxn cnxn;
    static {
        //Keep these two lines together to keep the initialization order explicit
        LOG = LoggerFactory.getLogger(ZooKeeper.class);
        Environment.logEnv("Client environment:", LOG);
    }
    public ZooKeeperSaslClient getSaslClient() {
        return cnxn.zooKeeperSaslClient;
    }
    //观察者管理器
    private final ZKWatchManager watchManager = new ZKWatchManager();
}
```
从上面可以看出Zookeeper主要有两个成员分别为客户端和watcher管理器。

先来看watcher管理ZKWatchManager

```java
private static class ZKWatchManager implements ClientWatchManager {
      //节点数据观察器
       private final Map<String, Set<Watcher>> dataWatches =
           new HashMap<String, Set<Watcher>>();
      //节点存在观察器
       private final Map<String, Set<Watcher>> existWatches =
           new HashMap<String, Set<Watcher>>();
      //节点孩子节点观察器
       private final Map<String, Set<Watcher>> childWatches =
           new HashMap<String, Set<Watcher>>();
       //默认观察期器
       private volatile Watcher defaultWatcher;
}
```
为了便于理解ZKWatchManager，我们来看ZKWatchManager的
```java
@Override
       public Set<Watcher> materialize(Watcher.Event.KeeperState state,
                                       Watcher.Event.EventType type,
                                       String clientPath)
       {
           Set<Watcher> result = new HashSet<Watcher>();

           switch (type) {
           case None:
               result.add(defaultWatcher);
               boolean clear = ClientCnxn.getDisableAutoResetWatch() &&
                       state != Watcher.Event.KeeperState.SyncConnected;

               synchronized(dataWatches) {
                   for(Set<Watcher> ws: dataWatches.values()) {
                       result.addAll(ws);
                   }
                   if (clear) {
                       dataWatches.clear();
                   }
               }

               synchronized(existWatches) {
                   for(Set<Watcher> ws: existWatches.values()) {
                       result.addAll(ws);
                   }
                   if (clear) {
                       existWatches.clear();
                   }
               }

               synchronized(childWatches) {
                   for(Set<Watcher> ws: childWatches.values()) {
                       result.addAll(ws);
                   }
                   if (clear) {
                       childWatches.clear();
                   }
               }

               return result;
           case NodeDataChanged:
           case NodeCreated:
               synchronized (dataWatches) {
                   addTo(dataWatches.remove(clientPath), result);
               }
               synchronized (existWatches) {
                   addTo(existWatches.remove(clientPath), result);
               }
               break;
           case NodeChildrenChanged:
               synchronized (childWatches) {
                   addTo(childWatches.remove(clientPath), result);
               }
               break;
           case NodeDeleted:
               synchronized (dataWatches) {
                   addTo(dataWatches.remove(clientPath), result);
               }
               // XXX This shouldn't be needed, but just in case
               synchronized (existWatches) {
                   Set<Watcher> list = existWatches.remove(clientPath);
                   if (list != null) {
                       addTo(existWatches.remove(clientPath), result);
                       LOG.warn("We are triggering an exists watch for delete! Shouldn't happen!");
                   }
               }
               synchronized (childWatches) {
                   addTo(childWatches.remove(clientPath), result);
               }
               break;
           default:
               String msg = "Unhandled watch event type " + type
                   + " with state " + state + " on path " + clientPath;
               LOG.error(msg);
               throw new RuntimeException(msg);
           }

           return result;
       }
   }
   final private void addTo(Set<Watcher> from, Set<Watcher> to) {
          if (from != null) {
              to.addAll(from);
          }
      }
```
再来看


## ZkClient
源码参见：[][]

[]: ""

```java
```


##
源码参见：[][]

[]: ""

```java
```

[zookeeper-demo]:https://github.com/Donaldhan/zookeeper-demo "zookeeper-demo"

[BeanDefinition接口][]

![BeanDefinition](/image/spring-context/BeanDefinition.png)

[BeanDefinition接口]:https://donaldhan.github.io/spring-framework/2017/12/26/BeanDefinition%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.html "BeanDefinition接口"

## 总结
