---
layout: page
title: HIVE客户端
subtitle: HIVE客户端
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

由于我们创建数据库时没有指定对应的数仓存储路径，默认为HDFS下的数仓目录user/hive/warehouse+数据库名+.db对应的文件夹。
如果数据库中有0或多个表时，不能直接删除，需要先删除表再删除数据库；如果想要删除含有表的数据库，在删除时加上cascade，可以级联删除（慎用）。
Hive表有两种，分别是内部表与外部表，如果是内部表，在删除时，MySQL中的元数据和HDFS中的数据都会被删除；
如果是外部表，在删除时，MySQL中的元数据会被删除，HDFS中的数据不会被删除；
加载文件数据到Hive表有两种方式，一种是从本地文件加载，一种从hdfs文件加载。如果加上LOCAL表示从本地加载数据，
默认不加，从hdfs中加载数据，添加OVERWRITE关键字，将会覆盖表中数据，及先删除，在加载。
从hdfs方式加载完数据，需要注意hdfs上的文件将会被删除，移动hdfs的垃圾箱中。
插入数据实际为一个MR任务。聚合类的操作（max，min，avg，count），都需要运行MR任务。
导出数据与加载数据LOCAL和OVERWRITE使用基本一致。如果加上LOCAL表示导出到本地默认不加，导出到hdfs，如果加OVERWRITE关键字，将会覆盖原文件中的数据，及先删除，在导出。
从本地加载文件到分区表时，实际上是，将本地文件放到hdfs上的数据库分区表文件夹（order_partition）下的分区字段+分区Value（event_month=2020-02）文价夹。
单级分区和多级分区唯一的区别就是多级分区在hdfs中的目录为多级。


# 目录
* [](#)
    * [](#)
    * [](#)
* [总结](#总结)


从搜索的结果来看，hive没有对应的client API，在大数据的场景中，应该没有这种java直接访问hive，要结合spark、今天我们只做一个简单的测试；



###

启动hive

hive --service metastore &

 

启动 hiveserver2 开启此服务后才能通过java api调用

 访问http://192.168.1.10:10002   进入hiveweb页面

###


## 总结


# 附
## 引用文献
使用hive客户端java api读写hive集群上的信息:<https://www.bbsmax.com/A/1O5EBAy7d7/>  
hdinsight-java-hive-jdbc:<https://github.com/Azure-Samples/hdinsight-java-hive-jdbc>  
HiveMetaStoreClient:<https://github.com/Re1tReddy/HiveMetaStoreClient>  