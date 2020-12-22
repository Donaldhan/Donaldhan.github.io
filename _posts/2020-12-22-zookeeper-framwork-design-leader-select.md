---
layout: page
title: Zookeeper框架设计及源码解读二（快速选举器及选举消息的发送与接收）
subtitle: Zookeeper框架设计及源码解读二（快速选举器及选举消息的发送与接收）
date: 2020-12-22 21:07:00
author: valuewithTime
catalog: true
category: Zookeeper
categories:
    - Zookeeper
tags:
    - Zookeeper
---

# 引言
Zookeeper整体架构主要分为数据的存储，消息，leader选举和数据同步这几个模块。leader选举主要是在集群处于混沌的状态下，从集群peer的提议中选择集群的leader，其他为follower或observer，维护集群peer的统一视图，保证整个集群的数据一致性，如果在leader选举成功后，存在follower日志落后的情况，则将事务日志同步给follower。针对消息模块，peer之间的通信包需要序列化和反序列才能发送和处理，具体的消息处理由集群相应角色的消息处理器链来处理。针对客户单的节点的创建，数据修改等操作，将会先写到内存数据库，如果有提交请求，则将数据写到事务日志，同时Zookeeper会定时将内存数据库写到快照日志，以防止没有提交的日志，在宕机的情况下丢失。数据同步模块将leader的事务日志同步给Follower，保证整个集群数据的一致性。

Zookeeper启动时，首先解析配置文件，根据配置文件选择启动单例还是集群模式。集群模式启动，首先从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中。然后启动ServerCnxnFactory,监听客户端的请求。实际上是启动了一个基于Netty服务，客户端发送的数据，交由NettyServerCnxn处理，NettyServerCnxn数据包的处理，实际委托给ZooKeeperServer。

今天我们来看一下leader选举。



# 目录
* [概要框架设计](#概要框架设计)
* [源码分析](#源码分析)
    * [启动Zookeeper](#启动zookeeper)
    * [Leader选举](#leader选举)
        * [快速选举策略](#快速选举策略)
            * [消息处理器](#消息处理器)
                * [发送消息工作线程WorkerSender](#发送消息工作线程workersender)
                * [消息接收线程WorkerReceiver](#消息接收线程WorkerReceiver)
    * [peer启动](#peer启动)
    * [数据同步](#数据同步)
    * [消息处理](#消息处理)
    * [数据存储](#数据存储)
* [总结](#总结)
* [附](#附)



# 概要框架设计


![zookeeper-framework](/image/zookeeper/zookeeper-framework-design.png)  


Zookeeper整体架构主要分为数据的存储，消息，leader选举和数据同步这几个模块。leader选举主要是在集群处于混沌的状态下，从集群peer的提议中选择集群的leader，其他为follower或observer，维护集群peer的统一视图，保证整个集群的数据一致性，如果在leader选举成功后，存在follower日志落后的情况，则将事务日志同步给follower。针对消息模块，peer之间的通信包需要序列化和反序列才能发送和处理，具体的消息处理由集群相应角色的消息处理器链来处理。针对客户单的节点的创建，数据修改等操作，将会先写到内存数据库，如果有提交请求，则将数据写到事务日志，同时Zookeeper会定时将内存数据库写到快照日志，以防止没有提交的日志，在宕机的情况下丢失。数据同步模块将leader的事务日志同步给Follower，保证整个集群数据的一致性。



# 源码分析
源码分析仓库，见
[zookeeper github](https://github.com/Donaldhan/zookeeper) 

## 启动Zookeeper
[Zookeeper框架设计及源码解读一（Zookeeper启动）](https://donaldhan.github.io/zookeeper/2020/12/17/zookeeper-framwork-design-zookeer-starter.html)


Zookeeper启动时，首先解析配置文件，根据配置文件选择启动单例还是集群模式。集群模式启动，首先从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中。然后启动ServerCnxnFactory,监听客户端的请求。实际上是启动了一个基于Netty服务，客户端发送的数据，交由NettyServerCnxn处理，NettyServerCnxn数据包的处理，实际委托给ZooKeeperServer。
## Leader选举


//QuorumPeer
```java
   /**
     *
     */
    synchronized public void startLeaderElection() {
        try {
            if (getPeerState() == ServerState.LOOKING) {
                //如果peer状态为LOOKING， 创建投票
                currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
            }
        } catch(IOException e) {
            RuntimeException re = new RuntimeException(e.getMessage());
            re.setStackTrace(e.getStackTrace());
            throw re;
        }

        this.electionAlg = createElectionAlgorithm(electionType);
    }
/**
     * org.apache.zookeeper.server.quorum.QuorumPeerConfig#electionAlg， 默认为3
     * @param electionAlgorithm
     * @return
     */
    @SuppressWarnings("deprecation")
    protected Election createElectionAlgorithm(int electionAlgorithm){
        Election le=null;

        //TODO: use a factory rather than a switch
        switch (electionAlgorithm) {
        case 1:
            le = new AuthFastLeaderElection(this);
            break;
        case 2:
            le = new AuthFastLeaderElection(this, true);
            break;
        case 3:
            qcm = createCnxnManager();
            //org.apache.zookeeper.server.quorum.QuorumCnxManager.Listener
            QuorumCnxManager.Listener listener = qcm.listener;
            if(listener != null){
                listener.start();
                //fast leader 选举策略
                FastLeaderElection fle = new FastLeaderElection(this, qcm);
                fle.start();
                le = fle;
            } else {
                LOG.error("Null listener when initializing cnx manager");
            }
            break;
        default:
            assert false;
        }
        return le;
    }
```
从上面可以看出，启动peer选举策略实际启动的为fast leader 选举策略，如果peer状态为LOOKING， 创建投票（最后提交的日志id，时间戳，peerId）。

再来看快速选举策略
## 快速选举策略
//FastLeaderElection
```java
public class FastLeaderElection implements Election {
   /**
     * 发送消息队列
     */
    LinkedBlockingQueue<ToSend> sendqueue;
    /**
     * 接收消息队列
     */
    LinkedBlockingQueue<Notification> recvqueue;
    /**
     * Constructor of FastLeaderElection. It takes two parameters, one
     * is the QuorumPeer object that instantiated this object, and the other
     * is the connection manager. Such an object should be created only once
     * by each peer during an instance of the ZooKeeper service.
     * 在ZooKeeper实例化是，每个peer应该创建的对象
     *
     * @param self  QuorumPeer that created this object
     * @param manager   Connection manager
     */
    public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager){
        this.stop = false;
        this.manager = manager;
        starter(self, manager);
    }

    /**
     * This method is invoked by the constructor. Because it is a
     * part of the starting procedure of the object that must be on
     * any constructor of this class, it is probably best to keep as
     * a separate method. As we have a single constructor currently,
     * it is not strictly necessary to have it separate.
     *
     * @param self      QuorumPeer that created this object
     * @param manager   Connection manager
     */
    private void starter(QuorumPeer self, QuorumCnxManager manager) {
        this.self = self;
        proposedLeader = -1;
        proposedZxid = -1;

        sendqueue = new LinkedBlockingQueue<ToSend>();
        recvqueue = new LinkedBlockingQueue<Notification>();
        //消息管理器 KEY
        this.messenger = new Messenger(manager);
    }
    /**
     * Multi-threaded implementation of message handler. Messenger
     * implements two sub-classes: WorkReceiver and  WorkSender. The
     * functionality of each is obvious from the name. Each of these
     * spawns a new thread.
     * 消息处理器的多线程实现。具体实现包括接收和发送工作线程。
     */

    protected class Messenger {
/**
         * Receives messages from instance of QuorumCnxManager on
         * method run(), and processes such messages.
         * 处理从QuorumCnxManager接收的消息
         */

        class WorkerReceiver extends ZooKeeperThread  {
            ...
        }
        
        /**
         * This worker simply dequeues a message to send and
         * and queues it on the manager's queue.
         * 发送消息工作线程，出列一个需要发送的消息，并把它放入
         * 管理器QuorumCnxManager的队列
         */

        class WorkerSender extends ZooKeeperThread {
            ...
        }
 /**
         * 发送消息工作线程
         */
        WorkerSender ws;
        /**
         * 接收消息工作线程
         */
        WorkerReceiver wr;
        Thread wsThread = null;
        Thread wrThread = null;

        /**
         * Constructor of class Messenger.
         *
         * @param manager   Connection manager
         */
        Messenger(QuorumCnxManager manager) {

            this.ws = new WorkerSender(manager);

            this.wsThread = new Thread(this.ws,
                    "WorkerSender[myid=" + self.getId() + "]");
            this.wsThread.setDaemon(true);

            this.wr = new WorkerReceiver(manager);

            this.wrThread = new Thread(this.wr,
                    "WorkerReceiver[myid=" + self.getId() + "]");
            this.wrThread.setDaemon(true);
        }

        /**
         * Starts instances of WorkerSender and WorkerReceiver
         */
        void start(){
            this.wsThread.start();
            this.wrThread.start();
        }
        ...
        
    }
}
```

fast leader 选举策略启动时实际上启动了一个消息处理器Messenger。 消息处理器内部有一个发送消息工作线程WorkerSender，出列一个需要发送的消息，并把它放入管理器QuorumCnxManager的队列；
一个消息接收线程WorkerReceiver处理从QuorumCnxManager接收的消息。

### 消息处理器

下面我们分别来看消息处理的两个工作线程

#### 发送消息工作线程WorkerSender
//
```java
/**
         * This worker simply dequeues a message to send and
         * and queues it on the manager's queue.
         * 发送消息工作线程，出列一个需要发送的消息，并把它放入
         * 管理器QuorumCnxManager的队列
         */

        class WorkerSender extends ZooKeeperThread {
            volatile boolean stop;
            QuorumCnxManager manager;

            WorkerSender(QuorumCnxManager manager){
                super("WorkerSender");
                this.stop = false;
                this.manager = manager;
            }

            public void run() {
                while (!stop) {
                    try {
                        //从发送消息队列拉取消息
                        ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
                        if(m == null) continue;

                        process(m);
                    } catch (InterruptedException e) {
                        break;
                    }
                }
                LOG.info("WorkerSender is down");
            }

            /**
             * Called by run() once there is a new message to send.
             *  消息处理
             * @param m     message to send
             */
            void process(ToSend m) {
                ByteBuffer requestBuffer = buildMsg(m.state.ordinal(),
                                                    m.leader,
                                                    m.zxid,
                                                    m.electionEpoch,
                                                    m.peerEpoch,
                                                    m.configData);

                manager.toSend(m.sid, requestBuffer);
            }
        }
```




//
```java
 public class QuorumCnxManager {
     /*
     * Local IP address
     * 本地ip地址
     */
    final long mySid;
    final int socketTimeout;
    final Map<Long, QuorumPeer.QuorumServer> view;
    final boolean listenOnAllIPs;
    private ThreadPoolExecutor connectionExecutor;
    private final Set<Long> inprogressConnections = Collections
            .synchronizedSet(new HashSet<Long>());
    private QuorumAuthServer authServer;
    private QuorumAuthLearner authLearner;
    private boolean quorumSaslAuthEnabled;
    /*
     * Counter to count connection processing threads.
     */
    private AtomicInteger connectionThreadCnt = new AtomicInteger(0);

    /*
     * Mapping from Peer to Thread number 消息发送Worker
     */
    final ConcurrentHashMap<Long, SendWorker> senderWorkerMap;
    /**
     * 发送消息队列
     */
    final ConcurrentHashMap<Long, ArrayBlockingQueue<ByteBuffer>> queueSendMap;
    final ConcurrentHashMap<Long, ByteBuffer> lastMessageSent;

    /*
     * Reception queue 接受消息队列
     */
    public final ArrayBlockingQueue<Message> recvQueue;
    /*
     * Object to synchronize access to recvQueue
     */
    private final Object recvQLock = new Object();

    /*
     * Shutdown flag
     */

    volatile boolean shutdown = false;

    /*
     * Listener thread
     */
    public final Listener listener;

    /*
     * Counter to count worker threads
     */
    private AtomicInteger threadCnt = new AtomicInteger(0);

    /*
     * Socket options for TCP keepalive
     */
    private final boolean tcpKeepAlive = Boolean.getBoolean("zookeeper.tcpKeepAlive");
     ...
/**
     * Processes invoke this message to queue a message to send. Currently, 
     * only leader election uses it.
     * 处理发送消息队列中的消息，当前leader选举会使用到
     */
    public void toSend(Long sid, ByteBuffer b) {
        /*
         * If sending message to myself, then simply enqueue it (loopback).
         * 如果目的地是自己，则添加消息到接收队列
         */
        if (this.mySid == sid) {
             b.position(0);
             addToRecvQueue(new Message(b.duplicate(), sid));
            /*
             * Otherwise send to the corresponding thread to send.
             */
        } else {
             /*
              * Start a new connection if doesn't have one already.
              * 开启一个新的连接，并建立peer sid和发送消息队列的关系
              */
             ArrayBlockingQueue<ByteBuffer> bq = new ArrayBlockingQueue<ByteBuffer>(
                SEND_CAPACITY);
             ArrayBlockingQueue<ByteBuffer> oldq = queueSendMap.putIfAbsent(sid, bq);
             if (oldq != null) {
                 addToSendQueue(oldq, b);
             } else {
                 addToSendQueue(bq, b);
             }
             //建立连接
             connectOne(sid);
                
        }
    }
    private void addToSendQueue(ArrayBlockingQueue<ByteBuffer> queue,
          ByteBuffer buffer) {
        if (queue.remainingCapacity() == 0) {
            try {
                queue.remove();
            } catch (NoSuchElementException ne) {
                // element could be removed by poll()
                LOG.debug("Trying to remove from an empty " +
                        "Queue. Ignoring exception " + ne);
            }
        }
        try {
            queue.add(buffer);
        } catch (IllegalStateException ie) {
            // This should never happen
            LOG.error("Unable to insert an element in the queue " + ie);
        }
    }
    /**
     * Try to establish a connection to server with id sid.
     *  尝试建立到sid的连接
     *  @param sid  server id
     */
    synchronized void connectOne(long sid){
        if (senderWorkerMap.get(sid) != null) {
            LOG.debug("There is a connection already for server " + sid);
            return;
        }
        synchronized (self.QV_LOCK) {
            boolean knownId = false;
            // Resolve hostname for the remote server before attempting to
            // connect in case the underlying ip address has changed.
            self.recreateSocketAddresses(sid);
            Map<Long, QuorumPeer.QuorumServer> lastCommittedView = self.getView();
            QuorumVerifier lastSeenQV = self.getLastSeenQuorumVerifier();
            Map<Long, QuorumPeer.QuorumServer> lastProposedView = lastSeenQV.getAllMembers();
            if (lastCommittedView.containsKey(sid)) {
                //上次提交的peer视图包含sid
                knownId = true;
                if (connectOne(sid, lastCommittedView.get(sid).electionAddr))
                    return;
            }
            if (lastSeenQV != null && lastProposedView.containsKey(sid)
                    && (!knownId || (lastProposedView.get(sid).electionAddr !=
                    lastCommittedView.get(sid).electionAddr))) {
                //上次提议的peer view
                knownId = true;
                if (connectOne(sid, lastProposedView.get(sid).electionAddr))
                    return;
            }
            if (!knownId) {
                LOG.warn("Invalid server id: " + sid);
                return;
            }
        }
    }
    /**
     * Try to establish a connection to server with id sid using its electionAddr.
     *  尝试使用选举地址建立连接
     *  @param sid  server id
     *  @return boolean success indication
     */
    synchronized private boolean connectOne(long sid, InetSocketAddress electionAddr){
        if (senderWorkerMap.get(sid) != null) {
            LOG.debug("There is a connection already for server " + sid);
            return true;
        }

        Socket sock = null;
        try {
             LOG.debug("Opening channel to server " + sid);
             sock = new Socket();
             setSockOpts(sock);
             //建立socket连接
             sock.connect(electionAddr, cnxTO);
             LOG.debug("Connected to server " + sid);
            // Sends connection request asynchronously if the quorum
            // sasl authentication is enabled. This is required because
            // sasl server authentication process may take few seconds to
            // finish, this may delay next peer connection requests.
            if (quorumSaslAuthEnabled) {
                initiateConnectionAsync(sock, sid);
            } else {
                initiateConnection(sock, sid);
            }
             return true;
         } catch (UnresolvedAddressException e) {
             // Sun doesn't include the address that causes this
             // exception to be thrown, also UAE cannot be wrapped cleanly
             // so we log the exception in order to capture this critical
             // detail.
             LOG.warn("Cannot open channel to " + sid
                     + " at election address " + electionAddr, e);
             closeSocket(sock);
             throw e;
         } catch (IOException e) {
             LOG.warn("Cannot open channel to " + sid
                     + " at election address " + electionAddr,
                     e);
             closeSocket(sock);
             return false;
         }
   
    }
    /**
     * If this server has initiated the connection, then it gives up on the
     * connection if it loses challenge. Otherwise, it keeps the connection.
     */
    public void initiateConnection(final Socket sock, final Long sid) {
        try {
            startConnection(sock, sid);
        } catch (IOException e) {
            LOG.error("Exception while connecting, id: {}, addr: {}, closing learner connection",
                    new Object[] { sid, sock.getRemoteSocketAddress() }, e);
            closeSocket(sock);
            return;
        }
    }
/**
     * 建立连接
     * @param sock
     * @param sid
     * @return
     * @throws IOException
     */
    private boolean startConnection(Socket sock, Long sid)
            throws IOException {
        DataOutputStream dout = null;
        DataInputStream din = null;
        try {
            // Use BufferedOutputStream to reduce the number of IP packets. This is
            // important for x-DC scenarios.
            BufferedOutputStream buf = new BufferedOutputStream(sock.getOutputStream());
            dout = new DataOutputStream(buf);

            // Sending id and challenge

            // represents protocol version (in other words - message type)
            //协议版本
            dout.writeLong(PROTOCOL_VERSION);
            //server id
            dout.writeLong(self.getId());
            String addr = self.getElectionAddress().getHostString() + ":" + self.getElectionAddress().getPort();
            byte[] addr_bytes = addr.getBytes();
            //选举地址
            dout.writeInt(addr_bytes.length);
            dout.write(addr_bytes);
            dout.flush();

            din = new DataInputStream(
                    new BufferedInputStream(sock.getInputStream()));
        } catch (IOException e) {
            LOG.warn("Ignoring exception reading or writing challenge: ", e);
            closeSocket(sock);
            return false;
        }

        // authenticate learner
        QuorumPeer.QuorumServer qps = self.getVotingView().get(sid);
        if (qps != null) {
            // TODO - investigate why reconfig makes qps null.
            authLearner.authenticate(sock, qps.hostname);
        }

        // If lost the challenge, then drop the new connection
        //如果挑战丢失，则关闭连接诶
        if (sid > self.getId()) {
            LOG.info("Have smaller server identifier, so dropping the " +
                     "connection: (" + sid + ", " + self.getId() + ")");
            closeSocket(sock);
            // Otherwise proceed with the connection
        } else {
            //发送线程
            SendWorker sw = new SendWorker(sock, sid);
            //接收工作线程
            RecvWorker rw = new RecvWorker(sock, din, sid, sw);
            sw.setRecv(rw);

            SendWorker vsw = senderWorkerMap.get(sid);
            
            if(vsw != null)
                vsw.finish();
            
            senderWorkerMap.put(sid, sw);
            queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(
                        SEND_CAPACITY));
            
            sw.start();
            rw.start();
            
            return true;    
            
        }
        return false;
    }
    ...
     public QuorumCnxManager(QuorumPeer self,
                            final long mySid,
                            Map<Long,QuorumPeer.QuorumServer> view,
                            QuorumAuthServer authServer,
                            QuorumAuthLearner authLearner,
                            int socketTimeout,
                            boolean listenOnAllIPs,
                            int quorumCnxnThreadsSize,
                            boolean quorumSaslAuthEnabled) {
        this.recvQueue = new ArrayBlockingQueue<Message>(RECV_CAPACITY);
        this.queueSendMap = new ConcurrentHashMap<Long, ArrayBlockingQueue<ByteBuffer>>();
        this.senderWorkerMap = new ConcurrentHashMap<Long, SendWorker>();
        this.lastMessageSent = new ConcurrentHashMap<Long, ByteBuffer>();
        
        String cnxToValue = System.getProperty("zookeeper.cnxTimeout");
        if(cnxToValue != null){
            this.cnxTO = Integer.parseInt(cnxToValue);
        }
        
        this.self = self;

        this.mySid = mySid;
        this.socketTimeout = socketTimeout;
        this.view = view;
        this.listenOnAllIPs = listenOnAllIPs;

        initializeAuth(mySid, authServer, authLearner, quorumCnxnThreadsSize,
                quorumSaslAuthEnabled);

        // Starts listener thread that waits for connection requests
        listener = new Listener();
        listener.setName("QuorumPeerListener");
    }
    /**
     * Thread to listen on some port
     */
    public class Listener extends ZooKeeperThread {

        volatile ServerSocket ss = null;

        public Listener() {
            // During startup of thread, thread name will be overridden to
            // specific election address
            super("ListenerThread");
        }

        /**
         * Sleeps on accept().
         * 开启监听
         */
        @Override
        public void run() {
            int numRetries = 0;
            InetSocketAddress addr;
            Socket client = null;
            while((!shutdown) && (numRetries < 3)){
                try {
                    ss = new ServerSocket();
                    ss.setReuseAddress(true);
                    if (self.getQuorumListenOnAllIPs()) {
                        int port = self.getElectionAddress().getPort();
                        addr = new InetSocketAddress(port);
                    } else {
                        // Resolve hostname for this server in case the
                        // underlying ip address has changed.
                        self.recreateSocketAddresses(self.getId());
                        addr = self.getElectionAddress();
                    }
                    LOG.info("My election bind port: " + addr.toString());
                    setName(addr.toString());
                    ss.bind(addr);
                    while (!shutdown) {
                        try {
                            client = ss.accept();
                            setSockOpts(client);
                            LOG.info("Received connection request "
                                     + client.getRemoteSocketAddress());
                            // Receive and handle the connection request
                            // asynchronously if the quorum sasl authentication is
                            // enabled. This is required because sasl server
                            // authentication process may take few seconds to finish,
                            // this may delay next peer connection requests.
                            if (quorumSaslAuthEnabled) {
                                receiveConnectionAsync(client);
                            } else {
                                receiveConnection(client);
                            }
                            numRetries = 0;
                        } catch (SocketTimeoutException e) {
                            LOG.warn("The socket is listening for the election accepted "
                                     + "and it timed out unexpectedly, but will retry."
                                     + "see ZOOKEEPER-2836");
                        }
                    }
                } catch (IOException e) {
                    if (shutdown) {
                        break;
                    }
                    LOG.error("Exception while listening", e);
                    numRetries++;
                    try {
                        ss.close();
                        Thread.sleep(1000);
                    } catch (IOException ie) {
                        LOG.error("Error closing server socket", ie);
                    } catch (InterruptedException ie) {
                        LOG.error("Interrupted while sleeping. " +
                            "Ignoring exception", ie);
                    }
                    closeSocket(client);
                }
            }
            LOG.info("Leaving listener");
            if (!shutdown) {
                LOG.error("As I'm leaving the listener thread, "
                        + "I won't be able to participate in leader "
                        + "election any longer: "
                        + self.getElectionAddress());
            } else if (ss != null) {
                // Clean up for shutdown.
                try {
                    ss.close();
                } catch (IOException ie) {
                    // Don't log an error for shutdown.
                    LOG.debug("Error closing server socket", ie);
                }
            }
        }

 }
```
从上面可以看出，发送消息工作线程WorkerSender，从FastLeaderElection的发送队列poll消息，并把它放入管理器QuorumCnxManager的队列，如果需要则建立消息关联的peer, 如果连接peer的id大于
当前peer的id，则关闭连接，否则启动发送工作线程SendWorker和接收线程RecvWorker。 同时QuorumCnxManager在启动时，启动监听，监听peer的连接。来看一下发送工作线程

//QuorumCnxManager：SendWorker
```java
/**
     * Thread to send messages. Instance waits on a queue, and send a message as
     * soon as there is one available. If connection breaks, then opens a new
     * one.
     * 发送消息线程，等下消息队列，有消息可用。如果连接断开，则打开一个新的
     */
    class SendWorker extends ZooKeeperThread {
        Long sid;
        Socket sock;
        /**
         * 接收工作线程
         */
        RecvWorker recvWorker;
        volatile boolean running = true;
        DataOutputStream dout;

        /**
         * An instance of this thread receives messages to send
         * through a queue and sends them to the server sid.
         * 
         * @param sock
         *            Socket to remote peer
         * @param sid
         *            Server identifier of remote peer
         */
        SendWorker(Socket sock, Long sid) {
            super("SendWorker:" + sid);
            this.sid = sid;
            this.sock = sock;
            recvWorker = null;
            try {
                dout = new DataOutputStream(sock.getOutputStream());
            } catch (IOException e) {
                LOG.error("Unable to access socket output stream", e);
                closeSocket(sock);
                running = false;
            }
            LOG.debug("Address of remote peer: " + this.sid);
        }
         @Override
        public void run() {
            threadCnt.incrementAndGet();
            try {
                /**
                 * If there is nothing in the queue to send, then we
                 * send the lastMessage to ensure that the last message
                 * was received by the peer. The message could be dropped
                 * in case self or the peer shutdown their connection
                 * (and exit the thread) prior to reading/processing
                 * the last message. Duplicate messages are handled correctly
                 * by the peer.
                 *
                 * If the send queue is non-empty, then we have a recent
                 * message than that stored in lastMessage. To avoid sending
                 * stale message, we should send the message in the send queue.
                 */
                ArrayBlockingQueue<ByteBuffer> bq = queueSendMap.get(sid);
                if (bq == null || isSendQueueEmpty(bq)) {
                   ByteBuffer b = lastMessageSent.get(sid);
                   if (b != null) {
                       LOG.debug("Attempting to send lastMessage to sid=" + sid);
                       send(b);
                   }
                }
            } catch (IOException e) {
                LOG.error("Failed to send last message. Shutting down thread.", e);
                this.finish();
            }
            
            try {
                while (running && !shutdown && sock != null) {

                    ByteBuffer b = null;
                    try {
                        ArrayBlockingQueue<ByteBuffer> bq = queueSendMap
                                .get(sid);
                        if (bq != null) {
                            b = pollSendQueue(bq, 1000, TimeUnit.MILLISECONDS);
                        } else {
                            LOG.error("No queue of incoming messages for " +
                                      "server " + sid);
                            break;
                        }

                        if(b != null){
                            lastMessageSent.put(sid, b);
                            send(b);
                        }
                    } catch (InterruptedException e) {
                        LOG.warn("Interrupted while waiting for message on queue",
                                e);
                    }
                }
            } catch (Exception e) {
                LOG.warn("Exception when using channel: for id " + sid
                         + " my id = " + QuorumCnxManager.this.mySid
                         + " error = " + e);
            }
            this.finish();
            LOG.warn("Send worker leaving thread " + " id " + sid + " my id = " + self.getId());
        }
         synchronized void send(ByteBuffer b) throws IOException {
            byte[] msgBytes = new byte[b.capacity()];
            try {
                b.position(0);
                b.get(msgBytes);
            } catch (BufferUnderflowException be) {
                LOG.error("BufferUnderflowException ", be);
                return;
            }
            dout.writeInt(b.capacity());
            dout.write(b.array());
            dout.flush();
        }

```
从上面可以看出发送消息线程，从消息队列拉取消息，并通过Socket的DataOutputStream，发送给peer。

#### 消息接收线程WorkerReceiver
//Messager：WorkerReceiver
```java
  * Receives messages from instance of QuorumCnxManager on
         * method run(), and processes such messages.
         * 处理从QuorumCnxManager接收的消息
         */

        class WorkerReceiver extends ZooKeeperThread  {
            volatile boolean stop;
            QuorumCnxManager manager;

            WorkerReceiver(QuorumCnxManager manager) {
                super("WorkerReceiver");
                this.stop = false;
                this.manager = manager;
            }

            public void run() {

                Message response;
                while (!stop) {
                    // Sleeps on receive
                    try {
                        //从接收队列，拉取消息
                        response = manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);
                        if(response == null) continue;

                        // The current protocol and two previous generations all send at least 28 bytes
                        //至少有28个字节，方可用
                        if (response.buffer.capacity() < 28) {
                            LOG.error("Got a short response: " + response.buffer.capacity());
                            continue;
                        }

                        // this is the backwardCompatibility mode in place before ZK-107
                        // It is for a version of the protocol in which we didn't send peer epoch
                        // With peer epoch and version the message became 40 bytes
                        boolean backCompatibility28 = (response.buffer.capacity() == 28);

                        // this is the backwardCompatibility mode for no version information 兼容模式
                        boolean backCompatibility40 = (response.buffer.capacity() == 40);
                        
                        response.buffer.clear();

                        // Instantiate Notification and set its attributes
                        Notification n = new Notification();

                        int rstate = response.buffer.getInt();
                        long rleader = response.buffer.getLong();
                        long rzxid = response.buffer.getLong();
                        long relectionEpoch = response.buffer.getLong();
                        long rpeerepoch;

                        int version = 0x0;
                        if (!backCompatibility28) {
                            rpeerepoch = response.buffer.getLong();
                            if (!backCompatibility40) {
                                /*
                                 * Version added in 3.4.6
                                 */
                                
                                version = response.buffer.getInt();
                            } else {
                                LOG.info("Backward compatibility mode (36 bits), server id: {}", response.sid);
                            }
                        } else {
                            LOG.info("Backward compatibility mode (28 bits), server id: {}", response.sid);
                            rpeerepoch = ZxidUtils.getEpochFromZxid(rzxid);
                        }

                        QuorumVerifier rqv = null;

                        // check if we have a version that includes config. If so extract config info from message.
                        if (version > 0x1) {
                            int configLength = response.buffer.getInt();
                            byte b[] = new byte[configLength];

                            response.buffer.get(b);
                                                       
                            synchronized(self) {
                                try {
                                    rqv = self.configFromString(new String(b));
                                    QuorumVerifier curQV = self.getQuorumVerifier();
                                    if (rqv.getVersion() > curQV.getVersion()) {
                                        LOG.info("{} Received version: {} my version: {}", self.getId(),
                                                Long.toHexString(rqv.getVersion()),
                                                Long.toHexString(self.getQuorumVerifier().getVersion()));
                                        if (self.getPeerState() == ServerState.LOOKING) {
                                            LOG.debug("Invoking processReconfig(), state: {}", self.getServerState());
                                            //重新配置，可能需要开启新一轮的选举
                                            self.processReconfig(rqv, null, null, false);
                                            if (!rqv.equals(curQV)) {
                                                LOG.info("restarting leader election");
                                                self.shuttingDownLE = true;
                                                self.getElectionAlg().shutdown();

                                                break;
                                            }
                                        } else {
                                            LOG.debug("Skip processReconfig(), state: {}", self.getServerState());
                                        }
                                    }
                                } catch (IOException e) {
                                    LOG.error("Something went wrong while processing config received from {}", response.sid);
                               } catch (ConfigException e) {
                                   LOG.error("Something went wrong while processing config received from {}", response.sid);
                               }
                            }                          
                        } else {
                            LOG.info("Backward compatibility mode (before reconfig), server id: {}", response.sid);
                        }
                       
                        /*
                         * If it is from a non-voting server (such as an observer or
                         * a non-voting follower), respond right away.
                         *
                         * 如果当前和下一轮投票的成员，不包含消息源sid
                         */
                        if(!validVoter(response.sid)) {
                            Vote current = self.getCurrentVote();
                            QuorumVerifier qv = self.getQuorumVerifier();
                            ToSend notmsg = new ToSend(ToSend.mType.notification,
                                    current.getId(),
                                    current.getZxid(),
                                    logicalclock.get(),
                                    self.getPeerState(),
                                    response.sid,
                                    current.getPeerEpoch(),
                                    qv.toString().getBytes());
                            //发送通知消息
                            sendqueue.offer(notmsg);
                        } else {
                            //peer处理新消息
                            // Receive new message
                            if (LOG.isDebugEnabled()) {
                                LOG.debug("Receive new notification message. My id = "
                                        + self.getId());
                            }

                            // State of peer that sent this message 确认peer状态
                            QuorumPeer.ServerState ackstate = QuorumPeer.ServerState.LOOKING;
                            switch (rstate) {
                            case 0:
                                ackstate = QuorumPeer.ServerState.LOOKING;
                                break;
                            case 1:
                                ackstate = QuorumPeer.ServerState.FOLLOWING;
                                break;
                            case 2:
                                ackstate = QuorumPeer.ServerState.LEADING;
                                break;
                            case 3:
                                ackstate = QuorumPeer.ServerState.OBSERVING;
                                break;
                            default:
                                continue;
                            }

                            n.leader = rleader;
                            n.zxid = rzxid;
                            n.electionEpoch = relectionEpoch;
                            n.state = ackstate;
                            n.sid = response.sid;
                            n.peerEpoch = rpeerepoch;
                            n.version = version;
                            n.qv = rqv;
                            /*
                             * Print notification info
                             */
                            if(LOG.isInfoEnabled()){
                                printNotification(n);
                            }

                            /*
                             * If this server is looking, then send proposed leader
                             * 如果server正在look，则发送提议leader
                             */

                            if(self.getPeerState() == QuorumPeer.ServerState.LOOKING){
                                //添加到消息队列
                                recvqueue.offer(n);

                                /*
                                 * Send a notification back if the peer that sent this
                                 * message is also looking and its logical clock is
                                 * lagging behind.
                                 * 如果发送消息的peer正常looker，并且逻辑时钟落后，则发送通知
                                 */
                                if((ackstate == QuorumPeer.ServerState.LOOKING)
                                        && (n.electionEpoch < logicalclock.get())){
                                    Vote v = getVote();
                                    QuorumVerifier qv = self.getQuorumVerifier();
                                    ToSend notmsg = new ToSend(ToSend.mType.notification,
                                            v.getId(),
                                            v.getZxid(),
                                            logicalclock.get(),
                                            self.getPeerState(),
                                            response.sid,
                                            v.getPeerEpoch(),
                                            qv.toString().getBytes());
                                    //添加到发送队列
                                    sendqueue.offer(notmsg);
                                }
                            } else {
                                /*
                                 * If this server is not looking, but the one that sent the ack
                                 * is looking, then send back what it believes to be the leader.
                                 * 若果当前peer为非looker，但是回复的peer为looker，则发送当前peer相信的leader信息
                                 */
                                Vote current = self.getCurrentVote();
                                if(ackstate == QuorumPeer.ServerState.LOOKING){
                                    if(LOG.isDebugEnabled()){
                                        LOG.debug("Sending new notification. My id ={} recipient={} zxid=0x{} leader={} config version = {}",
                                                self.getId(),
                                                response.sid,
                                                Long.toHexString(current.getZxid()),
                                                current.getId(),
                                                Long.toHexString(self.getQuorumVerifier().getVersion()));
                                    }

                                    QuorumVerifier qv = self.getQuorumVerifier();
                                    ToSend notmsg = new ToSend(
                                            ToSend.mType.notification,
                                            current.getId(),
                                            current.getZxid(),
                                            current.getElectionEpoch(),
                                            self.getPeerState(),
                                            response.sid,
                                            current.getPeerEpoch(),
                                            qv.toString().getBytes());
                                    //入列
                                    sendqueue.offer(notmsg);
                                }
                            }
                        }
                    } catch (InterruptedException e) {
                        LOG.warn("Interrupted Exception while waiting for new message" +
                                e.toString());
                    }
                }
                LOG.info("WorkerReceiver is down");
            }
        }
```
从上面看出，消息接收线程WorkerReceiver从QuorumCnxManager的接收队列中拉取消息，并解析出peer的状态（LOOKING, 观察，Follower，或者leader），
事务id，leaderId，leader选举时间戳，peer的时间戳等信息；如果peer不在当前投票的视图范围之内，同步当前peer的状态（构建通知消息（服务id，事务id，peer状态，时间戳等），并放到发送队列），
然后更新通知(事务id，leaderId，leader选举时间戳,peer时间戳)，如果当前peer的状态为LOOKING，则添加通知消息到peer的消息接收队列，如果peer状态为LOOKING，则同步当前节点的投票信息给peer，
 若果当前节点为非looker，而peer为looker，则发送当前peer相信的leader信息。

再来看一下消息接收线程



////QuorumCnxManager：RecvWorker
```java
  /**
     * Thread to receive messages. Instance waits on a socket read. If the
     * channel breaks, then removes itself from the pool of receivers.
     * 接收消息线程。等到从socket读取数据，如果通道断开，从接收器池中移除自己
     */
    class RecvWorker extends ZooKeeperThread {
        Long sid;
        Socket sock;
        volatile boolean running = true;
        final DataInputStream din;
        /**
         * 发送工作线程
         */
        final SendWorker sw;

        RecvWorker(Socket sock, DataInputStream din, Long sid, SendWorker sw) {
            super("RecvWorker:" + sid);
            this.sid = sid;
            this.sock = sock;
            this.sw = sw;
            this.din = din;
            try {
                // OK to wait until socket disconnects while reading.
                sock.setSoTimeout(0);
            } catch (IOException e) {
                LOG.error("Error while accessing socket for " + sid, e);
                closeSocket(sock);
                running = false;
            }
        }
          @Override
        public void run() {
            threadCnt.incrementAndGet();
            try {
                while (running && !shutdown && sock != null) {
                    /**
                     * Reads the first int to determine the length of the
                     * message
                     */
                    int length = din.readInt();
                     if (length <= 0 || length > PACKETMAXSIZE) {
                        throw new IOException(
                                "Received packet with invalid packet: "
                                        + length);
                    }
                    /**
                     * Allocates a new ByteBuffer to receive the message
                     */
                    byte[] msgArray = new byte[length];
                    din.readFully(msgArray, 0, length);
                    ByteBuffer message = ByteBuffer.wrap(msgArray);
                    addToRecvQueue(new Message(message.duplicate(), sid));
                }
            } catch (Exception e) {
                LOG.warn("Connection broken for id " + sid + ", my id = "
                         + QuorumCnxManager.this.mySid + ", error = " , e);
            } finally {
                LOG.warn("Interrupting SendWorker");
                sw.finish();
                closeSocket(sock);
            }
        }
          public void addToRecvQueue(Message msg) {
        synchronized(recvQLock) {
            if (recvQueue.remainingCapacity() == 0) {
                try {
                    recvQueue.remove();
                } catch (NoSuchElementException ne) {
                    // element could be removed by poll()
                     LOG.debug("Trying to remove from an empty " +
                         "recvQueue. Ignoring exception " + ne);
                }
            }
            try {
                recvQueue.add(msg);
            } catch (IllegalStateException ie) {
                // This should never happen
                LOG.error("Unable to insert element in the recvQueue " + ie);
            }
        }
    }
...
    }
```
从上面可以看出接收工作线程RecvWorker，主要是从Socket的Data输入流中读取数据，并组装成消息，放到QuorumCnxManager的消息接收队列，待消息接收线程WorkerReceiver处理。
今天现将到者，选举还有很多内容，在QuorumPeer中处理，我们放到下一篇。


## QuorumPeer



# 附

启动peer选举策略实际启动的为fast leader 选举策略，如果peer状态为LOOKING， 创建投票（最后提交的日志id，时间戳，peerId）。

fast leader 选举策略启动时实际上启动了一个消息处理器Messenger。 消息处理器内部有一个发送消息工作线程WorkerSender，出列一个需要发送的消息，并把它放入管理器QuorumCnxManager的队列；
一个消息接收线程WorkerReceiver处理从QuorumCnxManager接收的消息。

发送消息工作线程WorkerSender，从FastLeaderElection的发送队列poll消息，并把它放入管理器QuorumCnxManager的队列，如果需要则建立消息关联的peer，并发送协议版本，服务id及选举地址, 如果连接peer的id大于
当前peer的id，则关闭连接，否则启动发送工作线程SendWorker和接收线程RecvWorker。 同时QuorumCnxManager在启动时，启动监听，监听peer的连接。发送消息线程SendWorker，从消息队列拉取消息，并通过Socket的DataOutputStream，发送给peer。


消息接收线程WorkerReceiver从QuorumCnxManager的接收队列中拉取消息，并解析出peer的状态（LOOKING, 观察，Follower，或者leader），
事务id，leaderId，leader选举时间戳，peer的时间戳等信息；如果peer不在当前投票的视图范围之内，同步当前peer的状态（构建通知消息（服务id，事务id，peer状态，时间戳等），并放到发送队列），
然后更新通知(事务id，leaderId，leader选举时间戳,peer时间戳)，如果当前peer的状态为LOOKING，则添加通知消息到peer的消息接收队列，如果peer状态为LOOKING，则同步当前节点的投票信息给peer，
 若果当前节点为非looker，而peer为looker，则发送当前peer相信的leader信息。

接收工作线程RecvWorker，主要是从Socket的Data输入流中读取数据，并组装成消息，放到QuorumCnxManager的消息接收队列，待消息接收线程WorkerReceiver处理。