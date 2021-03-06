---
layout: page
title: incubator-gossip框架设计
subtitle: incubator-gossip框架设计
date: 2020-10-20 14:30:00
author: valuewithtime
catalog: true
category: BlockChain
categories:
    - BlockChain
tags:
    - p2p
---

# 引言

Gossip protocol 也叫 Epidemic Protocol （流行病协议），实际上它还有很多别名，比如：“流言算法”、“疫情传播算法”等。很多知名的 P2P 网络或区块链项目，比如 IPFS，Ethereum 等，都使用了 Kadmelia 算法，而大名鼎鼎的 Bitcoin 则是使用了 Gossip 协议来传播交易和区块信息。实际上，只要仔细分析一下场景就知道，Ethereum 使用 DHT 算法并不是很合理，因为它使用节点保存整个链数据，不像 IPFS 那样分片保存数据，因此 Ethereum 真正适合的协议应该像 Bitcoin 那样，是 Gossip 协议。


# 目录
* [gossip初探](#gossip初探)
* [框架设计](#框架设计)
* [源码分析](#源码分析)
    * [UDP管理器](#udp管理器)
    * [消息编解码](#消息编解码)
    * [消息处理器](#消息处理器)
    * [节点状态刷新器](#节点状态刷新器)
    * [数据同步器](#数据同步器)
    * [core组件](#core组件)
    * [gossip管理器](#gossip管理)
* [总结](#总结)

# gossip初探
关于Gossip 协议介绍，我们不做太多的讲解，因为网上已经有很多的文章，可以参考附录文献。在了解gossip一些的基础之上我们来探索gossip相关的协议实现框架，本文研究的框架为
[incubator-retired-gossip][]。

[incubator-retired-gossip]:https://github.com/Donaldhan/incubator-retired-gossip "incubator-retired-gossip"

incubator-retired-gossip框架提供了gossip协议的单机版示例，包括族的管理，基于CRDT的共享数据，PN计数器，数据中心，分布式锁的相关demo；

具体参见使用说明
[gossip-examples](https://github.com/Donaldhan/incubator-retired-gossip/tree/master/gossip-examples)

把demo跑一篇，我们再来看gossip的框架设计。


# 框架设计
![gossip-framework](/image/gossip/gossip-framework.png)

gossip是基于UDP协议进行通信的，底层是通过UDP管理器进行消息的收发；消息编解码器使用的是JSON序列化方式，节点接收消息后，有解码器进行解码，委托消息处理器处理，消息处理器有节点数据、共享数据、及宕机消息等。集群中的节点的状态同步通过节点状态刷新器进行同步。针对族活跃节点、宕机节点、节点数据，及共享数据的同步通过数据同步器完成。core组件对节点、共享数据进行管理，及消息的收发。gossip管理器持久节点状态、数据及共享数据。

# 源码分析

## UDP管理器
```java
/**
 * This class is constructed by reflection in GossipManager.
 * It manages transport (byte read/write) operations over UDP.
 * 管理基于UDP的数据传输操作
 */
public class UdpTransportManager extends AbstractTransportManager implements Runnable {
  public static final Logger LOGGER = Logger.getLogger(UdpTransportManager.class);
  /**
   *
   * The socket used for the passive thread of the gossip service.
   * gossip socket server
   *  */
  private final DatagramSocket server;
  /**
   * 获取数据超时时间
   */
  private final int soTimeout;
  
  private final Thread me;
  /**
   * 运行状态
   */
  private final AtomicBoolean keepRunning = new AtomicBoolean(true);
  
  /**
   * required for reflection to work!
   * @param gossipManager
   * @param gossipCore*/
  public UdpTransportManager(GossipManager gossipManager, GossipCore gossipCore) {
    super(gossipManager, gossipCore);
    soTimeout = gossipManager.getSettings().getGossipInterval() * 2;
    try {
      SocketAddress socketAddress = new InetSocketAddress(gossipManager.getMyself().getUri().getHost(),
              gossipManager.getMyself().getUri().getPort());
      server = new DatagramSocket(socketAddress);
    } catch (SocketException ex) {
      LOGGER.warn(ex);
      throw new RuntimeException(ex);
    }
    me = new Thread(this);
  }

  @Override
  public void run() {
    while (keepRunning.get()) {
      try {
        byte[] buf = read();
        try {
          //读取数据
          Base message = gossipManager.getProtocolManager().read(buf);
          //处理消息
          gossipCore.receive(message);
          //TODO this is suspect  GossipMemberStateRefresher
          gossipManager.getMemberStateRefresher().run();
        } catch (RuntimeException ex) {//TODO trap json exception
          LOGGER.error("Unable to process message", ex);
        }
      } catch (IOException e) {
        // InterruptedException are completely normal here because of the blocking lifecycle.
        if (!(e.getCause() instanceof InterruptedException)) {
          LOGGER.error(e);
        }
        keepRunning.set(false);
      }
    }
  }
  
  @Override
  public void shutdown() {
    keepRunning.set(false);
    server.close();
    super.shutdown();
    me.interrupt();
  }

  /**
   * blocking read a message.
   * 从缓存区读取消息数据
   * @return buffer of message contents.
   * @throws IOException
   */
  public byte[] read() throws IOException {
    byte[] buf = new byte[server.getReceiveBufferSize()];
    DatagramPacket p = new DatagramPacket(buf, buf.length);
    server.receive(p);
    debug(p.getData());
    return p.getData();
  }

  /**
   * 发送字节数据
   * @param endpoint
   * @param buf
   * @throws IOException
   */
  @Override
  public void send(URI endpoint, byte[] buf) throws IOException {
    // todo: investigate UDP socket reuse. It would save a little setup/teardown time wrt to the local socket.
    try (DatagramSocket socket = new DatagramSocket()){
      socket.setSoTimeout(soTimeout);
      InetAddress dest = InetAddress.getByName(endpoint.getHost());
      DatagramPacket payload = new DatagramPacket(buf, buf.length, dest, endpoint.getPort());
      socket.send(payload);
    }
  }
  

  /**
   * 启动endpoint
   */
  @Override
  public void startEndpoint() {
    me.start();
  }
  
}
```
从上面可以看出，UDP传输管理UdpTransportManager，主要发送字节数据，接收字节数据，并委托给core组件。在UDP传输管理器启动的过程中，同时启动节点状态刷新器。

我们再来看一下消息编解码器

## 消息编解码
```java
// this class is constructed by reflection in GossipManager.
public class JacksonProtocolManager implements ProtocolManager {
  
  private final ObjectMapper objectMapper;
  private final PrivateKey privKey;
  private final Meter signed;
  private final Meter unsigned;
  
  /** required for reflection to work!
   * @param settings
   * @param id
   * @param registry*/
  public JacksonProtocolManager(GossipSettings settings, String id, MetricRegistry registry) {
    // set up object mapper.
    objectMapper = buildObjectMapper(settings);
    
    // set up message signing. 如果需要加签消息，则加载节点公私钥
    if (settings.isSignMessages()){
      File privateKey = new File(settings.getPathToKeyStore(), id);
      File publicKey = new File(settings.getPathToKeyStore(), id + ".pub");
      if (!privateKey.exists()){
        throw new IllegalArgumentException("private key not found " + privateKey);
      }
      if (!publicKey.exists()){
        throw new IllegalArgumentException("public key not found " + publicKey);
      }
      try (FileInputStream keyfis = new FileInputStream(privateKey)) {
        byte[] encKey = new byte[keyfis.available()];
        keyfis.read(encKey);
        keyfis.close();
        PKCS8EncodedKeySpec privKeySpec = new PKCS8EncodedKeySpec(encKey);
        KeyFactory keyFactory = KeyFactory.getInstance("DSA");
        privKey = keyFactory.generatePrivate(privKeySpec);
      } catch (NoSuchAlgorithmException | InvalidKeySpecException | IOException e) {
        throw new RuntimeException("failed hard", e);
      }
    } else {
      privKey = null;
    }
    
    signed = registry.meter(PassiveGossipConstants.SIGNED_MESSAGE);
    unsigned = registry.meter(PassiveGossipConstants.UNSIGNED_MESSAGE);
  }

  /**
   * 转换消息为字节数组
   * @param message
   * @return
   * @throws IOException
   */
  @Override
  public byte[] write(Base message) throws IOException {
    byte[] json_bytes;
    if (privKey == null){
      json_bytes = objectMapper.writeValueAsBytes(message);
    } else {
      SignedPayload p = new SignedPayload();
      p.setData(objectMapper.writeValueAsString(message).getBytes());
      p.setSignature(sign(p.getData(), privKey));
      json_bytes = objectMapper.writeValueAsBytes(p);
    }
    return json_bytes;
  }

  /**
   * 转换字节数组为消息对象
   * @param buf
   * @return
   * @throws IOException
   */
  @Override
  public Base read(byte[] buf) throws IOException {
    Base activeGossipMessage = objectMapper.readValue(buf, Base.class);
    if (activeGossipMessage instanceof SignedPayload){
      SignedPayload s = (SignedPayload) activeGossipMessage;
      signed.mark();
      return objectMapper.readValue(s.getData(), Base.class);
    } else {
      unsigned.mark();
      return activeGossipMessage;
    }
  }

  public static ObjectMapper buildObjectMapper(GossipSettings settings) {
    ObjectMapper om = new ObjectMapper();
    om.enableDefaultTyping();
    // todo: should be specified in the configuration.
    om.registerModule(new CrdtModule());
    om.configure(JsonGenerator.Feature.WRITE_NUMBERS_AS_STRINGS, false);
    return om;
  }
  
  private static byte[] sign(byte [] bytes, PrivateKey pk){
    Signature dsa;
    try {
      dsa = Signature.getInstance("SHA1withDSA", "SUN");
      dsa.initSign(pk);
      dsa.update(bytes);
      return dsa.sign();
    } catch (NoSuchAlgorithmException | NoSuchProviderException | InvalidKeyException | SignatureException e) {
      throw new RuntimeException(e);
    } 
  }
}

```
从上面可以看出，消息编解码器JacksonProtocolManager是基于JSON的消息编解码，如果是RSA加签方式，则在收发消息，做响应的加解密。

## 消息处理器
消息处理这块，我们来看一下节点数据、共享数据、节点宕机消息的处理。

* 共享数据消息处理器

```java
public class SharedDataMessageHandler implements MessageHandler{
  
  /**
   * @param gossipCore context.
   * @param gossipManager context.
   * @param base message reference.
   * @return boolean indicating success.
   */
  @Override
  public boolean invoke(GossipCore gossipCore, GossipManager gossipManager, Base base) {
    UdpSharedDataMessage message = (UdpSharedDataMessage) base;
    //添加共享数据
    gossipCore.addSharedData(message);
    return true;
  }
}

```

* 节点数据消息处理器

```java
public class PerNodeDataMessageHandler implements MessageHandler {

  /**
   * @param gossipCore context.
   * @param gossipManager context.
   * @param base message reference.
   * @return boolean indicating success.
   */
  @Override
  public boolean invoke(GossipCore gossipCore, GossipManager gossipManager, Base base) {
    UdpPerNodeDataMessage message = (UdpPerNodeDataMessage) base;
    //添加节点数据消息
    gossipCore.addPerNodeData(message);
    return true;
  }
}
```

* 节点宕机消息处理器

```java
public class ShutdownMessageHandler implements MessageHandler {
  
  /**
   * @param gossipCore context.
   * @param gossipManager context.
   * @param base message reference.
   * @return boolean indicating success.
   */
  @Override
  public boolean invoke(GossipCore gossipCore, GossipManager gossipManager, Base base) {
    ShutdownMessage s = (ShutdownMessage) base;
    //转换宕机下消息，为节点数据消息
    PerNodeDataMessage m = new PerNodeDataMessage();
    m.setKey(ShutdownMessage.PER_NODE_KEY);
    m.setNodeId(s.getNodeId());
    m.setPayload(base);
    m.setTimestamp(System.currentTimeMillis());
    m.setExpireAt(System.currentTimeMillis() + 30L * 1000L);
    //添加节点数据消息
    gossipCore.addPerNodeData(m);
    return true;
  }
}
```
从上面可以共享数据和节点数据处理实际是委托给core组件，这个我们后面再来看，宕机消息转换为节点消息进行处理。


## 节点状态刷新器


```java
/**
 * 成员状态刷新线程
 */
public class GossipMemberStateRefresher {
  public static final Logger LOGGER = Logger.getLogger(GossipMemberStateRefresher.class);

  /**
   * gossip成员
   */
  private final Map<LocalMember, GossipState> members;
  /**
   * gossip配置
   */
  private final GossipSettings settings;
  /**
   * gossip监听器
   */
  private final List<GossipListener> listeners = new CopyOnWriteArrayList<>();
  /**
   * 系统时钟
   */
  private final Clock clock;
  private final BiFunction<String, String, PerNodeDataMessage> findPerNodeGossipData;
  /**
   * 监听器执行线程
   */
  private final ExecutorService listenerExecutor;
  /**
   * 条赌气
   */
  private final ScheduledExecutorService scheduledExecutor;
  /**
   * 任务队列
   */
  private final BlockingQueue<Runnable> workQueue;

  public GossipMemberStateRefresher(Map<LocalMember, GossipState> members, GossipSettings settings,
                                    GossipListener listener,
                                    BiFunction<String, String, PerNodeDataMessage> findPerNodeGossipData) {
    this.members = members;
    this.settings = settings;
    listeners.add(listener);
    this.findPerNodeGossipData = findPerNodeGossipData;
    clock = new SystemClock();
    workQueue = new ArrayBlockingQueue<>(1024);
    listenerExecutor = new ThreadPoolExecutor(1, 20, 1, TimeUnit.SECONDS, workQueue,
            new ThreadPoolExecutor.DiscardOldestPolicy());
    scheduledExecutor = Executors.newScheduledThreadPool(1);
  }

  /**
   * 调度成员状态刷新器
   */
  public void init() {
    scheduledExecutor.scheduleAtFixedRate(() -> run(), 0, 100, TimeUnit.MILLISECONDS);
  }

  /**
   *
   */
  public void run() {
    try {
      runOnce();
    } catch (RuntimeException ex) {
      LOGGER.warn("scheduled state had exception", ex);
    }
  }

  /**
   * gossip成员状态探测
   */
  public void runOnce() {
    for (Entry<LocalMember, GossipState> entry : members.entrySet()) {
      boolean userDown = processOptimisticShutdown(entry);
      if (userDown)
        continue;

      Double phiMeasure = entry.getKey().detect(clock.nanoTime());
      GossipState requiredState;
      //根据探测结果，判断节点状态
      if (phiMeasure != null) {
        requiredState = calcRequiredState(phiMeasure);
      } else {
        requiredState = calcRequiredStateCleanupInterval(entry.getKey(), entry.getValue());
      }

      if (entry.getValue() != requiredState) {
        members.put(entry.getKey(), requiredState);
        /* Call listeners asynchronously 异步触发节点状态监听器*/
        for (GossipListener listener: listeners)
          listenerExecutor.execute(() -> listener.gossipEvent(entry.getKey(), requiredState));
      }
    }
  }

  /**
   *
   * @param phiMeasure
   * @return
   */
  public GossipState calcRequiredState(Double phiMeasure) {
    //如果探测节点存活的时间间隔大于阈值，则节点down
    if (phiMeasure > settings.getConvictThreshold())
      return GossipState.DOWN;
    else
      return GossipState.UP;
  }

  /**
   * 首次探活状态
   * @param member
   * @param state
   * @return
   */
  public GossipState calcRequiredStateCleanupInterval(LocalMember member, GossipState state) {
    long now = clock.nanoTime();
    long nowInMillis = TimeUnit.MILLISECONDS.convert(now, TimeUnit.NANOSECONDS);
    //如果当前时间减去清理时间间隔，大于成员的心跳时间，则DOWN
    if (nowInMillis - settings.getCleanupInterval() > member.getHeartbeat()) {
      return GossipState.DOWN;
    } else {
      return state;
    }
  }

  /**
   * If we have a special key the per-node data that means that the node has sent us
   * a pre-emptive shutdown message. We process this so node is seen down sooner
   * 预处理节点状态
   * @param l member to consider
   * @return true if node forced down
   */
  public boolean processOptimisticShutdown(Entry<LocalMember, GossipState> l) {
    //获取节点宕机消息
    PerNodeDataMessage m = findPerNodeGossipData.apply(l.getKey().getId(), ShutdownMessage.PER_NODE_KEY);
    if (m == null) {
      return false;
    }
    ShutdownMessage s = (ShutdownMessage) m.getPayload();
    //如果节点宕机时间大于当前心跳时间
    if (s.getShutdownAtNanos() > l.getKey().getHeartbeat()) {
      members.put(l.getKey(), GossipState.DOWN);
      if (l.getValue() == GossipState.UP) {
        //如果状态变化，则通知监听器
        for (GossipListener listener: listeners)
          listenerExecutor.execute(() -> listener.gossipEvent(l.getKey(), GossipState.DOWN));
      }
      return true;
    }
    return false;
  }

  /**
   * 注解监听器
   * @param listener
   */
  public void register(GossipListener listener) {
    listeners.add(listener);
  }

  /**
   * 关闭gossip成员刷新器
   */
  public void shutdown() {
    scheduledExecutor.shutdown();
    try {
      scheduledExecutor.awaitTermination(5, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
      LOGGER.debug("Issue during shutdown", e);
    }
    listenerExecutor.shutdown();
    try {
      listenerExecutor.awaitTermination(5, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
      LOGGER.debug("Issue during shutdown", e);
    }
    listenerExecutor.shutdownNow();
  }
}
```
节点状态刷新器GossipMemberStateRefresher，通过宕机探测器FailureDetector，探测节点的存活状态，并更新节点状态，针对状态变更的，要通知响应的监听器。


## 数据同步器
```java
/**
 * Base implementation gossips randomly to live nodes periodically gossips to dead ones
 *
 */
public class SimpleActiveGossiper extends AbstractActiveGossiper {

  /**
   * 调度器
   */
  private ScheduledExecutorService scheduledExecutorService;
  /**
   * 任务队列
   */
  private final BlockingQueue<Runnable> workQueue;
  /**
   * 线程池执行器
   */
  private ThreadPoolExecutor threadService;

  /**
   * @param gossipManager
   * @param gossipCore
   * @param registry
   */
  public SimpleActiveGossiper(GossipManager gossipManager, GossipCore gossipCore,
                              MetricRegistry registry) {
    super(gossipManager, gossipCore, registry);
    scheduledExecutorService = Executors.newScheduledThreadPool(2);
    workQueue = new ArrayBlockingQueue<Runnable>(1024);
    threadService = new ThreadPoolExecutor(1, 30, 1, TimeUnit.SECONDS, workQueue,
            new ThreadPoolExecutor.DiscardOldestPolicy());
  }

  @Override
  public void init() {
    super.init();
    scheduledExecutorService.scheduleAtFixedRate(() -> {
      threadService.execute(() -> {
        //发送Live成员
        sendToALiveMember();
      });
    }, 0, gossipManager.getSettings().getGossipInterval(), TimeUnit.MILLISECONDS);
    scheduledExecutorService.scheduleAtFixedRate(() -> {
      //发送Dead成员
      sendToDeadMember();
    }, 0, gossipManager.getSettings().getGossipInterval(), TimeUnit.MILLISECONDS);
    scheduledExecutorService.scheduleAtFixedRate(
            //发送节点数据
            () -> sendPerNodeData(gossipManager.getMyself(),
                    selectPartner(gossipManager.getLiveMembers())),
            0, gossipManager.getSettings().getGossipInterval(), TimeUnit.MILLISECONDS);
    scheduledExecutorService.scheduleAtFixedRate(
            //发送共享数据
            () -> sendSharedData(gossipManager.getMyself(),
                    selectPartner(gossipManager.getLiveMembers())),
            0, gossipManager.getSettings().getGossipInterval(), TimeUnit.MILLISECONDS);
  }
  
  @Override
  public void shutdown() {
    super.shutdown();
    scheduledExecutorService.shutdown();
    try {
      scheduledExecutorService.awaitTermination(5, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
      LOGGER.debug("Issue during shutdown", e);
    }
    //发送关闭消息
    sendShutdownMessage();
    threadService.shutdown();
    try {
      threadService.awaitTermination(5, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
      LOGGER.debug("Issue during shutdown", e);
    }
  }

  /**
   *从当前节点的gossip成员列表中选择一个成员，发送Live节点信息
   */
  protected void sendToALiveMember(){
    LocalMember member = selectPartner(gossipManager.getLiveMembers());
    sendMembershipList(gossipManager.getMyself(), member);
  }

  /**
   * 从当前节点的gossip成员列表中选择一个成员，发送Dead节点信息
   */
  protected void sendToDeadMember(){
    LocalMember member = selectPartner(gossipManager.getDeadMembers());
    sendMembershipList(gossipManager.getMyself(), member);
  }
  
  /**
   * sends an optimistic shutdown message to several clusters nodes
   * 通知族节点，当前节点已关闭
   */
  protected void sendShutdownMessage(){
    List<LocalMember> l = gossipManager.getLiveMembers();
    int sendTo = l.size() < 3 ? 1 : l.size() / 2;
    for (int i = 0; i < sendTo; i++) {
      threadService.execute(() -> sendShutdownMessage(gossipManager.getMyself(), selectPartner(l)));
    }
  }
}
```
数据同步器，同步活跃、宕机成员，节点数据、共享数据。

我们再来看同步活跃节点

//SimpleActiveGossiper
```java
 /**
   *从当前节点的gossip成员列表中选择一个成员，发送Live节点信息
   */
  protected void sendToALiveMember(){
    LocalMember member = selectPartner(gossipManager.getLiveMembers());
    sendMembershipList(gossipManager.getMyself(), member);
  }
```


//AbstractActiveGossiper
```java
 /**
   * 返回本地的随机gossip成员
   * @param memberList
   *          An immutable list
   * @return The chosen LocalGossipMember to gossip with.
   */
  protected LocalMember selectPartner(List<LocalMember> memberList) {
    LocalMember member = null;
    if (memberList.size() > 0) {
      int randomNeighborIndex = random.nextInt(memberList.size());
      member = memberList.get(randomNeighborIndex);
    }
    return member;
  }
 /**
   * Performs the sending of the membership list, after we have incremented our own heartbeat.
   * 在自增心跳之后，发送成员列表
   */
  protected void sendMembershipList(LocalMember me, LocalMember member) {
    if (member == null){
      return;
    }
    long startTime = System.currentTimeMillis();
    me.setHeartbeat(System.nanoTime());
    //保活gossip消息
    UdpActiveGossipMessage message = new UdpActiveGossipMessage();
    message.setUriFrom(gossipManager.getMyself().getUri().toASCIIString());
    message.setUuid(UUID.randomUUID().toString());
    message.getMembers().add(convert(me));
    for (LocalMember other : gossipManager.getMembers().keySet()) {
      message.getMembers().add(convert(other));
    }
    Response r = gossipCore.send(message, member.getUri());
    if (r instanceof ActiveGossipOk){
      //maybe count metrics here
    } else {
      LOGGER.debug("Message " + message + " generated response " + r);
    }
    sendMembershipHistogram.update(System.currentTimeMillis() - startTime);
  }
```

发送活跃节点消息，实际随机从活跃节点的信息，将当前节点的活跃节点信息包装保活gossip消息发送给筛选出的节点。发送宕机节点的原理一致。发送消息委托给core组件。


再来看发送共享数据

//SimpleActiveGossiper
```java
 @Override
  public void init() {
    super.init();
    scheduledExecutorService.scheduleAtFixedRate(() -> {
      threadService.execute(() -> {
        //发送Live成员
        sendToALiveMember();
      });
    }, 0, gossipManager.getSettings().getGossipInterval(), TimeUnit.MILLISECONDS);
    scheduledExecutorService.scheduleAtFixedRate(() -> {
      //发送Dead成员
      sendToDeadMember();
    }, 0, gossipManager.getSettings().getGossipInterval(), TimeUnit.MILLISECONDS);
    scheduledExecutorService.scheduleAtFixedRate(
            //发送节点数据
            () -> sendPerNodeData(gossipManager.getMyself(),
                    selectPartner(gossipManager.getLiveMembers())),
            0, gossipManager.getSettings().getGossipInterval(), TimeUnit.MILLISECONDS);
    scheduledExecutorService.scheduleAtFixedRate(
            //发送共享数据
            () -> sendSharedData(gossipManager.getMyself(),
                    selectPartner(gossipManager.getLiveMembers())),
            0, gossipManager.getSettings().getGossipInterval(), TimeUnit.MILLISECONDS);
  }
```
//AbstractActiveGossiper
```java
 /**
   * 发送共享该数据
   * @param me
   * @param member
   */
  public final void sendSharedData(LocalMember me, LocalMember member) {
    if (member == null) {
      return;
    }
    long startTime = System.currentTimeMillis();
    if (gossipSettings.isBulkTransfer()) {
      sendSharedDataInBulkInternal(me, member);
    } else {
      sendSharedDataInternal(me, member);
    }
    sharedDataHistogram.update(System.currentTimeMillis() - startTime);
  }

 /**
   * Send shared data one entry at a time.
   * 发送共享数据
   * @param me
   * @param member
   */
  private void sendSharedDataInternal(LocalMember me, LocalMember member) {
    for (Entry<String, SharedDataMessage> innerEntry : gossipCore.getSharedData().entrySet()){
      if (innerEntry.getValue().getReplicable() != null && !innerEntry.getValue().getReplicable()
              .shouldReplicate(me, member, innerEntry.getValue())) {
        //不可复制，则直接do nothing
        continue;
      }
      //拷贝共享数据
      UdpSharedDataMessage message = new UdpSharedDataMessage();
      message.setUuid(UUID.randomUUID().toString());
      message.setUriFrom(me.getId());
      copySharedDataMessage(innerEntry.getValue(), message);
      gossipCore.sendOneWay(message, member.getUri());
    }
  } 
```
发送共享数据实际为通过core组件将共享数据保证成UdpSharedDataMessage并发送给peer节点。节点数据原理一致，只不过发送的是节点数据。


## core组件
```java
public class GossipCore implements GossipCoreConstants {

  class LatchAndBase {
    /**
     * 请求锁
     */
    private final CountDownLatch latch;
    /**
     * 消息
     */
    private volatile Base base;
    
    LatchAndBase(){
      latch = new CountDownLatch(1);
    }
    
  }
  public static final Logger LOGGER = Logger.getLogger(GossipCore.class);
  private final GossipManager gossipManager;
  /**
   * 可追溯的请求
   */
  private ConcurrentHashMap<String, LatchAndBase> requests;
  /**
   * 每个节点的数据
   */
  private final ConcurrentHashMap<String, ConcurrentHashMap<String, PerNodeDataMessage>> perNodeData;
  /**
   * 共享数据集
   */
  private final ConcurrentHashMap<String, SharedDataMessage> sharedData;
  private final Meter messageSerdeException;
  /**
   * 传输异常度量器
   */
  private final Meter transmissionException;
  /**
   * 传输成功度量器
   */
  private final Meter transmissionSuccess;
  private final DataEventManager eventManager;
  
  public GossipCore(GossipManager manager, MetricRegistry metrics){
    this.gossipManager = manager;
    requests = new ConcurrentHashMap<>();
    perNodeData = new ConcurrentHashMap<>();
    sharedData = new ConcurrentHashMap<>();
    eventManager = new DataEventManager(metrics);
    metrics.register(PER_NODE_DATA_SIZE, (Gauge<Integer>)() -> perNodeData.size());
    metrics.register(SHARED_DATA_SIZE, (Gauge<Integer>)() ->  sharedData.size());
    metrics.register(REQUEST_SIZE, (Gauge<Integer>)() ->  requests.size());
    messageSerdeException = metrics.meter(MESSAGE_SERDE_EXCEPTION);
    transmissionException = metrics.meter(MESSAGE_TRANSMISSION_EXCEPTION);
    transmissionSuccess = metrics.meter(MESSAGE_TRANSMISSION_SUCCESS);
  }
  ...
}
```
//GossipCore
```java
 /**
   * 接受消息
   * @param base
   */
  public void receive(Base base) {
    if (!gossipManager.getMessageHandler().invoke(this, gossipManager, base)) {
      LOGGER.warn("received message can not be handled");
    }
  }

```

//SharedDataMessageHandler
```java
public class SharedDataMessageHandler implements MessageHandler{
  
  /**
   * @param gossipCore context.
   * @param gossipManager context.
   * @param base message reference.
   * @return boolean indicating success.
   */
  @Override
  public boolean invoke(GossipCore gossipCore, GossipManager gossipManager, Base base) {
    UdpSharedDataMessage message = (UdpSharedDataMessage) base;
    //添加共享数据
    gossipCore.addSharedData(message);
    return true;
  }
}
```


//GossipCore
```java
 /**
   * 添加共享数据
   * @param message
   */
  @SuppressWarnings({ "unchecked", "rawtypes" })
  public void addSharedData(SharedDataMessage message) {
    while (true){
      SharedDataMessage previous = sharedData.putIfAbsent(message.getKey(), message);
      if (previous == null){
        eventManager.notifySharedData(message.getKey(), message.getPayload(), null);
        return;
      }
      if (message.getPayload() instanceof Crdt){
        //合并共享数据
        SharedDataMessage merged = new SharedDataMessage();
        merged.setExpireAt(message.getExpireAt());
        merged.setKey(message.getKey());
        merged.setNodeId(message.getNodeId());
        merged.setTimestamp(message.getTimestamp());
        Crdt mergedCrdt = ((Crdt) previous.getPayload()).merge((Crdt) message.getPayload());
        merged.setPayload(mergedCrdt);
        boolean replaced = sharedData.replace(message.getKey(), previous, merged);
        if (replaced){
          if(!merged.getPayload().equals(previous.getPayload())) {
            eventManager
                    .notifySharedData(message.getKey(), merged.getPayload(), previous.getPayload());
          }
          return;
        }
      } else {
        //非CRDT数据，已最新数据为准
        if (previous.getTimestamp() < message.getTimestamp()){
          boolean result = sharedData.replace(message.getKey(), previous, message);
          if (result){
            eventManager.notifySharedData(message.getKey(), message.getPayload(), previous.getPayload());
            return;
          }
        } else {
          return;
        }
      }
    }
  }
```
peer节点接收消息后，调用消息处理器，处理消息，针对共享数据，如果基于CRDT的数据则合并，否则更新，通知事件处理器；针对节点数据基本相同。


再来看发送数据

//GossipCore
```java
/**
   * @param message
   * @param uri
   * @return
   */
  public Response send(Base message, URI uri){
    if (LOGGER.isDebugEnabled()){
      LOGGER.debug("Sending " + message);
      LOGGER.debug("Current request queue " + requests);
    }

    final Trackable t;
    LatchAndBase latchAndBase = null;
    if (message instanceof Trackable){
      t = (Trackable) message;
      latchAndBase = new LatchAndBase();
      //放入请求集
      requests.put(t.getUuid() + "/" + t.getUriFrom(), latchAndBase);
    } else {
      t = null;
    }
    sendInternal(message, uri);
    if (latchAndBase == null){
      return null;
    }
    
    try {

      boolean complete = latchAndBase.latch.await(1, TimeUnit.SECONDS);
      if (complete){
        //等待结果
        return (Response) latchAndBase.base;
      } else{
        return null;
      }
    } catch (InterruptedException e) {
      throw new RuntimeException(e);
    } finally {
      if (latchAndBase != null){
        //移除高清球
        requests.remove(t.getUuid() + "/" + t.getUriFrom());
      }
    }
  }

  /**
   * Sends a message across the network while blocking. Catches and ignores IOException in transmission. Used
   * when the protocol for the message is not to wait for a response
   * 发送网络消息。忽略，并捕捉传输的io异常。一般用于不需要回复的协议消息
   * @param message the message to send
   * @param u the uri to send it to
   */
  public void sendOneWay(Base message, URI u) {
    try {
      sendInternal(message, u);
    } catch (RuntimeException ex) {
      LOGGER.debug("Send one way failed", ex);
    }
  }
   /**
   * Sends a blocking message.
   * todo: move functionality to TransportManager layer.
   * @param message
   * @param uri
   * @throws RuntimeException if data can not be serialized or in transmission error
   */
  private void sendInternal(Base message, URI uri) {
    byte[] json_bytes;
    try {
      json_bytes = gossipManager.getProtocolManager().write(message);
    } catch (IOException e) {
      messageSerdeException.mark();
      throw new RuntimeException(e);
    }
    try {
      gossipManager.getTransportManager().send(uri, json_bytes);
      transmissionSuccess.mark();
    } catch (IOException e) {
      transmissionException.mark();
      throw new RuntimeException(e);
    }
  }
```
core组件发送消息实际是委托底层的UDP传输管理器进行传输。

## gossip管理器
```java
public abstract class GossipManager {

  public static final Logger LOGGER = Logger.getLogger(GossipManager.class);
  
  // this mapper is used for ring and user-data persistence only. NOT messages.
  public static final ObjectMapper metdataObjectMapper = new ObjectMapper() {
    private static final long serialVersionUID = 1L;
  {
    enableDefaultTyping();
    configure(JsonGenerator.Feature.WRITE_NUMBERS_AS_STRINGS, false);
  }};

  /**
   * gossip 成员
   */
  private final ConcurrentSkipListMap<LocalMember, GossipState> members;
  /**
   * 当前节点信息
   */
  private final LocalMember me;
  /**
   * gossip协议配置
   */
  private final GossipSettings settings;
  /**
   * gossip服务运行状态
   */
  private final AtomicBoolean gossipServiceRunning;

  /**
   * gossip 节点管理
   */
  private TransportManager transportManager;
  /**
   * 协议管理器
   */
  private ProtocolManager protocolManager;
  
  private final GossipCore gossipCore;
  private final DataReaper dataReaper;
  /**
   * 时钟
   */
  private final Clock clock;
  /**
   * 调度器
   */
  private final ScheduledExecutorService scheduledServiced;
  private final MetricRegistry registry;
  /**
   * 节点gossip成员持久器
   */
  private final RingStatePersister ringState;
  /**
   * 用户数据持久器
   */
  private final UserDataPersister userDataState;
  /**
   * 成员状态刷新器
   */
  private final GossipMemberStateRefresher memberStateRefresher;

  /**
   * 消息处理器
   */
  private final MessageHandler messageHandler;
  private final LockManager lockManager;

  /**
   * @param cluster
   * @param uri
   * @param id
   * @param properties  成员属性
   * @param settings
   * @param gossipMembers
   * @param listener
   * @param registry
   * @param messageHandler
   */
  public GossipManager(String cluster,
                       URI uri, String id, Map<String, String> properties, GossipSettings settings,
                       List<Member> gossipMembers, GossipListener listener, MetricRegistry registry,
                       MessageHandler messageHandler) {
    this.settings = settings;
    this.messageHandler = messageHandler;
    //创建系统时钟
    clock = new SystemClock();
    //本地gossip成员
    me = new LocalMember(cluster, uri, id, clock.nanoTime(), properties,
            settings.getWindowSize(), settings.getMinimumSamples(), settings.getDistribution());
    gossipCore = new GossipCore(this, registry);
    this.lockManager = new LockManager(this, settings.getLockManagerSettings(), registry);
    dataReaper = new DataReaper(gossipCore, clock);
    members = new ConcurrentSkipListMap<>();
    for (Member startupMember : gossipMembers) {
      if (!startupMember.equals(me)) {
        LocalMember member = new LocalMember(startupMember.getClusterName(),
                startupMember.getUri(), startupMember.getId(),
                clock.nanoTime(), startupMember.getProperties(), settings.getWindowSize(),
                settings.getMinimumSamples(), settings.getDistribution());
        //TODO should members start in down state?
        members.put(member, GossipState.DOWN);
      }
    }
    gossipServiceRunning = new AtomicBoolean(true);
    this.scheduledServiced = Executors.newScheduledThreadPool(1);
    this.registry = registry;
    //gossip成员持久器
    this.ringState = new RingStatePersister(GossipManager.buildRingStatePath(this), this);
    //用户数据持久器
    this.userDataState = new UserDataPersister(
        gossipCore,
        GossipManager.buildPerNodeDataPath(this),
        GossipManager.buildSharedDataPath(this));
    //gossip成员状态刷新器
    this.memberStateRefresher = new GossipMemberStateRefresher(members, settings, listener, this::findPerNodeGossipData);
    //加载节点gossip成员
    readSavedRingState();
    //加载节点及共享数据
    readSavedDataState();
  }
...
}
```
gossip管理器，对集群的成员、消息处理、全局配置、节点状态持久器、用户数据持久器，协议管理器，udp传输管理器进行集成。我们来
简单看一下共享数据持久器；


```java
public class UserDataPersister implements Runnable {
  
  private static final Logger LOGGER = Logger.getLogger(UserDataPersister.class);
  private final GossipCore gossipCore;

  /**
   * 节点数据文件
   */
  private final File perNodePath;
  /**
   * 节点共享数据文件
   */
  private final File sharedPath;
  private final ObjectMapper objectMapper;
  
  UserDataPersister(GossipCore gossipCore, File perNodePath, File sharedPath) {
    this.gossipCore = gossipCore;
    this.objectMapper = GossipManager.metdataObjectMapper;
    this.perNodePath = perNodePath;
    this.sharedPath = sharedPath;
  }

  /**
   * 从磁盘加载节点数据
   * @return
   */
  @SuppressWarnings("unchecked")
  ConcurrentHashMap<String, ConcurrentHashMap<String, PerNodeDataMessage>> readPerNodeFromDisk(){
    if (!perNodePath.exists()) {
      return new ConcurrentHashMap<String, ConcurrentHashMap<String, PerNodeDataMessage>>();
    }
    try (FileInputStream fos = new FileInputStream(perNodePath)){
      return objectMapper.readValue(fos, ConcurrentHashMap.class);
    } catch (IOException e) {
      LOGGER.debug(e);
    }
    return new ConcurrentHashMap<String, ConcurrentHashMap<String, PerNodeDataMessage>>();
  }

  /**
   *
   */
  void writePerNodeToDisk(){
    try (FileOutputStream fos = new FileOutputStream(perNodePath)){
      objectMapper.writeValue(fos, gossipCore.getPerNodeData());
    } catch (IOException e) {
      LOGGER.warn(e);
    }
  }
  
  void writeSharedToDisk(){
    try (FileOutputStream fos = new FileOutputStream(sharedPath)){
      objectMapper.writeValue(fos, gossipCore.getSharedData());
    } catch (IOException e) {
      LOGGER.warn(e);
    }
  }

  /**
   * 从磁盘加载共享数据
   * @return
   */
  @SuppressWarnings("unchecked")
  ConcurrentHashMap<String, SharedDataMessage> readSharedDataFromDisk(){
    if (!sharedPath.exists()) {
      return new ConcurrentHashMap<>();
    }
    try (FileInputStream fos = new FileInputStream(sharedPath)){
      return objectMapper.readValue(fos, ConcurrentHashMap.class);
    } catch (IOException e) {
      LOGGER.debug(e);
    }
    return new ConcurrentHashMap<String, SharedDataMessage>();
  }
  
  /**
   * Writes all pernode and shared data to disk 
   */
  @Override
  public void run() {
    writePerNodeToDisk();
    writeSharedToDisk();
  }
}
```
用户数据持久UserDataPersister，主要是在节点宕机将用户数据持久化的磁盘，启动时，在从磁盘加载。节点状态持久化器原理相似。



# 总结

gossip是基于UDP协议进行通信的，底层是通过UDP管理器进行消息的收发；消息编解码器使用的是JSON序列化方式，节点接收消息后，有解码器进行解码，委托消息处理器处理，消息处理器有节点数据、共享数据、及宕机消息等。集群中的节点的状态同步通过节点状态刷新器进行同步。针对族活跃节点、宕机节点、节点数据，及共享数据的同步通过数据同步器完成。core组件对节点、共享数据进行管理，及消息的收发。gossip管理器持久节点状态、数据及共享数据。

UDP传输管理UdpTransportManager，主要发送字节数据，接收字节数据，并委托给core组件。在UDP传输管理器启动的过程中，同时启动节点状态刷新器。

消息编解码器JacksonProtocolManager是基于JSON的消息编解码，如果是RSA加签方式，则在收发消息，做响应的加解密。

共享数据和节点数据处理实际是委托给core组件，宕机消息转换为节点消息进行处理。

节点状态刷新器GossipMemberStateRefresher，通过宕机探测器FailureDetector，探测节点的存活状态，并更新节点状态，针对状态变更的，要通知响应的监听器。

数据同步器，同步活跃、宕机成员，节点数据、共享数据。发送活跃节点消息，实际随机从活跃节点的信息，将当前节点的活跃节点信息包装保活gossip消息发送给筛选出的节点。发送宕机节点的原理一致。发送消息委托给core组件。发送共享数据实际为通过core组件将共享数据保证成UdpSharedDataMessage并发送给peer节点。节点数据原理一致，只不过发送的是节点数据。

peer节点接收消息后，调用消息处理器，处理消息，针对共享数据，如果基于CRDT的数据则合并，否则更新，通知事件处理器；针对节点数据基本相同。core组件发送消息实际是委托底层的UDP传输管理器进行传输。

gossip管理器，对集群的成员、消息处理、全局配置、节点状态持久器、节点数据持久器、共享数据持久器，协议管理器，udp传输管理器进行集成。



# 附
## Gossip
[P2P 网络核心技术：Gossip 协议](https://zhuanlan.zhihu.com/p/41228196)   
[漫谈 Gossip 协议](https://segmentfault.com/a/1190000022957348)   
[浅谈集群版Redis和Gossip协议](https://juejin.im/post/6844904002098823181)
   

[incubator-retired-gossip](https://github.com/Donaldhan/incubator-retired-gossip)  
[apache gossip](http://incubator.apache.org/projects/gossip.html)  


[分布式一致性算法-CRDT一](https://kknews.cc/code/9oen8oj.html)   
[分布式一致性算法-CRDT二](https://kknews.cc/code/lekzp92.html)  
[分布式CRDT模型](https://www.jdon.com/artichect/crdt.html)   
[如何选择地理分布式双活策略：合并复制和CRDT之对比](https://www.infoq.cn/article/database-merge-replication-crdt)     
[何时使用基于CRDT的数据库](https://chi.small-business-tracker.com/when-use-crdt-based-database-542900)    
[基于TLA+的CRDT协议验证框架](https://zhuanlan.zhihu.com/p/86256851)    

[单实例比redis吞吐大12倍，Anna用了什么黑科技？](https://lawrencepeng.github.io/2018/08/08/%E5%8D%95%E5%AE%9E%E4%BE%8B%E6%AF%94redis%E5%90%9E%E5%90%90%E5%A4%A712%E5%80%8D%EF%BC%8CAnna%E7%94%A8%E4%BA%86%E4%BB%80%E4%B9%88%E9%BB%91%E7%A7%91%E6%8A%80/)  




## Kademlia
[P2P 网络核心技术：Kademlia 协议](https://zhuanlan.zhihu.com/p/40286711)   
[Kademlia协议](https://zhuanlan.zhihu.com/p/38425656)   
[Kademlia](https://github.com/Donaldhan/Kademlia)  
[OpenKAD github](https://github.com/Donaldhan/OpenKAD)  
[openkad google](https://code.google.com/archive/p/openkad/)