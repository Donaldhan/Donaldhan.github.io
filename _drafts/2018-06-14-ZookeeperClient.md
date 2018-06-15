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

[zookeeper-demo]:https://github.com/Donaldhan/zookeeper-demo "zookeeper-demo"



# 目录
* [原生API](#原生api)
  * [CRWDA操作](#crwda操作)
  * [ZKWatchManager](#zkwatchmanager)
  * [ClientCnxn](#clientcnxn)
* [ZkClient](#zkclient)
* [Curator](#curator)
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

## ZKWatchManager
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
```java
public interface Watcher {

    /**
     * This interface defines the possible states an Event may represent
     */
    public interface Event {
        /**
         * Enumeration of states the ZooKeeper may be at the event
         */
        public enum KeeperState {
            /** Unused, this state is never generated by the server */
            @Deprecated
            Unknown (-1),

            /** The client is in the disconnected state - it is not connected
             * to any server in the ensemble. */
            Disconnected (0),

            /** Unused, this state is never generated by the server */
            @Deprecated
            NoSyncConnected (1),

            /** The client is in the connected state - it is connected
             * to a server in the ensemble (one of the servers specified
             * in the host connection parameter during ZooKeeper client
             * creation). */
            SyncConnected (3),

            /**
             * Auth failed state
             */
            AuthFailed (4),

            /**
             * The client is connected to a read-only server, that is the
             * server which is not currently connected to the majority.
             * The only operations allowed after receiving this state is
             * read operations.
             * This state is generated for read-only clients only since
             * read/write clients aren't allowed to connect to r/o servers.
             */
            ConnectedReadOnly (5),

            /**
              * SaslAuthenticated: used to notify clients that they are SASL-authenticated,
              * so that they can perform Zookeeper actions with their SASL-authorized permissions.
              */
            SaslAuthenticated(6),

            /** The serving cluster has expired this session. The ZooKeeper
             * client connection (the session) is no longer valid. You must
             * create a new client connection (instantiate a new ZooKeeper
             * instance) if you with to access the ensemble. */
            Expired (-112);

            private final int intValue;     // Integer representation of valu
          }
      }
      /**
        * Enumeration of types of events that may occur on the ZooKeeper
        */
       public enum EventType {
           None (-1),
           NodeCreated (1),
           NodeDeleted (2),
           NodeDataChanged (3),
           NodeChildrenChanged (4);

           private final int intValue;     // Integer representation of value
                                           // for sending over wire
          }
        abstract public void process(WatchedEvent event);
}
```       
从上面可以看出，watcher观察者管理器ZKWatchManager，主要关注点的事件类型有节点创建NodeCreated，节点删除NodeDeleted，节点数据改变NodeDataChanged，节点子节点更新事件类型NodeChildrenChanged；客户端状态有：
同步连接SyncConnected，断开连接Disconnected，只读连接ConnectedReadOnly，验证失败AuthFailed，已验证SaslAuthenticated，会话过期Expired等状态。
Watcher观察器，主要根据事件类型，注册节点观察器，默认为节点数据观察器集，节点存在观察器集，节点孩子节点观察器集，默认观察期器集；如果是NodeCreated和NodeDeleted，则注册节点数据观察器集，节点存在观察器集；
如果是NodeDataChanged，则注册节点孩子节点观察器集；如果是NodeDeleted，则注册节点数据观察器集，节点存在观察器集，节点孩子节点观察器集。

我们再来看Zookeeper的另一个关键成员客户端ClientCnxn。

## ClientCnxn
```java
public class ClientCnxn {
    private static final Logger LOG = LoggerFactory.getLogger(ClientCnxn.class);

    /** This controls whether automatic watch resetting is enabled.
     * Clients automatically reset watches during session reconnect, this
     * option allows the client to turn off this behavior by setting
     * the environment variable "zookeeper.disableAutoWatchReset" to "true" */
    private static boolean disableAutoWatchReset;
    static {
        // this var should not be public, but otw there is no easy way
        // to test
        disableAutoWatchReset =
            Boolean.getBoolean("zookeeper.disableAutoWatchReset");
        if (LOG.isDebugEnabled()) {
            LOG.debug("zookeeper.disableAutoWatchReset is "
                    + disableAutoWatchReset);
        }
    }
    private final CopyOnWriteArraySet<AuthData> authInfo = new CopyOnWriteArraySet<AuthData>();

    /**
     * These are the packets that have been sent and are waiting for a response.
     */
    private final LinkedList<Packet> pendingQueue = new LinkedList<Packet>();

    /**
     * These are the packets that need to be sent.
     */
    private final LinkedList<Packet> outgoingQueue = new LinkedList<Packet>();

    private int connectTimeout;

    /**
     * The timeout in ms the client negotiated with the server. This is the
     * "real" timeout, not the timeout request by the client (which may have
     * been increased/decreased by the server which applies bounds to this
     * value.
     */
    private volatile int negotiatedSessionTimeout;

    private int readTimeout;

    private final int sessionTimeout;

    private final ZooKeeper zooKeeper;

    private final ClientWatchManager watcher;

    private long sessionId;

    private byte sessionPasswd[] = new byte[16];

    /**
     * If true, the connection is allowed to go to r-o mode. This field's value
     * is sent, besides other data, during session creation handshake. If the
     * server on the other side of the wire is partitioned it'll accept
     * read-only clients only.
     */
    private boolean readOnly;

    final String chrootPath;

    final SendThread sendThread;

    final EventThread eventThread;

    /**
     * Set to true when close is called. Latches the connection such that we
     * don't attempt to re-connect to the server if in the middle of closing the
     * connection (client sends session disconnect to server as part of close
     * operation)
     */
    private volatile boolean closing = false;

    /**
     * A set of ZooKeeper hosts this client could connect to.
     */
    private final HostProvider hostProvider;

    /**
     * Is set to true when a connection to a r/w server is established for the
     * first time; never changed afterwards.
     * <p>
     * Is used to handle situations when client without sessionId connects to a
     * read-only server. Such client receives "fake" sessionId from read-only
     * server, but this sessionId is invalid for other servers. So when such
     * client finds a r/w server, it sends 0 instead of fake sessionId during
     * connection handshake and establishes new, valid session.
     * <p>
     * If this field is false (which implies we haven't seen r/w server before)
     * then non-zero sessionId is fake, otherwise it is valid.
     */
    volatile boolean seenRwServerBefore = false;


    public ZooKeeperSaslClient zooKeeperSaslClient;
}
```
客户端ClientCnxn中最重要的是发送线程SendThread和事件线程EventThread，同时关联一个ZooKeeper，以及客户端watcher管理器ClientWatchManager，实际为ZKWatchManager，
还有一个我们需要关注的点是等待发送数据包队列pendingQueue（LinkedList<Packet>）和需要被发送的数据包队列outgoingQueue(LinkedList<Packet>)。

我们来看一下数据包 *Packet*。


```java
/**
     * This class allows us to pass the headers and the relevant records around.
     */
    static class Packet {
        //请求头部
        RequestHeader requestHeader;
        //响应头部
        ReplyHeader replyHeader;
        //请求
        Record request;
        //响应
        Record response;
        //字节缓冲区
        ByteBuffer bb;

        /** Client's view of the path (may differ due to chroot) **/
        String clientPath;
        /** Servers's view of the path (may differ due to chroot) **/
        String serverPath;

        boolean finished;
        //异步回调接口
        AsyncCallback cb;
        //包数据上下文
        Object ctx;
        //观察者注册器
        WatchRegistration watchRegistration;
}
```
从上面可以看出，数据包Packet主要有请求头部requestHeader（RequestHeader），响应头部replyHeader（ReplyHeader），请求request（Record），响应response（Record），字节缓冲区ByteBuffer，客户端路径clientPath，服务端路径serverPath，异步回调接口AsyncCallback，数据包上下文，观察者注册器watchRegistration。
再来看请求头部，响应头部，请求和响应record

```java
public class RequestHeader implements Record {
  private int xid;
  private int type;
}
```

```java
public class ReplyHeader implements Record {
  private int xid;
  private long zxid;
  private int err;//错误代码
}
```

```java
/**
 * Interface that is implemented by generated classes.
 *
 */
public interface Record {
    public void serialize(OutputArchive archive, String tag)
        throws IOException;
    public void deserialize(InputArchive archive, String tag)
        throws IOException;
}
```
Record主要作用为序列与反序列数据，再来看OutputArchive，InputArchive


```java
/**
 * Interface that alll the serializers have to implement.
 *
 */
public interface OutputArchive {
    public void writeByte(byte b, String tag) throws IOException;
    public void writeBool(boolean b, String tag) throws IOException;
    public void writeInt(int i, String tag) throws IOException;
    public void writeLong(long l, String tag) throws IOException;
    public void writeFloat(float f, String tag) throws IOException;
    public void writeDouble(double d, String tag) throws IOException;
    public void writeString(String s, String tag) throws IOException;
    public void writeBuffer(byte buf[], String tag)
        throws IOException;
    public void writeRecord(Record r, String tag) throws IOException;
    public void startRecord(Record r, String tag) throws IOException;
    public void endRecord(Record r, String tag) throws IOException;
    public void startVector(List v, String tag) throws IOException;
    public void endVector(List v, String tag) throws IOException;
    public void startMap(TreeMap v, String tag) throws IOException;
    public void endMap(TreeMap v, String tag) throws IOException;
}
```

```java
/**
 * Interface that all the Deserializers have to implement.
 *
 */
public interface InputArchive {
    public byte readByte(String tag) throws IOException;
    public boolean readBool(String tag) throws IOException;
    public int readInt(String tag) throws IOException;
    public long readLong(String tag) throws IOException;
    public float readFloat(String tag) throws IOException;
    public double readDouble(String tag) throws IOException;
    public String readString(String tag) throws IOException;
    public byte[] readBuffer(String tag) throws IOException;
    public void readRecord(Record r, String tag) throws IOException;
    public void startRecord(String tag) throws IOException;
    public void endRecord(String tag) throws IOException;
    public Index startVector(String tag) throws IOException;
    public void endVector(String tag) throws IOException;
    public Index startMap(String tag) throws IOException;
    public void endMap(String tag) throws IOException;
}
```

再来看异步回调接口：

```java
public interface AsyncCallback {
    interface StatCallback extends AsyncCallback {
        public void processResult(int rc, String path, Object ctx, Stat stat);
    }

    interface DataCallback extends AsyncCallback {
        public void processResult(int rc, String path, Object ctx, byte data[],
                Stat stat);
    }

    interface ACLCallback extends AsyncCallback {
        public void processResult(int rc, String path, Object ctx,
                List<ACL> acl, Stat stat);
    }

    interface ChildrenCallback extends AsyncCallback {
        public void processResult(int rc, String path, Object ctx,
                List<String> children);
    }

    interface Children2Callback extends AsyncCallback {
        public void processResult(int rc, String path, Object ctx,
                List<String> children, Stat stat);
    }

    interface StringCallback extends AsyncCallback {
        public void processResult(int rc, String path, Object ctx, String name);
    }

    interface VoidCallback extends AsyncCallback {
        public void processResult(int rc, String path, Object ctx);
    }
}
```
异步回调接口中AsyncCallback，定义了各种回调接口，比如数据变更，子节点变更等。

再来看观察者注册器：
```java
/**
     * Register a watcher for a particular path.
     */
    abstract class WatchRegistration {
        private Watcher watcher;//观察者
        private String clientPath;
        public WatchRegistration(Watcher watcher, String clientPath)
        {
            this.watcher = watcher;
            this.clientPath = clientPath;
        }

        abstract protected Map<String, Set<Watcher>> getWatches(int rc);

        /**
         * Register the watcher with the set of watches on path.
         * @param rc the result code of the operation that attempted to
         * add the watch on the path.
         */
        public void register(int rc) {
            if (shouldAddWatch(rc)) {
                Map<String, Set<Watcher>> watches = getWatches(rc);
                synchronized(watches) {
                    Set<Watcher> watchers = watches.get(clientPath);
                    if (watchers == null) {
                        watchers = new HashSet<Watcher>();
                        watches.put(clientPath, watchers);
                    }
                    watchers.add(watcher);
                }
            }
        }
        /**
         * Determine whether the watch should be added based on return code.
         * @param rc the result code of the operation that attempted to add the
         * watch on the node
         * @return true if the watch should be added, otw false
         */
        protected boolean shouldAddWatch(int rc) {
            return rc == 0;
        }
    }

```
再来看观察者注册器的几个实现：
```java
/** Handle the special case of exists watches - they add a watcher
     * even in the case where NONODE result code is returned.
     */
    class ExistsWatchRegistration extends WatchRegistration {
        public ExistsWatchRegistration(Watcher watcher, String clientPath) {
            super(watcher, clientPath);
        }

        @Override
        protected Map<String, Set<Watcher>> getWatches(int rc) {
            return rc == 0 ?  watchManager.dataWatches : watchManager.existWatches;
        }

        @Override
        protected boolean shouldAddWatch(int rc) {
            return rc == 0 || rc == KeeperException.Code.NONODE.intValue();
        }
    }

    class DataWatchRegistration extends WatchRegistration {
        public DataWatchRegistration(Watcher watcher, String clientPath) {
            super(watcher, clientPath);
        }

        @Override
        protected Map<String, Set<Watcher>> getWatches(int rc) {
            return watchManager.dataWatches;
        }
    }

    class ChildWatchRegistration extends WatchRegistration {
        public ChildWatchRegistration(Watcher watcher, String clientPath) {
            super(watcher, clientPath);
        }

        @Override
        protected Map<String, Set<Watcher>> getWatches(int rc) {
            return watchManager.childWatches;
        }
    }
```

Zookeeper客户端状态：
```java
public enum States {
        CONNECTING, ASSOCIATING, CONNECTED, CONNECTEDREADONLY,
        CLOSED, AUTH_FAILED, NOT_CONNECTED;

        public boolean isAlive() {
            return this != CLOSED && this != AUTH_FAILED;
        }

        /**
         * Returns whether we are connected to a server (which
         * could possibly be read-only, if this client is allowed
         * to go to read-only mode)
         * */
        public boolean isConnected() {
            return this == CONNECTED || this == CONNECTEDREADONLY;
        }
    }
```

以上这部分，我们不需要纠结，只需要了解就行。

再来看客户端ClientCnxn的发送线程：
```java
/**
     * This class services the outgoing request queue and generates the heart
     * beats. It also spawns the ReadThread.
     */
    class SendThread extends Thread {
        private long lastPingSentNs;
        private final ClientCnxnSocket clientCnxnSocket;//客户端Socket
        private Random r = new Random(System.nanoTime());        
        private boolean isFirstConnect = true;

        void readResponse(ByteBuffer incomingBuffer) throws IOException {
          ...
        }
        SendThread(ClientCnxnSocket clientCnxnSocket) {
            super(makeThreadName("-SendThread()"));
            state = States.CONNECTING;
            this.clientCnxnSocket = clientCnxnSocket;
            setUncaughtExceptionHandler(uncaughtExceptionHandler);
            setDaemon(true);
        }

        // TODO: can not name this method getState since Thread.getState()
        // already exists
        // It would be cleaner to make class SendThread an implementation of
        // Runnable
        /**
         * Used by ClientCnxnSocket
         *
         * @return
         */
        ZooKeeper.States getZkState() {
            return state;
        }

        ClientCnxnSocket getClientCnxnSocket() {
            return clientCnxnSocket;
        }
}
```
我们来看一下发送线程的ping方法：
```java
private void sendPing() {
           lastPingSentNs = System.nanoTime();
           RequestHeader h = new RequestHeader(-2, OpCode.ping);
          //添加请求头部到队列
           queuePacket(h, null, null, null, null, null, null, null, null);
       }
```
```java
Packet queuePacket(RequestHeader h, ReplyHeader r, Record request,
            Record response, AsyncCallback cb, String clientPath,
            String serverPath, Object ctx, WatchRegistration watchRegistration)
    {
        Packet packet = null;

        // Note that we do not generate the Xid for the packet yet. It is
        // generated later at send-time, by an implementation of ClientCnxnSocket::doIO(),
        // where the packet is actually sent.
        synchronized (outgoingQueue) {
            packet = new Packet(h, r, request, response, watchRegistration);
            packet.cb = cb;
            packet.ctx = ctx;
            packet.clientPath = clientPath;
            packet.serverPath = serverPath;
            if (!state.isAlive() || closing) {
                conLossPacket(packet);
            } else {
                // If the client is asking to close the session then
                // mark as closing
                if (h.getType() == OpCode.closeSession) {
                    closing = true;
                }
                //添加数据包到包队列
                outgoingQueue.add(packet);
            }
        }
        sendThread.getClientCnxnSocket().wakeupCnxn();
        return packet;
    }
```

再来看启动连接方法
```java
private void startConnect() throws IOException {
       state = States.CONNECTING;

       InetSocketAddress addr;
       if (rwServerAddress != null) {
           addr = rwServerAddress;
           rwServerAddress = null;
       } else {
           addr = hostProvider.next(1000);
       }

       setName(getName().replaceAll("\\(.*\\)",
               "(" + addr.getHostName() + ":" + addr.getPort() + ")"));
       try {
           zooKeeperSaslClient = new ZooKeeperSaslClient("zookeeper/"+addr.getHostName());
       } catch (LoginException e) {
           // An authentication error occurred when the SASL client tried to initialize:
           // for Kerberos this means that the client failed to authenticate with the KDC.
           // This is different from an authentication error that occurs during communication
           // with the Zookeeper server, which is handled below.
           LOG.warn("SASL configuration failed: " + e + " Will continue connection to Zookeeper server without "
             + "SASL authentication, if Zookeeper server allows it.");
           eventThread.queueEvent(new WatchedEvent(
             Watcher.Event.EventType.None,
             Watcher.Event.KeeperState.AuthFailed, null));
           saslLoginFailed = true;
       }
       logStartConnect(addr);
       //委托给内存socket客户端
       clientCnxnSocket.connect(addr);
   }
   private void logStartConnect(InetSocketAddress addr) {
     String msg = "Opening socket connection to server " + addr;
     if (zooKeeperSaslClient != null) {
       msg += ". " + zooKeeperSaslClient.getConfigStatus();
     }
     LOG.info(msg);
}
```

再来看发送线程主逻辑

```java
public void run() {
     clientCnxnSocket.introduce(this,sessionId);
     clientCnxnSocket.updateNow();
     clientCnxnSocket.updateLastSendAndHeard();
     int to;
     long lastPingRwServer = System.currentTimeMillis();
     while (state.isAlive()) {
         try {
             if (!clientCnxnSocket.isConnected()) {
                 if(!isFirstConnect){
                     try {
                         Thread.sleep(r.nextInt(1000));
                     } catch (InterruptedException e) {
                         LOG.warn("Unexpected exception", e);
                     }
                 }
                 // don't re-establish connection if we are closing
                 if (closing || !state.isAlive()) {
                     break;
                 }
                 startConnect();
                 clientCnxnSocket.updateLastSendAndHeard();
             }

             if (state.isConnected()) {
                 // determine whether we need to send an AuthFailed event.
                 if (zooKeeperSaslClient != null) {
                     boolean sendAuthEvent = false;
                     if (zooKeeperSaslClient.getSaslState() == ZooKeeperSaslClient.SaslState.INITIAL) {
                         try {
                             zooKeeperSaslClient.initialize(ClientCnxn.this);
                         } catch (SaslException e) {
                            LOG.error("SASL authentication with Zookeeper Quorum member failed: " + e);
                             state = States.AUTH_FAILED;
                             sendAuthEvent = true;
                         }
                     }
                     KeeperState authState = zooKeeperSaslClient.getKeeperState();
                     if (authState != null) {
                         if (authState == KeeperState.AuthFailed) {
                             // An authentication error occurred during authentication with the Zookeeper Server.
                             state = States.AUTH_FAILED;
                             sendAuthEvent = true;
                         } else {
                             if (authState == KeeperState.SaslAuthenticated) {
                                 sendAuthEvent = true;
                             }
                         }
                     }

                     if (sendAuthEvent == true) {
                         eventThread.queueEvent(new WatchedEvent(
                               Watcher.Event.EventType.None,
                               authState,null));
                     }
                 }
                 to = readTimeout - clientCnxnSocket.getIdleRecv();
             } else {
                 to = connectTimeout - clientCnxnSocket.getIdleRecv();
             }

             if (to <= 0) {
                 throw new SessionTimeoutException(
                         "Client session timed out, have not heard from server in "
                                 + clientCnxnSocket.getIdleRecv() + "ms"
                                 + " for sessionid 0x"
                                 + Long.toHexString(sessionId));
             }
             if (state.isConnected()) {
                 int timeToNextPing = readTimeout / 2
                         - clientCnxnSocket.getIdleSend();
                 if (timeToNextPing <= 0) {
                     sendPing();
                     clientCnxnSocket.updateLastSend();
                 } else {
                     if (timeToNextPing < to) {
                         to = timeToNextPing;
                     }
                 }
             }

             // If we are in read-only mode, seek for read/write server
             if (state == States.CONNECTEDREADONLY) {
                 long now = System.currentTimeMillis();
                 int idlePingRwServer = (int) (now - lastPingRwServer);
                 if (idlePingRwServer >= pingRwTimeout) {
                     lastPingRwServer = now;
                     idlePingRwServer = 0;
                     pingRwTimeout =
                         Math.min(2*pingRwTimeout, maxPingRwTimeout);
                     pingRwServer();
                 }
                 to = Math.min(to, pingRwTimeout - idlePingRwServer);
             }
             //关键点，在这，发送数据包队列
             clientCnxnSocket.doTransport(to, pendingQueue, outgoingQueue, ClientCnxn.this);
         } catch (Throwable e) {
             if (closing) {
                 if (LOG.isDebugEnabled()) {
                     // closing so this is expected
                     LOG.debug("An exception was thrown while closing send thread for session 0x"
                             + Long.toHexString(getSessionId())
                             + " : " + e.getMessage());
                 }
                 break;
             } else {
                 // this is ugly, you have a better way speak up
                 if (e instanceof SessionExpiredException) {
                     LOG.info(e.getMessage() + ", closing socket connection");
                 } else if (e instanceof SessionTimeoutException) {
                     LOG.info(e.getMessage() + RETRY_CONN_MSG);
                 } else if (e instanceof EndOfStreamException) {
                     LOG.info(e.getMessage() + RETRY_CONN_MSG);
                 } else if (e instanceof RWServerFoundException) {
                     LOG.info(e.getMessage());
                 } else {
                     LOG.warn(
                             "Session 0x"
                                     + Long.toHexString(getSessionId())
                                     + " for server "
                                     + clientCnxnSocket.getRemoteSocketAddress()
                                     + ", unexpected error"
                                     + RETRY_CONN_MSG, e);
                 }
                 cleanup();
                 if (state.isAlive()) {
                     eventThread.queueEvent(new WatchedEvent(
                             Event.EventType.None,
                             Event.KeeperState.Disconnected,
                             null));
                 }
                 clientCnxnSocket.updateNow();
                 clientCnxnSocket.updateLastSendAndHeard();
             }
         }
     }
     cleanup();
     clientCnxnSocket.close();
     if (state.isAlive()) {
         eventThread.queueEvent(new WatchedEvent(Event.EventType.None,
                 Event.KeeperState.Disconnected, null));
     }
     ZooTrace.logTraceMessage(LOG, ZooTrace.getTextTraceLevel(),
                              "SendThread exitedloop.");
}
```
从上面可以看出发送线程主要的作用是发送客户端请求数据包，实际委托给内部的clientCnxnSocket。


再来看客户端Socket的定义
```java
abstract class ClientCnxnSocket {
    private static final Logger LOG = LoggerFactory.getLogger(ClientCnxnSocket.class);

    protected boolean initialized;

    /**
     * This buffer is only used to read the length of the incoming message.
     */
    protected final ByteBuffer lenBuffer = ByteBuffer.allocateDirect(4);

    /**
     * After the length is read, a new incomingBuffer is allocated in
     * readLength() to receive the full message.
     */
    protected ByteBuffer incomingBuffer = lenBuffer;
    protected long sentCount = 0;
    protected long recvCount = 0;
    protected long lastHeard;
    protected long lastSend;
    protected long now;
    protected ClientCnxn.SendThread sendThread;

    /**
     * The sessionId is only available here for Log and Exception messages.
     * Otherwise the socket doesn't need to know it.
     */
    protected long sessionId;
    void updateLastSend() {
        this.lastSend = now;
    }

    void updateLastSendAndHeard() {
        this.lastSend = now;
        this.lastHeard = now;
    }

    protected void readLength() throws IOException {
        int len = incomingBuffer.getInt();
        if (len < 0 || len >= ClientCnxn.packetLen) {
            throw new IOException("Packet len" + len + " is out of range!");
        }
        incomingBuffer = ByteBuffer.allocate(len);
    }

    void readConnectResult() throws IOException {
        if (LOG.isTraceEnabled()) {
            StringBuilder buf = new StringBuilder("0x[");
            for (byte b : incomingBuffer.array()) {
                buf.append(Integer.toHexString(b) + ",");
            }
            buf.append("]");
            LOG.trace("readConnectResult " + incomingBuffer.remaining() + " "
                    + buf.toString());
        }
        ByteBufferInputStream bbis = new ByteBufferInputStream(incomingBuffer);
        BinaryInputArchive bbia = BinaryInputArchive.getArchive(bbis);
        ConnectResponse conRsp = new ConnectResponse();
        conRsp.deserialize(bbia, "connect");

        // read "is read-only" flag
        boolean isRO = false;
        try {
            isRO = bbia.readBool("readOnly");
        } catch (IOException e) {
            // this is ok -- just a packet from an old server which
            // doesn't contain readOnly field
            LOG.warn("Connected to an old server; r-o mode will be unavailable");
        }

        this.sessionId = conRsp.getSessionId();
        sendThread.onConnected(conRsp.getTimeOut(), this.sessionId,
                conRsp.getPasswd(), isRO);
    }

    abstract boolean isConnected();

    abstract void connect(InetSocketAddress addr) throws IOException;

    abstract SocketAddress getRemoteSocketAddress();

    abstract SocketAddress getLocalSocketAddress();

    abstract void cleanup();

    abstract void close();

    abstract void wakeupCnxn();

    abstract void enableWrite();

    abstract void disableWrite();

    abstract void enableReadWriteOnly();
    //调度数据包队列
    abstract void doTransport(int waitTimeOut, List<Packet> pendingQueue,
            LinkedList<Packet> outgoingQueue, ClientCnxn cnxn)
            throws IOException, InterruptedException;

    abstract void testableCloseSocket() throws IOException;
    //发送数据包发送数据包和
    abstract void sendPacket(Packet p) throws IOException;
  }
```
从上面可以看出，客户端socket的主要功能为发送数据包sendPacket和调度数据包队列doTransport。

我们来看默认socket客户端的实现类：
```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
           long sessionId, byte[] sessionPasswd, boolean canBeReadOnly)
       throws IOException
   {
       ...
       cnxn = new ClientCnxn(connectStringParser.getChrootPath(),
               hostProvider, sessionTimeout, this, watchManager,
               getClientCnxnSocket(), sessionId, sessionPasswd, canBeReadOnly);
       cnxn.seenRwServerBefore = true; // since user has provided sessionId
       cnxn.start();
   }
private static ClientCnxnSocket getClientCnxnSocket() throws IOException {
       String clientCnxnSocketName = System
               .getProperty(ZOOKEEPER_CLIENT_CNXN_SOCKET);
       if (clientCnxnSocketName == null) {
           clientCnxnSocketName = ClientCnxnSocketNIO.class.getName();
       }
       try {
           return (ClientCnxnSocket) Class.forName(clientCnxnSocketName)
                   .newInstance();
       } catch (Exception e) {
           IOException ioe = new IOException("Couldn't instantiate "
                   + clientCnxnSocketName);
           ioe.initCause(e);
           throw ioe;
       }
}
```
从上面可以看出客户端socket的默认实现为ClientCnxnSocketNIO。

我们来看ClientCnxnSocketNIO

```java
public class ClientCnxnSocketNIO extends ClientCnxnSocket {
    private static final Logger LOG = LoggerFactory
            .getLogger(ClientCnxnSocketNIO.class);

    private final Selector selector = Selector.open();

    private SelectionKey sockKey;

    ClientCnxnSocketNIO() throws IOException {
        super();
    }
  }
```
从上面可以看出，客户端Socket的实现ClientCnxnSocketNIO，内部主要使用nio的选择器和选择key。


再来看创建socket

```java
/**
    * create a socket channel.
    * @return the created socket channel
    * @throws IOException
    */
   SocketChannel createSock() throws IOException {
       SocketChannel sock;
       sock = SocketChannel.open();
       sock.configureBlocking(false);
       sock.socket().setSoLinger(false, -1);
       sock.socket().setTcpNoDelay(true);
       return sock;
   }
```


再来看发送数据包：
```java
@Override
   void sendPacket(Packet p) throws IOException {
       SocketChannel sock = (SocketChannel) sockKey.channel();
       if (sock == null) {
           throw new IOException("Socket is null!");
       }
       p.createBB();
       ByteBuffer pbb = p.bb;
       sock.write(pbb);
}
```
发送数据包，实际委托给内Socket通道。

再来看调度数据包队列：
```java
@Override
    void doTransport(int waitTimeOut, List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue,
                     ClientCnxn cnxn)
            throws IOException, InterruptedException {
        selector.select(waitTimeOut);
        Set<SelectionKey> selected;
        synchronized (this) {
            selected = selector.selectedKeys();
        }
        // Everything below and until we get back to the select is
        // non blocking, so time is effectively a constant. That is
        // Why we just have to do this once, here
        updateNow();
        for (SelectionKey k : selected) {
            SocketChannel sc = ((SocketChannel) k.channel());
            if ((k.readyOps() & SelectionKey.OP_CONNECT) != 0) {
                if (sc.finishConnect()) {
                    updateLastSendAndHeard();
                    sendThread.primeConnection();
                }
            } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
              //实际调度数据包队列
                doIO(pendingQueue, outgoingQueue, cnxn);
            }
        }
        if (sendThread.getZkState().isConnected()) {
            synchronized(outgoingQueue) {
                if (findSendablePacket(outgoingQueue,
                        cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                    enableWrite();
                }
            }
        }
        selected.clear();
    }
    /**
    * @return true if a packet was received
    * @throws InterruptedException
    * @throws IOException
    */
   void doIO(List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue, ClientCnxn cnxn)
     throws InterruptedException, IOException {
       SocketChannel sock = (SocketChannel) sockKey.channel();
       if (sock == null) {
           throw new IOException("Socket is null!");
       }
       //读数据
       if (sockKey.isReadable()) {
           int rc = sock.read(incomingBuffer);
           if (rc < 0) {
               throw new EndOfStreamException(
                       "Unable to read additional data from server sessionid 0x"
                               + Long.toHexString(sessionId)
                               + ", likely server has closed socket");
           }
           if (!incomingBuffer.hasRemaining()) {
               incomingBuffer.flip();
               if (incomingBuffer == lenBuffer) {
                   recvCount++;
                   readLength();
               } else if (!initialized) {
                   readConnectResult();
                   enableRead();
                   if (findSendablePacket(outgoingQueue,
                           cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                       // Since SASL authentication has completed (if client is configured to do so),
                       // outgoing packets waiting in the outgoingQueue can now be sent.
                       enableWrite();
                   }
                   lenBuffer.clear();
                   incomingBuffer = lenBuffer;
                   updateLastHeard();
                   initialized = true;
               } else {
                 //读取数据，解析为响应Record
                   sendThread.readResponse(incomingBuffer);
                   lenBuffer.clear();
                   incomingBuffer = lenBuffer;
                   updateLastHeard();
               }
           }
       }
       //写数据
       if (sockKey.isWritable()) {
           synchronized(outgoingQueue) {
               Packet p = findSendablePacket(outgoingQueue,
                       cnxn.sendThread.clientTunneledAuthenticationInProgress());

               if (p != null) {
                   updateLastSend();
                   // If we already started writing p, p.bb will already exist
                   if (p.bb == null) {
                       if ((p.requestHeader != null) &&
                               (p.requestHeader.getType() != OpCode.ping) &&
                               (p.requestHeader.getType() != OpCode.auth)) {
                           p.requestHeader.setXid(cnxn.getXid());
                       }
                       p.createBB();
                   }
                   //写数据包
                   sock.write(p.bb);
                   if (!p.bb.hasRemaining()) {
                       sentCount++;
                       outgoingQueue.removeFirstOccurrence(p);
                       if (p.requestHeader != null
                               && p.requestHeader.getType() != OpCode.ping
                               && p.requestHeader.getType() != OpCode.auth) {
                           synchronized (pendingQueue) {
                               pendingQueue.add(p);
                           }
                       }
                   }
               }
               if (outgoingQueue.isEmpty()) {
                   // No more packets to send: turn off write interest flag.
                   // Will be turned on later by a later call to enableWrite(),
                   // from within ZooKeeperSaslClient (if client is configured
                   // to attempt SASL authentication), or in either doIO() or
                   // in doTransport() if not.
                   disableWrite();
               } else {
                   // Just in case
                   enableWrite();
               }
           }
       }
   }
```

调度数据包队列，实际委托给内Socket通道，如果是响应消息，则转化为响应Record，如果是发送数据包，则委托给内部的socket通道。


到这里我们报客户端Socket的发送线程分析完，小节一下：

客户端ClientCnxn中最重要的是发送线程SendThread和事件线程EventThread，同时关联一个ZooKeeper，以及客户端watcher管理器ClientWatchManager，实际为ZKWatchManager，
还有一个我们需要关注的点是等待发送数据包队列pendingQueue（LinkedList<Packet>）和需要被发送的数据包队列outgoingQueue(LinkedList<Packet>)。

数据包Packet主要有请求头部requestHeader（RequestHeader），响应头部replyHeader（ReplyHeader），请求request（Record），响应response（Record），字节缓冲区ByteBuffer，客户端路径clientPath，服务端路径serverPath，异步回调接口AsyncCallback，数据包上下文，观察者注册器watchRegistration。

发送线程SendThread主要的作用是发送客户端请求数据包，实际委托给内部的clientCnxnSocket。

客户端socket的主要功能为发送数据包sendPacket和调度数据包队列doTransport。

客户端Socket的实现ClientCnxnSocketNIO，内部主要使用nio的选择器和选择key。

发送数据包，实际委托给内Socket通道。

调度数据包队列，实际委托给内Socket通道，如果是响应消息，则转化为响应Record，如果是发送数据包，则委托给内部的socket通道。


再来看客户端ClientCnxn中的事件线程EventThread：


```java
class EventThread extends Thread {
      //待处理事件队列
       private final LinkedBlockingQueue<Object> waitingEvents =
           new LinkedBlockingQueue<Object>();

       /** This is really the queued session state until the event
        * thread actually processes the event and hands it to the watcher.
        * But for all intents and purposes this is the state.
        */
       private volatile KeeperState sessionState = KeeperState.Disconnected;

      private volatile boolean wasKilled = false;
      private volatile boolean isRunning = false;

       EventThread() {
           super(makeThreadName("-EventThread"));
           setUncaughtExceptionHandler(uncaughtExceptionHandler);
           setDaemon(true);
       }
}
```
来看事件线程的主流程：

```java
@Override
public void run() {
   try {
      isRunning = true;
      while (true) {
         //消费队列事件
         Object event = waitingEvents.take();
         if (event == eventOfDeath) {
            wasKilled = true;
         } else {
           //处理事件
            processEvent(event);
         }
         if (wasKilled)
            synchronized (waitingEvents) {
               if (waitingEvents.isEmpty()) {
                  isRunning = false;
                  break;
               }
            }
      }
   } catch (InterruptedException e) {
      LOG.error("Event thread exiting due to interruption", e);
   }
    LOG.info("EventThread shut down");
}
```

再来看处理事件：

```java
private void processEvent(Object event) {
      try {
          if (event instanceof WatcherSetEventPair) {
              // each watcher will process the event
              WatcherSetEventPair pair = (WatcherSetEventPair) event;
              for (Watcher watcher : pair.watchers) {
                  try {
                    //如果是事件观察者对，则条用观察者处理事件
                      watcher.process(pair.event);
                  } catch (Throwable t) {
                      LOG.error("Error while calling watcher ", t);
                  }
              }
          } else {
            //否则为数据包，转换事件
              Packet p = (Packet) event;
              int rc = 0;
              //客户端路径
              String clientPath = p.clientPath;
              if (p.replyHeader.getErr() != 0) {
                  rc = p.replyHeader.getErr();
              }
              if (p.cb == null) {//异步回调为空
                  LOG.warn("Somehow a null cb got to EventThread!");
              } else if (p.response instanceof ExistsResponse
                      || p.response instanceof SetDataResponse
                      || p.response instanceof SetACLResponse) {
                  //状态回调
                  StatCallback cb = (StatCallback) p.cb;
                  if (rc == 0) {
                      if (p.response instanceof ExistsResponse) {
                        //节点存在检查响应
                          cb.processResult(rc, clientPath, p.ctx,
                                  ((ExistsResponse) p.response)
                                          .getStat());
                      } else if (p.response instanceof SetDataResponse) {
                        //设置节点数据响应
                          cb.processResult(rc, clientPath, p.ctx,
                                  ((SetDataResponse) p.response)
                                          .getStat());
                      } else if (p.response instanceof SetACLResponse) {
                        //安全控制响应
                          cb.processResult(rc, clientPath, p.ctx,
                                  ((SetACLResponse) p.response)
                                          .getStat());
                      }
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null);
                  }
              } else if (p.response instanceof GetDataResponse) {
                //获取数据响应
                  DataCallback cb = (DataCallback) p.cb;
                  GetDataResponse rsp = (GetDataResponse) p.response;
                  if (rc == 0) {
                      cb.processResult(rc, clientPath, p.ctx, rsp
                              .getData(), rsp.getStat());
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null,
                              null);
                  }
              } else if (p.response instanceof GetACLResponse) {
                //获取安全控制响应
                  ACLCallback cb = (ACLCallback) p.cb;
                  GetACLResponse rsp = (GetACLResponse) p.response;
                  if (rc == 0) {
                      cb.processResult(rc, clientPath, p.ctx, rsp
                              .getAcl(), rsp.getStat());
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null,
                              null);
                  }
              } else if (p.response instanceof GetChildrenResponse) {
                //获取节点子节点响应
                  ChildrenCallback cb = (ChildrenCallback) p.cb;
                  GetChildrenResponse rsp = (GetChildrenResponse) p.response;
                  if (rc == 0) {
                      cb.processResult(rc, clientPath, p.ctx, rsp
                              .getChildren());
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null);
                  }
              } else if (p.response instanceof GetChildren2Response) {
                  Children2Callback cb = (Children2Callback) p.cb;
                  GetChildren2Response rsp = (GetChildren2Response) p.response;
                  if (rc == 0) {
                      cb.processResult(rc, clientPath, p.ctx, rsp
                              .getChildren(), rsp.getStat());
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null, null);
                  }
              } else if (p.response instanceof CreateResponse) {
                //创建节点响应
                  StringCallback cb = (StringCallback) p.cb;
                  CreateResponse rsp = (CreateResponse) p.response;
                  if (rc == 0) {
                      cb.processResult(rc, clientPath, p.ctx,
                              (chrootPath == null
                                      ? rsp.getPath()
                                      : rsp.getPath()
                                .substring(chrootPath.length())));
                  } else {
                      cb.processResult(rc, clientPath, p.ctx, null);
                  }
              } else if (p.cb instanceof VoidCallback) {
                  VoidCallback cb = (VoidCallback) p.cb;
                  cb.processResult(rc, clientPath, p.ctx);
              }
          }
      } catch (Throwable t) {
          LOG.error("Caught unexpected throwable", t);
      }
   }
}
private static class WatcherSetEventPair {
       private final Set<Watcher> watchers;//事件观察者
       private final WatchedEvent event;//事件

       public WatcherSetEventPair(Set<Watcher> watchers, WatchedEvent event) {
           this.watchers = watchers;
           this.event = event;
       }
}
public class WatchedEvent {
   final private KeeperState keeperState;
   final private EventType eventType;
   private String path;
}
```

从上面可以看出事件线程主要处理创建、设值,获取节点数据和获取节点子节点数据，检查节点是否存在，删除节点等事件，并处理。

再来看一下创建Zookeeper客户端：
```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
         long sessionId, byte[] sessionPasswd, boolean canBeReadOnly)
     throws IOException
{
     ...
     cnxn = new ClientCnxn(connectStringParser.getChrootPath(),
             hostProvider, sessionTimeout, this, watchManager,
             getClientCnxnSocket(), sessionId, sessionPasswd, canBeReadOnly);
     cnxn.seenRwServerBefore = true; // since user has provided sessionId
     cnxn.start();//启动客户端
}
```
来看启动客户端Socket：

```java
public class ClientCnxn {
  private Object eventOfDeath = new Object();
  public void start() {
        sendThread.start();
        eventThread.start();
  }
}
```

从上面可以看出，启动客户端Socket，实际上启动发送数据包线程（处理数据的请求和响应）和事件线程（处理crwda相关事件）。

## CRWDA操作

再来看一下，创建节点，设置，获取节点数据。

1. 创建节点
```java
public String create(final String path, byte data[], List<ACL> acl,
         CreateMode createMode)
     throws KeeperException, InterruptedException
{
     final String clientPath = path;
     PathUtils.validatePath(clientPath, createMode.isSequential());

     final String serverPath = prependChroot(clientPath);
     //创建请求和响应
     RequestHeader h = new RequestHeader();
     h.setType(ZooDefs.OpCode.create);
     CreateRequest request = new CreateRequest();
     CreateResponse response = new CreateResponse();
     request.setData(data);
     request.setFlags(createMode.toFlag());
     request.setPath(serverPath);
     if (acl != null && acl.size() == 0) {
         throw new KeeperException.InvalidACLException();
     }
     request.setAcl(acl);
     //委托给socket客户端，发送创建节点操作
     ReplyHeader r = cnxn.submitRequest(h, request, response, null);
     if (r.getErr() != 0) {
         throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                 clientPath);
     }
     if (cnxn.chrootPath == null) {
         return response.getPath();
     } else {
         return response.getPath().substring(cnxn.chrootPath.length());
     }
}
```
#### ClientCnxn
```java
public ReplyHeader submitRequest(RequestHeader h, Record request,
            Record response, WatchRegistration watchRegistration)
            throws InterruptedException {
        ReplyHeader r = new ReplyHeader();
        Packet packet = queuePacket(h, r, request, response, null, null, null,
                    null, watchRegistration);
        synchronized (packet) {
            while (!packet.finished) {
                packet.wait();
            }
        }
        return r;
    }
```
创建节点，创建创建请求和响应，委托给socket客户端，发送创建节点操作。
再来看创建节点，异步回调处理结果：
```java
/**
 * The asynchronous version of create.
 *
 * @see #create(String, byte[], List, CreateMode)
 */

public void create(final String path, byte data[], List<ACL> acl,
        CreateMode createMode,  StringCallback cb, Object ctx)
{
    final String clientPath = path;
    PathUtils.validatePath(clientPath, createMode.isSequential());

    final String serverPath = prependChroot(clientPath);

    RequestHeader h = new RequestHeader();
    h.setType(ZooDefs.OpCode.create);
    CreateRequest request = new CreateRequest();
    CreateResponse response = new CreateResponse();
    ReplyHeader r = new ReplyHeader();
    request.setData(data);
    request.setFlags(createMode.toFlag());
    request.setPath(serverPath);
    request.setAcl(acl);
    //委托给socket客户端
    cnxn.queuePacket(h, r, request, response, cb, clientPath,
            serverPath, ctx, null);
}
```
2. 设置
```java
public Stat setData(final String path, byte data[], int version)
        throws KeeperException, InterruptedException
    {
        final String clientPath = path;
        PathUtils.validatePath(clientPath);

        final String serverPath = prependChroot(clientPath);

        RequestHeader h = new RequestHeader();
        h.setType(ZooDefs.OpCode.setData);
        SetDataRequest request = new SetDataRequest();
        request.setPath(serverPath);
        request.setData(data);
        request.setVersion(version);
        SetDataResponse response = new SetDataResponse();
        ReplyHeader r = cnxn.submitRequest(h, request, response, null);
        if (r.getErr() != 0) {
            throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                    clientPath);
        }
        return response.getStat();
    }

```

3. 获取节点数据
```java
public byte[] getData(String path, boolean watch, Stat stat)
           throws KeeperException, InterruptedException {
       return getData(path, watch ? watchManager.defaultWatcher : null, stat);
}
/**
     * The asynchronous version of getData.
     *
     * @see #getData(String, Watcher, Stat)
     */
    public void getData(final String path, Watcher watcher,
            DataCallback cb, Object ctx)
    {
        final String clientPath = path;
        PathUtils.validatePath(clientPath);

        // the watch contains the un-chroot path
        WatchRegistration wcb = null;
        if (watcher != null) {
            wcb = new DataWatchRegistration(watcher, clientPath);
        }

        final String serverPath = prependChroot(clientPath);

        RequestHeader h = new RequestHeader();
        h.setType(ZooDefs.OpCode.getData);
        GetDataRequest request = new GetDataRequest();
        request.setPath(serverPath);
        request.setWatch(watcher != null);
        GetDataResponse response = new GetDataResponse();
        //委托给socket客户端
        cnxn.queuePacket(h, new ReplyHeader(), request, response, cb,
                clientPath, serverPath, ctx, wcb);
    }
```

从上面可以看出，Zk的crwda的相关操作，首先创建相应类型的请求和响应，然后委托给socket客户端，处理响应的操作，并解析响应消息。

其他操作，我们不在讲解，放在附中。

由于篇幅问题，另外两种客户端ZkClient和Curator，我们放在后面的文章中再讲。

## ZkClient

## Curator


## 总结
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

## 附
```java
public void delete(final String path, int version)
        throws InterruptedException, KeeperException
    {
        final String clientPath = path;
        PathUtils.validatePath(clientPath);

        final String serverPath;

        // maintain semantics even in chroot case
        // specifically - root cannot be deleted
        // I think this makes sense even in chroot case.
        if (clientPath.equals("/")) {
            // a bit of a hack, but delete(/) will never succeed and ensures
            // that the same semantics are maintained
            serverPath = clientPath;
        } else {
            serverPath = prependChroot(clientPath);
        }

        RequestHeader h = new RequestHeader();
        h.setType(ZooDefs.OpCode.delete);
        DeleteRequest request = new DeleteRequest();
        request.setPath(serverPath);
        request.setVersion(version);
        ReplyHeader r = cnxn.submitRequest(h, request, null, null);
        if (r.getErr() != 0) {
            throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                    clientPath);
        }
}
public List<String> getChildren(final String path, Watcher watcher)
        throws KeeperException, InterruptedException
    {
        final String clientPath = path;
        PathUtils.validatePath(clientPath);

        // the watch contains the un-chroot path
        WatchRegistration wcb = null;
        if (watcher != null) {
            wcb = new ChildWatchRegistration(watcher, clientPath);
        }

        final String serverPath = prependChroot(clientPath);

        RequestHeader h = new RequestHeader();
        h.setType(ZooDefs.OpCode.getChildren);
        GetChildrenRequest request = new GetChildrenRequest();
        request.setPath(serverPath);
        request.setWatch(watcher != null);
        GetChildrenResponse response = new GetChildrenResponse();
        ReplyHeader r = cnxn.submitRequest(h, request, response, wcb);
        if (r.getErr() != 0) {
            throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                    clientPath);
        }
        return response.getChildren();
}
public void exists(final String path, Watcher watcher,
            StatCallback cb, Object ctx)
    {
        final String clientPath = path;
        PathUtils.validatePath(clientPath);

        // the watch contains the un-chroot path
        WatchRegistration wcb = null;
        if (watcher != null) {
            wcb = new ExistsWatchRegistration(watcher, clientPath);
        }

        final String serverPath = prependChroot(clientPath);

        RequestHeader h = new RequestHeader();
        h.setType(ZooDefs.OpCode.exists);
        ExistsRequest request = new ExistsRequest();
        request.setPath(serverPath);
        request.setWatch(watcher != null);
        SetDataResponse response = new SetDataResponse();
        cnxn.queuePacket(h, new ReplyHeader(), request, response, cb,
                clientPath, serverPath, ctx, wcb);
}
```
