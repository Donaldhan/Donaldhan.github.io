---
layout: page
title: Nacos框架设计
subtitle: Nacos框架设计
date: 2026-11-27 14:51:00
author: Ravitn
catalog: true
category: Microservices
categories:
    - Microservices
tags:
    - Nacos
---

# 引言

随着微服务架构的普及，服务治理与配置管理成为分布式系统的核心挑战。Nacos 作为阿里巴巴开源的动态服务发现、配置管理和服务管理平台，致力于帮助开发者构建云原生应用，实现服务的自动注册与发现、配置的动态更新、以及服务的健康检查与流量管理。Nacos 以其"一个平台解决两大问题"的设计理念，成为 Spring Cloud Alibaba 生态中不可或缺的基础设施组件。

Nacos 提供了两大核心能力：
- **配置管理**：动态配置服务，支持配置的发布、灰度、版本管理与监听推送。
- **服务发现**：服务注册与发现，支持临时实例（AP）与持久实例（CP）双轨制。

本文基于 Nacos 2.x 源码进行框架设计分析，涵盖从启动入口、客户端 SDK、通信层到核心业务模块（config/naming）以及一致性协议（JRaft/Distro）的完整链路。

源码的阅读参见：
[Nacos github](https://github.com/Donaldhan/nacos)

# 目录
* [Nacos框架设计](#Nacos框架设计)
* [启动入口](#启动入口)
* [配置管理流程](#配置管理流程)
* [服务发现流程](#服务发现流程)
* [集群一致性协议](#集群一致性协议)
* [通信层](#通信层)
* [总结](#总结)
* [附](#附)


# Nacos框架设计

Nacos 框架主要角色有客户端 Client、服务端 Server、数据库 DB 三大核心。客户端 Client 通过 SDK（NacosConfigService、NacosNamingService）与服务端建立 gRPC 长连接，完成配置的发布/监听与服务的注册/订阅。服务端 Server 由 console 模块启动，核心业务包括配置管理（config）与服务发现（naming）两大模块，底层依赖一致性协议（JRaft 用于持久数据、Distro 用于临时数据）保证集群数据一致性。数据库 DB 支持两种存储模式：外部 MySQL（External）与内嵌 Derby（Embedded + JRaft），分别对应 AP 与 CP 的不同一致性语义。

Nacos 框架的核心模块关系：

```
distribution (发布包)
    └── console (控制台入口, 主类 Nacos)
        ├── config (配置管理)
        │   ├── ConfigController (HTTP入口)
        │   ├── ConfigOperationService (业务逻辑)
        │   ├── ConfigInfoPersistService (持久化, External/Embedded双实现)
        │   ├── DumpService (本地缓存与磁盘)
        │   ├── AsyncNotifyService (集群同步)
        │   └── RpcConfigChangeNotifier / LongPollingService (客户端推送)
        ├── naming (服务发现)
        │   ├── InstanceRequestHandler / PersistentInstanceRequestHandler (实例注册)
        │   ├── EphemeralClientOperationServiceImpl (临时实例, Distro/AP)
        │   ├── PersistentClientOperationServiceImpl (持久实例, JRaft/CP)
        │   ├── ClientServiceIndexesManager (索引管理)
        │   └── NamingSubscriberServiceV2Impl (订阅推送)
        ├── core (核心基础设施)
        │   ├── ServerMemberManager (集群成员管理)
        │   ├── JRaftProtocol / JRaftServer (Raft协议)
        │   ├── GrpcSdkServer / GrpcRequestAcceptor (gRPC通信)
        │   └── ConnectionManager (连接管理)
        └── persistence (持久化, 支持MySQL/Derby)

client (客户端 SDK)
    ├── NacosConfigService (配置客户端)
    ├── NacosNamingService (服务发现客户端)
    └── NamingGrpcRedoService (断线重连补偿)
```

框架的底层基于 gRPC 双向流通信，替代了 1.x 时代的 HTTP 短轮询。客户端通过 `RpcClient` 与服务端建立长连接，所有请求（注册、订阅、配置发布、配置监听）都复用这条连接。服务端通过 `RequestHandlerRegistry` 按请求类型路由到对应的处理器，实现了配置、命名、健康检查等多业务共享同一套通信通道。


# 启动入口

Nacos Server 的启动入口为 console 模块下的 `com.alibaba.nacos.Nacos`：

```java
@SpringBootApplication
@ComponentScan(basePackages = "com.alibaba.nacos", excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = {NacosTypeExcludeFilter.class}),
        @Filter(type = FilterType.CUSTOM, classes = {TypeExcludeFilter.class}),
        @Filter(type = FilterType.CUSTOM, classes = {AutoConfigurationExcludeFilter.class})})
@ServletComponentScan
public class Nacos {
    public static void main(String[] args) {
        SpringApplication.run(Nacos.class, args);
    }
}
```

启动流程：

```
startup.sh
  → java -jar nacos-server.jar
    → Nacos.main()
      → SpringApplication.run()
        → 启动 Spring Boot 应用
          → ServerMemberManager.init() (集群成员初始化)
          → GrpcSdkServer.start() (gRPC服务启动)
          → JRaftProtocol.init() (Raft协议初始化)
          → DumpService.init() (配置缓存初始化)
```

关键初始化链路：

1. **ServerMemberManager**：集群成员管理，通过三种寻址模式（Standalone/FileConfig/AddressServer）获取集群列表，启动后通过 `MemberInfoReportTask` 每 2 秒游标轮转上报节点状态，维护 UP/SUSPICIOUS/DOWN 状态机。
2. **GrpcSdkServer**：启动 gRPC 服务端，注册连接拦截器与请求处理器，处理客户端的连接建立与请求路由。
3. **JRaftProtocol**：初始化 Raft 协议，创建独立的 Raft Group（naming_persistent_service_v2、config_data 等），启动 Leader 选举。
4. **DumpService**：根据存储模式（External/Embedded）执行配置数据的全量 dump，将数据库配置加载到本地磁盘与内存缓存。


# 配置管理流程

配置管理涵盖配置发布与配置监听两大核心链路。

## 配置发布链路

```
客户端/控制台          Nacos Server              数据库              其他节点              客户端监听器
    |                      |                       |                   |                      |
    |--POST /configs------>|                       |                   |                      |
    |                      |                       |                   |                      |
    |                  [ConfigController]          |                   |                      |
    |                  参数校验 + 加密              |                   |                      |
    |                      |                       |                   |                      |
    |                  [ConfigOperationService]    |                   |                      |
    |                      |--INSERT/UPDATE------->|                   |                      |
    |                      |<------OK-------------|                   |                      |
    |                      |                       |                   |                      |
    |                  [ConfigChangePublisher]     |                   |                      |
    |                      |--发布事件------------->|                   |                      |
    |                      |                       |                   |                      |
    |                  [DumpService]               |                   |                      |
    |                      |--Dump到本地---------->|                   |                      |
    |                      |--更新缓存------------>|                   |                      |
    |                      |                       |                   |                      |
    |                  [AsyncNotifyService]        |                   |                      |
    |                      |                       |                   |--gRPC同步配置变更---->|
    |                      |                       |                   |                      |
    |                      |                       |               [其他节点Dump]              |
    |                      |                       |                   |                      |
    |                      |                       |               [发布LocalDataChangeEvent] |
    |                      |                       |                   |                      |
    |                  [LongPollingService]        |                   |                      |
    |                      |--返回变更Group--------|-------------------|--------------------->|
    |                      |                       |                   |                      |
    |                  [RpcConfigChangeNotifier]   |                   |                      |
    |                      |--gRPC推送通知--------|-------------------|--------------------->|
    |                      |                       |                   |                      |
    |<-----HTTP响应--------|                       |                   |                      |
    |                      |                       |                   |                      |
    |                  客户端重新拉取配置           |                   |                      |
    |<--配置内容-----------|                       |                   |                      |
```

**发布配置**

1. 客户端通过 HTTP POST `/nacos/v1/cs/configs` 或 gRPC `ConfigPublishRequest` 发起配置发布请求；
2. `ConfigController.publishConfig()` 进行参数校验、加密处理、命名空间处理；
3. `ConfigOperationService.publishConfig()` 判断发布类型（Beta 灰度、Tag 灰度、CAS 乐观锁、普通发布），调用 `ConfigInfoPersistService.addConfigInfo()` 持久化到数据库；
4. 发布 `ConfigDataChangeEvent` 事件到事件总线；
5. **DumpService** 订阅该事件，将配置从数据库 dump 到本地磁盘（`/nacos/data/config-data/`），更新内存缓存的 MD5，发布 `LocalDataChangeEvent`；
6. **AsyncNotifyService** 订阅该事件，通过 gRPC `ConfigChangeClusterSyncRequest` 同步到集群其他节点；
7. 其他节点收到同步请求后执行相同的 dump 操作；
8. **RpcConfigChangeNotifier**（gRPC）和 **LongPollingService**（HTTP 长轮询）订阅 `LocalDataChangeEvent`，主动推送配置变更通知给客户端；
9. 客户端收到变更通知后，通过 gRPC 拉取最新配置内容，更新本地缓存，触发配置监听器回调。

## 两种存储模式的区别

Nacos 配置持久化支持两种模式，核心区别在于"原子性与复制由谁提供"：

| 维度 | External（MySQL） | Embedded（Derby + JRaft） |
|------|------------------|--------------------------|
| 原子性手段 | Spring 编程式事务（TransactionTemplate） | 一条 Raft 日志包含整包 SQL |
| 集群同步 | 共享 MySQL 天然提供 | Raft 多数派复制 + 各节点 apply |
| 主键来源 | MySQL 自增（KeyHolder） | 本地雪花算法（SnowFlowerIdGenerator） |
| 集群通知 | `ConfigChangePublisher` 广播 `ConfigDataChangeEvent` | 通知被短路，dump 作为日志回放的副产品 |
| 读一致性 | 事务内读自己的写 | ReadIndex 线性一致读 |

**External 模式**：SQL 立刻打到共享 MySQL，靠 Spring 事务保证原子性，集群一致性由"大家共用一个库"天然提供。

**Embedded 模式**：SQL 先不执行，攒进 ThreadLocal，最后把整包 SQL 序列化成一条 JRaft 日志提交，由每个节点的 `onApply` 在本地事务里重放。一致性由 Raft 多数派提供。

## 客户端配置监听

客户端通过 `NacosConfigService` 监听配置：

1. 客户端从服务端拉取配置，保存到本地快照目录（`~/nacos/config/`）；
2. 注册监听到客户端工作线程（`ClientWorker`），添加配置缓存（`CacheData`）；
3. 标记缓存与服务端不一致，通知客户端工作线程存在变更；
4. 客户端工作线程通过 gRPC 拉取配置变更，更新本地缓存，触发监听器回调 `Listener.receiveConfigInfo`；
5. 服务端配置变更时，通过 gRPC 推送 `ConfigChangeNotifyRequest` 通知客户端，客户端标记 `receiveNotifyChanged=true`，重新拉取配置。


# 服务发现流程

服务发现涵盖服务注册、服务订阅、服务查询三大核心链路，分为临时实例（AP/Distro）与持久实例（CP/JRaft）双轨制。

## 临时实例注册流程（AP/Distro）

```
业务应用                客户端SDK                Nacos Server              集群其他节点
    |                      |                       |                         |
    |--registerInstance-->|                       |                         |
    |                      |                       |                         |
    |              [NacosNamingService]            |                         |
    |              参数校验+group前缀归一化          |                         |
    |                      |                       |                         |
    |              [NamingClientProxyDelegate]    |                         |
    |              通道选择(gRPC/HTTP)              |                         |
    |                      |                       |                         |
    |              [NamingGrpcClientProxy]         |                         |
    |              ① cacheInstanceForRedo          |                         |
    |              ② doRegisterService             |                         |
    |                      |                       |                         |
    |              [RpcClient.request]------------>|                         |
    |                      |               [InstanceRequestHandler]           |
    |                      |               [EphemeralClientOperationServiceImpl]
    |                      |               client.addServiceInstance(内存)   |
    |                      |                       |                         |
    |                      |            事件扇出:    |                         |
    |                      |            ① ClientChangedEvent → Distro同步    |
    |                      |            ② ClientRegisterServiceEvent → 索引  |
    |                      |            ③ InstanceMetadataEvent → 元数据      |
    |                      |                       |                         |
    |                      |              [DistroClientDataProcessor]-------->|
    |                      |               同步整条Client的所有publisher       |
    |                      |                       |                         |
    |                      |              [ClientServiceIndexesManager]     |
    |                      |              publisherIndexes[service].add(clientId)
    |                      |                       |                         |
    |                      |              [NamingSubscriberServiceV2Impl]     |
    |                      |              PushDelayTask(500ms合并窗口)           |
    |                      |              gRPC推送订阅客户端                   |
    |<--注册成功-----------|<--InstanceResponse------|                         |
    |                      |                       |                         |
    |              ③ instanceRegistered=true       |                         |
```

**临时实例注册**

1. 客户端 `NacosNamingService.registerInstance()` 进行参数校验（心跳时间关系、clusterName 正则）与 group 前缀归一化；
2. `NamingClientProxyDelegate` 根据实例类型（ephemeral=true）与服务端能力位选择 gRPC 通道；
3. `NamingGrpcClientProxy.registerServiceForEphemeral()` 先调用 `cacheInstanceForRedo()` 写入 redo 状态表（`expectedRegistered=true, registered=false`），再发送 `InstanceRequest`；
4. 服务端 `InstanceRequestHandler.handle()` 接收请求，`EphemeralClientOperationServiceImpl.registerInstance()` 将实例以压缩后的 `InstancePublishInfo` 写入 `Client.publishers`（全程内存，无数据库操作）；
5. 事件扇出三条链路：
   - **ClientChangedEvent**：`DistroClientDataProcessor` 同步整条 Client 的所有 publisher 到集群其他节点（AP 一致性）；
   - **ClientRegisterServiceEvent**：`ClientServiceIndexesManager` 更新 `publisherIndexes`，发布 `ServiceChangedEvent`，触发 `PushDelayTask` 推送订阅客户端；
   - **InstanceMetadataEvent**：`NamingMetadataManager` 维护服务元数据过期信息；
6. 客户端收到成功响应后调用 `instanceRegistered()` 标记 `registered=true`。

**Redo 机制（断线自愈）**：

- 连接断开时，`NamingGrpcRedoService.onDisConnect()` 将所有 `registered` 标记为 `false`；
- 定时任务（默认 3 秒）扫描 redo 表，根据三维状态机（`registered/unregistering/expectedRegistered`）判定 `RedoType.REGISTER/UNREGISTER/REMOVE`，自动补偿重发；
- 保证"至少一次投递 + 最终一致"的语义。

## 持久实例注册流程（CP/JRaft）

```
业务应用                客户端SDK                Nacos Server(Leader)        集群Follower
    |                      |                       |                         |
    |--registerInstance-->|                       |                         |
    |                      |                       |                         |
    |              [NamingGrpcClientProxy]         |                         |
    |              doRegisterServiceForPersistent   |                         |
    |                      |                       |                         |
    |              [PersistentInstanceRequest]---->|                         |
    |                      |               [PersistentInstanceRequestHandler]|
    |                      |               clientId = ip:port#false          |
    |                      |               (丢弃connectionId)                 |
    |                      |                       |                         |
    |                      |               [PersistentClientOperationServiceImpl]
    |                      |               ① Hessian序列化InstanceStoreRequest
    |                      |               ② 构建WriteRequest(ADD)           |
    |                      |               ③ protocol.write() 同步等10s       |
    |                      |                       |                         |
    |                      |               [JRaftProtocol.write]               |
    |                      |               [JRaftServer.commit]              |
    |                      |               本机是Leader?                      |
    |                      |               是 → applyOperation → node.apply |
    |                      |               否 → invokeToLeader RPC转发        |
    |                      |                       |                         |
    |                      |               写本地log → 复制到Follower ------>|
    |                      |               多数派ACK                            |
    |                      |                       |                         |
    |                      |               [NacosStateMachine.onApply]       |
    |                      |               onInstanceRegister:              |
    |                      |               client.addServiceInstance           |
    |                      |               HealthCheckInstancePublishInfo     |
    |                      |                       |<------Follower同样onApply-|
    |                      |                       |                         |
    |                      |               事件扇出(与临时实例共用后半程):       |
    |                      |               ClientRegisterServiceEvent        |
    |                      |               → publisherIndexes                  |
    |                      |               → ServiceChangedEvent                |
    |                      |               → PushDelayTask → 推送               |
    |<--注册成功-----------|<--InstanceResponse------|                         |
```

**持久实例注册**

1. 客户端通过 `PersistentInstanceRequest` 发送持久实例注册请求；
2. 服务端 `PersistentInstanceRequestHandler` 将 clientId 设为 `ip:port#false`（丢弃 connectionId），身份绑定为逻辑 ip:port 而非物理连接；
3. `PersistentClientOperationServiceImpl` 是一个双重身份的类：对上是 `ClientOperationService`（业务接口），对下是 `RequestProcessor4CP`（Raft 状态机回调）；
4. 将 `InstanceStoreRequest` Hessian 序列化为 `WriteRequest(operation=ADD)`，调用 `protocol.write()` 同步等待 Raft 多数派提交（最长 10 秒）；
5. Leader 节点本地 `applyOperation` 写日志，Follower 通过日志复制接收；
6. 每个节点的 `NacosStateMachine.onApply` 反序列化并落地到 `IpPortBasedClient.publishers`，升级为 `HealthCheckInstancePublishInfo`（启动健康检查）；
7. 事件扇出（与临时实例共用后半程）：`ClientRegisterServiceEvent` → 索引更新 → `ServiceChangedEvent` → 推送订阅客户端。

**临时实例 vs 持久实例对比**：

| 维度 | 临时实例（AP/Distro） | 持久实例（CP/JRaft） |
|------|---------------------|---------------------|
| 身份绑定 | gRPC connectionId | ip:port#false（逻辑会话） |
| 断连后果 | 实例消失，靠 redo 重注册 | 无影响，实例永久存在 |
| 一致性 | Distro 异步复制（AP） | Raft 多数派提交（CP） |
| 健康判定 | 客户端心跳/连接状态 | 服务端主动探测（HealthCheckTaskV2） |
| 一进程多实例 | 同连接同服务只能存 1 个 | 每个 ip:port 独立 client |
| 过期回收 | 按时间超时 | 永不超时，靠显式 DELETE |

## 服务订阅流程

Nacos 的订阅是"客户端本地过滤 + 服务端定向推送 + 轮询兜底"的三层结构：

1. **客户端**：`subscribe(svc, group, clusters, listener)` 创建 `ClusterSelector`（clusters 只在本地过滤，不上网络），先注册本地监听器，再通过 gRPC 发送 `SubscribeServiceRequest`（clusters 传空串）；
2. **服务端**：`SubscribeServiceRequestHandler` 将订阅关系记录在 `Client.subscribers` 与 `subscriberIndexes` 反向索引中（**订阅关系不参与 Distro 集群同步**，只存在于连接所在节点）；
3. **首次订阅**：服务端同步返回当前服务实例列表；后续变更通过 `PushDelayTask`（500ms 合并窗口）定向推送 `NotifySubscriberRequest` 给订阅客户端；
4. **客户端**：收到推送后 `ServiceInfoHolder.processServiceInfo` 进行 diff 检测，发布 `InstancesChangeEvent`，通过 `NamingSelectorWrapper` 二次过滤（按 cluster），最终回调用户 `EventListener`。

**关键设计**：订阅关系只属于连接所在节点，不复制到集群。客户端连哪个节点，就只被那个节点推送，因此 gRPC 长连接天然黏性，LB 不能每个请求换一个节点。


# 集群一致性协议

Nacos 采用双一致性协议体系：

## JRaft（CP 协议）

用于持久数据的强一致保证：
- **持久实例注册**（naming_persistent_service_v2 group）
- **配置数据**（Derby 集群模式，config_data group）
- **SwitchManager 集群开关**

JRaft 关键设计：
1. 每个 `RequestProcessor4CP` 独占一个 Raft Group，独立的 log/snapshot/meta 目录；
2. Leader 本地直接 `applyOperation` 写日志，Follower 通过 `invokeToLeader` RPC 转发；
3. 写日志时前置 2 字节 request-type 标签，回放侧自描述还原；
4. 快照写以 client 为单位全量 dump，带 CRC64 校验防磁盘静默腐坏；
5. 健康状态变更也写成 Raft 日志（CHANGE 操作），全集群强一致的 healthy。

## Distro（AP 协议）

用于临时数据的最终一致保证：
- **临时实例注册**（EphemeralClient）
- **服务元数据**

Distro 关键设计：
1. 同步单位是整条 Client 的所有 publisher（一次打包，减少 RPC 次数）；
2. 只有连接归属节点（`isResponsibleClient`）才发起复制，避免多节点重复广播；
3. 读侧懒聚合：写路径只维护索引（O(1)），"聚合实例列表"推迟到查询/推送时做；
4. 订阅关系不参与 Distro 同步（连接级本地状态）。


# 通信层

Nacos 2.x 基于 gRPC 双向流通信，替代 1.x 的 HTTP 短轮询：

```
客户端                          服务端
  |                              |
  |---GrpcClient.connectToServer-->|
  |---ConnectionSetupRequest------>|
  |                    [GrpcBiStreamRequestAcceptor]
  |                    ConnectionManager.register
  |                              |
  |<--连接建立成功-----------------|
  |                              |
  |---InstanceRequest------------>|
  |                    [GrpcRequestAcceptor]
  |                    RequestHandlerRegistry.getByRequestType
  |                    → InstanceRequestHandler.handle
  |                              |
  |<--InstanceResponse------------|
  |                              |
  |<--NotifySubscriberRequest----|  (服务端主动推送)
  |<--ConfigChangeNotifyRequest--|
  |---NotifySubscriberResponse-->|
```

**gRPC 请求处理器路由表**：

| Handler | Request 类型 | 作用 |
|---------|-------------|------|
| `InstanceRequestHandler` | InstanceRequest | 临时实例注册/心跳/注销 |
| `PersistentInstanceRequestHandler` | PersistentInstanceRequest | 持久实例操作 |
| `BatchInstanceRequestHandler` | BatchInstanceRequest | 批量实例操作 |
| `ServiceQueryRequestHandler` | ServiceQueryRequest | 服务实例查询 |
| `SubscribeServiceRequestHandler` | SubscribeServiceRequest | 服务订阅 |
| `ConfigChangeBatchListenRequestHandler` | ConfigChangeBatchListenRequest | 配置监听/注销 |
| `ConfigQueryRequestHandler` | ConfigQueryRequest | 配置查询 |
| `ConfigPublishRequestHandler` | ConfigPublishRequest | 配置发布 |
| `ConfigChangeClusterSyncRequestHandler` | ConfigChangeClusterSyncRequest | 集群配置变更同步 |
| `HealthCheckRequestHandler` | HealthCheckRequest | 健康检查 |
| `DistroDataRequestHandler` | DistroDataRequest | Distro 协议数据同步 |
| `MemberReportHandler` | MemberReportRequest | 集群节点状态上报 |

**双通道兼容机制**：
- 客户端与服务端建连时握手返回 `ServerAbilities`（能力矩阵）；
- `MemberInfoReportTask` 根据对端能力自动选择 gRPC 或 HTTP 通道上报；
- HTTP 失败自动重试 gRPC，gRPC 返回 `NO_HANDLER` 自动回退 HTTP，实现滚动升级兼容。


# 总结

Nacos 框架的优点主要是：

1. **双一致性协议**：临时实例走 Distro（AP），持久实例走 JRaft（CP），满足不同业务场景的一致性需求；
2. **事件驱动架构**：两级事件机制（`ConfigDataChangeEvent` → 集群同步 + 本地 dump；`LocalDataChangeEvent` → 客户端推送），解耦业务逻辑与处理逻辑；
3. **长连接替代短轮询**：gRPC 双向流通信，支持服务端主动推送，降低延迟与网络开销；
4. **Redo 自愈机制**：客户端三维状态机保证"至少一次投递 + 最终一致"，断线自动重注册；
5. **懒聚合读优化**：写路径只维护索引，读时反查聚合，避免注册风暴下的写放大；
6. **双通道推送**：gRPC 推送（新版本）+ HTTP 长轮询（老版本兼容），平滑迁移；
7. **能力协商**：基于 `ServerAbilities` 的协议自适应，支持混合版本集群滚动升级。

**潜在缺点**：
- 持久实例注册延迟受 Raft 多数派 RTT 支配，P99 较高；
- Embedded 模式下错误语义在 apply 边界处存在断裂（业务异常被吞成 success=false，可能假成功）；
- 同连接同服务只能存一个实例（`publishers` 以 Service 为 key），多实例需多个 NacosNamingService；
- 订阅关系不跨节点，客户端黏性连接，LB 不能随意换节点。


# 附

源码代码阅读入口

## Nacos Server 启动入口
com.alibaba.nacos.Nacos

## 集群成员管理
### ServerMemberManager
com.alibaba.nacos.core.cluster.ServerMemberManager.init

com.alibaba.nacos.core.cluster.ServerMemberManager.memberChange

com.alibaba.nacos.core.cluster.ServerMemberManager.MemberInfoReportTask

## gRPC 通信层
### GrpcSdkServer
com.alibaba.nacos.core.remote.grpc.GrpcSdkServer

com.alibaba.nacos.core.remote.grpc.GrpcBiStreamRequestAcceptor.requestBiStream

com.alibaba.nacos.core.remote.grpc.GrpcRequestAcceptor.request

com.alibaba.nacos.core.remote.RequestHandlerRegistry.getByRequestType

## JRaft 协议
### JRaftProtocol
com.alibaba.nacos.core.distributed.raft.JRaftProtocol.init

com.alibaba.nacos.core.distributed.raft.JRaftProtocol.write

com.alibaba.nacos.core.distributed.raft.JRaftServer.commit

com.alibaba.nacos.core.distributed.raft.JRaftServer.applyOperation

com.alibaba.nacos.core.distributed.raft.JRaftServer.invokeToLeader

### NacosStateMachine
com.alibaba.nacos.core.distributed.raft.NacosStateMachine.onApply

## 配置管理
### ConfigController
com.alibaba.nacos.config.server.controller.ConfigController.publishConfig

### ConfigOperationService
com.alibaba.nacos.config.server.service.ConfigOperationService.publishConfig

### ConfigChangePublisher
com.alibaba.nacos.config.server.service.ConfigChangePublisher.notifyConfigChange

### DumpService
com.alibaba.nacos.config.server.service.dump.DumpService.handleConfigDataChange

com.alibaba.nacos.config.server.service.dump.DumpService.dump

### AsyncNotifyService
com.alibaba.nacos.config.server.service.notify.AsyncNotifyService.handleConfigDataChangeEvent

### RpcConfigChangeNotifier
com.alibaba.nacos.config.server.remote.RpcConfigChangeNotifier.onEvent

### LongPollingService
com.alibaba.nacos.config.server.service.LongPollingService

## 配置持久化（双实现）
### ExternalConfigInfoPersistServiceImpl
com.alibaba.nacos.config.server.service.repository.extrnal.ExternalConfigInfoPersistServiceImpl.addConfigInfo

### EmbeddedConfigInfoPersistServiceImpl
com.alibaba.nacos.config.server.service.repository.embedded.EmbeddedConfigInfoPersistServiceImpl.addConfigInfo

com.alibaba.nacos.core.persistence.DistributedDatabaseOperateImpl.onApply

## 服务发现
### NacosNamingService
com.alibaba.nacos.client.naming.NacosNamingService.registerInstance

com.alibaba.nacos.client.naming.NacosNamingService.subscribe

### NamingGrpcClientProxy
com.alibaba.nacos.client.naming.remote.gprc.NamingGrpcClientProxy.registerServiceForEphemeral

com.alibaba.nacos.client.naming.remote.gprc.NamingGrpcClientProxy.doRegisterService

### NamingGrpcRedoService
com.alibaba.nacos.client.naming.remote.gprc.redo.NamingGrpcRedoService.cacheInstanceForRedo

com.alibaba.nacos.client.naming.remote.gprc.redo.NamingGrpcRedoService.onDisConnect

com.alibaba.nacos.client.naming.remote.gprc.redo.RedoScheduledTask.run

### InstanceRequestHandler（临时实例）
com.alibaba.nacos.naming.remote.rpc.handler.InstanceRequestHandler.handle

com.alibaba.nacos.naming.core.v2.service.impl.EphemeralClientOperationServiceImpl.registerInstance

com.alibaba.nacos.naming.core.v2.client.AbstractClient.addServiceInstance

### PersistentInstanceRequestHandler（持久实例）
com.alibaba.nacos.naming.remote.rpc.handler.PersistentInstanceRequestHandler.handle

com.alibaba.nacos.naming.core.v2.service.impl.PersistentClientOperationServiceImpl.registerInstance

com.alibaba.nacos.naming.core.v2.service.impl.PersistentClientOperationServiceImpl.onApply

### Distro 协议
com.alibaba.nacos.naming.consistency.ephemeral.distro.v2.DistroClientDataProcessor.onEvent

com.alibaba.nacos.core.distributed.distro.DistroProtocol.sync

com.alibaba.nacos.core.distributed.distro.task.execute.DistroSyncChangeTask.doExecute

### 索引与推送
com.alibaba.nacos.naming.core.v2.index.ClientServiceIndexesManager.handleClientOperation

com.alibaba.nacos.naming.push.v2.NamingSubscriberServiceV2Impl.onEvent

com.alibaba.nacos.naming.push.v2.task.PushExecuteTask.run

### 订阅服务
com.alibaba.nacos.naming.remote.rpc.handler.SubscribeServiceRequestHandler.handle

com.alibaba.nacos.naming.core.v2.service.impl.EphemeralClientOperationServiceImpl.subscribeService

## 配置客户端
### NacosConfigService
com.alibaba.nacos.client.config.NacosConfigService.publishConfig

com.alibaba.nacos.client.config.NacosConfigService.getConfigAndSignListener

### ClientWorker
com.alibaba.nacos.client.config.impl.ClientWorker.ConfigRpcTransportClient.handleConfigChangeNotifyRequest

com.alibaba.nacos.client.config.impl.ClientWorker.ConfigRpcTransportClient.executeConfigListen

com.alibaba.nacos.client.config.impl.CacheData.checkListenerMd5
