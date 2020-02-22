---
layout: page
title: HIVE高可用环境的搭建
subtitle: HIVE高可用环境的搭建
date: 2020-02-22 20:57:00
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

```
donaldhan@nameNode:/bdp$ scp -r ./hive/ donaldhan@resourceManager:/bdp/
haproxy-1.7.9.tar.gz                                                                                                                                                      100% 1707KB   1.7MB/s   00:00    
mysql-connector-java-5.1.41.jar                                                                                                                                           100%  970KB 969.5KB/s   00:00    
mysql-server_5.7.11-1ubuntu15.10_amd64.deb-bundle.tar                                                                                                                     100%  180MB  25.7MB/s   00:07    
apache-hive-2.3.4-bin.tar.gz                                                                                                                                              100%  221MB  14.8MB/s   00:15    
donaldhan@nameNode:/bdp$ 
```


```
donaldhan@secondlyNamenode:/bdp/hive$ tar -xvf apache-hive-2.3.4-bin.tar.gz 
donaldhan@secondlyNamenode:/bdp/hive$ ls
apache-hive-2.3.4-bin  apache-hive-2.3.4-bin.tar.gz  haproxy-1.7.9.tar.gz  mysql-connector-java-5.1.41.jar  mysql-server_5.7.11-1ubuntu15.10_amd64.deb-bundle.tar
```

```
donaldhan@nameNode:/bdp/hive$ cp mysql-connector-java-5.1.41.jar  apache-hive-2.3.4-bin/lib/
donaldhan@nameNode:/bdp/hive$ ls -al apache-hive-2.3.4-bin/lib | grep mysql
-rw-rw-r--  1 donaldhan donaldhan   992805 Feb 13 22:00 mysql-connector-java-5.1.41.jar
-rw-r--r--  1 donaldhan donaldhan     7954 Oct 25  2018 mysql-metadata-storage-0.9.2.jar
donaldhan@nameNode:/bdp/hive$ 
```

```
donaldhan@nameNode:/bdp/hive$ tar -xvf mysql-server_5.7.11-1ubuntu15.10_amd64.deb-bundle.tar -C  mysql-server-5.7.11/
mysql-community-server_5.7.11-1ubuntu15.10_amd64.deb
libmysqlclient-dev_5.7.11-1ubuntu15.10_amd64.deb
libmysqld-dev_5.7.11-1ubuntu15.10_amd64.deb
mysql-client_5.7.11-1ubuntu15.10_amd64.deb
mysql-server_5.7.11-1ubuntu15.10_amd64.deb
mysql-community-client_5.7.11-1ubuntu15.10_amd64.deb
mysql-common_5.7.11-1ubuntu15.10_amd64.deb
mysql-community-test_5.7.11-1ubuntu15.10_amd64.deb
mysql-community-source_5.7.11-1ubuntu15.10_amd64.deb
mysql-community_5.7.11-1ubuntu15.10_amd64.changes
libmysqlclient20_5.7.11-1ubuntu15.10_amd64.deb
mysql-testsuite_5.7.11-1ubuntu15.10_amd64.deb
donaldhan@nameNode:/bdp/hive$ ls
apache-hive-2.3.4-bin  apache-hive-2.3.4-bin.tar.gz  haproxy-1.7.9.tar.gz  mysql-connector-java-5.1.41.jar  mysql-server-5.7.11  mysql-server_5.7.11-1ubuntu15.10_amd64.deb-bundle.tar
donaldhan@nameNode:/bdp/hive$ cd mysql-server-5.7.11/
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ ls
libmysqlclient20_5.7.11-1ubuntu15.10_amd64.deb    mysql-common_5.7.11-1ubuntu15.10_amd64.deb            mysql-community-source_5.7.11-1ubuntu15.10_amd64.deb
libmysqlclient-dev_5.7.11-1ubuntu15.10_amd64.deb  mysql-community_5.7.11-1ubuntu15.10_amd64.changes     mysql-community-test_5.7.11-1ubuntu15.10_amd64.deb
libmysqld-dev_5.7.11-1ubuntu15.10_amd64.deb       mysql-community-client_5.7.11-1ubuntu15.10_amd64.deb  mysql-server_5.7.11-1ubuntu15.10_amd64.deb
mysql-client_5.7.11-1ubuntu15.10_amd64.deb        mysql-community-server_5.7.11-1ubuntu15.10_amd64.deb  mysql-testsuite_5.7.11-1ubuntu15.10_amd64.deb
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ 

```

```
sudo dpkg -i {libmysqlclient20,libmysqlclient-dev,libmysqld-dev}_*.deb
sudo dpkg -i mysql-{common,community-client,client,community-server,server}_*.deb
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ dpkg -l | grep mysql
ii  libmysqlclient-dev                            5.7.11-1ubuntu15.10                        amd64        MySQL development headers
ii  libmysqlclient20:amd64                        5.7.11-1ubuntu15.10                        amd64        MySQL shared client libraries
ii  libmysqld-dev                                 5.7.11-1ubuntu15.10                        amd64        MySQL embedded server library
ii  mysql-client                                  5.7.11-1ubuntu15.10                        amd64        MySQL Client meta package depending on latest version
ii  mysql-common                                  5.7.11-1ubuntu15.10                        amd64        MySQL configuration for client and server
ii  mysql-community-client                        5.7.11-1ubuntu15.10                        amd64        MySQL Client and client tools
rc  mysql-community-server                        5.7.11-1ubuntu
```



```
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ cat /etc/hosts
127.0.0.1	localhost
192.168.5.135  nameNode
192.168.5.136 secondlyNameNode
192.168.5.137 resourceManager
192.168.5.135 ns
192.168.3.106 mysqldb

```

# 配置全局HIVE HOME路径
```
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ tail -n -6 ~/.bashrc 
export HADOOP_COMMON_LIB_NATIVE_DIR=${HADOOP}/lib/native
export YARN_HOME=${HADOOP_HOME}
export HADOOP_OPT="-Djava.library.path=${HADOOP_HOME}/lib/native"
export HIVE_HOME=/bdp/hive/apache-hive-2.3.4-bin
export PATH=${JAVA_HOME}/bin:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:${ZOOKEEPER_HOME}/bin:${HBASE_HOME}/bin:${HIVE_HOME}/bin:${PATH}

donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ 

```

# 配置hive环境变量
```
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ cp hive-env.sh.template hive-env.sh
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ vim hive-env.sh
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ tail -n 5 hive-env.sh
# export HIVE_AUX_JARS_PATH=
HADOOP_HOME=/bdp/hadoop/hadoop-2.7.1
HIVE_CONF_DIR=/bdp/hive/apache-hive-2.3.4-bin/conf
HIVE_AUX_JARS_PATH=/bdp/hive/apache-hive-2.3.4-bin/lib
```

# 配置hive-site
```
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ cp hive-default.xml.template hive-site.xml
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ vim hive-site.xml 
```
具体如下：
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?><!--
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
--><configuration>
  <!-- WARNING!!! This file is auto generated for documentation purposes ONLY! -->
  <!-- WARNING!!! Any changes you make to this file will be ignored by Hive.   -->
  <!-- WARNING!!! You must make your changes in hive-site.xml instead.         -->
  <!-- Hive Execution Parameters -->
i <property>
    <name>hive.exec.scratchdir</name>
    <value>/user/hive/tmp</value>
    <description>HDFS root scratch dir for Hive jobs which gets created with write all (733) permission. For each connecting user, an HDFS scratch dir: ${hive.exec.scratchdir}/&lt;username&gt; is created, with ${hive.scratch.dir.permission}.</description>
  </property>
 <property>
    <name>hive.exec.local.scratchdir</name>
    <value>/bdp/hive/jobslog/${system:user.name}</value>
    <description>Local scratch space for Hive jobs</description>
  </property>
  <property>
    <name>hive.downloaded.resources.dir</name>
    <value>/bdp/hive/jobslog/${hive.session.id}_resources</value>
    <description>Temporary local directory for added resources in the remote file system.</description>
  </property>
<!-- warehouse config -->
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
    <description>location of default database for the warehouse</description>
  </property>

  <!-- metastore config -->
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://mysqldb:3306/hive_db?createDatabaseIfNotExist=true</value>
    <description>
      JDBC connect string for a JDBC metastore.
      To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
      For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
    </description>
  </property>
<property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
    <description>Username to use against metastore database</description>
  </property>
 <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>123456</value>
    <description>password to use against metastore database</description>
  </property>
  <!-- HiveServer2 HA config -->
<property>
    <name>hive.zookeeper.quorum</name>
    <value>
      nameNode:2181,secondlyNameNode:2181,resourceManager:2181
    </value>
    <description>
      List of ZooKeeper servers to talk to. This is needed for:
      1. Read/write locks - when hive.lock.manager is set to
      org.apache.hadoop.hive.ql.lockmgr.zookeeper.ZooKeeperHiveLockManager,
      2. When HiveServer2 supports service discovery via Zookeeper.
      3. For delegation token storage if zookeeper store is used, if
      hive.cluster.delegation.token.store.zookeeper.connectString is not set
      4. LLAP daemon registry service
      5. Leader selection for privilege synchronizer
    </description>
  </property>
<property>
    <name>hive.server2.support.dynamic.service.discovery</name>
    <value>true</value>
    <description>Whether HiveServer2 supports dynamic service discovery for its clients. To support this, each instance of HiveServer2 currently uses ZooKeeper to register itself, when it is brought up. JDBC/ODBC clients should use the ZooKeeper ensemble: hive.zookeeper.quorum in their connection string.</description>
  </property>
  <property>
    <name>hive.server2.zookeeper.namespace</name>
    <value>hiveserver2_zk</value>
    <description>The parent node in ZooKeeper used by HiveServer2 when supporting dynamic service discovery.</description>
  </property>
  <property>
    <name>hive.server2.zookeeper.publish.configs</name>
    <value>true</value>
    <description>Whether we should publish HiveServer2's configs to ZooKeeper.</description>
</property>
</configuration>

```

# 修改日志目录
```
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ cp hive-log4j2.properties.template hive-log4j2.properties
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ vim hive-log4j2.propertie

property.hive.log.dir = /bdp/hive/log


donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ cp hive-exec-log4j2.properties.template hive-e
hive-env.sh                           hive-env.sh.template                  hive-exec-log4j2.properties.template  
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ cp hive-exec-log4j2.properties.template hive-exec-log4j2.properties
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ vim hive-exec-log4j2.properties
property.hive.log.dir = /bdp/hive/exelog
```

# copy 配置文件到其他两台机
```
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ scp /home/donaldhan/.bashrc donaldhan@secondlyNameNode:/home/donaldhan/
.bashrc                                       100% 4512     4.4KB/s   00:00    
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ scp /home/donaldhan/.bashrc donaldhan@resourceManager:/home/donaldhan/
.bashrc                                       100% 4512     4.4KB/s   00:00    
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ scp /bdp/hive/apache-hive-2.3.4-bin/conf/hive-env.sh donaldhan@secondlyNameNode:/bdp/hive/apache-hive-2.3.4-bin/conf/
hive-env.sh                                   100% 2509     2.5KB/s   00:00    
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ scp /bdp/hive/apache-hive-2.3.4-bin/conf/hive-env.sh donaldhan@resourceManager:/bdp/hive/apache-hive-2.3.4-bin/conf/
hive-env.sh                                   100% 2509     2.5KB/s   00:00    
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ scp /bdp/hive/apache-hive-2.3.4-bin/conf/hive-site.xml  donaldhan@resourceManager:/bdp/hive/apache-hive-2.3.4-bin/conf/
hive-site.xml                                 100% 4824     4.7KB/s   00:00    
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ scp /bdp/hive/apache-hive-2.3.4-bin/conf/hive-site.xml  donaldhan@secondlyNameNode:/bdp/hive/apache-hive-2.3.4-bin/conf/
hive-site.xml                                 100% 4824     4.7KB/s   00:00    
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ scp /bdp/hive/apache-hive-2.3.4-bin/conf/hive-log4j2.properties  donaldhan@secondlyNameNode:/bdp/hive/apache-hive-2.3.4-bin/conf/
hive-log4j2.properties                        100% 2900     2.8KB/s   00:00    
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ scp /bdp/hive/apache-hive-2.3.4-bin/conf/hive-log4j2.properties  donaldhan@resourceManager:/bdp/hive/apache-hive-2.3.4-bin/conf/
hive-log4j2.properties                        100% 2900     2.8KB/s   00:00    
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ scp /bdp/hive/apache-hive-2.3.4-bin/conf/hive-exec-log4j2.properties   donaldhan@resourceManager:/bdp/hive/apache-hive-2.3.4-bin/conf/
hive-exec-log4j2.properties                   100% 2252     2.2KB/s   00:00    
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ scp /bdp/hive/apache-hive-2.3.4-bin/conf/hive-exec-log4j2.properties   donaldhan@secondlyNameNode:/bdp/hive/apache-hive-2.3.4-bin/conf/
hive-exec-log4j2.properties                   100% 2252     2.2KB/s   00:00    
donaldhan@nameNode:/bdp/hive/apache-hive-2.3.4-bin/conf$ 

```

hdfs-site.xml  文件配置
```xml
<property>
<name>dfs.webhdfs.enabled</name>
<value>true</value>
</property>
```

core-site.xml 配置 这里配置很重要  还有要注意这里的name页签中的root 这里是hdfs登录的具体的用户名，写错了就会报错，访问hive的时候会报错

```xml
<property>
     <name>hadoop.proxyuser.root.hosts</name>
     <value>*</value>
   </property>
   <property>
    <name>hadoop.proxyuser.root.groups</name>
    <value>*</value>
</property>
```


启动HIVE
首先在分布式环境中启动zk
zkServer.sh start
启动hadoop集群
start-dfs.sh



```
donaldhan@nameNode:/bdp/hadoop/hadoop-2.7.1/etc$ jps
3650 NameNode
4051 JournalNode
4389 Jps
3800 DataNode
3433 QuorumPeerMain
4271 DFSZKFailoverController
```
创建HIVE数仓目录和job中间目录
```
donaldhan@nameNode:/bdp/hadoop/hadoop-2.7.1/etc$ hdfs dfs -mkdir -p /user/hive/warehouse
donaldhan@nameNode:/bdp/hadoop/hadoop-2.7.1/etc$ hdfs dfs -mkdir -p /user/hive/tmp
donaldhan@nameNode:/bdp/hadoop/hadoop-2.7.1/etc$ hdfs dfs -chmod 777 /user/hive/tmp 
donaldhan@nameNode:/bdp/hadoop/hadoop-2.7.1/etc$ hdfs dfs -chmod 777 /user/hive/warehouse
donaldhan@nameNode:/bdp/hadoop/hadoop-2.7.1/etc$ hdfs dfs -ls /user/hive/Found 2 items
drwxrwxrwx   - donaldhan supergroup          0 2020-02-22 22:59 /user/hive/tmp
drwxrwxrwx   - donaldhan supergroup          0 2020-02-22 22:59 /user/hive/warehouse
donaldhan@nameNode:/bdp/hadoop/hadoop-2.7.1/etc$ 
```

```
donaldhan@nameNode:/bdp/hadoop/hadoop-2.7.1/etc$ jps
3650 NameNode
4051 JournalNode
4708 RunJar
4888 Jps
3800 DataNode
3433 QuorumPeerMain
4271 DFSZKFailoverController
donaldhan@nameNode:/bdp/hadoop/hadoop-2.7.1/etc$ 
```


操作指令

      1.后台启动服务. 在hive节点上启动即可

          nohup hiveserver2 -hiveconf hive.root.logger=DEBUG,console  1> hive.log 2>&1 &

     2.客户端访问  belline 敲入以下指令登录

       !connect jdbc:hive2://hdp04:2181,hdp05:2181,hdp06:2181,hdp07:2181,hdp08:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2_zk root  "root"




HIVE AdministratorDocumentation config：<https://cwiki.apache.org/confluence/display/Hive/Home#Home-AdministratorDocumentation>

Apache Hive TM:h<ttps://cloud.tencent.com/developer/article/1561886>  
HIVE快速入门教程2Hive架构:<https://www.jianshu.com/p/eeb65dcfcc6a>   
Hive的使用:<https://www.jianshu.com/p/7bf9a390d7e6>   
Hive教程:<https://www.yiibai.com/hive/>   
HIVE Metastore:<http://www.pianshen.com/article/8317243978/>  
Hive Metastore的故事:<https://zhuanlan.zhihu.com/p/100585524>    
Hive MetaStore的结构:<https://www.jianshu.com/p/420ddb3bde7f>   
Hive Metastore原理及配置:<https://blog.csdn.net/qq_40990732/article/details/80914873>  
Hive为什么要启用Metastore:<https://blog.csdn.net/qq_35440040/article/details/82462269> 

Hive HA使用说明及Hive使用HAProxy配置HA(高可用)：
<https://www.aboutyun.com/thread-10938-1-1.html>  

HAProxy用法详解 全网最详细中文文档：
<http://www.ttlsa.com/linux/haproxy-study-tutorial/>   

HiveMetaStore高可用性(HA)配置:<https://blog.csdn.net/rotkang/article/details/78683626>  
一个失败，连接另外一个

HiveServer2 高可用配置：<https://www.jianshu.com/p/3dfa4b4e7ce0>   

构建高可用Hive HA和整合HBase开发环境:<https://blog.csdn.net/pysense/article/details/102987186>

hive 3.1.1 高可用集群搭建（与zookeeper集成）搭建笔记：<https://blog.csdn.net/liuhuabing760596103/article/details/89175063>

Hive集群部署:<https://www.alongparty.cn/hive-cluster-deployment.html> 

Centos7.6+Hadoop 3.1.2(HA)+Zookeeper3.4.13+Hbase1.4.9(HA)+Hive2.3.4+Spark2.4.0(HA)高可用集群搭建:<https://mshk.top/2019/03/centos-hadoop-zookeeper-hbase-hive-spark-high-availability/>   


Dynamic HA Provider Configuration:<https://cwiki.apache.org/confluence/display/KNOX/Dynamic+HA+Provider+Configuration>   

knox:<http://knox.apache.org/>  

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
    




###



###


## 总结


# 附

## 安装mysql缺少so包
```
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ sudo dpkg -i  mysql-community-server_5.7.11-1ubuntu15.10_amd64.deb
(Reading database ... 160808 files and directories currently installed.)
Preparing to unpack mysql-community-server_5.7.11-1ubuntu15.10_amd64.deb ...
.
Unpacking mysql-community-server (5.7.11-1ubuntu15.10) over (5.7.11-1ubuntu15.10) ...
dpkg: dependency problems prevent configuration of mysql-community-server:
 mysql-community-server depends on mysql-client (= 5.7.11-1ubuntu15.10); however:
  Package mysql-client is not installed.
 mysql-community-server depends on libaio1 (>= 0.3.93); however:
  Package libaio1 is not installed.
 mysql-community-server depends on libmecab2v5 (>= 0.996-1.1ubuntu1); however:
  Package libmecab2v5 is not installed.

dpkg: error processing package mysql-community-server (--install):
 dependency problems - leaving unconfigured
Processing triggers for ureadahead (0.100.0-19) ...
Processing triggers for systemd (225-1ubuntu9) ...
Processing triggers for man-db (2.7.4-1) ...
Errors were encountered while processing:
 mysql-community-server
```

```
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ apt-get install libaio1
E: Could not open lock file /var/lib/dpkg/lock - open (13: Permission denied)
E: Unable to lock the administration directory (/var/lib/dpkg/), are you root?
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ ^C
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ sudo apt-get install ^C
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ sudo apt-get install libmecab2v5
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Package libmecab2v5 is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source

E: Package 'libmecab2v5' has no installation candidate
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ sudo apt-get update 
E: Could not get lock /var/lib/apt/lists/lock - open (11: Resource temporarily unavailable)
E: Unable to lock directory /var/lib/apt/lists/
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ sudo apt-get upgrade
Reading package lists... Done
Building dependency tree       
Reading state information... Done
You might want to run 'apt-get -f install' to correct these.
The following packages have unmet dependencies:
 mysql-community-client : Depends: libaio1 (>= 0.3.93) but it is not installed
 mysql-community-server : Depends: libaio1 (>= 0.3.93) but it is not installed
                          Depends: libmecab2v5 (>= 0.996-1.1ubuntu1) but it is not installable
E: Unmet dependencies. Try using -f.
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ sudo apt-get upgrade -f 

```
https://bugs.mysql.com/bug.php?id=79798

https://www.fatalerrors.org/a/mysql-community-server-depends-on-libmecab2-however-libmecab2-is-not-installed.html

```
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ sudo apt-get -f install
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Correcting dependencies... Done
The following package was automatically installed and is no longer required:
  libdbusmenu-gtk4
Use 'apt-get autoremove' to remove it.
The following packages will be REMOVED:
  mysql-community-server mysql-server
0 upgraded, 0 newly installed, 2 to remove and 10 not upgraded.
2 not fully installed or removed.
After this operation, 136 MB disk space will be freed.
Do you want to continue? [Y/n] y
(Reading database ... 160904 files and directories currently installed.)
Removing mysql-server (5.7.11-1ubuntu15.10) ...
Removing mysql-community-server (5.7.11-1ubuntu15.10) ...
Processing triggers for man-db (2.7.4-1) ...
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ sudo dpkg -l|grep mysql
ii  libmysqlclient-dev                            5.7.11-1ubuntu15.10                        amd64        MySQL development headers
ii  libmysqlclient20:amd64                        5.7.11-1ubuntu15.10                        amd64        MySQL shared client libraries
ii  libmysqld-dev                                 5.7.11-1ubuntu15.10                        amd64        MySQL embedded server library
ii  mysql-client                                  5.7.11-1ubuntu15.10                        amd64        MySQL Client meta package depending on latest version
ii  mysql-common                                  5.7.11-1ubuntu15.10                        amd64        MySQL configuration for client and server
ii  mysql-community-client                        5.7.11-1ubuntu15.10                        amd64        MySQL Client and client tools
rc  mysql-community-server                        5.7.11-1ubuntu15.10                        amd64        MySQL Server and server tools
donaldhan@nameNode:/bdp/hive/mysql-server-5.7.11$ 


sudo rm /var/lib/mysql/ -R
sudo rm /etc/mysql/ -R
sudo apt-get autoremove mysql* --purge
sudo apt-get remove apparmor # select Yes in this step
sudo apt-get install mysql-server mysql-common # Reenter password

```


https://serverfault.com/questions/752063/how-can-i-install-mysql-5-7-9-to-ubuntu-14-04

package=mysql-apt-config_0.8.11-1_all.deb
wget http://dev.mysql.com/get/$package
sudo dpkg -i $package
sudo apt-get update
sudo apt-get install mysql-community-server mysql-server



package=mysql-apt-config_0.6.0-1_all.deb

 http://dev.mysql.com/get/mysql-apt-config_0.6.0-1_all.deb


 lock file
 rm lock file

 ```
 donaldhan@nameNode:/bdp/hive$ sudo apt-get install libmecab2
Reading package lists... Done
Building dependency tree       
Reading state information... Done
libmecab2 is already the newest version.
0 upgraded, 0 newly installed, 0 to remove and 32 not upgraded.
donaldhan@nameNode:/bdp/hive$ 

 ```



 ```
  sudo apt-get install software-properties-common
$ sudo add-apt-repository -y ppa:ondrej/mysql-5.7
$ sudo apt-get update
$ sudo apt-get install mysql-server
 ```


 #  Couldn't create directory /bdb/hive/local/jobslog/
 ```
 2020-02-22T23:07:31,704  WARN [main] server.HiveServer2: Error starting HiveServer2 on attempt 5, will retry in 60000ms
java.lang.RuntimeException: Error applying authorization policy on hive configuration: Couldn't create directory /bdb/hive/local/jobslog/46fb3bec-497a-4029-8c10-13e409c15214_resources
        at org.apache.hive.service.cli.CLIService.init(CLIService.java:117) ~[hive-service-2.3.4.jar:2.3.4]
        at org.apache.hive.service.CompositeService.init(CompositeService.java:59) ~[hive-service-2.3.4.jar:2.3.4]
        at org.apache.hive.service.server.HiveServer2.init(HiveServer2.java:142) ~[hive-service-2.3.4.jar:2.3.4]
        at org.apache.hive.service.server.HiveServer2.startHiveServer2(HiveServer2.java:607) [hive-service-2.3.4.jar:2.3.4]
        at org.apache.hive.service.server.HiveServer2.access$700(HiveServer2.java:100) [hive-service-2.3.4.jar:2.3.4]
        at org.apache.hive.service.server.HiveServer2$StartOptionExecutor.execute(HiveServer2.java:855) [hive-service-2.3.4.jar:2.3.4]
        at org.apache.hive.service.server.HiveServer2.main(HiveServer2.java:724) [hive-service-2.3.4.jar:2.3.4]
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_191]
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_191]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_191]
        at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_191]
        at org.apache.hadoop.util.RunJar.run(RunJar.java:221) [hadoop-common-2.7.1.jar:?]
        at org.apache.hadoop.util.RunJar.main(RunJar.java:136) [hadoop-common-2.7.1.jar:?]
Caused by: java.lang.RuntimeException: Couldn't create directory /bdb/hive/local/jobslog/46fb3bec-497a-4029-8c10-13e409c15214_resources
        at org.apache.hadoop.hive.ql.util.ResourceDownloader.ensureDirectory(ResourceDownloader.java:116) ~[hive-exec-2.3.4.jar:2.3.4]
        at org.apache.hadoop.hive.ql.util.ResourceDownloader.<init>(ResourceDownloader.java:47) ~[hive-exec-2.3.4.jar:2.3.4]
        at org.apache.hadoop.hive.ql.session.SessionState.<init>(SessionState.java:397) ~[hive-exec-2.3.4.jar:2.3.4]
        at org.apache.hadoop.hive.ql.session.SessionState.<init>(SessionState.java:370) ~[hive-exec-2.3.4.jar:2.3.4]
        at org.apache.hive.service.cli.CLIService.applyAuthorizationConfigPolicy(CLIService.java:127) ~[hive-service-2.3.4.jar:2.3.4]
        at org.apache.hive.service.cli.CLIService.init(CLIService.java:114) ~[hive-service-2.3.4.jar:2.3.4]
```

# 
```
2020-02-22T23:19:57,488  WARN [main] metastore.MetaStoreDirectSql: Self-test query [select "DB_ID" from "DBS"] failed; direct SQL is disabled
javax.jdo.JDODataStoreException: Error executing SQL query "select "DB_ID" from "DBS"".
	at org.datanucleus.api.jdo.NucleusJDOHelper.getJDOExceptionForNucleusException(NucleusJDOHelper.java:543) ~[datanucleus-api-jdo-4.2.4.jar:?]
	at org.datanucleus.api.jdo.JDOQuery.executeInternal(JDOQuery.java:391) ~[datanucleus-api-jdo-4.2.4.jar:?]
	at org.datanucleus.api.jdo.JDOQuery.execute(JDOQuery.java:216) ~[datanucleus-api-jdo-4.2.4.jar:?]
```