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

下载sqoop-1.4.7.jar，avro-1.8.1.jar放到sqoop的lib文件加下
```
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7$ ls lib/
ant-contrib-1.0b3.jar       mysql-connector-java-5.1.41.jar
ant-eclipse-1.0-jvm1.2.jar  sqoop-1.4.7.jar
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.
```



添加Sqoop路径到系统路径
```
export SQOOP_HOME=/bdp/sqoop/sqoop-1.4.7
export PATH=${SQOOP_HOME}/bin:${JAVA_HOME}/bin:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:${ZOOKEEPER_HOME}/bin:${HBASE_HOME}/bin:${HIVE_HOME}/bin:${FLUME_HOME}/bin:${PATH}

```

使用以下命令
```
sqoop version
```

查看是否安装成功
```
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7$ sqoop version
...
20/03/18 22:33:24 INFO sqoop.Sqoop: Running Sqoop version: 1.4.7
Sqoop 1.4.7
git commit id 04c9c16335b71d44fdc7d4f1deabe6f575620b94
Compiled by ec2-user on Tue Jun 12 19:50:32 UTC 2018

```
## Sqoop的使用

### 查看数据库的名称：
```
sqoop list-databases --connect jdbc:mysql://ip:3306/ --username 用户名--password 密码
```

```
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7$ sqoop list-databases --connect jdbc:mysql://192.168.3.107:3306/ --username root --password 123456
...
20/03/18 23:01:49 INFO manager.MySQLManager: Preparing to use a MySQL streaming resultset.
information_schema
hive_db
mysql
performance_schema
single_hive_db
test
```

### 列举出数据库中的表名：
```
sqoop list-tables --connect jdbc:mysql://ip:3306/数据库名称 --username 用户名 --password 密码
```

```
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7$ sqoop list-tables --connect jdbc:mysql://192.168.3.107:3306/test --username root --password 123456
...
20/03/18 23:06:09 INFO sqoop.Sqoop: Running Sqoop version: 1.4.7
20/03/18 23:06:09 WARN tool.BaseSqoopTool: Setting your password on the command-line is insecure. Consider using -P instead.
20/03/18 23:06:09 INFO manager.MySQLManager: Preparing to use a MySQL streaming resultset.
blogs
comments
users
donaldhan@pseduoDisHadoop:/bdp/sqoop/sqoop-1.4.7$ 
```



### 导入数据
从关系数据导出数据到HDFS，HIVE，或HBASE；



具体命令解释如下



```
sqoop import  
--connect jdbc:mysql://ip:3306/databasename  #指定JDBC的URL 其中database指的是(Mysql或者Oracle)中的数据库名
--table  tablename  #要读取数据库database中的表名           
--username root      #用户名 
--password  123456  #密码    
--target-dir   /path  #指的是HDFS中导入表的存放目录(注意：是目录)
--fields-terminated-by '\t'   #设定导入数据后每个字段的分隔符，默认；分隔
--lines-terminated-by '\n'    #设定导入数据后每行的分隔符
--m,--num-mappers 1  #并发的map数量1,如果不设置默认启动4个map task执行数据导入，则需要指定一个列来作为划分map task任务的依据
-- where ’查询条件‘   #导入查询出来的内容，表的子集
-e，--query	‘取数sql’
--incremental  append  #增量导入
--check-column：column_id   #指定增量导入时的参考列
--last-value：num   #上一次导入column_id的最后一个值
--null-string ‘’   #导入的字段为空时，用指定的字符进行替换
```


准备数据

我们在mysql的test库中创建如下表
```
CREATE TABLE `books` (
  `id` int(11) NOT NULL,
  `book_name` varchar(64) DEFAULT NULL,
  `book_price` decimal(10,0) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```
插入如下数据
```
INSERT INTO `test`.`books` (`id`, `book_name`, `book_price`) VALUES ('1', '贫穷的本质', '39');
INSERT INTO `test`.`books` (`id`, `book_name`, `book_price`) VALUES ('2', '聪明的投资者', '42');
```

```
mysql> select * from books;
+----+--------------+------------+
| id | book_name    | book_price |
+----+--------------+------------+
|  1 | 贫穷的本质   | 39         |
|  2 | 聪明的投资者 | 42         |
+----+--------------+------------+
2 rows in set

mysql> 
```

启动hadoop
```
start-dfs.sh 
```
启动yarn
```
start-yarn.sh 
```

创建hdfs目录，用户存放导出的数据
```
donaldhan@pseduoDisHadoop:~$ hdfs dfs -mkdir /user/sqoop
donaldhan@pseduoDisHadoop:~$ hdfs dfs -ls /user
Found 3 items
drwxr-xr-x   - donaldhan supergroup          0 2019-03-24 22:08 /user/donaldhan
drwxr-xr-x   - donaldhan supergroup          0 2020-02-27 22:07 /user/hive
drwxr-xr-x   - donaldhan supergroup          0 2020-03-19 22:40 /user/sqoop

```

```
sqoop import --connect  jdbc:mysql://192.168.3.107:3306/test --username 'root' --password '123456' --table books --target-dir /user/sqoop/books2 --fields-terminated-by '\t' --m 1
```

查看导入的数据文件
```
donaldhan@pseduoDisHadoop:~$ hdfs dfs -ls /user/sqoop/books
Found 3 items
-rw-r--r--   1 donaldhan supergroup          0 2020-03-19 23:24 /user/sqoop/books/_SUCCESS
-rw-r--r--   1 donaldhan supergroup         21 2020-03-19 23:24 /user/sqoop/books/part-m-00000
-rw-r--r--   1 donaldhan supergroup         24 2020-03-19 23:24 /user/sqoop/books/part-m-00001
donaldhan@pseduoDisHadoop:~$ hdfs dfs -cat /user/sqoop/books/*01
2	聪明的投资者	42
donaldhan@pseduoDisHadoop:~$ hdfs dfs -cat /user/sqoop/books/*00
1	贫穷的本质	39
donaldhan@pseduoDisHadoop:~$ 
donaldhan@pseduoDisHadoop:~$ hdfs dfs -cat /user/sqoop/books/*0*
1	贫穷的本质	39
2	聪明的投资者	42
```


从输出日志来看每次只抓取一条记录


```
20/03/19 23:34:25 INFO manager.SqlManager: Executing SQL statement: SELECT t.* FROM `books` AS t LIMIT 1
20/03/19 23:34:25 INFO manager.SqlManager: Executing SQL statement: SELECT t.* FROM `books` AS t LIMIT 1
```


我们可以通过--fetch-size参数每次控制抓取大小

```
--fetch-size  2
```

执行的过程中，我们发现是无效的，因为被Mysql的驱动忽略

```
20/03/19 23:34:24 INFO manager.MySQLManager: Argument '--fetch-size 2' will probably get ignored by MySQL JDBC driver.
```

是否可以使用--m控制map任务数量，进而将所有的数据集中到一个文件中
```
--m 1
```


```
sqoop import --connect  jdbc:mysql://192.168.3.107:3306/test --username 'root' --password '123456' --table books --target-dir /user/sqoop/books2 --fields-terminated-by '\t' --m 1
```

查看hdfs文件

```
donaldhan@pseduoDisHadoop:~$ hdfs dfs -ls /user/sqoop/books2
Found 2 items
-rw-r--r--   1 donaldhan supergroup          0 2020-03-19 23:39 /user/sqoop/books2/_SUCCESS
-rw-r--r--   1 donaldhan supergroup         45 2020-03-19 23:39 /user/sqoop/books2/part-m-00000
donaldhan@pseduoDisHadoop:~$ hdfs dfs -cat /user/sqoop/books2/*0
1	贫穷的本质	39
2	聪明的投资者	42
```

从hdfs目录文件来看，答案是可定的。












```
sqoop import --connect  jdbc:mysql://192.168.3.107:3306/test --username 'root' --password '123456' --table books --target-dir /user/sqoop --fields-terminated-by '\t'   --lines-terminated-by '\n'  -m 1 


```
sqoop import --connect  jdbc:mysql://192.168.3.107:3306/test --username 'root' --password '123456' --table books --target-dir /user/sqoop/test --fields-terminated-by '\t'  --lines-terminated-by '\n'  --incremental  append   --check-column：id   --last-value：1   --null-string '\\N'
```





```
sqoop import  
--connect jdbc:mysql://ip:3306/databasename
--table  tablename      
--username root    
--password  123456 
--target-dir   /path  
--fields-terminated-by '\t'  
--lines-terminated-by '\n'  
--m 1  
-- where 
--incremental  append 
--check-column：column_id 
--last-value：num   
--null-string ‘’  
```



# Sqoop2
![sqoop2_framwork](/image/sqoop/sqoop2_framwork.webp)

###


## 总结
将mysql导入到hdfs，可以使用--m控制map任务数量，进而将所有的数据集中到一个文件中；

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
问题原因：缺失sqoop-1.4.7.jar包，下载放到lib即可。

[Sqoop找不到主类 Error: Could not find or load main class org.apache.sqoop.Sqoop](https://www.cnblogs.com/hxsyl/p/6552701.html)    
[运行sqoop 报 Could not find or load main class org.apache.sqoop.Sqoop](http://blog.chinaunix.net/uid-22948773-id-3563685.html)


### Caused by: java.lang.ClassNotFoundException: org.apache.avro.
```
Caused by: java.lang.ClassNotFoundException: org.apache.avro.LogicalType
	at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:349)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	... 10 more

```

问题原因：缺失avro-1.8.1.jar包，下载放到lib即可。



### Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/commons/lang3/StringUtils

缺失commons-lang3-3.8.1.jar，下载放到lib即可。
