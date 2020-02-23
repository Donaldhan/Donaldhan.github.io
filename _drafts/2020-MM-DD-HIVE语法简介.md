---
layout: page
title: my blog
subtitle: sub title
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

上一篇，我们使用Zookeeper在3台机上搭建了Hive的HA高可用环境，同时使用HIVE CLI和Beeline, 体验了一些简单的DDL和DML。
在配置的过程中metastore，要先初始化；另外hive.server2.thrift.bind.host配置，不同的机器，绑定的主机名为相应的主机，HiveServer2就是单点模式了。最重要的注意hive.server2.thrift.client.user配置用户，要与hadoop的core-site.xml中的代理用户名要一致。


单机版，创建表，从本地文件加载数据，从hdfs加载数据。



# 目录
* [](#)
    * [](#)
    * [](#)
* [总结](#总结)


Hive DDL DML及SQL操作:<https://blog.csdn.net/HG_Harvey/article/details/77488314>  
LanguageManual DDL:<https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL>  
LanguageManual DML:<https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DML>


###



###


## 总结
