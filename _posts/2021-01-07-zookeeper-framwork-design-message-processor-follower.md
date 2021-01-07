---
layout: page
title: Zookeeper框架设计及源码解读七（跟随者观察者消息处理器）
subtitle: Zookeeper框架设计及源码解读七（跟随者观察者消息处理器）
date: 2021-01-07 20:33:00
author: valuewithTime
catalog: true
category: Zookeeper
categories:
    - Zookeeper
tags:
    - Zookeeper
---

# 引言
观察者、跟随者、和领导者启动server，分别为，LeaderZooKeeperServer,FollowerZooKeeperServer,ObserverZooKeeperServer. Leader的消息处处理器链为LeaderRequestProcessor->PrepRequestProcessor->ProposalRequestProcessor->CommitProcessor->ToBeAppliedRequestProcessor->FinalRequestProcessor; Follower的消息处理器链为SendAckRequestProcessor->SyncRequestProcessor->FollowerRequestProcessor->CommitProcessor->FinalRequestProcessor. Observer的消息处处理器链为SyncRequestProcessor->ObserverRequestProcessor->CommitProcessor->FinalRequestProcessor;

LeaderRequestProcessor处理器主要做的本地会话检查，并更新会话保活信息。 PrepRequestProcessor处理消息，首先添加到内部的提交请求队列，然后启动线程预处理请求。 预处理消息，主要是针对事务性的CUD， 则构建响应的请求，比如CreateRequest，SetDataRequest等。针对非事务性R，则检查会话的有效性。 事务性预处理请求，主要是将请求包装成事件变更记录ZooKeeperServer，并保存到Zookeeper的请求变更记录集outstandingChanges中。 ProposalRequestProcessor处理器，主要处理同步请求消息，针对同步请求，则发送消息到响应的server。 CommitProcessor，主要是过滤出需要提交的请求，比如CRUD等，并交由下一个处理器处理。 ToBeAppliedRequestProcessor处理器，主要是保证提议为最新。 FinalRequestProcessor首先由ZooKeeperServer处理CUD相关的请求操作，针对R类的相关操作，直接查询ZooKeeperServer的内存数据库。 ZooKeeperServer处理CUD操作，委托表给ZKDatabase，ZKDatabase委托给DataTree， DataTree根据CUD相关请求操作，CUD相应路径的 DataNode。针对R类操作，获取dataTree的DataNode的相关信息。



今天我们来看一下跟随者的消息处理。



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

## 消息处理


### Leader消息处理

[Zookeeper框架设计及源码解读六（Leader消息处理）
](https://donaldhan.github.io/zookeeper/2021/01/05/zookeeper-framwork-design-message-processor-leader.html) 


先来回顾一下各角色的消息处理链


观察者、跟随者、和领导者启动server，分别为，LeaderZooKeeperServer,FollowerZooKeeperServer,ObserverZooKeeperServer.
Leader的消息处处理器链为LeaderRequestProcessor->PrepRequestProcessor->ProposalRequestProcessor->CommitProcessor->ToBeAppliedRequestProcessor->FinalRequestProcessor;
Follower的消息处理器链为SendAckRequestProcessor->SyncRequestProcessor->FollowerRequestProcessor->CommitProcessor->FinalRequestProcessor.
Observer的消息处处理器链为SyncRequestProcessor->ObserverRequestProcessor->CommitProcessor->FinalRequestProcessor;

我们来看Follower消息处理
### Follower消息处理

#### SendAckRequestProcessor
//
```java
public class SendAckRequestProcessor implements RequestProcessor, Flushable {
    private static final Logger LOG = LoggerFactory.getLogger(SendAckRequestProcessor.class);

    Learner learner;

    SendAckRequestProcessor(Learner peer) {
        this.learner = peer;
    }

    public void processRequest(Request si) {
        if(si.type != OpCode.sync){
            QuorumPacket qp = new QuorumPacket(Leader.ACK, si.getHdr().getZxid(), null,
                null);
            try {
                learner.writePacket(qp, false);
            } catch (IOException e) {
                LOG.warn("Closing connection to leader, exception during packet send", e);
                try {
                    if (!learner.sock.isClosed()) {
                        learner.sock.close();
                    }
                } catch (IOException e1) {
                    // Nothing to do, we are shutting things down, so an exception here is irrelevant
                    LOG.debug("Ignoring error closing the connection", e1);
                }
            }
        }
    }
    ...
}
```
SendAckRequestProcessor处理器，针对非同步操作，回复ACK。

再来看同步处理器


#### SyncRequestProcessor

//SyncRequestProcessor
```java
public class SyncRequestProcessor extends ZooKeeperCriticalThread implements
        RequestProcessor {
    private static final Logger LOG = LoggerFactory.getLogger(SyncRequestProcessor.class);
    private final ZooKeeperServer zks;
    /**
     * 请求队列
     */
    private final LinkedBlockingQueue<Request> queuedRequests =
        new LinkedBlockingQueue<Request>();
    /**
     *  后继请求处理器
     */
    private final RequestProcessor nextProcessor;

    private Thread snapInProcess = null;
    volatile private boolean running;

    /**
     * Transactions that have been written and are waiting to be flushed to
     * disk. Basically this is the list of SyncItems whose callbacks will be
     * invoked after flush returns successfully.
     */
    private final LinkedList<Request> toFlush = new LinkedList<Request>();
    private final Random r = new Random(System.nanoTime());
    /**
     * The number of log entries to log before starting a snapshot
     */
    private static int snapCount = ZooKeeperServer.getSnapCount();

    private final Request requestOfDeath = Request.requestOfDeath;

    public SyncRequestProcessor(ZooKeeperServer zks,
            RequestProcessor nextProcessor) {
        super("SyncThread:" + zks.getServerId(), zks
                .getZooKeeperServerListener());
        this.zks = zks;
        this.nextProcessor = nextProcessor;
        running = true;
    }
    ...
    @Override
    public void run() {
        try {
            int logCount = 0;

            // we do this in an attempt to ensure that not all of the servers
            // in the ensemble take a snapshot at the same time
            int randRoll = r.nextInt(snapCount/2);
            while (true) {
                Request si = null;
                if (toFlush.isEmpty()) {
                    si = queuedRequests.take();
                } else {
                    si = queuedRequests.poll();
                    if (si == null) {
                        flush(toFlush);
                        continue;
                    }
                }
                if (si == requestOfDeath) {
                    break;
                }
                if (si != null) {
                    // track the number of records written to the log
                    if (zks.getZKDatabase().append(si)) {
                        logCount++;
                        if (logCount > (snapCount / 2 + randRoll)) {
                            randRoll = r.nextInt(snapCount/2);
                            // roll the log ， 切割日志
                            zks.getZKDatabase().rollLog();
                            // take a snapshot
                            if (snapInProcess != null && snapInProcess.isAlive()) {
                                LOG.warn("Too busy to snap, skipping");
                            } else {
                                snapInProcess = new ZooKeeperThread("Snapshot Thread") {
                                        public void run() {
                                            try {
                                                //拍摄zk数据树快照
                                                zks.takeSnapshot();
                                            } catch(Exception e) {
                                                LOG.warn("Unexpected exception", e);
                                            }
                                        }
                                    };
                                snapInProcess.start();
                            }
                            logCount = 0;
                        }
                    } else if (toFlush.isEmpty()) {
                        // optimization for read heavy workloads
                        // iff this is a read, and there are no pending
                        // flushes (writes), then just pass this to the next
                        // processor
                        if (nextProcessor != null) {
                            nextProcessor.processRequest(si);
                            if (nextProcessor instanceof Flushable) {
                                ((Flushable)nextProcessor).flush();
                            }
                        }
                        continue;
                    }
                    toFlush.add(si);
                    if (toFlush.size() > 1000) {
                        flush(toFlush);
                    }
                }
            }
        } catch (Throwable t) {
            handleException(this.getName(), t);
        } finally{
            running = false;
        }
        LOG.info("SyncRequestProcessor exited!");
    }
private void flush(LinkedList<Request> toFlush)
        throws IOException, RequestProcessorException
    {
        if (toFlush.isEmpty())
            return;

        zks.getZKDatabase().commit();
        while (!toFlush.isEmpty()) {
            Request i = toFlush.remove();
            if (nextProcessor != null) {
                nextProcessor.processRequest(i);
            }
        }
        if (nextProcessor != null && nextProcessor instanceof Flushable) {
            ((Flushable)nextProcessor).flush();
        }
    }
 }
```
SyncRequestProcessor同步请求处理从请求队列拉取请求,针对刷新队列不为空的情况，如果请求队列为空，则提交请求日志，并刷新到磁盘，否则根据日志计数器和快照计数器计算是否需要拍摄快照。



再来看FollowerRequestProcessor处理器
#### FollowerRequestProcessor

//FollowerRequestProcessor
```java
public class FollowerRequestProcessor extends ZooKeeperCriticalThread implements
        RequestProcessor {
    private static final Logger LOG = LoggerFactory.getLogger(FollowerRequestProcessor.class);

    FollowerZooKeeperServer zks;

    RequestProcessor nextProcessor;

    LinkedBlockingQueue<Request> queuedRequests = new LinkedBlockingQueue<Request>();

    boolean finished = false;
    ...
     @Override
    public void run() {
        try {
            while (!finished) {
                Request request = queuedRequests.take();
                if (LOG.isTraceEnabled()) {
                    ZooTrace.logRequest(LOG, ZooTrace.CLIENT_REQUEST_TRACE_MASK,
                            'F', request, "");
                }
                if (request == Request.requestOfDeath) {
                    break;
                }
                // We want to queue the request to be processed before we submit
                // the request to the leader so that we are ready to receive
                // the response
                nextProcessor.processRequest(request);

                // We now ship the request to the leader. As with all
                // other quorum operations, sync also follows this code
                // path, but different from others, we need to keep track
                // of the sync operations this follower has pending, so we
                // add it to pendingSyncs.
                switch (request.type) {
                case OpCode.sync:
                    zks.pendingSyncs.add(request);
                    zks.getFollower().request(request);
                    break;
                case OpCode.create:
                case OpCode.create2:
                case OpCode.createTTL:
                case OpCode.createContainer:
                case OpCode.delete:
                case OpCode.deleteContainer:
                case OpCode.setData:
                case OpCode.reconfig:
                case OpCode.setACL:
                case OpCode.multi:
                case OpCode.check:
                    zks.getFollower().request(request);
                    break;
                case OpCode.createSession:
                case OpCode.closeSession:
                    // Don't forward local sessions to the leader.
                    if (!request.isLocalSession()) {
                        zks.getFollower().request(request);
                    }
                    break;
                }
            }
        } catch (Exception e) {
            handleException(this.getName(), e);
        }
        LOG.info("FollowerRequestProcessor exited loop!");
    }
    ...
}
```
从上面可看出FollowerRequestProcessor处理器，从请求队列拉取请求，如果请求为同步请求，则添加请求到同步队列, 并转发请求给leader，如果为CRUD相关的操作，直接转发请求给leader。



//Learner
```java
/**
     * send a request packet to the leader
     * 发送请求到leader
     * @param request
     *                the request from the client
     * @throws IOException
     */
    void request(Request request) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        DataOutputStream oa = new DataOutputStream(baos);
        oa.writeLong(request.sessionId);
        oa.writeInt(request.cxid);
        oa.writeInt(request.type);
        if (request.request != null) {
            request.request.rewind();
            int len = request.request.remaining();
            byte b[] = new byte[len];
            request.request.get(b);
            request.request.rewind();
            oa.write(b);
        }
        oa.close();
        QuorumPacket qp = new QuorumPacket(Leader.REQUEST, -1, baos
                .toByteArray(), request.authInfo);
        writePacket(qp, true);
    }
    
```
然后发送同步请求到leader。


follower的其他两个处理器我们在上一篇leader消息处理时，已经分析过，这里不再赘述，有兴趣的看前一篇的内容。


再来回顾一下follower的消息处理链

Follower的消息处理器链为SendAckRequestProcessor->SyncRequestProcessor->FollowerRequestProcessor->CommitProcessor->FinalRequestProcessor.

再来看观察者的消息处理链
### Observer消息处理

Observer的消息处处理器链为SyncRequestProcessor->ObserverRequestProcessor->CommitProcessor->FinalRequestProcessor; 其他几个消息处理器，我们已经关注，
我们重点关注一下ObserverRequestProcessor处理器


#### ObserverRequestProcessor
//ObserverRequestProcessor
```java
/**
 * This RequestProcessor forwards any requests that modify the state of the
 * system to the Leader.
 */
public class ObserverRequestProcessor extends ZooKeeperCriticalThread implements
        RequestProcessor {
    private static final Logger LOG = LoggerFactory.getLogger(ObserverRequestProcessor.class);

    ObserverZooKeeperServer zks;

    RequestProcessor nextProcessor;

    // We keep a queue of requests. As requests get submitted they are
    // stored here. The queue is drained in the run() method.
    LinkedBlockingQueue<Request> queuedRequests = new LinkedBlockingQueue<Request>();

    boolean finished = false;

    /**
     * Constructor - takes an ObserverZooKeeperServer to associate with
     * and the next processor to pass requests to after we're finished.
     * @param zks
     * @param nextProcessor
     */
    public ObserverRequestProcessor(ObserverZooKeeperServer zks,
            RequestProcessor nextProcessor) {
        super("ObserverRequestProcessor:" + zks.getServerId(), zks
                .getZooKeeperServerListener());
        this.zks = zks;
        this.nextProcessor = nextProcessor;
    }

    @Override
    public void run() {
        try {
            while (!finished) {
                Request request = queuedRequests.take();
                if (LOG.isTraceEnabled()) {
                    ZooTrace.logRequest(LOG, ZooTrace.CLIENT_REQUEST_TRACE_MASK,
                            'F', request, "");
                }
                if (request == Request.requestOfDeath) {
                    break;
                }
                // We want to queue the request to be processed before we submit
                // the request to the leader so that we are ready to receive
                // the response
                nextProcessor.processRequest(request);

                // We now ship the request to the leader. As with all
                // other quorum operations, sync also follows this code
                // path, but different from others, we need to keep track
                // of the sync operations this Observer has pending, so we
                // add it to pendingSyncs.
                //观察者同步请求给leader
                switch (request.type) {
                case OpCode.sync:
                    //发送同步请求
                    zks.pendingSyncs.add(request);
                    zks.getObserver().request(request);
                    break;
                case OpCode.create:
                case OpCode.create2:
                case OpCode.createTTL:
                case OpCode.createContainer:
                case OpCode.delete:
                case OpCode.deleteContainer:
                case OpCode.setData:
                case OpCode.reconfig:
                case OpCode.setACL:
                case OpCode.multi:
                case OpCode.check:
                    zks.getObserver().request(request);
                    break;
                case OpCode.createSession:
                case OpCode.closeSession:
                    // Don't forward local sessions to the leader.
                    if (!request.isLocalSession()) {
                        zks.getObserver().request(request);
                    }
                    break;
                }
            }
        } catch (Exception e) {
            handleException(this.getName(), e);
        }
        LOG.info("ObserverRequestProcessor exited loop!");
    }
    ...
        }
```
从上可以看出，观察者请求处理器，从请求队列拉取请求，如果请求为同步请求，则添加请求到同步队列, 并转发请求给leader，如果为CRUD相关的操作，直接转发请求给leader。


# 总结

针对跟随者，SendAckRequestProcessor处理器，针对非同步操作，回复ACK。
SyncRequestProcessor处理器从请求队列拉取请求,针对刷新队列不为空的情况，如果请求队列为空，则提交请求日志，并刷新到磁盘，否则根据日志计数器和快照计数器计算是否需要拍摄快照。
FollowerRequestProcessor处理器，从请求队列拉取请求，如果请求为同步请求，则添加请求到同步队列, 并转发请求给leader，如果为CRUD相关的操作，直接转发请求给leader。


针对观察者，观察者请求处理器，从请求队列拉取请求，如果请求为同步请求，则添加请求到同步队列, 并转发请求给leader，如果为CRUD相关的操作，直接转发请求给leader。



# 附