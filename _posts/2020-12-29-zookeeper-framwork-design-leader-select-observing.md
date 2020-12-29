---
layout: page
title: Zookeeper框架设计及源码解读四（观察者观察leader）
subtitle: Zookeeper框架设计及源码解读四（观察者观察leader）
date: 2020-12-29 19:58:00
author: valuewithTime
catalog: true
category: Zookeeper
categories:
    - Zookeeper
tags:
    - Zookeeper
---

# 引言
peer状态有四种LOOKING， OBSERVING，FOLLOWING和LEADING几种状态；LOOKING为初态，Leader还没有选举成功，其他为终态。

当前QuorumPeer处于LOOKING提议投票阶段，启动一个ReadOnlyZooKeeperServer服务，并设置当前peer投票。
ReadOnlyZooKeeperServer内部的处理器链为ReadOnlyRequestProcessor->PrepRequestProcessor->FinalRequestProcessor。
，只读处理器ReadOnlyRequestProcessor，对CRUD相关的操作，进行忽略，只处理check请求，并通过NettyServerCnxn发送ReplyHeader，头部主要的信息为内存数据库的最大事务id。

创建投票，首先更新当前的投票信息，如果peer为参与者，首先投自己一票（当前peer的serverId，最大事务id，以及时间戳）,并发送通知到所有投票peer; 如果peer状态为LOOKING，且选举没有结束，则从接收消息队列拉取通知, 如果通知为空，则发送投票提议通知到所有投票peer, 否则判断下一轮投票视图是否包括当前通知的server和提议leader, 则判断peer的状态(LOOKING,OBSERVING,FOLLOWING,LEADING)。当前peer状态为LOOKING时,，如果通知的时间点，大于当前server时间点，则更新投票提议，并发送通知消息到所有投票peer。如果当前节点的Quorum Peer都进行投票回复，然后从接收队列中拉取通知投票消息，如果为空，则投票结束，更新当前投票状态为LEADING。当peer为OBSERVING,FOLLOWING状态，什么都不做;当peer状态为leading，则如果投票的时间戳和当前节点的投票时间戳一致，并且所有peer都回复，则结束投票。


一句总结，就是peer首先推举自己为leader，直到如果所有的投票成员全部赞同，则选举结束；选举的过程中以最新投票为准。


今天我们来看一下，选举过程中观察者的处理策略



# 目录
* [概要框架设计](#概要框架设计)
* [源码分析](#源码分析)
    * [启动Zookeeper](#启动zookeeper)
    * [Leader选举](#leader选举)
        * [QuorumPeer选举机制及处理策略](#quorumpeer选举机制及处理策略)
        * [LOOKING提议投票阶段](#looking提议投票阶段)
        * [OBSERVING观察者状态](#observing观察者状态)
        * [FOLLOWING跟随者状态](#following跟随者状态)
        * [LEADING领导者状态](#leading领导者状态)
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


## Leader选举

[Zookeeper框架设计及源码解读二（快速选举策略及选举消息的发送与接收）](https://donaldhan.github.io/zookeeper/2020/12/22/zookeeper-framwork-design-leader-select.html) 

### LOOKING提议投票阶段
[Zookeeper框架设计及源码解读三（leader选举LOOKING阶段）](https://donaldhan.github.io/zookeeper/2020/12/23/zookeeper-framwork-design-leader-select-quorum-peer-looking.html)  

### OBSERVING观察者状态
//QuorumPeer
```java
case OBSERVING:
                    //观察节点
                    try {
                        LOG.info("OBSERVING");
                        //org.apache.zookeeper.server.quorum.Observer
                        //org.apache.zookeeper.server.quorum.ObserverZooKeeperServer
                        setObserver(makeObserver(logFactory));
                        observer.observeLeader();
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e );
                    } finally {
                        observer.shutdown();
                        setObserver(null);  
                       updateServerState();
                    }
                    break;
```

//QuorumPeer
```java
 /**
     * 观察者
     * @param logFactory
     * @return
     * @throws IOException
     */
    protected Observer makeObserver(FileTxnSnapLog logFactory) throws IOException {
        return new Observer(this, new ObserverZooKeeperServer(logFactory, this, this.zkDb));
    }
```
//
```java
public class Observer extends Learner{

    Observer(QuorumPeer self,ObserverZooKeeperServer observerZooKeeperServer) {
        this.self = self;
        this.zk=observerZooKeeperServer;
    }
    ...
    /**
     * the main method called by the observer to observe the leader
     * 观察leader
     * @throws Exception 
     */
    void observeLeader() throws Exception {
        zk.registerJMX(new ObserverBean(this, zk), self.jmxLocalPeerBean);

        try {
            //获取leader server
            QuorumServer leaderServer = findLeader();
            LOG.info("Observing " + leaderServer.addr);
            try {
                //与leader建立连接，尝试5次
                connectToLeader(leaderServer.addr, leaderServer.hostname);
                //一旦与leader建立连接，则执行握手协议，建立following / observing连接
                long newLeaderZxid = registerWithLeader(Leader.OBSERVERINFO);
                if (self.isReconfigStateChange())
                   throw new Exception("learned about role change");
                //同步Leader历史日志，保持一致
                syncWithLeader(newLeaderZxid);
                QuorumPacket qp = new QuorumPacket();
                while (this.isRunning()) {
                    readPacket(qp);
                    //处理器请求包
                    processPacket(qp);
                }
            } catch (Exception e) {
                LOG.warn("Exception when observing the leader", e);
                try {
                    sock.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }

                // clear pending revalidations
                pendingRevalidations.clear();
            }
        } finally {
            zk.unregisterJMX(this);
        }
    }
}
```
关键有两步，1：同步leader历史日志保持一致；2：处理请求；

分别来看这两步

#### 同步leader历史日志
//Learner
```java
public class Learner {
    **
     * Finally, synchronize our history with the Leader. 
     * @param newLeaderZxid
     * @throws IOException
     * @throws InterruptedException
     */
    protected void syncWithLeader(long newLeaderZxid) throws Exception{
        QuorumPacket ack = new QuorumPacket(Leader.ACK, 0, null, null);
        QuorumPacket qp = new QuorumPacket();
        long newEpoch = ZxidUtils.getEpochFromZxid(newLeaderZxid);
        
        QuorumVerifier newLeaderQV = null;
        
        // In the DIFF case we don't need to do a snapshot because the transactions will sync on top of any existing snapshot
        // For SNAP and TRUNC the snapshot is needed to save that history
        boolean snapshotNeeded = true;
        boolean syncSnapshot = false;
        readPacket(qp);
        LinkedList<Long> packetsCommitted = new LinkedList<Long>();
        //未提交的数据包
        LinkedList<PacketInFlight> packetsNotCommitted = new LinkedList<PacketInFlight>();
        synchronized (zk) {
            if (qp.getType() == Leader.DIFF) {
                LOG.info("Getting a diff from the leader 0x{}", Long.toHexString(qp.getZxid()));
                snapshotNeeded = false;
            }
            else if (qp.getType() == Leader.SNAP) {
                LOG.info("Getting a snapshot from leader 0x" + Long.toHexString(qp.getZxid()));
                // The leader is going to dump the database
                // db is clear as part of deserializeSnapshot()
                zk.getZKDatabase().deserializeSnapshot(leaderIs);
                // ZOOKEEPER-2819: overwrite config node content extracted
                // from leader snapshot with local config, to avoid potential
                // inconsistency of config node content during rolling restart.
                if (!QuorumPeerConfig.isReconfigEnabled()) {
                    LOG.debug("Reset config node content from local config after deserialization of snapshot.");
                    //没有开启配置，则初始化配置
                    zk.getZKDatabase().initConfigInZKDatabase(self.getQuorumVerifier());
                }
                String signature = leaderIs.readString("signature");
                if (!signature.equals("BenWasHere")) {
                    LOG.error("Missing signature. Got " + signature);
                    throw new IOException("Missing signature");                   
                }
                zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());

                // immediately persist the latest snapshot when there is txn log gap
                syncSnapshot = true;
            } else if (qp.getType() == Leader.TRUNC) {
                //we need to truncate the log to the lastzxid of the leader
                LOG.warn("Truncating log to get in sync with the leader 0x"
                        + Long.toHexString(qp.getZxid()));
                //节点事务日志
                boolean truncated=zk.getZKDatabase().truncateLog(qp.getZxid());
                if (!truncated) {
                    // not able to truncate the log
                    LOG.error("Not able to truncate the log "
                            + Long.toHexString(qp.getZxid()));
                    System.exit(13);
                }
                //更新last日志物质
                zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());

            }
            else {
                LOG.error("Got unexpected packet from leader: {}, exiting ... ",
                          LearnerHandler.packetToString(qp));
                System.exit(13);

            }
            zk.getZKDatabase().initConfigInZKDatabase(self.getQuorumVerifier());
            zk.createSessionTracker();            
            
            long lastQueued = 0;

            // in Zab V1.0 (ZK 3.4+) we might take a snapshot when we get the NEWLEADER message, but in pre V1.0
            // we take the snapshot on the UPDATE message, since Zab V1.0 also gets the UPDATE (after the NEWLEADER)
            // we need to make sure that we don't take the snapshot twice.
            boolean isPreZAB1_0 = true;
            //If we are not going to take the snapshot be sure the transactions are not applied in memory
            // but written out to the transaction log
            boolean writeToTxnLog = !snapshotNeeded;
            // we are now going to start getting transactions to apply followed by an UPTODATE
            outerLoop:
            while (self.isRunning()) {
                readPacket(qp);
                switch(qp.getType()) {
                case Leader.PROPOSAL:
                    //提议
                    PacketInFlight pif = new PacketInFlight();
                    pif.hdr = new TxnHeader();
                    pif.rec = SerializeUtils.deserializeTxn(qp.getData(), pif.hdr);
                    if (pif.hdr.getZxid() != lastQueued + 1) {
                    LOG.warn("Got zxid 0x"
                            + Long.toHexString(pif.hdr.getZxid())
                            + " expected 0x"
                            + Long.toHexString(lastQueued + 1));
                    }
                    lastQueued = pif.hdr.getZxid();
                    
                    if (pif.hdr.getType() == OpCode.reconfig){                
                        SetDataTxn setDataTxn = (SetDataTxn) pif.rec;       
                       QuorumVerifier qv = self.configFromString(new String(setDataTxn.getData()));
                       self.setLastSeenQuorumVerifier(qv, true);                               
                    }
                    
                    packetsNotCommitted.add(pif);
                    break;
                case Leader.COMMIT:
                case Leader.COMMITANDACTIVATE:
                    //提议提交
                    pif = packetsNotCommitted.peekFirst();
                    if (pif.hdr.getZxid() == qp.getZxid() && qp.getType() == Leader.COMMITANDACTIVATE) {
                        QuorumVerifier qv = self.configFromString(new String(((SetDataTxn) pif.rec).getData()));
                        boolean majorChange = self.processReconfig(qv, ByteBuffer.wrap(qp.getData()).getLong(),
                                qp.getZxid(), true);
                        if (majorChange) {
                            throw new Exception("changes proposed in reconfig");
                        }
                    }
                    if (!writeToTxnLog) {
                        if (pif.hdr.getZxid() != qp.getZxid()) {
                            LOG.warn("Committing " + qp.getZxid() + ", but next proposal is " + pif.hdr.getZxid());
                        } else {
                            //没有写到事务日志则，处理消息包
                            zk.processTxn(pif.hdr, pif.rec);
                            packetsNotCommitted.remove();
                        }
                    } else {
                        packetsCommitted.add(qp.getZxid());
                    }
                    break;
                case Leader.INFORM:
                case Leader.INFORMANDACTIVATE:
                    //观察者通知提议
                    PacketInFlight packet = new PacketInFlight();
                    packet.hdr = new TxnHeader();

                    if (qp.getType() == Leader.INFORMANDACTIVATE) {
                        ByteBuffer buffer = ByteBuffer.wrap(qp.getData());
                        long suggestedLeaderId = buffer.getLong();
                        byte[] remainingdata = new byte[buffer.remaining()];
                        buffer.get(remainingdata);
                        packet.rec = SerializeUtils.deserializeTxn(remainingdata, packet.hdr);
                        QuorumVerifier qv = self.configFromString(new String(((SetDataTxn)packet.rec).getData()));
                        boolean majorChange =
                                self.processReconfig(qv, suggestedLeaderId, qp.getZxid(), true);
                        if (majorChange) {
                            throw new Exception("changes proposed in reconfig");
                        }
                    } else {
                        packet.rec = SerializeUtils.deserializeTxn(qp.getData(), packet.hdr);
                        // Log warning message if txn comes out-of-order
                        if (packet.hdr.getZxid() != lastQueued + 1) {
                            LOG.warn("Got zxid 0x"
                                    + Long.toHexString(packet.hdr.getZxid())
                                    + " expected 0x"
                                    + Long.toHexString(lastQueued + 1));
                        }
                        lastQueued = packet.hdr.getZxid();
                    }
                    if (!writeToTxnLog) {
                        // Apply to db directly if we haven't taken the snapshot
                        zk.processTxn(packet.hdr, packet.rec);
                    } else {
                        packetsNotCommitted.add(packet);
                        packetsCommitted.add(qp.getZxid());
                    }

                    break;                
                case Leader.UPTODATE:
                    //leader 通知follower ，可以响应客户端的请求
                    LOG.info("Learner received UPTODATE message");                                      
                    if (newLeaderQV!=null) {
                       boolean majorChange =
                           self.processReconfig(newLeaderQV, null, null, true);
                       if (majorChange) {
                           throw new Exception("changes proposed in reconfig");
                       }
                    }
                    if (isPreZAB1_0) {
                        //如果需要，拍摄快照
                        zk.takeSnapshot(syncSnapshot);
                        self.setCurrentEpoch(newEpoch);
                    }
                    self.setZooKeeperServer(zk);
                    self.adminServer.setZooKeeperServer(zk);
                    break outerLoop;
                case Leader.NEWLEADER: // Getting NEWLEADER here instead of in discovery 
                    // means this is Zab 1.0
                   LOG.info("Learner received NEWLEADER message");
                   if (qp.getData()!=null && qp.getData().length > 1) {
                       try {                       
                           QuorumVerifier qv = self.configFromString(new String(qp.getData()));
                           self.setLastSeenQuorumVerifier(qv, true);
                           newLeaderQV = qv;
                       } catch (Exception e) {
                           e.printStackTrace();
                       }
                   }

                   if (snapshotNeeded) {
                       zk.takeSnapshot(syncSnapshot);
                   }
                   
                    self.setCurrentEpoch(newEpoch);
                    writeToTxnLog = true; //Anything after this needs to go to the transaction log, not applied directly in memory
                    isPreZAB1_0 = false;
                    writePacket(new QuorumPacket(Leader.ACK, newLeaderZxid, null, null), true);
                    break;
                }
            }
        }
        ack.setZxid(ZxidUtils.makeZxid(newEpoch, 0));
        writePacket(ack, true);
        sock.setSoTimeout(self.tickTime * self.syncLimit);
        zk.startup();
        /*
         * Update the election vote here to ensure that all members of the
         * ensemble report the same vote to new servers that start up and
         * send leader election notifications to the ensemble.
         * 
         * @see https://issues.apache.org/jira/browse/ZOOKEEPER-1732
         */
        self.updateElectionVote(newEpoch);

        // We need to log the stuff that came in between the snapshot and the uptodate
        if (zk instanceof FollowerZooKeeperServer) {
            FollowerZooKeeperServer fzk = (FollowerZooKeeperServer)zk;
            for(PacketInFlight p: packetsNotCommitted) {
                //日志请求
                fzk.logRequest(p.hdr, p.rec);
            }
            for(Long zxid: packetsCommitted) {
                //提交事务
                fzk.commit(zxid);
            }
        } else if (zk instanceof ObserverZooKeeperServer) {
            // Similar to follower, we need to log requests between the snapshot
            // and UPTODATE
            ObserverZooKeeperServer ozk = (ObserverZooKeeperServer) zk;
            for (PacketInFlight p : packetsNotCommitted) {
                Long zxid = packetsCommitted.peekFirst();
                if (p.hdr.getZxid() != zxid) {
                    // log warning message if there is no matching commit
                    // old leader send outstanding proposal to observer
                    LOG.warn("Committing " + Long.toHexString(zxid)
                            + ", but next proposal is "
                            + Long.toHexString(p.hdr.getZxid()));
                    continue;
                }
                packetsCommitted.remove();
                Request request = new Request(null, p.hdr.getClientId(),
                        p.hdr.getCxid(), p.hdr.getType(), null, null);
                request.setTxn(p.rec);
                request.setHdr(p.hdr);
                ozk.commitRequest(request);
            }
        } else {
            // New server type need to handle in-flight packets
            throw new UnsupportedOperationException("Unknown server type");
        }
    }
    ...
}
```
观察者同步leader，首先从输入流中读取数据包，如果是快照同步，则从leader同步快照信息，并添加DataTree；如果是TUNC命令，则截取日志（观察者日志快于Leader），然后从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中。如果观察者节点没有启动，则启动ZookeeperServer（ObserverZooKeeperServer，FollowerZooKeeperServer，LeaderZooKeeperServer），并设置消息处理器链，更新启动状态，然后更新选举时间戳；如果server为FollowerZooKeeperServer，则log请求未提交的请求，并添加到请求队列，同时唤起syncProcessor处理请求，针对已提交的请求，交由commitProcessor处理。如果是ObserverZooKeeperServer，则只处理为提交的请求，交由commitProcessor处理。如果Server已经启动，针对重新选举这种情形，从输入流中读取消息，如果是提议消息，则添加到未提交消息队列；如果是为提交消息，则从待提交队列，拉取消息，并添加到提交消息队列。如果为通知消息，针对没有写到日志的情形，则委托server处理请求，否则将消息添加待提交和提交消息队列。如果为UPTODATE消息，则leader通知follower ，可以响应客户端的请求，如果需要则拍摄快照，并跟新选举时间戳。如果为NEWLEADER请求，则更新选举时间戳，发送回复消息，如果需要拍摄快照，则takeSnapshot。


回到观察者处理请求
#### 处理请求
//Observer
```java
 /**
     * Controls the response of an observer to the receipt of a quorumpacket
     * @param qp
     * @throws Exception 
     */
    protected void processPacket(QuorumPacket qp) throws Exception{
        switch (qp.getType()) {
        case Leader.PING:
            ping(qp);
            break;
        case Leader.PROPOSAL:
            LOG.warn("Ignoring proposal");
            break;
        case Leader.COMMIT:
            LOG.warn("Ignoring commit");
            break;
        case Leader.UPTODATE:
            LOG.error("Received an UPTODATE message after Observer started");
            break;
        case Leader.REVALIDATE:
            //校验会话
            revalidate(qp);
            break;
        case Leader.SYNC:
            //处理同步请求
            ((ObserverZooKeeperServer)zk).sync();
            break;
        case Leader.INFORM:
            //提交提议通知
            TxnHeader hdr = new TxnHeader();
            Record txn = SerializeUtils.deserializeTxn(qp.getData(), hdr);
            Request request = new Request (hdr.getClientId(),  hdr.getCxid(), hdr.getType(), hdr, txn, 0);
            ObserverZooKeeperServer obs = (ObserverZooKeeperServer)zk;
            obs.commitRequest(request);
            break;
        case Leader.INFORMANDACTIVATE:
            //重新配置通知
            hdr = new TxnHeader();
            
           // get new designated leader from (current) leader's message
            ByteBuffer buffer = ByteBuffer.wrap(qp.getData());    
           long suggestedLeaderId = buffer.getLong();
           
            byte[] remainingdata = new byte[buffer.remaining()];
            buffer.get(remainingdata);
            txn = SerializeUtils.deserializeTxn(remainingdata, hdr);
            QuorumVerifier qv = self.configFromString(new String(((SetDataTxn)txn).getData()));
            
            request = new Request (hdr.getClientId(),  hdr.getCxid(), hdr.getType(), hdr, txn, 0);
            obs = (ObserverZooKeeperServer)zk;
                        
            boolean majorChange = 
                self.processReconfig(qv, suggestedLeaderId, qp.getZxid(), true);
           
            obs.commitRequest(request);                                 

            if (majorChange) {
               throw new Exception("changes proposed in reconfig");
           }            
            break;
        default:
            LOG.warn("Unknown packet type: {}", LearnerHandler.packetToString(qp));
            break;
        }
    }

```
从上面可以看出，观察者，只处理ping，同步，及通知请求。

# 总结
观察者同步leader，首先从输入流中读取数据包，如果是快照同步，则从leader同步快照信息，并添加DataTree；如果是TUNC命令，则截取日志（观察者日志快于Leader），然后从磁盘中加载数据到内存数据树DataTree， 并添加committed交易日志到DataTree中。如果观察者节点没有启动，则启动ZookeeperServer（ObserverZooKeeperServer，FollowerZooKeeperServer，LeaderZooKeeperServer），并设置消息处理器链，更新启动状态，然后更新选举时间戳；如果server为FollowerZooKeeperServer，则log请求未提交的请求，并添加到请求队列，同时唤起syncProcessor处理请求，针对已提交的请求，交由commitProcessor处理。如果是ObserverZooKeeperServer，则只处理为提交的请求，交由commitProcessor处理。如果Server已经启动，针对重新选举这种情形，从输入流中读取消息，如果是提议消息，则添加到未提交消息队列；如果是为提交消息，则从待提交队列，拉取消息，并添加到提交消息队列。如果为通知消息，针对没有写到日志的情形，则委托server处理请求，否则将消息添加待提交和提交消息队列。如果为UPTODATE消息，则leader通知follower ，可以响应客户端的请求，如果需要则拍摄快照，并跟新选举时间戳。如果为NEWLEADER请求，则更新选举时间戳，发送回复消息，如果需要拍摄快照，则takeSnapshot。观察者，只处理ping，同步，及通知请求。



# 附