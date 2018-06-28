---
layout: page
title: Zookeeper连接客户端总结
subtitle: Zookeeper连接客户端总结
date: 2018-06-28 20:58:19
author: donaldhan
catalog: true
category: Zookeeper
categories:
    - Zookeeper
tags:
    - ZkClient
---

[zookeeper-demo]:https://github.com/Donaldhan/zookeeper-demo "zookeeper-demo"
[Zookeeper原生API]:https://donaldhan.github.io/zookeeper/2018/06/14/Zookeeper%E5%8E%9F%E7%94%9FAPI.html "Zookeeper原生API"
[ZkClient]:https://donaldhan.github.io/zookeeper/2018/11/04/ZkClient.html "ZkClient"
[Curator]:https://donaldhan.github.io/zookeeper/2018/06/18/Curator.html "Curator"


# 目录
* [Zookeeper原生API](#zookeeper原生api)
* [ZkClient](#zkclient)
* [Curator](#curator)
* [总结](#总结)

## Zookeeper原生API
Apache [Zookeeper原生API][],的缺点：

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

Zookeeper主要有两个成员分别为客户端和watcher管理器。watcher观察器，主要关注点的事件类型有节点创建NodeCreated，节点删除NodeDeleted，节点数据改变NodeDataChanged，
节点子节点更新事件类型NodeChildrenChanged；客户端状态有：同步连接SyncConnected，断开连接Disconnected，只读连接ConnectedReadOnly，验证失败AuthFailed，已验证SaslAuthenticated，会话过期Expired等状态。
Watcher观察者管理器ZKWatchManager，主要根据事件类型，注册节点观察器，默认为节点数据观察器集，节点存在观察器集，节点孩子节点观察器集，默认观察期器集；如果是NodeCreated和NodeDeleted，则注册节点数据观察器集，节点存在观察器集；
如果是NodeDataChanged，则注册节点孩子节点观察器集；如果是NodeDeleted，则注册节点数据观察器集，节点存在观察器集，节点孩子节点观察器集。

客户端ClientCnxn中最重要的是发送线程SendThread和事件线程EventThread，同时关联一个ZooKeeper，以及客户端watcher管理器ClientWatchManager，实际为ZKWatchManager，
还有一个我们需要关注的点是等待发送数据包队列pendingQueue（LinkedList<Packet>）和需要被发送的数据包队列outgoingQueue(LinkedList<Packet>)。

数据包Packet主要有请求头部requestHeader（RequestHeader），响应头部replyHeader（ReplyHeader），请求request（Record），响应response（Record），字节缓冲区ByteBuffer，客户端路径clientPath，服务端路径serverPath，异步回调接口AsyncCallback，数据包上下文，观察者注册器watchRegistration。

发送线程SendThread主要的作用是发送客户端请求数据包，实际委托给内部的clientCnxnSocket。

客户端socket的主要功能为发送数据包sendPacket和调度数据包队列doTransport。

客户端Socket的实现ClientCnxnSocketNIO，内部主要使用nio的选择器和选择key。

发送数据包，实际委托给内Socket通道。

调度数据包队列，实际委托给内Socket通道，如果是响应消息，则转化为响应Record，如果是发送数据包，则委托给内部的socket通道。

事件线程主要处理创建、设值,获取节点数据和获取节点子节点数据，检查节点是否存在，删除节点等事件，并处理。

启动客户端Socket，实际上启动发送数据包线程（处理数据的请求和响应）和事件线程（处理crwda相关事件）。

创建节点，创建创建请求和响应，委托给socket客户端，发送创建节点操作。

Zk的crwda的相关操作，首先创建相应类型的请求和响应，然后委托给socket客户端，处理响应的操作，并解析响应消息。

## ZkClient

[ZkClient][]是由Datameer的工程师开发的开源客户端，对Zookeeper的原生API进行了包装。
相对于原生api优势：
1. 实现了超时重连、Watcher反复注册等功能。
2. 添加序列化支持。
3. 同时可以递归创建和删除路径。

Zk客户端ZkClient主要的成员变量为,客户端连接IZkConnection，子节点监听器集IZkChildListener，节点数据监听器集IZkDataListener，当前状态KeeperState，事件锁ZkLock，
客户端状态监听器集IZkStateListener，事件线程ZkEventThread，序列化器ZkSerializer,最要的一点实现了 *Watcher* 接口。

节点数据监听器IZkDataListener,主要监控节点数据的变化，包括创建，变更，和删除事件。

子节点监听器IZkChildListener，监控路径子节点的变化，包括创建，变更，和删除事件。

客户端状态监听器IZkStateListener，处理连接状态的变更，并在会话过期时，重新创建连接。

事件锁，为可重入锁，有三个条件，分别为节点数据变更，会话状态变更，节点事件条件。

序列化器ZkSerializer,用于序列化，发送给Zkserver的数据，反序列化，从zk服务器接受的数据。

Zkclient的构造，主要是初始化Zk会话连接，会话超时时间和会话连接超时时间。默认的序列化器为SerializableSerializer，同时我们可以自己实现字节的序列化器。

会话接口IZkConnection，主要提供了ZK的CRWDA操作，这个与[Zk原生API的客户端socket][]作用相同。

ZkClient会话客户端ZkConnection，主要成员变量，一个为远程Zk客户端ZooKeeper，一个用户控制会话连接与关闭的可重入锁ReentrantLock。
连接操作，主要是创建原生Zookeeper客户端，关闭操作实际，是关闭原生Zookeeper客户端。
CDRWA操作实际委托给内部的原生Zookeeper客户端，ZkClient会话客户端连接ZkConnection，面向的能染是字节流。
创建zk目录时，我们可以根据布尔参数createParents，来决定是否需要创建父目录，实际操作委托给内部的ZkClient会话连接。
删除操作，当会话失去连接时，重新连接，通过回调再执行删除目录操作，实际操作委托给内部的ZkClient会话连接。
检查目录是否存在操作，当会话失去连接时，重新连接，通过回调再执行检查目录操作，实际操作委托给内部的ZkClient会话连接。
读操作的如果失去连接，则重新连接，连接成功后，通过回调，委托ZkClient会话读取目录数据，如果存在目录监听器，则触发目录监听器，同时反序列化读取的字节序列。
写操作先序列化数据，如果失去连接，则重新连接，连接成功后，通过回调，委托ZkClient会话写目录数据。

事件线程ZkEventThread内部有一个zk事件ZkEvent队列LinkedBlockingQueue<ZkEvent>，事件线程的主要任务是，消费zk事件ZkEvent队列中的
事件，并执行相应的事件。

ZkClient实现Watcher的目的主要处理目录变更和会话状态变更相关事件，对于在会话关闭时，触发的事件，直接丢弃。
状态变更事件处理，主要是将触发状态监听任务保证成ZK事件ZkEvent，放入事件线程的事件队列中，如果会话过期，则重新连接。

触发目录变更及子目录变更事件的原理和状态变更基本相同，都是将触发监听器操作包装成包装成ZK事件ZkEvent，放入事件线程ZkEventThread的事件队列中，对于目录变更事件，则重新注册监听器，
从而避免了原生API的重复注册的弊端。


## Curator
[Curator][]框架工厂CuratorFrameworkFactory内部，主要成员变量为默认的会话超时与连接超时时间，本地地址，字节压缩器GzipCompressionProvider，
默认的Zookeeper工厂DefaultZookeeperFactory，默认ACL提供器DefaultACLProvider。GzipCompressionProvider用于压缩字节流。
默认的Zookeeper工厂DefaultZookeeperFactory，用于创建原生Zookeeper客户端。DefaultACLProvider主要用户获取节点的ACL权限。


CuratorFrameworkFactory内部构建器Builder，除了会话超时与连接超时时间，字节压缩器，原生API客户端工厂，
ACL提供器之外，还有线程工程ThreadFactory，验证方式，及验证值，及重试策略RetryPolicy。
ExponentialBackoffRetry主要用户控制会话超时重连的次数和下次尝试时间。
内部构建器Builder，创建的实际为CuratorFrameworkImpl。

CuratorFramework主要提供了启动关闭客户端操作，及CDRWA相关的构建器，如创建节点CreateBuilder，删除节点DeleteBuilder，获取节点数据GetDataBuilder，设置节点数据SetACLBuilder，
，检查节点ExistsBuilder，同步数据构建器SyncBuilder， 事物构建器CuratorTransaction，ACL构建器GetACLBuilder、SetACLBuilder，提供了客户端连接状态监听器Listenable<ConnectionStateListener>，
客户端监听器Listenable<CuratorListener> ，无处理错误监听器Listenable<UnhandledErrorListener>操作，同时提供了获取zk客户端和CuratorZookeeperClient和确保路径的操作EnsurePath。


Curator zk客户端CuratorZookeeperClient主要用于获取原生API ZK客户端，以及用于重新创建失效会话，执行相应的CDRWA操作。
创建构建器CreateBuilder，主要提供了创建持久化和临时节点的操作。

Curator框架实现CuratorFrameworkImpl，创建目录实际上委托给Curator框架内部的原生API zk客户端，如果需要创建建父目录，并且父目录不存在，则创建父目录。
如果会话失效，则重新建立会话，如果建立会话成功，则调用创建目录回调Callable。

删除构建器DeleteBuilder，删除目录，实际操作在一个重试循环中，如果会话过期，则重新连接会话，并将实际删除操作委托给Curator框架内部的原生API zk客户端。

Curator框架实现CuratorFrameworkImpl的获取目录数据操作，检查目录和设置目录数据的原理与创建、删除操作基本相同实际操作委托给Curator框架内部的原生API zk客户端，
并保证会话有效。
