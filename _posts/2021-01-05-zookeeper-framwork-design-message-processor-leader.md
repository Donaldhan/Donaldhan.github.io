---
layout: page
title: Zookeeper框架设计及源码解读六（Leader消息处理）
subtitle: Zookeeper框架设计及源码解读六（Leader消息处理）
date: 2021-01-05 19:37:00
author: valuewithTime
catalog: true
category: Zookeeper
categories:
    - Zookeeper
tags:
    - Zookeeper
---

# 引言
跟随者，跟随领导者首先连接leader，注册follower状态，在leader连接的过程中，如果发现消息队列中有LEADERINFO请求，则响应leader然后同步leader，这部分逻辑和观察者一致，主要有同步leader日志快照，如果为TUNC命令，则截取事务日志。
跟随者处理消息包，如果为提议消息，则log请求，提交消息，则委托给commit处理器处理，如果为同步请求则从同步请求队列拉取消息，并委托给commit处理器处理。
领导者首先加载数据到内存数据库,创建新leader数据包，然后等待所有Quorum， peer 全部投票完毕, 启动server(LeaderZooKeeperServer)。
节点状态的确定逻辑为集群没有配置，则为LOOING状态，如果节点投注的serverId为当前节点，则为Leader，如果学习的类型为参与者，则为节点状态为跟踪者，如果学习类型为观察状态，则为观察者。

今天我们来看一下消息的处理




# 目录
* [概要框架设计](#概要框架设计)
* [源码分析](#源码分析)
    * [启动Zookeeper](#启动zookeeper)
    * [Leader选举](#leader选举)
        * [QuorumPeer选举机制及处理策略](#quorumpeer选举机制及处理策略)
        * [LOOKING提议投票阶段](#looking提议投票阶段)
        * [OBSERVING观察者同步leader](#observing观察者同步leader)
        * [FOLLOWING跟随者状态](#following跟随者状态)
        * [LEADING领导者状态](#leading领导者状态)
    * [消息处理](#消息处理)
        * [Leader消息处理](#Leader消息处理)
        * [Follower消息处理](#follower消息处理)
        * [Observer消息处理](#observer消息处理)
    * [数据同步](#数据同步)
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


## Leader选举

[Zookeeper框架设计及源码解读二（快速选举策略及选举消息的发送与接收）](https://donaldhan.github.io/zookeeper/2020/12/22/zookeeper-framwork-design-leader-select.html) 

### LOOKING提议投票阶段
[Zookeeper框架设计及源码解读三（leader选举LOOKING阶段）](https://donaldhan.github.io/zookeeper/2020/12/23/zookeeper-framwork-design-leader-select-quorum-peer-looking.html)  


#### OBSERVING观察者同步leader
[Zookeeper框架设计及源码解读四（观察者观察leader）](https://donaldhan.github.io/zookeeper/2020/12/29/zookeeper-framwork-design-leader-select-observing.html) 


### FOLLOWING跟随者状态
### LEADING领导者状态
[Zookeeper框架设计及源码解读五（跟随者状态、领导者状态）](https://donaldhan.github.io/zookeeper/2020/12/30/zookeeper-framwork-design-leader-select-following-lead.html) 




## 消息的处理

消息的处理涉及到3中角色观察者、跟随者、和领导者，我们分别来看这个3个角色的消息处理。 在选举结束之后，观察者、跟随者、和领导者都会启动一个server，分别为，LeaderZooKeeperServer,FollowerZooKeeperServer,ObserverZooKeeperServer.
我们先来看消息各个角色的消息处理器链；

### 观察者、跟随者、和领导者消息处理链

先看LeaderZooKeeperServer的启动

//LeaderZooKeeperServer
```java
public class LeaderZooKeeperServer extends QuorumZooKeeperServer {
    ...
     @Override
    public synchronized void startup() {
        super.startup();
        if (containerManager != null) {
            containerManager.start();
        }
    }
    ...
      @Override
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        RequestProcessor toBeAppliedProcessor = new Leader.ToBeAppliedRequestProcessor(finalProcessor, getLeader());
        commitProcessor = new CommitProcessor(toBeAppliedProcessor,
                Long.toString(getServerId()), false,
                getZooKeeperServerListener());
        commitProcessor.start();
        ProposalRequestProcessor proposalProcessor = new ProposalRequestProcessor(this,
                commitProcessor);
        proposalProcessor.initialize();
        prepRequestProcessor = new PrepRequestProcessor(this, proposalProcessor);
        prepRequestProcessor.start();
        firstProcessor = new LeaderRequestProcessor(this, prepRequestProcessor);

        setupContainerManager();
    }
}
```


//ZooKeeperServer
```java
/**
     * 启动
     */
    public synchronized void startup() {
        if (sessionTracker == null) {
            createSessionTracker();
        }
        //启动会话追踪
        startSessionTracker();
        //设置请求处理器
        setupRequestProcessors();

        registerJMX();
        setState(State.RUNNING);
        notifyAll();
    }
```

从上面可以看出Leader的处理器链为LeaderRequestProcessor->PrepRequestProcessor->ProposalRequestProcessor->CommitProcessor->ToBeAppliedRequestProcessor->FinalRequestProcessor;


再来看FollowerZooKeeperServer

//FollowerZooKeeperServer
```java
public class FollowerZooKeeperServer extends LearnerZooKeeperServer {
    private static final Logger LOG =
        LoggerFactory.getLogger(FollowerZooKeeperServer.class);

    /*
     * Pending sync requests
     * 同步请求队列
     */
    ConcurrentLinkedQueue<Request> pendingSyncs;

    /**
     * @param logFactory
     * @param self
     * @param zkDb
     * @throws IOException
     */
    FollowerZooKeeperServer(FileTxnSnapLog logFactory,QuorumPeer self,
            ZKDatabase zkDb) throws IOException {
        super(logFactory, self.tickTime, self.minSessionTimeout,
                self.maxSessionTimeout, zkDb, self);
        this.pendingSyncs = new ConcurrentLinkedQueue<Request>();
    }

    public Follower getFollower(){
        return self.follower;
    }

    @Override
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        commitProcessor = new CommitProcessor(finalProcessor,
                Long.toString(getServerId()), true, getZooKeeperServerListener());
        commitProcessor.start();
        firstProcessor = new FollowerRequestProcessor(this, commitProcessor);
        ((FollowerRequestProcessor) firstProcessor).start();
        syncProcessor = new SyncRequestProcessor(this,
                new SendAckRequestProcessor((Learner)getFollower()));
        syncProcessor.start();
    }

    LinkedBlockingQueue<Request> pendingTxns = new LinkedBlockingQueue<Request>();

    /**
     * @param hdr
     * @param txn
     */
    public void logRequest(TxnHeader hdr, Record txn) {
        Request request = new Request(hdr.getClientId(), hdr.getCxid(), hdr.getType(), hdr, txn, hdr.getZxid());
        if ((request.zxid & 0xffffffffL) != 0) {
            pendingTxns.add(request);
        }
        syncProcessor.processRequest(request);
    }
    ...
}
```
从上可以看出follower的消息处理器链为SendAckRequestProcessor->SyncRequestProcessor->FollowerRequestProcessor->CommitProcessor->FinalRequestProcessor.


再来看ObserverZooKeeperServer

//ObserverZooKeeperServer
```java
public class ObserverZooKeeperServer extends LearnerZooKeeperServer {
    private static final Logger LOG =
        LoggerFactory.getLogger(ObserverZooKeeperServer.class);        
    
    /**
     * Enable since request processor for writing txnlog to disk and
     * take periodic snapshot. Default is ON.
     */
    
    private boolean syncRequestProcessorEnabled = this.self.getSyncEnabled();
    
    /*
     * Pending sync requests
     * 同步请求
     */
    ConcurrentLinkedQueue<Request> pendingSyncs = 
        new ConcurrentLinkedQueue<Request>();

    ObserverZooKeeperServer(FileTxnSnapLog logFactory, QuorumPeer self, ZKDatabase zkDb) throws IOException {
        super(logFactory, self.tickTime, self.minSessionTimeout, self.maxSessionTimeout, zkDb, self);
        LOG.info("syncEnabled =" + syncRequestProcessorEnabled);
    }
    ...
    /**
     * Set up the request processors for an Observer:
     * firstProcesor->commitProcessor->finalProcessor
     */
    @Override
    protected void setupRequestProcessors() {      
        // We might consider changing the processor behaviour of 
        // Observers to, for example, remove the disk sync requirements.
        // Currently, they behave almost exactly the same as followers.
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        commitProcessor = new CommitProcessor(finalProcessor,
                Long.toString(getServerId()), true,
                getZooKeeperServerListener());
        commitProcessor.start();
        firstProcessor = new ObserverRequestProcessor(this, commitProcessor);
        ((ObserverRequestProcessor) firstProcessor).start();

        /*
         * Observer should write to disk, so that the it won't request
         * too old txn from the leader which may lead to getting an entire
         * snapshot.
         *
         * However, this may degrade performance as it has to write to disk
         * and do periodic snapshot which may double the memory requirements
         */
        if (syncRequestProcessorEnabled) {
            syncProcessor = new SyncRequestProcessor(this, null);
            syncProcessor.start();
        }
    }
}
```
从上面可以看出，ObserverZooKeeperServer的处理器链为SyncRequestProcessor->ObserverRequestProcessor->CommitProcessor->FinalRequestProcessor;

LeaderZooKeeperServer,FollowerZooKeeperServer,ObserverZooKeeperServer.


观察者、跟随者、和领导者启动server，分别为，LeaderZooKeeperServer,FollowerZooKeeperServer,ObserverZooKeeperServer.
Leader的消息处处理器链为LeaderRequestProcessor->PrepRequestProcessor->ProposalRequestProcessor->CommitProcessor->ToBeAppliedRequestProcessor->FinalRequestProcessor;
Follower的消息处理器链为SendAckRequestProcessor->SyncRequestProcessor->FollowerRequestProcessor->CommitProcessor->FinalRequestProcessor.
Observer的消息处处理器链为SyncRequestProcessor->ObserverRequestProcessor->CommitProcessor->FinalRequestProcessor;



### Leader消息处理

首先来看LeaderRequestProcessor
#### LeaderRequestProcessor
//LeaderRequestProcessor
```java
public class LeaderRequestProcessor implements RequestProcessor {
    private static final Logger LOG = LoggerFactory
            .getLogger(LeaderRequestProcessor.class);

    private final LeaderZooKeeperServer lzks;

    private final RequestProcessor nextProcessor;

    public LeaderRequestProcessor(LeaderZooKeeperServer zks,
            RequestProcessor nextProcessor) {
        this.lzks = zks;
        this.nextProcessor = nextProcessor;
    }

    @Override
    public void processRequest(Request request)
            throws RequestProcessorException {
        // Check if this is a local session and we are trying to create
        // an ephemeral node, in which case we upgrade the session
        //检查是否为本地会话，并尝试创建一个临时会话，在这个过程中，我们有可能升级会话
        Request upgradeRequest = null;
        try {
            upgradeRequest = lzks.checkUpgradeSession(request);
        } catch (KeeperException ke) {
            if (request.getHdr() != null) {
                LOG.debug("Updating header");
                request.getHdr().setType(OpCode.error);
                request.setTxn(new ErrorTxn(ke.code().intValue()));
            }
            request.setException(ke);
            LOG.info("Error creating upgrade request " + ke.getMessage());
        } catch (IOException ie) {
            LOG.error("Unexpected error in upgrade", ie);
        }
        if (upgradeRequest != null) {
            nextProcessor.processRequest(upgradeRequest);
        }

        nextProcessor.processRequest(request);
    }

...
}
```
LeaderRequestProcessor处理器主要做的本地会话检查，并更新会话保活信息。 再来看PrepRequestProcessor

#### PrepRequestProcessor
//PrepRequestProcessor
```java
public class PrepRequestProcessor extends ZooKeeperCriticalThread implements
        RequestProcessor {
             /**
     * 事务请求对垒
     */
    LinkedBlockingQueue<Request> submittedRequests = new LinkedBlockingQueue<Request>();

    private final RequestProcessor nextProcessor;

    ZooKeeperServer zks;
    ...
     /**
     * 添加请求数据
     * @param request
     */
    public void processRequest(Request request) {
        submittedRequests.add(request);
    }
    @Override
    public void run() {
        try {
            while (true) {
                //从队列拉取请求
                Request request = submittedRequests.take();
                long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
                if (request.type == OpCode.ping) {
                    traceMask = ZooTrace.CLIENT_PING_TRACE_MASK;
                }
                if (LOG.isTraceEnabled()) {
                    ZooTrace.logRequest(LOG, traceMask, 'P', request, "");
                }
                if (Request.requestOfDeath == request) {
                    break;
                }
                //预处理请求
                pRequest(request);
            }
        } catch (RequestProcessorException e) {
            if (e.getCause() instanceof XidRolloverException) {
                LOG.info(e.getCause().getMessage());
            }
            handleException(this.getName(), e);
        } catch (Exception e) {
            handleException(this.getName(), e);
        }
        LOG.info("PrepRequestProcessor exited loop!");
    }
    ...
        }
```
从上可以看出， PrepRequestProcessor处理消息，首先添加到内部的提交请求队列，然后启动线程预处理请求。
再来看预处理请求的详细操作：

```java
/**
     * This method will be called inside the ProcessRequestThread, which is a
     * singleton, so there will be a single thread calling this code.
     *
     * @param request
     */
    protected void pRequest(Request request) throws RequestProcessorException {
        // LOG.info("Prep>>> cxid = " + request.cxid + " type = " +
        // request.type + " id = 0x" + Long.toHexString(request.sessionId));
        request.setHdr(null);
        request.setTxn(null);

        try {
            switch (request.type) {
            case OpCode.createContainer:
            case OpCode.create:
            case OpCode.create2:
                //创建请求
                CreateRequest create2Request = new CreateRequest();
                pRequest2Txn(request.type, zks.getNextZxid(), request, create2Request, true);
                break;
            case OpCode.createTTL:
                //创建TTL请求
                CreateTTLRequest createTtlRequest = new CreateTTLRequest();
                pRequest2Txn(request.type, zks.getNextZxid(), request, createTtlRequest, true);
                break;
            case OpCode.deleteContainer:
            case OpCode.delete:
                //删除
                DeleteRequest deleteRequest = new DeleteRequest();
                pRequest2Txn(request.type, zks.getNextZxid(), request, deleteRequest, true);
                break;
            case OpCode.setData:
                //设置数据
                SetDataRequest setDataRequest = new SetDataRequest();                
                pRequest2Txn(request.type, zks.getNextZxid(), request, setDataRequest, true);
                break;
            case OpCode.reconfig:
                ReconfigRequest reconfigRequest = new ReconfigRequest();
                ByteBufferInputStream.byteBuffer2Record(request.request, reconfigRequest);
                pRequest2Txn(request.type, zks.getNextZxid(), request, reconfigRequest, true);
                break;
            case OpCode.setACL:
                //
                SetACLRequest setAclRequest = new SetACLRequest();                
                pRequest2Txn(request.type, zks.getNextZxid(), request, setAclRequest, true);
                break;
            case OpCode.check:
                CheckVersionRequest checkRequest = new CheckVersionRequest();              
                pRequest2Txn(request.type, zks.getNextZxid(), request, checkRequest, true);
                break;
            case OpCode.multi:
                //提交事务
                MultiTransactionRecord multiRequest = new MultiTransactionRecord();
                try {
                    ByteBufferInputStream.byteBuffer2Record(request.request, multiRequest);
                } catch(IOException e) {
                    request.setHdr(new TxnHeader(request.sessionId, request.cxid, zks.getNextZxid(),
                            Time.currentWallTime(), OpCode.multi));
                    throw e;
                }
                List<Txn> txns = new ArrayList<Txn>();
                //Each op in a multi-op must have the same zxid!
                long zxid = zks.getNextZxid();
                KeeperException ke = null;

                //Store off current pending change records in case we need to rollback
                Map<String, ChangeRecord> pendingChanges = getPendingChanges(multiRequest);

                for(Op op: multiRequest) {
                    Record subrequest = op.toRequestRecord();
                    int type;
                    Record txn;

                    /* If we've already failed one of the ops, don't bother
                     * trying the rest as we know it's going to fail and it
                     * would be confusing in the logfiles.
                     */
                    if (ke != null) {
                        type = OpCode.error;
                        txn = new ErrorTxn(Code.RUNTIMEINCONSISTENCY.intValue());
                    }

                    /* Prep the request and convert to a Txn */
                    else {
                        try {
                            pRequest2Txn(op.getType(), zxid, request, subrequest, false);
                            type = request.getHdr().getType();
                            txn = request.getTxn();
                        } catch (KeeperException e) {
                            ke = e;
                            type = OpCode.error;
                            txn = new ErrorTxn(e.code().intValue());

                            LOG.info("Got user-level KeeperException when processing "
                                    + request.toString() + " aborting remaining multi ops."
                                    + " Error Path:" + e.getPath()
                                    + " Error:" + e.getMessage());

                            request.setException(e);

                            /* Rollback change records from failed multi-op */
                            rollbackPendingChanges(zxid, pendingChanges);
                        }
                    }

                    //FIXME: I don't want to have to serialize it here and then
                    //       immediately deserialize in next processor. But I'm
                    //       not sure how else to get the txn stored into our list.
                    ByteArrayOutputStream baos = new ByteArrayOutputStream();
                    BinaryOutputArchive boa = BinaryOutputArchive.getArchive(baos);
                    txn.serialize(boa, "request") ;
                    ByteBuffer bb = ByteBuffer.wrap(baos.toByteArray());

                    txns.add(new Txn(type, bb.array()));
                }

                request.setHdr(new TxnHeader(request.sessionId, request.cxid, zxid,
                        Time.currentWallTime(), request.type));
                request.setTxn(new MultiTxn(txns));

                break;

            //create/close session don't require request record
            case OpCode.createSession:
            case OpCode.closeSession:
                if (!request.isLocalSession()) {
                    pRequest2Txn(request.type, zks.getNextZxid(), request,
                                 null, true);
                }
                break;

            //All the rest don't need to create a Txn - just verify session
                //这些不需要创建事务，仅仅需要校验会话
            case OpCode.sync:
            case OpCode.exists:
            case OpCode.getData:
            case OpCode.getACL:
            case OpCode.getChildren:
            case OpCode.getChildren2:
            case OpCode.ping:
            case OpCode.setWatches:
            case OpCode.checkWatches:
            case OpCode.removeWatches:
                zks.sessionTracker.checkSession(request.sessionId,
                        request.getOwner());
                break;
            default:
                LOG.warn("unknown type " + request.type);
                break;
            }
        } catch (KeeperException e) {
            if (request.getHdr() != null) {
                request.getHdr().setType(OpCode.error);
                request.setTxn(new ErrorTxn(e.code().intValue()));
            }
            LOG.info("Got user-level KeeperException when processing "
                    + request.toString()
                    + " Error Path:" + e.getPath()
                    + " Error:" + e.getMessage());
            request.setException(e);
        } catch (Exception e) {
            // log at error level as we are returning a marshalling
            // error to the user
            LOG.error("Failed to process " + request, e);

            StringBuilder sb = new StringBuilder();
            ByteBuffer bb = request.request;
            if(bb != null){
                bb.rewind();
                while (bb.hasRemaining()) {
                    sb.append(Integer.toHexString(bb.get() & 0xff));
                }
            } else {
                sb.append("request buffer is null");
            }

            LOG.error("Dumping request buffer: 0x" + sb.toString());
            if (request.getHdr() != null) {
                request.getHdr().setType(OpCode.error);
                request.setTxn(new ErrorTxn(Code.MARSHALLINGERROR.intValue()));
            }
        }
        request.zxid = zks.getZxid();
        nextProcessor.processRequest(request);
    }
```
预处理消息，主要是针对事务性的CUD， 则构建响应的请求，比如CreateRequest，SetDataRequest等。针对非事务性R，则检查会话的有效性。

针对事务性消息，要处理为相应的事务：

```java
 /**
     * This method will be called inside the ProcessRequestThread, which is a
     * singleton, so there will be a single thread calling this code.
     *  预处理请求为事务
     * @param type
     * @param zxid
     * @param request
     * @param record
     */
    protected void pRequest2Txn(int type, long zxid, Request request,
                                Record record, boolean deserialize)
        throws KeeperException, IOException, RequestProcessorException
    {
        request.setHdr(new TxnHeader(request.sessionId, request.cxid, zxid,
                Time.currentWallTime(), type));

        switch (type) {
            case OpCode.create:
            case OpCode.create2:
            case OpCode.createTTL:
            case OpCode.createContainer: {
                //创建事务
                pRequest2TxnCreate(type, request, record, deserialize);
                break;
            }
            case OpCode.deleteContainer: {
                String path = new String(request.request.array());
                String parentPath = getParentPathAndValidate(path);
                ChangeRecord parentRecord = getRecordForPath(parentPath);
                ChangeRecord nodeRecord = getRecordForPath(path);
                if (nodeRecord.childCount > 0) {
                    throw new KeeperException.NotEmptyException(path);
                }
                if (EphemeralType.get(nodeRecord.stat.getEphemeralOwner()) == EphemeralType.NORMAL) {
                    throw new KeeperException.BadVersionException(path);
                }
                request.setTxn(new DeleteTxn(path));
                parentRecord = parentRecord.duplicate(request.getHdr().getZxid());
                parentRecord.childCount--;
                addChangeRecord(parentRecord);
                addChangeRecord(new ChangeRecord(request.getHdr().getZxid(), path, null, -1, null));
                break;
            }
            case OpCode.delete:
                //删除事物
                zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
                DeleteRequest deleteRequest = (DeleteRequest)record;
                if(deserialize)
                    ByteBufferInputStream.byteBuffer2Record(request.request, deleteRequest);
                String path = deleteRequest.getPath();
                String parentPath = getParentPathAndValidate(path);
                ChangeRecord parentRecord = getRecordForPath(parentPath);
                ChangeRecord nodeRecord = getRecordForPath(path);
                checkACL(zks, request.cnxn, parentRecord.acl, ZooDefs.Perms.DELETE, request.authInfo, path, null);
                checkAndIncVersion(nodeRecord.stat.getVersion(), deleteRequest.getVersion(), path);
                if (nodeRecord.childCount > 0) {
                    throw new KeeperException.NotEmptyException(path);
                }
                //DeleteTxn
                request.setTxn(new DeleteTxn(path));
                parentRecord = parentRecord.duplicate(request.getHdr().getZxid());
                parentRecord.childCount--;
                addChangeRecord(parentRecord);
                addChangeRecord(new ChangeRecord(request.getHdr().getZxid(), path, null, -1, null));
                break;
            case OpCode.setData:
                //设置数据
                zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
                SetDataRequest setDataRequest = (SetDataRequest)record;
                if(deserialize)
                    ByteBufferInputStream.byteBuffer2Record(request.request, setDataRequest);
                path = setDataRequest.getPath();
                validatePath(path, request.sessionId);
                nodeRecord = getRecordForPath(path);
                checkACL(zks, request.cnxn, nodeRecord.acl, ZooDefs.Perms.WRITE, request.authInfo, path, null);
                int newVersion = checkAndIncVersion(nodeRecord.stat.getVersion(), setDataRequest.getVersion(), path);
                //SetDataTxn
                request.setTxn(new SetDataTxn(path, setDataRequest.getData(), newVersion));
                nodeRecord = nodeRecord.duplicate(request.getHdr().getZxid());
                nodeRecord.stat.setVersion(newVersion);
                addChangeRecord(nodeRecord);
                break;
                ...
        }
```
从上面看出事务性预处理请求，主要是将请求包装成事件变更记录ZooKeeperServer，并保存到Zookeeper的请求变更记录集outstandingChanges中。

再来看协议请求处理器

#### ProposalRequestProcessor
//ProposalRequestProcessor
```java
public class ProposalRequestProcessor implements RequestProcessor {
    private static final Logger LOG =
        LoggerFactory.getLogger(ProposalRequestProcessor.class);

    LeaderZooKeeperServer zks;

    RequestProcessor nextProcessor;

    SyncRequestProcessor syncProcessor;
    ...
    public void processRequest(Request request) throws RequestProcessorException {
        // LOG.warn("Ack>>> cxid = " + request.cxid + " type = " +
        // request.type + " id = " + request.sessionId);
        // request.addRQRec(">prop");


        /* In the following IF-THEN-ELSE block, we process syncs on the leader.
         * If the sync is coming from a follower, then the follower
         * handler adds it to syncHandler. Otherwise, if it is a client of
         * the leader that issued the sync command, then syncHandler won't
         * contain the handler. In this case, we add it to syncHandler, and
         * call processRequest on the next processor.
         */

        if (request instanceof LearnerSyncRequest){
            //同步请求
            zks.getLeader().processSync((LearnerSyncRequest)request);
        } else {
            nextProcessor.processRequest(request);
            if (request.getHdr() != null) {
                // We need to sync and get consensus on any transactions
                try {
                    //请求地址不为空，则创建一个提议，并发送给其他成员
                    zks.getLeader().propose(request);
                } catch (XidRolloverException e) {
                    throw new RequestProcessorException(e.getMessage(), e);
                }
                syncProcessor.processRequest(request);
            }
        }
    }
}
```
ProposalRequestProcessor处理器，主要处理同步请求消息，针对同步请求，则发送消息到响应的server。


再来看CommitProcessor
#### CommitProcessor 
//CommitProcessor
```java
public class CommitProcessor extends ZooKeeperCriticalThread implements
        RequestProcessor {
    private static final Logger LOG = LoggerFactory.getLogger(CommitProcessor.class);

    /** Default: numCores */
    public static final String ZOOKEEPER_COMMIT_PROC_NUM_WORKER_THREADS =
        "zookeeper.commitProcessor.numWorkerThreads";
    /** Default worker pool shutdown timeout in ms: 5000 (5s) */
    public static final String ZOOKEEPER_COMMIT_PROC_SHUTDOWN_TIMEOUT =
        "zookeeper.commitProcessor.shutdownTimeout";

    /**
     * Incoming requests.
     */
    protected LinkedBlockingQueue<Request> queuedRequests =
        new LinkedBlockingQueue<Request>();

    /**
     * Requests that have been committed.
     * 已提交的请求队列
     */
    protected final LinkedBlockingQueue<Request> committedRequests =
        new LinkedBlockingQueue<Request>();

    /**
     * Requests that we are holding until commit comes in. Keys represent
     * session ids, each value is a linked list of the session's requests.
     */
    protected final Map<Long, LinkedList<Request>> pendingRequests =
            new HashMap<Long, LinkedList<Request>>(10000);

    /** The number of requests currently being processed */
    protected final AtomicInteger numRequestsProcessing = new AtomicInteger(0);

    RequestProcessor nextProcessor;

    /** For testing purposes, we use a separated stopping condition for the
     * outer loop.*/
    protected volatile boolean stoppedMainLoop = true; 
    protected volatile boolean stopped = true;
    private long workerShutdownTimeoutMS;
    protected WorkerService workerPool;
    private Object emptyPoolSync = new Object();

    /**
     * This flag indicates whether we need to wait for a response to come back from the
     * leader or we just let the sync operation flow through like a read. The flag will
     * be false if the CommitProcessor is in a Leader pipeline.
     */
    boolean matchSyncs;

    public CommitProcessor(RequestProcessor nextProcessor, String id,
                           boolean matchSyncs, ZooKeeperServerListener listener) {
        super("CommitProcessor:" + id, listener);
        this.nextProcessor = nextProcessor;
        this.matchSyncs = matchSyncs;
    }
    ...
    @Override
    public void run() {
        try {
            /*
             * In each iteration of the following loop we process at most
             * requestsToProcess requests of queuedRequests. We have to limit
             * the number of request we poll from queuedRequests, since it is
             * possible to endlessly poll read requests from queuedRequests, and
             * that will lead to a starvation of non-local committed requests.
             * 在每一个循环中，处理队列请求
             *
             */
            int requestsToProcess = 0;
            boolean commitIsWaiting = false;
			do {
                /*
                 * Since requests are placed in the queue before being sent to
                 * the leader, if commitIsWaiting = true, the commit belongs to
                 * the first update operation in the queuedRequests or to a
                 * request from a client on another server (i.e., the order of
                 * the following two lines is important!).
                 * 提交队列不为空，等待提交
                 */
                commitIsWaiting = !committedRequests.isEmpty();
                requestsToProcess =  queuedRequests.size();
                // Avoid sync if we have something to do
                if (requestsToProcess == 0 && !commitIsWaiting){
                    // Waiting for requests to process ，没有需要处理的请求，则等待
                    synchronized (this) {
                        while (!stopped && requestsToProcess == 0
                                && !commitIsWaiting) {
                            wait();
                            commitIsWaiting = !committedRequests.isEmpty();
                            requestsToProcess = queuedRequests.size();
                        }
                    }
                }
                /*
                 * Processing up to requestsToProcess requests from the incoming
                 * queue (queuedRequests), possibly less if a committed request
                 * is present along with a pending local write. After the loop,
                 * we process one committed request if commitIsWaiting.
                 * 处理请求队列中的消息。如果有待提价的请求，则处理之
                 */
                Request request = null;
                while (!stopped && requestsToProcess > 0
                        && (request = queuedRequests.poll()) != null) {
                    requestsToProcess--;
                    //处理需要提交的请求
                    if (needCommit(request)
                            || pendingRequests.containsKey(request.sessionId)) {
                        // Add request to pending
                        LinkedList<Request> requests = pendingRequests
                                .get(request.sessionId);
                        if (requests == null) {
                            requests = new LinkedList<Request>();
                            pendingRequests.put(request.sessionId, requests);
                        }
                        requests.addLast(request);
                    }
                    else {
                        sendToNextProcessor(request);
                    }
                    /*
                     * Stop feeding the pool if there is a local pending update
                     * and a committed request that is ready. Once we have a
                     * pending request with a waiting committed request, we know
                     * we can process the committed one. This is because commits
                     * for local requests arrive in the order they appeared in
                     * the queue, so if we have a pending request and a
                     * committed request, the committed request must be for that
                     * pending write or for a write originating at a different
                     * server.
                     * 等待提交
                     */
                    if (!pendingRequests.isEmpty() && !committedRequests.isEmpty()){
                        /*
                         * We set commitIsWaiting so that we won't check
                         * committedRequests again.
                         */
                        commitIsWaiting = true;
                        break;
                    }
                }

                // Handle a single committed request
                if (commitIsWaiting && !stopped){
                    waitForEmptyPool();

                    if (stopped){
                        return;
                    }

                    // Process committed head
                    if ((request = committedRequests.poll()) == null) {
                        throw new IOException("Error: committed head is null");
                    }

                    /*
                     * Check if request is pending, if so, update it with the committed info
                     */
                    LinkedList<Request> sessionQueue = pendingRequests
                            .get(request.sessionId);
                    if (sessionQueue != null) {
                        // If session queue != null, then it is also not empty.
                        Request topPending = sessionQueue.poll();
                        if (request.cxid != topPending.cxid) {
                            /*
                             * TL;DR - we should not encounter this scenario often under normal load.
                             * We pass the commit to the next processor and put the pending back with a warning.
                             *
                             * Generally, we can get commit requests that are not at the queue head after
                             * a session moved (see ZOOKEEPER-2684). Let's denote the previous server of the session
                             * with A, and the server that the session moved to with B (keep in mind that it is
                             * possible that the session already moved from B to a new server C, and maybe C=A).
                             * 1. If request.cxid < topPending.cxid : this means that the session requested this update
                             * from A, then moved to B (i.e., which is us), and now B receives the commit
                             * for the update after the session already performed several operations in B
                             * (and therefore its cxid is higher than that old request).
                             * 2. If request.cxid > topPending.cxid : this means that the session requested an updated
                             * from B with cxid that is bigger than the one we know therefore in this case we
                             * are A, and we lost the connection to the session. Given that we are waiting for a commit
                             * for that update, it means that we already sent the request to the leader and it will
                             * be committed at some point (in this case the order of cxid won't follow zxid, since zxid
                             * is an increasing order). It is not safe for us to delete the session's queue at this
                             * point, since it is possible that the session has newer requests in it after it moved
                             * back to us. We just leave the queue as it is, and once the commit arrives (for the old
                             * request), the finalRequestProcessor will see a closed cnxn handle, and just won't send a
                             * response.
                             * Also note that we don't have a local session, therefore we treat the request
                             * like any other commit for a remote request, i.e., we perform the update without sending
                             * a response.
                             */
                            LOG.warn("Got request " + request +
                                    " but we are expecting request " + topPending);
                            sessionQueue.addFirst(topPending);
                        } else {
                            /*
                             * Generally, we want to send to the next processor our version of the request,
                             * since it contains the session information that is needed for post update processing.
                             * In more details, when a request is in the local queue, there is (or could be) a client
                             * attached to this server waiting for a response, and there is other bookkeeping of
                             * requests that are outstanding and have originated from this server
                             * (e.g., for setting the max outstanding requests) - we need to update this info when an
                             * outstanding request completes. Note that in the other case (above), the operation
                             * originated from a different server and there is no local bookkeeping or a local client
                             * session that needs to be notified.
                             */
                            topPending.setHdr(request.getHdr());
                            topPending.setTxn(request.getTxn());
                            topPending.zxid = request.zxid;
                            request = topPending;
                        }
                    }

                    sendToNextProcessor(request);

                    waitForEmptyPool();

                    /*
                     * Process following reads if any, remove session queue if
                     * empty.
                     */
                    if (sessionQueue != null) {
                        while (!stopped && !sessionQueue.isEmpty()
                                && !needCommit(sessionQueue.peek())) {
                            sendToNextProcessor(sessionQueue.poll());
                        }
                        // Remove empty queues
                        if (sessionQueue.isEmpty()) {
                            pendingRequests.remove(request.sessionId);
                        }
                    }
                }
            } while (!stoppedMainLoop);
        } catch (Throwable e) {
            handleException(this.getName(), e);
        }
        LOG.info("CommitProcessor exited loop!");
    }

  /**
     * @param request
     * @return
     */
    protected boolean needCommit(Request request) {
        switch (request.type) {
            case OpCode.create:
            case OpCode.create2:
            case OpCode.createTTL:
            case OpCode.createContainer:
            case OpCode.delete:
            case OpCode.deleteContainer:
            case OpCode.setData:
            case OpCode.reconfig:
            case OpCode.multi:
            case OpCode.setACL:
                return true;
            case OpCode.sync:
                return matchSyncs;    
            case OpCode.createSession:
            case OpCode.closeSession:
                return !request.isLocalSession();
            default:
                return false;
        }
    }
    ...
        }
```
从上面可以看出CommitProcessor，主要是过滤出需要提交的请求，比如CRUD等，并交由下一个处理器处理。
#### ToBeAppliedRequestProcessor

//Leader.ToBeAppliedRequestProcessor
```java
static class ToBeAppliedRequestProcessor implements RequestProcessor {
        private final RequestProcessor next;

        private final Leader leader;


        /*
         * (non-Javadoc)
         *
         * @see org.apache.zookeeper.server.RequestProcessor#processRequest(org.apache.zookeeper.server.Request)
         */
        public void processRequest(Request request) throws RequestProcessorException {
            next.processRequest(request);

            // The only requests that should be on toBeApplied are write
            // requests, for which we will have a hdr. We can't simply use
            // request.zxid here because that is set on read requests to equal
            // the zxid of the last write op.
            if (request.getHdr() != null) {
                long zxid = request.getHdr().getZxid();
                Iterator<Proposal> iter = leader.toBeApplied.iterator();
                if (iter.hasNext()) {
                    Proposal p = iter.next();
                    if (p.request != null && p.request.zxid == zxid) {
                        iter.remove();
                        return;
                    }
                }
                LOG.error("Committed request not found on toBeApplied: "
                          + request);
            }
        }
        ...
}
```
ToBeAppliedRequestProcessor处理器，主要是保证提议为最新。

最后来看实际的消息处理

#### FinalRequestProcessor
//FinalRequestProcessor

```java
public class FinalRequestProcessor implements RequestProcessor {
    private static final Logger LOG = LoggerFactory.getLogger(FinalRequestProcessor.class);

    ZooKeeperServer zks;
    ...
     /**
     * @param request
     */
    public void processRequest(Request request) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Processing request:: " + request);
        }
        // request.addRQRec(">final");
        long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
        if (request.type == OpCode.ping) {
            traceMask = ZooTrace.SERVER_PING_TRACE_MASK;
        }
        if (LOG.isTraceEnabled()) {
            ZooTrace.logRequest(LOG, traceMask, 'E', request, "");
        }
        ProcessTxnResult rc = null;
        synchronized (zks.outstandingChanges) {
            // Need to process local session requests， 处理本地会话请求, KEY
            rc = zks.processTxn(request);
            ...
             // do not add non quorum packets to the queue.
            if (request.isQuorum()) {
                zks.getZKDatabase().addCommittedProposal(request);
            }
        }
        ServerCnxn cnxn = request.cnxn;

        String lastOp = "NA";
        zks.decInProcess();
        Code err = Code.OK;
        Record rsp = null;
        try {
            ...
            case OpCode.create: {
                lastOp = "CREA";
                rsp = new CreateResponse(rc.path);
                err = Code.get(rc.err);
                break;
            }
            case OpCode.create2:
            case OpCode.createTTL:
            case OpCode.createContainer: {
                lastOp = "CREA";
                rsp = new Create2Response(rc.path, rc.stat);
                err = Code.get(rc.err);
                break;
            }
            case OpCode.delete:
            case OpCode.deleteContainer: {
                lastOp = "DELE";
                err = Code.get(rc.err);
                break;
            }
            case OpCode.setData: {
                lastOp = "SETD";
                rsp = new SetDataResponse(rc.stat);
                err = Code.get(rc.err);
                break;
            }
            ...
            case OpCode.exists: {
                lastOp = "EXIS";
                // TODO we need to figure out the security requirement for this!
                ExistsRequest existsRequest = new ExistsRequest();
                ByteBufferInputStream.byteBuffer2Record(request.request,
                        existsRequest);
                String path = existsRequest.getPath();
                if (path.indexOf('\0') != -1) {
                    throw new KeeperException.BadArgumentsException();
                }
                Stat stat = zks.getZKDatabase().statNode(path, existsRequest
                        .getWatch() ? cnxn : null);
                rsp = new ExistsResponse(stat);
                break;
            }
            case OpCode.getData: {
                lastOp = "GETD";
                GetDataRequest getDataRequest = new GetDataRequest();
                ByteBufferInputStream.byteBuffer2Record(request.request,
                        getDataRequest);
                DataNode n = zks.getZKDatabase().getNode(getDataRequest.getPath());
                if (n == null) {
                    throw new KeeperException.NoNodeException();
                }
                PrepRequestProcessor.checkACL(zks, request.cnxn, zks.getZKDatabase().aclForNode(n),
                        ZooDefs.Perms.READ,
                        request.authInfo, getDataRequest.getPath(), null);
                Stat stat = new Stat();
                byte b[] = zks.getZKDatabase().getData(getDataRequest.getPath(), stat,
                        getDataRequest.getWatch() ? cnxn : null);
                rsp = new GetDataResponse(b, stat);
                break;
            }
            ...
            case OpCode.getChildren: {
                lastOp = "GETC";
                GetChildrenRequest getChildrenRequest = new GetChildrenRequest();
                ByteBufferInputStream.byteBuffer2Record(request.request,
                        getChildrenRequest);
                DataNode n = zks.getZKDatabase().getNode(getChildrenRequest.getPath());
                if (n == null) {
                    throw new KeeperException.NoNodeException();
                }
                PrepRequestProcessor.checkACL(zks, request.cnxn, zks.getZKDatabase().aclForNode(n),
                        ZooDefs.Perms.READ,
                        request.authInfo, getChildrenRequest.getPath(), null);
                List<String> children = zks.getZKDatabase().getChildren(
                        getChildrenRequest.getPath(), null, getChildrenRequest
                                .getWatch() ? cnxn : null);
                rsp = new GetChildrenResponse(children);
                break;
            }
            ...
            long lastZxid = zks.getZKDatabase().getDataTreeLastProcessedZxid();
        ReplyHeader hdr =
            new ReplyHeader(request.cxid, lastZxid, err.intValue());

        zks.serverStats().updateLatency(request.createTime);
        cnxn.updateStatsForResponse(request.cxid, lastZxid, lastOp,
                    request.createTime, Time.currentElapsedTime());

        try {
            //发送响应
            cnxn.sendResponse(hdr, rsp, "response");
            if (request.type == OpCode.closeSession) {
                cnxn.sendCloseSession();
            }
        } catch (IOException e) {
            LOG.error("FIXMSG",e);
        }
        ...
}
```
从上面可以看出，FinalRequestProcessor首先由ZooKeeperServer处理CUD相关的请求操作，针对R类的相关操作，直接查询ZooKeeperServer的内存数据库。
再来看ZooKeeperServer的事务性消息的处理

// ZooKeeperServer
```java
 /**
     * entry point for FinalRequestProcessor.java
     * @param request
     * @return
     */
    public ProcessTxnResult processTxn(Request request) {
        return processTxn(request, request.getHdr(), request.getTxn());
    }

    /**
     * 处理数据事务请求
     * @param request
     * @param hdr
     * @param txn
     * @return
     */
    private ProcessTxnResult processTxn(Request request, TxnHeader hdr,
                                        Record txn) {
        ProcessTxnResult rc;
        int opCode = request != null ? request.type : hdr.getType();
        long sessionId = request != null ? request.sessionId : hdr.getClientId();
        if (hdr != null) {
            //委托个数据库，处理事务
            rc = getZKDatabase().processTxn(hdr, txn);
        } else {
            rc = new ProcessTxnResult();
        }
        if (opCode == OpCode.createSession) {
            //创建会话
            if (hdr != null && txn instanceof CreateSessionTxn) {
                CreateSessionTxn cst = (CreateSessionTxn) txn;
                sessionTracker.addGlobalSession(sessionId, cst.getTimeOut());
            } else if (request != null && request.isLocalSession()) {
                request.request.rewind();
                int timeout = request.request.getInt();
                request.request.rewind();
                sessionTracker.addSession(request.sessionId, timeout);
            } else {
                LOG.warn("*****>>>>> Got "
                        + txn.getClass() + " "
                        + txn.toString());
            }
        } else if (opCode == OpCode.closeSession) {
            //关闭会话
            sessionTracker.removeSession(sessionId);
        }
        return rc;
    }
```


//ZKDatabase
```java
 /**
     * the process txn on the data
     * @param hdr the txnheader for the txn
     * @param txn the transaction that needs to be processed
     * @return the result of processing the transaction on this
     * datatree/zkdatabase
     */
    public ProcessTxnResult processTxn(TxnHeader hdr, Record txn) {
        return dataTree.processTxn(hdr, txn);
    }

```

//DataTree
```java
 /**
     * 处理事务
     * @param header
     * @param txn
     * @return
     */
    public ProcessTxnResult processTxn(TxnHeader header, Record txn)
    {
        ProcessTxnResult rc = new ProcessTxnResult();

        try {
            rc.clientId = header.getClientId();
            rc.cxid = header.getCxid();
            rc.zxid = header.getZxid();
            rc.type = header.getType();
            rc.err = 0;
            rc.multiResult = null;
            switch (header.getType()) {
                //创建节点
                case OpCode.create:
                    CreateTxn createTxn = (CreateTxn) txn;
                    rc.path = createTxn.getPath();
                    createNode(
                            createTxn.getPath(),
                            createTxn.getData(),
                            createTxn.getAcl(),
                            createTxn.getEphemeral() ? header.getClientId() : 0,
                            createTxn.getParentCVersion(),
                            header.getZxid(), header.getTime(), null);
                    break;
                case OpCode.create2:
                    CreateTxn create2Txn = (CreateTxn) txn;
                    rc.path = create2Txn.getPath();
                    Stat stat = new Stat();
                    createNode(
                            create2Txn.getPath(),
                            create2Txn.getData(),
                            create2Txn.getAcl(),
                            create2Txn.getEphemeral() ? header.getClientId() : 0,
                            create2Txn.getParentCVersion(),
                            header.getZxid(), header.getTime(), stat);
                    rc.stat = stat;
                    break;
                    //TTL node
                case OpCode.createTTL:
                    CreateTTLTxn createTtlTxn = (CreateTTLTxn) txn;
                    rc.path = createTtlTxn.getPath();
                    stat = new Stat();
                    createNode(
                            createTtlTxn.getPath(),
                            createTtlTxn.getData(),
                            createTtlTxn.getAcl(),
                            EphemeralType.ttlToEphemeralOwner(createTtlTxn.getTtl()),
                            createTtlTxn.getParentCVersion(),
                            header.getZxid(), header.getTime(), stat);
                    rc.stat = stat;
                    break;
                case OpCode.createContainer:
                    CreateContainerTxn createContainerTxn = (CreateContainerTxn) txn;
                    rc.path = createContainerTxn.getPath();
                    stat = new Stat();
                    createNode(
                            createContainerTxn.getPath(),
                            createContainerTxn.getData(),
                            createContainerTxn.getAcl(),
                            EphemeralType.CONTAINER_EPHEMERAL_OWNER,
                            createContainerTxn.getParentCVersion(),
                            header.getZxid(), header.getTime(), stat);
                    rc.stat = stat;
                    break;
                    //delete
                case OpCode.delete:
                case OpCode.deleteContainer:
                    DeleteTxn deleteTxn = (DeleteTxn) txn;
                    rc.path = deleteTxn.getPath();
                    deleteNode(deleteTxn.getPath(), header.getZxid());
                    break;
                    //set data
                    ...
            }
            ...
        }
        ...
        /*
         * Snapshots are taken lazily. It can happen that the child
         * znodes of a parent are created after the parent
         * is serialized. Therefore, while replaying logs during restore, a
         * create might fail because the node was already
         * created.
         *
         * After seeing this failure, we should increment
         * the cversion of the parent znode since the parent was serialized
         * before its children.
         *
         * Note, such failures on DT should be seen only during
         * restore.
         */
        if (header.getType() == OpCode.create &&
                rc.err == Code.NODEEXISTS.intValue()) {
            LOG.debug("Adjusting parent cversion for Txn: " + header.getType() +
                    " path:" + rc.path + " err: " + rc.err);
            int lastSlash = rc.path.lastIndexOf('/');
            String parentName = rc.path.substring(0, lastSlash);
            CreateTxn cTxn = (CreateTxn)txn;
            try {
                setCversionPzxid(parentName, cTxn.getParentCVersion(),
                        header.getZxid());
            } catch (KeeperException.NoNodeException e) {
                LOG.error("Failed to set parent cversion for: " +
                      parentName, e);
                rc.err = e.code().intValue();
            }
        } else if (rc.err != Code.OK.intValue()) {
            LOG.debug("Ignoring processTxn failure hdr: " + header.getType() +
                  " : error: " + rc.err);
        }
        return rc;
```
从上面可以看出，ZooKeeperServer处理CUD操作，委托表给ZKDatabase，ZKDatabase委托给DataTree， DataTree根据CUD相关请求操作，CUD相应路径的
DataNode。针对R类操作，获取dataTree的DataNode的相关信息。


LeaderRequestProcessor处理器主要做的本地会话检查，并更新会话保活信息。
PrepRequestProcessor处理消息，首先添加到内部的提交请求队列，然后启动线程预处理请求。
预处理消息，主要是针对事务性的CUD， 则构建响应的请求，比如CreateRequest，SetDataRequest等。针对非事务性R，则检查会话的有效性。
事务性预处理请求，主要是将请求包装成事件变更记录ZooKeeperServer，并保存到Zookeeper的请求变更记录集outstandingChanges中。
ProposalRequestProcessor处理器，主要处理同步请求消息，针对同步请求，则发送消息到响应的server。
CommitProcessor，主要是过滤出需要提交的请求，比如CRUD等，并交由下一个处理器处理。
ToBeAppliedRequestProcessor处理器，主要是保证提议为最新。
FinalRequestProcessor首先由ZooKeeperServer处理CUD相关的请求操作，针对R类的相关操作，直接查询ZooKeeperServer的内存数据库。
ZooKeeperServer处理CUD操作，委托表给ZKDatabase，ZKDatabase委托给DataTree， DataTree根据CUD相关请求操作，CUD相应路径的
DataNode。针对R类操作，获取dataTree的DataNode的相关信息。


# 总结

观察者、跟随者、和领导者启动server，分别为，LeaderZooKeeperServer,FollowerZooKeeperServer,ObserverZooKeeperServer.
Leader的消息处处理器链为LeaderRequestProcessor->PrepRequestProcessor->ProposalRequestProcessor->CommitProcessor->ToBeAppliedRequestProcessor->FinalRequestProcessor;
Follower的消息处理器链为SendAckRequestProcessor->SyncRequestProcessor->FollowerRequestProcessor->CommitProcessor->FinalRequestProcessor.
Observer的消息处处理器链为SyncRequestProcessor->ObserverRequestProcessor->CommitProcessor->FinalRequestProcessor;





LeaderRequestProcessor处理器主要做的本地会话检查，并更新会话保活信息。
PrepRequestProcessor处理消息，首先添加到内部的提交请求队列，然后启动线程预处理请求。
预处理消息，主要是针对事务性的CUD， 则构建响应的请求，比如CreateRequest，SetDataRequest等。针对非事务性R，则检查会话的有效性。
事务性预处理请求，主要是将请求包装成事件变更记录ZooKeeperServer，并保存到Zookeeper的请求变更记录集outstandingChanges中。
ProposalRequestProcessor处理器，主要处理同步请求消息，针对同步请求，则发送消息到响应的server。
CommitProcessor，主要是过滤出需要提交的请求，比如CRUD等，并交由下一个处理器处理。
ToBeAppliedRequestProcessor处理器，主要是保证提议为最新。
FinalRequestProcessor首先由ZooKeeperServer处理CUD相关的请求操作，针对R类的相关操作，直接查询ZooKeeperServer的内存数据库。
ZooKeeperServer处理CUD操作，委托表给ZKDatabase，ZKDatabase委托给DataTree， DataTree根据CUD相关请求操作，CUD相应路径的
DataNode。针对R类操作，获取dataTree的DataNode的相关信息。

# 附