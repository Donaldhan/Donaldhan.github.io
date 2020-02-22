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


```
```





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