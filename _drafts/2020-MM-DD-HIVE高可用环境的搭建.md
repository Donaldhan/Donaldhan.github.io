---
layout: page
title: HIVE高可用环境的搭建
subtitle: HIVE高可用环境的搭建
date: 2020-11-04 15:17:19
author: donaldhan
catalog: true
category: BigData
categories:
    - BigData
tags:
    - Hive
---

# 引言




# 目录
* [](#)
    * [](#)
    * [](#)
* [总结](#总结)




Apache Hive TM:https://cloud.tencent.com/developer/article/1561886
HIVE快速入门教程2Hive架构:https://www.jianshu.com/p/eeb65dcfcc6a
Hive的使用:https://www.jianshu.com/p/7bf9a390d7e6
Hive教程:https://www.yiibai.com/hive/
HIVE --- Metastore:http://www.pianshen.com/article/8317243978/
Hive Metastore的故事:https://zhuanlan.zhihu.com/p/100585524
Hive MetaStore的结构:https://www.jianshu.com/p/420ddb3bde7f
Hive Metastore原理及配置:https://blog.csdn.net/qq_40990732/article/details/80914873
Hive为什么要启用Metastore:https://blog.csdn.net/qq_35440040/article/details/82462269

Hive HA使用说明及Hive使用HAProxy配置HA(高可用)：
https://www.aboutyun.com/thread-10938-1-1.html

HAProxy用法详解 全网最详细中文文档：
http://www.ttlsa.com/linux/haproxy-study-tutorial/

HiveMetaStore高可用性(HA)配置:https://blog.csdn.net/rotkang/article/details/78683626

一个失败，连接另外一个

HiveServer2 高可用配置：https://www.jianshu.com/p/3dfa4b4e7ce0

Dynamic HA Provider Configuration:https://cwiki.apache.org/confluence/display/KNOX/Dynamic+HA+Provider+Configuration

knox:http://knox.apache.org/

# WebHDFS
    Gateway: https://{gateway-host}:{gateway-port}/{gateway-path}/{cluster-name}/webhdfs
    Cluster: http://{webhdfs-host}:50070/webhdfs

# WebHCat (Templeton)
    Gateway: https://{gateway-host}:{gateway-port}/{gateway-path}/{cluster-name}/templeton
    Cluster: http://{webhcat-host}:50111/templeton}
# Oozie
    Gateway: https://{gateway-host}:{gateway-port}/{gateway-path}/{cluster-name}/oozie
    Cluster: http://{oozie-host}:11000/oozie}
# HBase
    Gateway: https://{gateway-host}:{gateway-port}/{gateway-path}/{cluster-name}/hbase
    Cluster: http://{hbase-host}:8080
# Hive JDBC
    Gateway: jdbc:hive2://{gateway-host}:{gateway-port}/;ssl=true;sslTrustStore={gateway-trust-store-path};trustStorePassword={gateway-trust-store-password};transportMode=http;httpPath={gateway-path}/{cluster-name}/hive
    Cluster: http://{hive-host}:10001/cliservice
    

构建高可用Hive HA和整合HBase开发环境:https://blog.csdn.net/pysense/article/details/102987186

hive 3.1.1 高可用集群搭建（与zookeeper集成）搭建笔记：https://blog.csdn.net/liuhuabing760596103/article/details/89175063




###



###


## 总结
