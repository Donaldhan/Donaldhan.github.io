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
上面一篇文章[HIVE单机环境搭建]，我们搭建了Hive的HA版本和单机版，今天我们来使用单机来看一下HIVE的相关DDL和DML语法。


[HIVE单机环境搭建]:
 "HIVE单机环境搭建"


# 目录
* [DDL](#ddl)
    * [创建数据库](#创建数据库)
    * [查看数据库](#查看数据库)
    * [修改数据库](#修改数据库)
    * [删除数据库](#删除数据库)
* [DML](#dml)
    * [创建表](#创建表)
    * [查询表](#查询表)
    * [修改表](#修改表)
    * [删除表](#删除表)

* [DML](#dml)
* [创建表](#创建表)
* [查询表](#查询表)
* [修改表](#修改表)
* [删除表](#删除表)
* [总结](#总结)

单机版，创建表，从本地文件加载数据，从hdfs加载数据。

我们使用Beeline进行我们相关语法的测试, 先开启命令行
```
donaldhan@pseduoDisHadoop:~$ beeline 
beeline> !connect jdbc:hive2://pseduoDisHadoop:10000
Connecting to jdbc:hive2://pseduoDisHadoop:10000
Enter username for jdbc:hive2://pseduoDisHadoop:10000: hadoop
Enter password for jdbc:hive2://pseduoDisHadoop:10000: ******
Connected to: Apache Hive (version 2.3.4)
Driver: Hive JDBC (version 2.3.4)
Transaction isolation: TRANSACTION_REPEATABLE_READ
```

# DDL


## 创建数据库
简单创建数据模式；
```
create database test;
```
查看对应的数据库存储位置；

```
0: jdbc:hive2://pseduoDisHadoop:10000> desc database test;
+----------+----------+----------------------------------------------------+-------------+-------------+-------------+
| db_name  | comment  |                      location                      | owner_name  | owner_type  | parameters  |
+----------+----------+----------------------------------------------------+-------------+-------------+-------------+
| test     |          | hdfs://pseduoDisHadoop:9000/user/hive/warehouse/test.db | donaldhan   | USER        |             |
+----------+----------+----------------------------------------------------+-------------+-------------+-------------+
1 row selected (0.275 seconds)
```
由于我们创建数据库时没有指定对应的数仓存储路径，默认为HDFS下的数仓目录user/hive/warehouse+数据库名+.db对应的文件夹。


指定数据库存储位置方式，创建数据库
```
0: jdbc:hive2://pseduoDisHadoop:10000> create database test1 location '/db/test1';
No rows affected (0.689 seconds)

0: jdbc:hive2://pseduoDisHadoop:10000> desc database test1;
+----------+----------+---------------------------------------+-------------+-------------+-------------+
| db_name  | comment  |               location                | owner_name  | owner_type  | parameters  |
+----------+----------+---------------------------------------+-------------+-------------+-------------+
| test1    |          | hdfs://pseduoDisHadoop:9000/db/test1  | donaldhan   | USER        |             |
+----------+----------+---------------------------------------+-------------+-------------+-------------+
1 row selected (0.278 seconds)

```

如果test数据库不存在再创建
```
0: jdbc:hive2://pseduoDisHadoop:10000> create database if not exists test1;
No rows affected (0.247 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> desc database test1
. . . . . . . . . . . . . . . . . . .> ;
+----------+----------+---------------------------------------+-------------+-------------+-------------+
| db_name  | comment  |               location                | owner_name  | owner_type  | parameters  |
+----------+----------+---------------------------------------+-------------+-------------+-------------+
| test1    |          | hdfs://pseduoDisHadoop:9000/db/test1  | donaldhan   | USER        |             |
+----------+----------+---------------------------------------+-------------+-------------+-------------+
1 row selected (0.264 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> create database if not exists test2;
No rows affected (0.424 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> desc database test2;
+----------+----------+----------------------------------------------------+-------------+-------------+-------------+
| db_name  | comment  |                      location                      | owner_name  | owner_type  | parameters  |
+----------+----------+----------------------------------------------------+-------------+-------------+-------------+
| test2    |          | hdfs://pseduoDisHadoop:9000/user/hive/warehouse/test2.db | donaldhan   | USER        |             |
+----------+----------+----------------------------------------------------+-------------+-------------+-------------+
1 row selected (0.289 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

创建数据库，并为数据库添加描述信息
```
0: jdbc:hive2://pseduoDisHadoop:10000> create database test3 comment 'my test3 db' with dbproperties ('creator'='donaldhan','date'='2020-02-25');
No rows affected (0.524 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> desc database extended test3;
+----------+--------------+----------------------------------------------------+-------------+-------------+---------------------------------------+
| db_name  |   comment    |                      location                      | owner_name  | owner_type  |              parameters               |
+----------+--------------+----------------------------------------------------+-------------+-------------+---------------------------------------+
| test3    | my test3 db  | hdfs://pseduoDisHadoop:9000/user/hive/warehouse/test3.db | donaldhan   | USER        | {date=2020-02-25, creator=donaldhan}  |
+----------+--------------+----------------------------------------------------+-------------+-------------+---------------------------------------+
1 row selected (0.271 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```


## 查看数据库

查看所有数据
```
0: jdbc:hive2://pseduoDisHadoop:10000> show databases;
+----------------+
| database_name  |
+----------------+
| default        |
| test           |
+----------------+
2 rows selected (3.638 seconds)
```

模糊查看数据库
```
0: jdbc:hive2://pseduoDisHadoop:10000> show databases like 'test*';
+----------------+
| database_name  |
+----------------+
| test           |
| test1          |
| test2          |
| test3          |
+----------------+
4 rows selected (0.225 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```


查看指定数据库

```
0: jdbc:hive2://pseduoDisHadoop:10000> desc database test3;
+----------+--------------+----------------------------------------------------+-------------+-------------+-------------+
| db_name  |   comment    |                      location                      | owner_name  | owner_type  | parameters  |
+----------+--------------+----------------------------------------------------+-------------+-------------+-------------+
| test3    | my test3 db  | hdfs://pseduoDisHadoop:9000/user/hive/warehouse/test3.db | donaldhan   | USER        |             |
+----------+--------------+----------------------------------------------------+-------------+-------------+-------------+
1 row selected (0.31 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> desc database extended test3;
+----------+--------------+----------------------------------------------------+-------------+-------------+---------------------------------------+
| db_name  |   comment    |                      location                      | owner_name  | owner_type  |              parameters               |
+----------+--------------+----------------------------------------------------+-------------+-------------+---------------------------------------+
| test3    | my test3 db  | hdfs://pseduoDisHadoop:9000/user/hive/warehouse/test3.db | donaldhan   | USER        | {date=2020-02-25, creator=donaldhan}  |
+----------+--------------+----------------------------------------------------+-------------+-------------+---------------------------------------+
1 row selected (0.271 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```

*desc database extended test* 和*desc database test* 效果一样，但在HIVE CLI下，加extended的查询的结果想详细。



## 修改数据库
```
0: jdbc:hive2://pseduoDisHadoop:10000> desc database extended test3;
+----------+--------------+----------------------------------------------------+-------------+-------------+----------------------------------------------------+
| db_name  |   comment    |                      location                      | owner_name  | owner_type  |                     parameters                     |
+----------+--------------+----------------------------------------------------+-------------+-------------+----------------------------------------------------+
| test3    | my test3 db  | hdfs://pseduoDisHadoop:9000/user/hive/warehouse/test3.db | donaldhan   | USER        | {date=2020-02-25, creator=donaldhan, modifier=rain} |
+----------+--------------+----------------------------------------------------+-------------+-------------+----------------------------------------------------+
1 row selected (0.234 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```

## 删除数据库

删除数据库test3, 有如下两种方式：

直接删除
```
drop database test3
```

存在则删除
```
drop database if exists test3
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> drop database test3;
No rows affected (0.819 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> drop database if exists test3;
No rows affected (0.043 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> show databases like 'test*';
+----------------+
| database_name  |
+----------------+
| test           |
| test1          |
| test2          |
+----------------+
3 rows selected (0.248 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> drop database test3;
Error: Error while compiling statement: FAILED: SemanticException [Error 10072]: Database does not exist: test3 (state=42000,code=10072)
```

如果数据库不存在删除，则回报：
```
Error: Error while compiling statement: FAILED: SemanticException [Error 10072]: Database does not exist: test3
```


如果数据库中有0或多个表时，不能直接删除，需要先删除表再删除数据库，否则回报如下错误
```
InvalidOperationException(message:Database test3 is not empty. One or more tables exist.)
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> use test1;
No rows affected (0.255 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> create table `user`(
. . . . . . . . . . . . . . . . . . .> id int,
. . . . . . . . . . . . . . . . . . .> name string)
. . . . . . . . . . . . . . . . . . .> row format delimited fields terminated by '\t';
No rows affected (0.808 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> drop database test1;
Error: org.apache.hive.service.cli.HiveSQLException: Error while processing statement: FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.DDLTask. InvalidOperationException(message:Database test1 is not empty. One or more tables exist.)
...
```


如果想要删除含有表的数据库，在删除时加上cascade，表示级联删除（慎用），可以使用如下命令
```
0: jdbc:hive2://pseduoDisHadoop:10000> drop database if exists test1 cascade;
No rows affected (1.36 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> show databases like 'test*';
+----------------+
| database_name  |
+----------------+
| test           |
+----------------+
1 row selected (0.201 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```




# DML
## 创建表
```
0: jdbc:hive2://pseduoDisHadoop:10000> use test;
No rows affected (0.227 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> create table `emp`(
. . . . . . . . . . . . . . . . . . .> empno int,
. . . . . . . . . . . . . . . . . . .> ename string,
. . . . . . . . . . . . . . . . . . .> job string,
. . . . . . . . . . . . . . . . . . .> mgr int,
. . . . . . . . . . . . . . . . . . .> hiredate string,
. . . . . . . . . . . . . . . . . . .> sal double,
. . . . . . . . . . . . . . . . . . .> comm double,
. . . . . . . . . . . . . . . . . . .> deptno int
. . . . . . . . . . . . . . . . . . .> )
. . . . . . . . . . . . . . . . . . .> row format delimited fields terminated by '\t';
No rows affected (1.406 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> show tables;
+-----------+
| tab_name  |
+-----------+
| emp       |
+-----------+
1 row selected (0.207 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> show tables;
+-----------+
| tab_name  |
+-----------+
| emp       |
+-----------+
1 row selected (0.207 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> desc emp;
+-----------+------------+----------+
| col_name  | data_type  | comment  |
+-----------+------------+----------+
| empno     | int        |          |
| ename     | string     |          |
| job       | string     |          |
| mgr       | int        |          |
| hiredate  | string     |          |
| sal       | double     |          |
| comm      | double     |          |
| deptno    | int        |          |
+-----------+------------+----------+
8 rows selected (0.289 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> desc extended emp;
+-----------------------------+----------------------------------------------------+-----------------+
|          col_name           |                     data_type                      |     comment     |
+-----------------------------+----------------------------------------------------+-----------------+
| empno                       | int                                                |                 |
| ename                       | string                                             |                 |
| job                         | string                                             |                 |
| mgr                         | int                                                |                 |
| hiredate                    | string                                             |                 |
| sal                         | double                                             |                 |
| comm                        | double                                             |                 |
| deptno                      | int                                                |                 |
|                             | NULL                                               | NULL            |
| Detailed Table Information  | Table(tableName:emp, dbName:test, owner:donaldhan, createTime:1582640961, lastAccessTime:0, retention:0, sd:StorageDescriptor(cols:[FieldSchema(name:empno, type:int, comment:null), FieldSchema(name:ename, type:string, comment:null), FieldSchema(name:job, type:string, comment:null), FieldSchema(name:mgr, type:int, comment:null), FieldSchema(name:hiredate, type:string, comment:null), FieldSchema(name:sal, type:double, comment:null), FieldSchema(name:comm, type:double, comment:null), FieldSchema(name:deptno, type:int, comment:null)], location:hdfs://pseduoDisHadoop:9000/user/hive/warehouse/test.db/emp, inputFormat:org.apache.hadoop.mapred.TextInputFormat, outputFormat:org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat, compressed:false, numBuckets:-1, serdeInfo:SerDeInfo(name:null, serializationLib:org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe, parameters:{serialization.format= | , field.delim=  |
+-----------------------------+----------------------------------------------------+-----------------+
10 rows selected (0.391 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

从本地文件加载数据到数据表

```
0: jdbc:hive2://pseduoDisHadoop:10000> load data local inpath '/bdp/hive/hiveLocalTables/emp.txt' overwrite into table emp;
```

## 查询表
## 修改表
## 删除表



## 总结
由于我们创建数据库时没有指定对应的数仓存储路径，默认为HDFS下的数仓目录user/hive/warehouse+数据库名+.db对应的文件夹。

如果数据库中有0或多个表时，不能直接删除，需要先删除表再删除数据库；如果想要删除含有表的数据库，在删除时加上cascade，可以级联删除（慎用）。


# 附
## 参考文献
Hive DDL DML及SQL操作:<https://blog.csdn.net/HG_Harvey/article/details/77488314>  


LanguageManual DDL:<https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL>  
LanguageManual DML:<https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DML>
HiveServer2 Clients:<https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients#HiveServer2Clients-BeelineHiveCommands>


## Error while compiling statement: FAILED: ParseException
Error: Error while compiling statement: FAILED: ParseException line 1:13 cannot recognize input near 'user' '(' 'id' in table name (state=42000,code=40000)

### 解决方式
主要是因为表明无法识别，表明要用引号`括住。

```
create table `user`(
id int,
name string)
row format delimited fields terminated by '\t';
```


## 从字段分隔符为制表符的文件加载数据，为空的情况
具体原因由于HIVE无法识别文件中的制表符，具体原因，参考如下链接：  

hive建表指定字段分隔符为制表符，之后上传文件，文件内容未被hive表正确识别问题 :<https://blog.csdn.net/u012443641/article/details/80021226>