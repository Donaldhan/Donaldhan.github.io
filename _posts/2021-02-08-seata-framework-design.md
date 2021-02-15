---
layout: page
title: Seata分布式事务框架设计
subtitle: Seata分布式事务框架设计
date: 2021-02-08 14:51:00
author: Ravitn
catalog: true
category: Seata
categories:
    - Seata
tags:
    - Seata
---

# 引言

随着应用体量的增加，微服务作为上帝之手为超级单体打开了屏障；同时使用整个应用服务显得更加清晰。服务寄托于数据，如何解决分布式事务的ACID，成为分布式事务亟待解决的问题。比较有名的分布式事务规范有XA（2PC），TCC(3PC), SAGA, 基于BASE理论的本地事务表重试达到最终一致性解决方案；基于MQ的2PC+补偿机制（事务回查）解决方案。Seata 作为一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。



此文中事务模型为**AT事务**，关于AT模式的demo可以参加下文：  
[Seata分布式事务AT模式初体验](https://donaldhan.github.io/seata/2021/01/19/seata-quick-start.html) 

源码的阅读参见：   
[seata github](https://github.com/Donaldhan/seata)

# 目录
* [Seata框架设计](#Seata框架设计)
* [全局事务流程](#全局事务流程)
* [附](#附)


# Seata框架设计

![seata-framwork-design](/image/seata/seata-framework-design.png)

Seata框架主要角色有事务管理器TM，资源管理器RM，事务协调器TC。事务管理器TM的核心模块为全局事务管理器，主要负责全局事务的开启，提交，回滚。事务协调器TC根据
事务管理器TM的全局事务的开启，提交，回滚请求，创建全局事务，驱动分支事务的提交和回滚，并维护全局和分支事务的状态。事务管理器TM开启全局事务后，执行实际的业务逻辑，实际业务由资源管理器RM的分支事务完成。资源管理器RM执行事务时，由对应的执行器去执行，包括重做日志，SQL的执行；实际的SQL由连接代理ConnectProxy执行，连接代理ConnectProxy会注册事物分支到事务协调器TC， 执行完实际SQL后，上报分支事务状态给事务协调器TC。
配置中心存储各种配置,比如Seata Client端(TM,RM),Seata Server(TC)的全局事务开关,事务会话存储模式等信息。注册中心维护服务和服务地址的映射关系。比如Seata Client端(TM,RM),发现Seata Server(TC)集群的地址,彼此通信。框架的底层基于Netty通信，通过编码器，解码器，编解码请求与响应。相应的消息交由TM的TMClient，RMClient及TC Server的处理器去处理。


# 全局事务流程

![/seata-at-transaction](/image/seata/seata-at-transaction.png)

**begin**
1. TM开启一个全局事务（->TC），TC生成一个全局事务（事务上下文[RootContext]）；
2. TM 执行实际的业务开始；
3. RM加入全局事务（Dubbo Filter RPCContext），注册事物分支, TC保存事务分支及事务行锁（包括需要加锁的行主键Key）；
4. RM执行本地事务（事务执行器TransactionalExecutor、连接代理ConnectionProxy）；并存储事务undo log（关联行数据的前后镜像）到本地重做日志表，并上报分支事务状态到TC；
5. 待所有分支事务执行完（TM实际业务结束）；
6. 如果在所有RM的事务分支，执行过程中没有异常，走7， 否则走13；
**commit**
7. TM发送提交全局事务请求到TC；
8. TC发送分支事务提交请求到所有的RM事务分支；
9. RM删除Undo log；
10. RM事务分支提交成功，TC从全局事务中、删除事务分支，同时从分支事务表中删除；
11. 待所有RM处理完（分支事务提交请求）， TC结束全局事务；
12. 更新全局事务状态为已提交{@link GlobalStatus#Committed}，释放全局事务相关的锁，并从会话管理器中移除全局会话；  
**rollback**
13. 异常发生，TM发起事务回滚请求到TC；
14. TC发送分支事务回滚请求到所有RM；
15. RM根据重做日志（数据前后镜像），回滚数据；
16. 待所有RM回滚完，TC结束全局；
17. 更新全局事务回滚（回滚成功Rollbacked，或回滚超时TimeoutRollbacked），释放全局事务相关的锁，并从会话管理器中移除全局会话，同时全局事务表中删除；

# 总结
Seata框架的优点主要是避免了手动处理分布式事务，只需要简单配置，注解即可解决分布式事务的ACID的问题，开发者只需要关注具体的业务实现；
同时Seata支持RCP框架Dubbo，主流配置中心和注册中心；唯一的缺点可能是性能方面会有问题，针对高性能的业务的场景，可能并不适用，多用于传统金融
级的业务；可性能的场景，我们可以使用MQ解耦，本地消息事务表保证事务的幂等性及最终一致性提高性能。

# 附
源码代码阅读入口
## SeataAutoConfiguration   

## GlobalTransactionScanner  
io.seata.spring.annotation.GlobalTransactionScanner#initClient

io.seata.spring.annotation.GlobalTransactionScanner#wrapIfNecessary



<!-- 开启全局事务 -->
### GlobalTransactional
io.seata.spring.annotation.GlobalTransactional  

### GlobalTransactionalInterceptor
io.seata.spring.annotation.GlobalTransactionalInterceptor#invoke

#### TransactionalTemplate
io.seata.tm.api.TransactionalTemplate#execute

#### DefaultGlobalTransaction
io.seata.tm.api.DefaultGlobalTransaction

<!-- 全局事务 -->

### TMClient  
io.seata.tm.TMClient#init(java.lang.String, java.lang.String, java.lang.String, java.lang.String)


#### TmNettyRemotingClient
io.seata.core.rpc.netty.AbstractNettyRemoting#processMessage
### ClientOnResponseProcessor  
io.seata.core.rpc.processor.client.ClientOnResponseProcessor#process

###  NettyClientBootstrap
io.seata.core.rpc.netty.NettyClientBootstrap#start

#### ProtocolV1Decoder

#### ProtocolV1Encoder
#### ClientHandler
io.seata.core.rpc.netty.AbstractNettyRemotingClient.ClientHandler


### RMClient


io.seata.rm.RMClient#init
####  RmNettyRemotingClient
io.seata.core.rpc.netty.RmNettyRemotingClient#init

io.seata.core.rpc.netty.AbstractNettyRemoting#processMessage
##### DefaultResourceManager

* BranchType.AT， DataSourceManager
* TCC,TCCResourceManager
* SAGA,SagaResourceManager
* XA, ResourceManagerXA

SeataAutoDataSourceProxyCreator
SeataAutoDataSourceProxyAdvice， SeataDataSourceProxy

#### DataSourceProxy

SeataDataSourceBeanPostProcessor

io.seata.rm.datasource.DataSourceProxy#init

io.seata.spring.annotation.datasource.DataSourceProxyHolder#putDataSource

io.seata.spring.annotation.datasource.SeataDataSourceBeanPostProcessor#postProcessAfterInitialization

##### ConnectionProxy

<!-- 基于DUBBO的AT服务，主要根据RootContext的XID和事务分支类型，决定是否处于全局事务中，及执行的事务类型 -->
ApacheDubboTransactionPropagationFilter
org.apache.dubbo.rpc.RpcContext

io.seata.rm.datasource.ConnectionProxy#commit  
io.seata.rm.datasource.ConnectionProxy#processGlobalTransactionCommit  

io.seata.rm.datasource.ConnectionProxy#register  

io.seata.rm.datasource.ConnectionProxy#rollback  
io.seata.rm.datasource.ConnectionProxy#setAutoCommit  



<!-- 构建undolog -->
io.seata.rm.datasource.exec.BaseTransactionalExecutor#prepareUndoLog

io.seata.rm.datasource.ConnectionProxy#appendUndoLog
io.seata.rm.datasource.ConnectionProxy#appendLockKey


**AbstractConnectionProxy**
io.seata.rm.datasource.AbstractConnectionProxy#prepareStatement(java.lang.String, int, int)

**PreparedStatementProxy**
io.seata.rm.datasource.PreparedStatementProxy#executeUpdate

**ExecuteTemplate**
io.seata.rm.datasource.exec.ExecuteTemplate#execute(io.seata.rm.datasource.StatementProxy<S>, io.seata.rm.datasource.exec.StatementCallback<T,S>, java.lang.Object...)

##### BaseTransactionalExecutor


<!-- 绑定全局事务id， 执行DML操作 -->
io.seata.rm.datasource.exec.BaseTransactionalExecutor#execute
io.seata.rm.datasource.exec.AbstractDMLBaseExecutor#doExecute

* MySQLInsertExecutor
* UpdateExecutor
* DeleteExecutor
* SelectForUpdateExecutor
* PlainExecutor
* MultiExecutor

**DruidSQLRecognizerFactoryImpl** 
io.seata.sqlparser.druid.DruidSQLRecognizerFactoryImpl#create

**MySQLOperateRecognizerHolder**
* MySQLDeleteRecognizer
* MySQLInsertRecognizer
* MySQLUpdateRecognizer
* MySQLSelectForUpdateRecognizer
##### DefaultRMHandler
io.seata.rm.DefaultRMHandler#initRMHandlers

* AT:RMHandlerAT
* TCC:RMHandlerTCC
* SAGA:RMHandlerSaga
* XA:RMHandlerXA

##### RMHandlerAT
io.seata.rm.RMHandlerAT#handle

<!-- 提交消息，回滚消息处理；委托给ConnectionProxy#setAutoCommit/ -->

io.seata.rm.AbstractRMHandler#doBranchCommit
io.seata.rm.AbstractRMHandler#doBranchRollback
##### RmBranchCommitProcessor
io.seata.rm.datasource.DataSourceManager#branchCommit 
#### RmBranchRollbackProcessor

io.seata.rm.datasource.undo.AbstractUndoLogManager#undo
#### RmUndoLogProcessor

####  ClientOnResponseProcessor

# Server(TC)


```sh
exec "$JAVACMD" $JAVA_OPTS -server -Xmx2048m -Xms2048m -Xmn1024m -Xss512k -XX:SurvivorRatio=10 -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=1024m -XX:-OmitStackTraceInFastThrow -XX:-UseAdaptiveSizePolicy -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath="$BASEDIR"/logs/java_heapdump.hprof -XX:+DisableExplicitGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=75 -Xloggc:"$BASEDIR"/logs/seata_gc.log -verbose:gc -Dio.netty.leakDetectionLevel=advanced -Dlogback.color.disable-for-bat=true \
  -classpath "$CLASSPATH" \
  -Dapp.name="seata-server" \
  -Dapp.pid="$$" \
  -Dapp.repo="$REPO" \
  -Dapp.home="$BASEDIR" \
  -Dbasedir="$BASEDIR" \
  io.seata.server.Server \
  "$@"
```

```cmd
seata-server.bat -p 8091 -h 127.0.0.1 -m db
```

## Server
io.seata.server.Server

### SessionHolder
io.seata.server.session.SessionHolder#init

会话管理器
DataBaseSessionManager
FileSessionManager
RedisSessionManager

 锁

DataBaseLockManager
FileLockManager
RedisLockManager
#### GlobalSession

#### BranchSession

RowLock
### DefaultCoordinator

io.seata.server.coordinator.DefaultCoordinator#onRequest
#### DefaultCore

ATCore  
XACore  
TccCore  
SagaCore  

<!-- 核心 -->

处理所有事务的请求，开启，提交，回滚，注册

### NettyRemotingServer

io.seata.core.rpc.netty.NettyRemotingServer#init

ServerOnRequestProcessor  
ServerOnResponseProcessor  
RegRmProcessor  
RegTmProcessor  
ServerHeartbeatProcessor  

## spring-cloud-starter-alibaba-seata
