---
layout: page
title: Zookeeper框架设计及源码解读四（）
subtitle: Zookeeper框架设计及源码解读四（）
date: 2020-12-16 19:41:00
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
    * [消息处理](#消息处理)
         * [OBSERVING观察者](#observing观察者)
        * [FOLLOWING跟随者](#following跟随者)
        * [LEADING领导者](#leading领导者)
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

先来看一下同步请求处理器

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
同步请求处理从请求队列请求,针对刷新队列不为空的情况，如果请求队列为空，则提交请求日志，并刷新到磁盘，否则根据日志计数器和快照计数器计算是否需要拍摄快照。


再来看观察者请求处理器
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

再来看

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

关于FinalRequestProcessor处理器，我们会单独在开一篇再讲。


# 总结
ObserverZooKeeperServer的处理器链为SyncRequestProcessor->ObserverRequestProcessor->CommitProcessor->FinalRequestProcessor;

同步请求处理从请求队列请求,针对刷新队列不为空的情况，如果请求队列为空，则提交请求日志，并刷新到磁盘，否则根据日志计数器和快照计数器计算是否需要拍摄快照。

观察者请求处理器，从请求队列拉取请求，如果请求为同步请求，则添加请求到同步队列, 并转发请求给leader，如果为CRUD相关的操作，直接转发请求给leader。
CommitProcessor，主要是过滤出需要提交的请求，比如CRUD等，并交由下一个处理器处理。


# 附