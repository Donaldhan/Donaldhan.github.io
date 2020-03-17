---
layout: page
title: Sqoop简介
subtitle: Sqoop简介
date: 2020-11-04 15:17:19
author: donaldhan
catalog: true
category: BigData
    - BigData
tags:
    - Sqoop
---

# 引言

在业务系统中，我们将业务数据存储在关系型数据库。在业务数据累计一定量的时候，如何对这些数据进行分析，是我们需要解决的一个问题。sqoop作为连接关系型数据库和hadoop的桥梁，支持全量和增量更新的方式，将数据导入到Hadoop的BigTable体系中，比如如 Hive和HBase；以便进行分析。


# 目录
* [ Sqoop的优点](#sqoop的优点)
* [Sqoop1](#sqoop1)
    * [](#)
* [Sqoop2](#sqoop2)
* [总结](#总结)


# Sqoop的优点

1. 可以高效、可控的利用资源，可以通过调整任务数来控制任务的并发度。
2. 可以自动的完成数据映射和转换。由于导入数据库是有类型的，它可以自动根据数据库中的类型转换到Hadoop 中，当然用户也可以自定义它们之间的映射关系
3. 支持多种数据库，如mysql，orcale等数据库
4. 插拔式Connector架构， Connector是与特定数据源相关的组件， 主要负责(从特定数据源中)抽取和加载数据。用户可选择Sqoop自带的Connector， 或者数据库提供的native Connector。



**sqoop工作的机制**

将导入或导出命令翻译成MapReduce程序来实现在翻译出的，性能高；可以MapReduce 中主要是对InputFormat和OutputFormat进行定制.

sqoop的有sqoop1和sqoop2是两个不同的版本，它们是完全不兼容的，apache1.4.X之后的版本是1, 1.99.0之上的版本是2。

我们先来看sqoop1
# Sqoop1


![sqoop1_framwork](/image/sqoop/sqoop1_framwork.webp)


Sqoop1客户端工具 不需要启动任何服务，直接拉起MapReuce作业(实际只有Map操作), 使用方便， 只有命令行交互方式。

缺陷：
* 仅支持JDBC的Connector
* 要求依赖软件必须安装在客户端上(包括Mysql/Hadoop/Oracle客户端， JDBC驱动，数据库厂商提供的Connector等)。
* 安全性差： 需要用户提供明文密码

下面我们来体验一下数据传输。当前使用的版本为sqoop-1.4.7.tar.gz, hadoop为2.7.1， hive环境搭建见下文。

[HIVE单机环境搭建](https://donaldhan.github.io/bigdata/2020/02/25/HIVE%E5%8D%95%E6%9C%BA%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA.html)

## sqoop安装配置
下载sqoop-1.4.7， 并解压
```
donaldhan@pseduoDisHadoop:/bdp/sqoop$ ls
sqoop-1.4.7  sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz  sqoop-1.4.7.tar.gz
donaldhan@pseduoDisHadoop:/bdp/sqoop$ cd sqoop-1.4.7/
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7$ pwd
/bdp/sqoop/sqoop-1.4.7
donaldhan@pseduoDisHadoop:/
```

配置Sqoop环境变量

```
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7$ cd conf/
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7/conf$ ls
oraoop-site-template.xml  sqoop-env-template.sh
sqoop-env-template.cmd    sqoop-site-template.xml
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7/conf$ cp sqoop-env-template.sh  sqoop-env.sh
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7/conf$ vim sqoop-env.sh 
```

具体配置如下：
```
# included in all the hadoop scripts with source command
# should not be executable directly
# also should not be passed any arguments, since we need original $*

# Set Hadoop-specific environment variables here.

#Set path to where bin/hadoop is available
export HADOOP_COMMON_HOME=/bdp/hadoop/hadoop-2.7.1

#Set path to where hadoop-*-core.jar is available
export HADOOP_MAPRED_HOME=/bdp/hadoop/hadoop-2.7.1

#set the path to where bin/hbase is available
#export HBASE_HOME=

#Set the path to where bin/hive is available
export HIVE_HOME=/bdp/hive/apache-hive-2.3.4-bin

#Set the path for where zookeper config dir is
#export ZOOCFGDIR=
```

我们主要测试mysql到HIVE， 由于Sqoop作业用的是MR，所以需要配置Hadoop的路径。

将mysql的驱动包放到sqoop的lib目录下
```
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7$ cp /bdp/hive/mysql-connector-java-5.1.41.jar lib/
```

添加Sqoop路径到系统路径
```
export SQOOP_HOME=/bdp/sqoop/sqoop-1.4.7
export PATH=${SQOOP_HOME}/bin:${JAVA_HOME}/bin:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:${ZOOKEEPER_HOME}/bin:${HBASE_HOME}/bin:${HIVE_HOME}/bin:${FLUME_HOME}/bin:${PATH}

```
## Sqoop的使用

1. 查看数据库的名称：

sqoop list-databases --connect jdbc:mysql://ip:3306/ --username 用户名--password 密码

sqoop list-databases --connect jdbc:mysql://mysqlDb:3306/ --username root --password 123456

2. 列举出数据库中的表名：

sqoop list-tables --connect jdbc:mysql://ip:3306/数据库名称 --username 用户名 --password 密码


# Sqoop2
![sqoop2_framwork](/image/sqoop/sqoop2_framwork.webp)

###


## 总结


# 附
## 参考文献
[sqoop](http://sqoop.apache.org/)   
[sqoop github](https://github.com/apache/sqoop)  
[真正了解sqoop的一切](https://www.jianshu.com/p/ec9003d8918c)   
[Sqoop简介](https://www.jianshu.com/p/4250382abbc6)   
[Sqoop1.4.7UserGuide](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html)   
[]()   


## 问题集

### Error: Could not find or load main class org.apache.sqoop.Sqoop
```
Error: Could not find or load main class org.apache.sqoop.Sqoop
```

[Sqoop找不到主类 Error: Could not find or load main class org.apache.sqoop.Sqoop](https://www.cnblogs.com/hxsyl/p/6552701.html)    
[运行sqoop 报 Could not find or load main class org.apache.sqoop.Sqoop](http://blog.chinaunix.net/uid-22948773-id-3563685.html)