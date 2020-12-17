---
layout: page
title: Zookeeper框架设计及源码解读一（Zookeeper启动）
subtitle: Zookeeper框架设计及源码解读一（Zookeeper启动）
date: 2020-12-17 20:42:00
author: valuewithTime
catalog: true
category: Zookeeper
categories:
    - Zookeeper
tags:
    - Zookeeper
---

# 引言
从Hadoop的高可用环境，接触到Zookeeper。Zookeeper在高可用集群架构中扮演者重要的角色。除此之外，在微服务盛行的当前，Dubbo默认采用Zookeeper最为注册中心。TBSchedule使用它存储定时任务，控制任务的并发执行。同时Zookeeper作为Raft 一致性协议的经典之作，接下来我们将一探究竟。



# 目录
* [概要框架设计](#概要框架设计)
* [源码分析](#源码分析)
    * [启动Zookeeper](#启动zookeeper)
    * [Leader选举](#leader选举)
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

从Zkserver.sh脚本中，可以看到入口为QuorumPeerMain, 

```java
public class QuorumPeerMain {
     protected QuorumPeer quorumPeer;

    /**
     * To start the replicated server specify the configuration file name on
     * the command line.
     * 启动一个可复制的server
     * @param args path to the configfile
     */
    public static void main(String[] args) {
        QuorumPeerMain main = new QuorumPeerMain();
        try {
            main.initializeAndRun(args);
        }
        /**
     * @param args
     * @throws ConfigException
     * @throws IOException
     * @throws AdminServerException
     */
    protected void initializeAndRun(String[] args)
        throws ConfigException, IOException, AdminServerException
    {
        //解析配置
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        }

        // Start and schedule the the purge task TODO read 开启清除快照定时任务
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();
        if (args.length == 1 && config.isDistributed()) {
            //分布式
            runFromConfig(config);
        } else {
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone， 单机版
            ZooKeeperServerMain.main(args);
        }
    }
    ...
     /**
     * @param config
     * @throws IOException
     * @throws AdminServerException
     */
    public void runFromConfig(QuorumPeerConfig config)
            throws IOException, AdminServerException
    {
        ...
         quorumPeer.start();
         ...
    }
}
```
从上面可以看出，QuorumPeerMain启动时解析配置文件，并启动quorum peer；


//QuorumPeer
```java
 @Override
    public synchronized void start() {
        if (!getView().containsKey(myid)) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
         }
         //加载数据数据树DataTree
        loadDataBase();
        //NettyServerCnxnFactory, NIOServerCnxnFactory
        startServerCnxnFactory();
        try {
            //管理服务器
            adminServer.start();
        } catch (AdminServerException e) {
            LOG.warn("Problem starting AdminServer", e);
            System.out.println(e);
        }
        //启动leader选举
        startLeaderElection();
        super.start();
    }
```
启动QuorumPeer，做几点重要的事情，加载数据到内存数据库，启动服务监听请求和开始leader选举。下面我们分别来看这几点。


1. 加载数据到内存数据库


//QuorumPeer
```java
/**
     * 从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中
     */
    private void loadDataBase() {
        try {
            //从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中
            zkDb.loadDataBase();

            // load the epochs
            long lastProcessedZxid = zkDb.getDataTree().lastProcessedZxid;
            long epochOfZxid = ZxidUtils.getEpochFromZxid(lastProcessedZxid);
            try {
                //读取当前Epoch
                currentEpoch = readLongFromFile(CURRENT_EPOCH_FILENAME);
            } catch(FileNotFoundException e) {
            	// pick a reasonable epoch number
            	// this should only happen once when moving to a
            	// new code version
            	currentEpoch = epochOfZxid;
            	LOG.info(CURRENT_EPOCH_FILENAME
            	        + " not found! Creating with a reasonable default of {}. This should only happen when you are upgrading your installation",
            	        currentEpoch);
            	writeLongToFile(CURRENT_EPOCH_FILENAME, currentEpoch);
            }
            if (epochOfZxid > currentEpoch) {
                throw new IOException("The current epoch, " + ZxidUtils.zxidToString(currentEpoch) + ", is older than the last zxid, " + lastProcessedZxid);
            }
            try {
                acceptedEpoch = readLongFromFile(ACCEPTED_EPOCH_FILENAME);
            } catch(FileNotFoundException e) {
            	// pick a reasonable epoch number
            	// this should only happen once when moving to a
            	// new code version
            	acceptedEpoch = epochOfZxid;
            	writeLongToFile(ACCEPTED_EPOCH_FILENAME, acceptedEpoch);
            }
            if (acceptedEpoch < currentEpoch) {
                throw new IOException("The accepted epoch, " + ZxidUtils.zxidToString(acceptedEpoch) + " is less than the current epoch, " + ZxidUtils.zxidToString(currentEpoch));
            }
        } 
    }
```

//ZKDatabase
```java
 /**
     * load the database from the disk onto memory and also add
     * the transactions to the committedlog in memory.
     * 从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中
     * @return the last valid zxid on disk
     * @throws IOException
     */
    public long loadDataBase() throws IOException {
        long zxid = snapLog.restore(dataTree, sessionsWithTimeouts, commitProposalPlaybackListener);
        initialized = true;
        return zxid;
    }
```

//FileTxnSnapLog
```java
/**
     * this function restores the server
     * database after reading from the
     * snapshots and transaction logs
     * 从快照和交易日志重构数据库
     * @param dt the datatree to be restored
     * @param sessions the sessions to be restored
     * @param listener the playback listener to run on the
     * database restoration
     * @return the highest zxid restored
     * @throws IOException
     */
    public long restore(DataTree dt, Map<Long, Integer> sessions,
            PlayBackListener listener) throws IOException {
        long deserializeResult = snapLog.deserialize(dt, sessions);
        ...
          return fastForwardFromEdits(dt, sessions, listener);
    }
```

```java
/**
     * deserialize a data tree from the most recent snapshot
     * @return the zxid of the snapshot
     */
    public long deserialize(DataTree dt, Map<Long, Integer> sessions)
            throws IOException {
        // we run through 100 snapshots (not all of them)
        // if we cannot get it running within 100 snapshots
        // we should  give up
        List<File> snapList = findNValidSnapshots(100);
        if (snapList.size() == 0) {
            return -1L;
        }
        File snap = null;
        boolean foundValid = false;
        for (int i = 0, snapListSize = snapList.size(); i < snapListSize; i++) {
            snap = snapList.get(i);
            LOG.info("Reading snapshot " + snap);
            try (InputStream snapIS = new BufferedInputStream(new FileInputStream(snap));
                 CheckedInputStream crcIn = new CheckedInputStream(snapIS, new Adler32())) {
                InputArchive ia = BinaryInputArchive.getArchive(crcIn);
                deserialize(dt, sessions, ia);
                long checkSum = crcIn.getChecksum().getValue();
                long val = ia.readLong("val");
                if (val != checkSum) {
                    throw new IOException("CRC corruption in snapshot :  " + snap);
                }
                foundValid = true;
                break;
            } catch (IOException e) {
                LOG.warn("problem reading snap file " + snap, e);
            }
        }
        if (!foundValid) {
            throw new IOException("Not able to find valid snapshots in " + snapDir);
        }
        dt.lastProcessedZxid = Util.getZxidFromName(snap.getName(), SNAPSHOT_FILE_PREFIX);
        return dt.lastProcessedZxid;
    }

```
从上面可以看出，初始化数据库主要是从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中

2. 启动服务监听请求
//QuorumPeer
```java
/**
     * 启动server
     */
    private void startServerCnxnFactory() {
        if (cnxnFactory != null) {
            cnxnFactory.start();
        }
        if (secureCnxnFactory != null) {
            secureCnxnFactory.start();
        }
    }

```


//NettyServerCnxnFactory
```java
public class NettyServerCnxnFactory extends ServerCnxnFactory {
    private static final Logger LOG = LoggerFactory.getLogger(NettyServerCnxnFactory.class);

    /**
     *
     */
    ServerBootstrap bootstrap;
    /**
     *
     */
    Channel parentChannel;
    /**
     *
     */
    ChannelGroup allChannels = new DefaultChannelGroup("zkServerCnxns");
    /**
     *
     */
    Map<InetAddress, Set<NettyServerCnxn>> ipMap =
        new HashMap<InetAddress, Set<NettyServerCnxn>>( );
    /**
     *
     */
    InetSocketAddress localAddress;
    int maxClientCnxns = 60;
NettyServerCnxnFactory() {
        bootstrap = new ServerBootstrap(
                new NioServerSocketChannelFactory(
                        Executors.newCachedThreadPool(),
                        Executors.newCachedThreadPool()));
        // parent channel
        bootstrap.setOption("reuseAddress", true);
        // child channels
        bootstrap.setOption("child.tcpNoDelay", true);
        /* set socket linger to off, so that socket close does not block */
        bootstrap.setOption("child.soLinger", -1);
        bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
            @Override
            public ChannelPipeline getPipeline() throws Exception {
                ChannelPipeline p = Channels.pipeline();
                if (secure) {
                    initSSL(p);
                }
                //通道处理器
                p.addLast("servercnxnfactory", channelHandler);

                return p;
            }
        });
    }
      public void start() {
        LOG.info("binding to port " + localAddress);
        parentChannel = bootstrap.bind(localAddress);
    }
     /**
     * 通道处理器
     */
    CnxnChannelHandler channelHandler = new CnxnChannelHandler();
     /**
     * This is an inner class since we need to extend SimpleChannelHandler, but
     * NettyServerCnxnFactory already extends ServerCnxnFactory. By making it inner
     * this class gets access to the member variables and methods.
     */
    @Sharable
    class CnxnChannelHandler extends SimpleChannelHandler {

        @Override
        public void channelClosed(ChannelHandlerContext ctx, ChannelStateEvent e)
            throws Exception
        {
            if (LOG.isTraceEnabled()) {
                LOG.trace("Channel closed " + e);
            }
            allChannels.remove(ctx.getChannel());
        }

        @Override
        public void channelConnected(ChannelHandlerContext ctx,
                ChannelStateEvent e) throws Exception
        {
            if (LOG.isTraceEnabled()) {
                LOG.trace("Channel connected " + e);
            }

            NettyServerCnxn cnxn = new NettyServerCnxn(ctx.getChannel(),
                    zkServer, NettyServerCnxnFactory.this);
            ctx.setAttachment(cnxn);

            if (secure) {
                //开启ssl验证
                SslHandler sslHandler = ctx.getPipeline().get(SslHandler.class);
                ChannelFuture handshakeFuture = sslHandler.handshake();
                handshakeFuture.addListener(new CertificateVerifier(sslHandler, cnxn));
            } else {
                allChannels.add(ctx.getChannel());
                addCnxn(cnxn);
            }
        }

        @Override
        public void channelDisconnected(ChannelHandlerContext ctx,
                ChannelStateEvent e) throws Exception
        {
            if (LOG.isTraceEnabled()) {
                LOG.trace("Channel disconnected " + e);
            }
            NettyServerCnxn cnxn = (NettyServerCnxn) ctx.getAttachment();
            if (cnxn != null) {
                if (LOG.isTraceEnabled()) {
                    LOG.trace("Channel disconnect caused close " + e);
                }
                cnxn.close();
            }
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, ExceptionEvent e)
            throws Exception
        {
            LOG.warn("Exception caught " + e, e.getCause());
            NettyServerCnxn cnxn = (NettyServerCnxn) ctx.getAttachment();
            if (cnxn != null) {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("Closing " + cnxn);
                }
                cnxn.close();
            }
        }

        /**
         * @param ctx
         * @param e
         * @throws Exception
         */
        @Override
        public void messageReceived(ChannelHandlerContext ctx, MessageEvent e)
            throws Exception
        {
            if (LOG.isTraceEnabled()) {
                LOG.trace("message received called " + e.getMessage());
            }
            try {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("New message " + e.toString()
                            + " from " + ctx.getChannel());
                }
                NettyServerCnxn cnxn = (NettyServerCnxn)ctx.getAttachment();
                synchronized(cnxn) {
                    //处理消息
                    processMessage(e, cnxn);
                }
            } catch(Exception ex) {
                LOG.error("Unexpected exception in receive", ex);
                throw ex;
            }
        }

        /**
         * @param e
         * @param cnxn
         */
        private void processMessage(MessageEvent e, NettyServerCnxn cnxn) {
            if (LOG.isDebugEnabled()) {
                LOG.debug(Long.toHexString(cnxn.sessionId) + " queuedBuffer: "
                        + cnxn.queuedBuffer);
            }

            if (e instanceof NettyServerCnxn.ResumeMessageEvent) {
                LOG.debug("Received ResumeMessageEvent");
                if (cnxn.queuedBuffer != null) {
                    if (LOG.isTraceEnabled()) {
                        LOG.trace("processing queue "
                                + Long.toHexString(cnxn.sessionId)
                                + " queuedBuffer 0x"
                                + ChannelBuffers.hexDump(cnxn.queuedBuffer));
                    }
                    //
                    cnxn.receiveMessage(cnxn.queuedBuffer);
                    if (!cnxn.queuedBuffer.readable()) {
                        LOG.debug("Processed queue - no bytes remaining");
                        cnxn.queuedBuffer = null;
                    } else {
                        LOG.debug("Processed queue - bytes remaining");
                    }
                } else {
                    LOG.debug("queue empty");
                }
                cnxn.channel.setReadable(true);
            } else {
                ChannelBuffer buf = (ChannelBuffer)e.getMessage();
                if (LOG.isTraceEnabled()) {
                    LOG.trace(Long.toHexString(cnxn.sessionId)
                            + " buf 0x"
                            + ChannelBuffers.hexDump(buf));
                }
                
                if (cnxn.throttled) {
                    LOG.debug("Received message while throttled");
                    // we are throttled, so we need to queue
                    if (cnxn.queuedBuffer == null) {
                        LOG.debug("allocating queue");
                        cnxn.queuedBuffer = dynamicBuffer(buf.readableBytes());
                    }
                    cnxn.queuedBuffer.writeBytes(buf);
                    if (LOG.isTraceEnabled()) {
                        LOG.trace(Long.toHexString(cnxn.sessionId)
                                + " queuedBuffer 0x"
                                + ChannelBuffers.hexDump(cnxn.queuedBuffer));
                    }
                } else {
                    LOG.debug("not throttled");
                    if (cnxn.queuedBuffer != null) {
                        if (LOG.isTraceEnabled()) {
                            LOG.trace(Long.toHexString(cnxn.sessionId)
                                    + " queuedBuffer 0x"
                                    + ChannelBuffers.hexDump(cnxn.queuedBuffer));
                        }
                        cnxn.queuedBuffer.writeBytes(buf);
                        if (LOG.isTraceEnabled()) {
                            LOG.trace(Long.toHexString(cnxn.sessionId)
                                    + " queuedBuffer 0x"
                                    + ChannelBuffers.hexDump(cnxn.queuedBuffer));
                        }

                        cnxn.receiveMessage(cnxn.queuedBuffer);
                        if (!cnxn.queuedBuffer.readable()) {
                            LOG.debug("Processed queue - no bytes remaining");
                            cnxn.queuedBuffer = null;
                        } else {
                            LOG.debug("Processed queue - bytes remaining");
                        }
                    } else {
                        cnxn.receiveMessage(buf);
                        if (buf.readable()) {
                            if (LOG.isTraceEnabled()) {
                                LOG.trace("Before copy " + buf);
                            }
                            cnxn.queuedBuffer = dynamicBuffer(buf.readableBytes()); 
                            cnxn.queuedBuffer.writeBytes(buf);
                            if (LOG.isTraceEnabled()) {
                                LOG.trace("Copy is " + cnxn.queuedBuffer);
                                LOG.trace(Long.toHexString(cnxn.sessionId)
                                        + " queuedBuffer 0x"
                                        + ChannelBuffers.hexDump(cnxn.queuedBuffer));
                            }
                        }
                    }
                }
            }
        }

        @Override
        public void writeComplete(ChannelHandlerContext ctx,
                WriteCompletionEvent e) throws Exception
        {
            if (LOG.isTraceEnabled()) {
                LOG.trace("write complete " + e);
            }
        }

        /**
         *
         */
        private final class CertificateVerifier
                implements ChannelFutureListener {
            /**
             *
             */
            private final SslHandler sslHandler;
            /**
             *
             */
            private final NettyServerCnxn cnxn;

            CertificateVerifier(SslHandler sslHandler, NettyServerCnxn cnxn) {
                this.sslHandler = sslHandler;
                this.cnxn = cnxn;
            }

            /**
             * Only allow the connection to stay open if certificate passes auth
             */
            public void operationComplete(ChannelFuture future)
                    throws SSLPeerUnverifiedException {
                if (future.isSuccess()) {
                    LOG.debug("Successful handshake with session 0x{}",
                            Long.toHexString(cnxn.sessionId));
                    SSLEngine eng = sslHandler.getEngine();
                    SSLSession session = eng.getSession();
                    cnxn.setClientCertificateChain(session.getPeerCertificates());

                    String authProviderProp
                            = System.getProperty(ZKConfig.SSL_AUTHPROVIDER, "x509");

                    X509AuthenticationProvider authProvider =
                            (X509AuthenticationProvider)
                                    ProviderRegistry.getProvider(authProviderProp);

                    if (authProvider == null) {
                        LOG.error("Auth provider not found: {}", authProviderProp);
                        cnxn.close();
                        return;
                    }

                    if (KeeperException.Code.OK !=
                            authProvider.handleAuthentication(cnxn, null)) {
                        LOG.error("Authentication failed for session 0x{}",
                                Long.toHexString(cnxn.sessionId));
                        cnxn.close();
                        return;
                    }

                    allChannels.add(future.getChannel());
                    addCnxn(cnxn);
                } else {
                    LOG.error("Unsuccessful handshake with session 0x{}",
                            Long.toHexString(cnxn.sessionId));
                    cnxn.close();
                }
            }
        }
    }
...
}
```

//
```java
public class NettyServerCnxn extends ServerCnxn {
    private static final Logger LOG = LoggerFactory.getLogger(NettyServerCnxn.class);
    Channel channel;
    ChannelBuffer queuedBuffer;
    /**
     *
     */
    volatile boolean throttled;
    ByteBuffer bb;
    ByteBuffer bbLen = ByteBuffer.allocate(4);
    long sessionId;
    int sessionTimeout;
    AtomicLong outstandingCount = new AtomicLong();
    Certificate[] clientChain;
    volatile boolean closingChannel;

    /** The ZooKeeperServer for this connection. May be null if the server
     * is not currently serving requests (for example if the server is not
     * an active quorum participant.
     */
    private volatile ZooKeeperServer zkServer;

    /**
     *
     */
    NettyServerCnxnFactory factory;
    boolean initialized;
    
    NettyServerCnxn(Channel channel, ZooKeeperServer zks, NettyServerCnxnFactory factory) {
        this.channel = channel;
        this.closingChannel = false;
        this.zkServer = zks;
        this.factory = factory;
        if (this.factory.login != null) {
            this.zooKeeperSaslServer = new ZooKeeperSaslServer(factory.login);
        }
    }
    ...
/**
     * 接受消息
     * @param message
     */
    public void receiveMessage(ChannelBuffer message) {
        try {
            while(message.readable() && !throttled) {
                if (bb != null) {
                    if (LOG.isTraceEnabled()) {
                        LOG.trace("message readable " + message.readableBytes()
                                + " bb len " + bb.remaining() + " " + bb);
                        ByteBuffer dat = bb.duplicate();
                        dat.flip();
                        LOG.trace(Long.toHexString(sessionId)
                                + " bb 0x"
                                + ChannelBuffers.hexDump(
                                        ChannelBuffers.copiedBuffer(dat)));
                    }

                    if (bb.remaining() > message.readableBytes()) {
                        int newLimit = bb.position() + message.readableBytes();
                        bb.limit(newLimit);
                    }
                    message.readBytes(bb);
                    bb.limit(bb.capacity());

                    if (LOG.isTraceEnabled()) {
                        LOG.trace("after readBytes message readable "
                                + message.readableBytes()
                                + " bb len " + bb.remaining() + " " + bb);
                        ByteBuffer dat = bb.duplicate();
                        dat.flip();
                        LOG.trace("after readbytes "
                                + Long.toHexString(sessionId)
                                + " bb 0x"
                                + ChannelBuffers.hexDump(
                                        ChannelBuffers.copiedBuffer(dat)));
                    }
                    //读完缓冲区
                    if (bb.remaining() == 0) {
                        packetReceived();
                        bb.flip();

                        ZooKeeperServer zks = this.zkServer;
                        if (zks == null || !zks.isRunning()) {
                            throw new IOException("ZK down");
                        }
                        if (initialized) {
                            //ZkServer， 处理数据包
                            zks.processPacket(this, bb);

                            if (zks.shouldThrottle(outstandingCount.incrementAndGet())) {
                                disableRecvNoWait();
                            }
                        } else {
                            LOG.debug("got conn req request from "
                                    + getRemoteSocketAddress());
                            zks.processConnectRequest(this, bb);
                            initialized = true;
                        }
                        bb = null;
                    }
                } else {
                    if (LOG.isTraceEnabled()) {
                        LOG.trace("message readable "
                                + message.readableBytes()
                                + " bblenrem " + bbLen.remaining());
                        ByteBuffer dat = bbLen.duplicate();
                        dat.flip();
                        LOG.trace(Long.toHexString(sessionId)
                                + " bbLen 0x"
                                + ChannelBuffers.hexDump(
                                        ChannelBuffers.copiedBuffer(dat)));
                    }

                    if (message.readableBytes() < bbLen.remaining()) {
                        bbLen.limit(bbLen.position() + message.readableBytes());
                    }
                    message.readBytes(bbLen);
                    bbLen.limit(bbLen.capacity());
                    if (bbLen.remaining() == 0) {
                        bbLen.flip();

                        if (LOG.isTraceEnabled()) {
                            LOG.trace(Long.toHexString(sessionId)
                                    + " bbLen 0x"
                                    + ChannelBuffers.hexDump(
                                            ChannelBuffers.copiedBuffer(bbLen)));
                        }
                        int len = bbLen.getInt();
                        if (LOG.isTraceEnabled()) {
                            LOG.trace(Long.toHexString(sessionId)
                                    + " bbLen len is " + len);
                        }

                        bbLen.clear();
                        if (!initialized) {
                            if (checkFourLetterWord(channel, message, len)) {
                                return;
                            }
                        }
                        if (len < 0 || len > BinaryInputArchive.maxBuffer) {
                            throw new IOException("Len error " + len);
                        }
                        bb = ByteBuffer.allocate(len);
                    }
                }
            }
        } catch(IOException e) {
            LOG.warn("Closing connection to " + getRemoteSocketAddress(), e);
            close();
        }
    }
    ...
}
```
从上面可以看出，启动ServerCnxnFactory,实际上是启动了一个基于Netty服务，客户端发送的数据，交由NettyServerCnxn处理，NettyServerCnxn数据包的处理，实际委托给ZooKeeperServer。消息的处理后面单独在将。

小节一下

Zookeeper启动时，首先解析配置文件，根据配置文件选择启动单例还是集群模式。集群模式启动，首先从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中。然后启动ServerCnxnFactory,监听客户端的请求。实际上是启动了一个基于Netty服务，客户端发送的数据，交由NettyServerCnxn处理，NettyServerCnxn数据包的处理，实际委托给ZooKeeperServer。


leader选举内容较多，我们单开一偏。


# 总
Zookeeper整体架构主要分为数据的存储，消息，leader选举和数据同步这几个模块。leader选举主要是在集群处于混沌的状态下，从集群peer的提议中选择集群的leader，其他为follower或observer，维护集群peer的统一视图，保证整个集群的数据一致性，如果在leader选举成功后，存在follower日志落后的情况，则将事务日志同步给follower。针对消息模块，peer之间的通信包需要序列化和反序列才能发送和处理，具体的消息处理由集群相应角色的消息处理器链来处理。针对客户单的节点的创建，数据修改等操作，将会先写到内存数据库，如果有提交请求，则将数据写到事务日志，同时Zookeeper会定时将内存数据库写到快照日志，以防止没有提交的日志，在宕机的情况下丢失。数据同步模块将leader的事务日志同步给Follower，保证整个集群数据的一致性。

Zookeeper启动时，首先解析配置文件，根据配置文件选择启动单例还是集群模式。集群模式启动，首先从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中。然后启动ServerCnxnFactory,监听客户端的请求。实际上是启动了一个基于Netty服务，客户端发送的数据，交由NettyServerCnxn处理，NettyServerCnxn数据包的处理，实际委托给ZooKeeperServer。


# 附