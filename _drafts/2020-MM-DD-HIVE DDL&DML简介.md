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
    * [创建表](#创建表)
        * [加载本地文件数据到数据表](#加载本地文件数据到数据表)
        * [复制表](#复制表)
    * [修改表](#修改表)
    * [删除表](#删除表)

* [DML](#dml)
    * [内部表和外部表](#内部表和外部表)
    * [表数据加载load](#表数据加载load)
    * [插入数据](#插入数据)
    * [查询数据](#查询数据)
    * [导出数据](#导出数据)
    * [分区表](#分区表)
    
* [复杂数据类型操作](#复杂数据类型操作)
    * [Array](#Array)
    * [Map](#Map)
    * [Struct](#Struct)
    * [Array](#Array)


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

### 加载本地文件数据到数据表

编辑emp数据text文本，以制表符为字段分割附
```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ vim emp.txt 
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ cat emp.txt 
123	donald	lawer	4568	2019-02-26	8000.0	3000.0	7896
456	rain	teacher	1314	2018-01-25	5000.1	2000.2	4654
789	jamel	cleaner		20170609	3000.0	1000.3	7895
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ 

```

**注意此文件内容不能简单拷贝，简单拷贝会出现，加载数据为空的情况， 使用vim的时候，要使用 *set list* 把制表符显示为^I ,用$标示行尾（使用list分辨尾部的字符是tab还是空格), 两个tab之间为空，则字段为Null**



从本地文件加载数据到数据表
```
0: jdbc:hive2://pseduoDisHadoop:10000>  load data local inpath '/bdp/hive/hiveLocalTables/emp.txt' overwrite into table emp;
No rows affected (4.38 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp;
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| 123        | donald     | lawer    | 4568     | 2019-02-26    | 8000.0   | 3000.0    | 7896        |
| 456        | rain       | teacher  | 1314     | 2018-01-25    | 5000.1   | 2000.2    | 4654        |
| 789        | jamel      | cleaner  | NULL     | 20170609      | 3000.0   | 1000.3    | 7895        |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
3 rows selected (4.226 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```

### 复制表

* 只拷贝表结构
```
0: jdbc:hive2://pseduoDisHadoop:10000> create table emp2 like emp;
No rows affected (1.005 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> desc emp2;
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
8 rows selected (0.4 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp2;
+-------------+-------------+-----------+-----------+----------------+-----------+------------+--------------+
| emp2.empno  | emp2.ename  | emp2.job  | emp2.mgr  | emp2.hiredate  | emp2.sal  | emp2.comm  | emp2.deptno  |
+-------------+-------------+-----------+-----------+----------------+-----------+------------+--------------+
+-------------+-------------+-----------+-----------+----------------+-----------+------------+--------------+
No rows selected (0.419 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

* 拷贝表结构及数据
会运行MapReduce作业
```
0: jdbc:hive2://pseduoDisHadoop:10000> create table emp3 as select * from emp;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (357.932 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp3;
+-------------+-------------+-----------+-----------+----------------+-----------+------------+--------------+
| emp3.empno  | emp3.ename  | emp3.job  | emp3.mgr  | emp3.hiredate  | emp3.sal  | emp3.comm  | emp3.deptno  |
+-------------+-------------+-----------+-----------+----------------+-----------+------------+--------------+
| 123         | donald      | lawer     | 4568      | 2019-02-26     | 8000.0    | 3000.0     | 7896         |
| 456         | rain        | teacher   | 1314      | 2018-01-25     | 5000.1    | 2000.2     | 4654         |
| 789         | jamel       | cleaner   | NULL      | 20170609       | 3000.0    | 1000.3     | 7895         |
+-------------+-------------+-----------+-----------+----------------+-----------+------------+--------------+
3 rows selected (0.522 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

**注意由于要运行MapReduce作业，需要开启hadoop的yarn；**

```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ start-yarn.sh 
starting yarn daemons
starting resourcemanager, logging to /bdp/hadoop/hadoop-2.7.1/logs/yarn-donaldhan-resourcemanager-pseduoDisHadoop.out
localhost: starting nodemanager, logging to /bdp/hadoop/hadoop-2.7.1/logs/yarn-donaldhan-nodemanager-pseduoDisHadoop.out
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ jps 
2962 RunJar
3655 ResourceManager
3113 RunJar
2570 DataNode
3787 NodeManager
2428 NameNode
2765 SecondaryNameNode
4333 Jps

```

* 拷贝表的限定列
创建表emp到emp_copy，emp_copy中只包含三列：empno，ename，job
```
0: jdbc:hive2://pseduoDisHadoop:10000> create table emp4 as select empno,ename,job from emp;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (78.651 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp4;
+-------------+-------------+-----------+
| emp4.empno  | emp4.ename  | emp4.job  |
+-------------+-------------+-----------+
| 123         | donald      | lawer     |
| 456         | rain        | teacher   |
| 789         | jamel       | cleaner   |
+-------------+-------------+-----------+
3 rows selected (0.332 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```

此种拷贝方式也会执行MR任务，注意在HIVE 2中，HIVE-on-MR已经丢弃。在将来的版本中将不可用。可以考虑使用其他执行引擎或者使用HIVE 1.x版本。

## 查询表

### 查看数据库下的所有表
```
0: jdbc:hive2://pseduoDisHadoop:10000> show tables;
+-----------+
| tab_name  |
+-----------+
| emp       |
| emp2      |
| emp3      |
| emp4      |
+-----------+
4 rows selected (0.282 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> show tables 'emp*';
+-----------+
| tab_name  |
+-----------+
| emp       |
| emp2      |
| emp3      |
| emp4      |
+-----------+
4 rows selected (0.24 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```

### 查看表结构
```
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
8 rows selected (0.297 seconds)
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
10 rows selected (0.448 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```

*desc extended *方式，相比*desc*可以查看表的更详细信息，比如存储位置，所属db，ower，创建时间，上次访问时间等。

### 查看表的创建语句
```
0: jdbc:hive2://pseduoDisHadoop:10000> show create table emp;
+----------------------------------------------------+
|                   createtab_stmt                   |
+----------------------------------------------------+
| CREATE TABLE `emp`(                                |
|   `empno` int,                                     |
|   `ename` string,                                  |
|   `job` string,                                    |
|   `mgr` int,                                       |
|   `hiredate` string,                               |
|   `sal` double,                                    |
|   `comm` double,                                   |
|   `deptno` int)                                    |
| ROW FORMAT SERDE                                   |
|   'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'  |
| WITH SERDEPROPERTIES (                             |
|   'field.delim'='\t',                              |
|   'serialization.format'='\t')                     |
| STORED AS INPUTFORMAT                              |
|   'org.apache.hadoop.mapred.TextInputFormat'       |
| OUTPUTFORMAT                                       |
|   'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat' |
| LOCATION                                           |
|   'hdfs://pseduoDisHadoop:9000/user/hive/warehouse/test.db/emp' |
| TBLPROPERTIES (                                    |
|   'transient_lastDdlTime'='1582724633')            |
+----------------------------------------------------+
22 rows selected (8.453 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```


## 修改表


修改表名emp2为emp_bak
```
0: jdbc:hive2://pseduoDisHadoop:10000> alter table emp4 rename to emp_bak;
No rows affected (0.704 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> show tables;
+-----------+
| tab_name  |
+-----------+
| emp       |
| emp2      |
| emp3      |
| emp_bak   |
+-----------+
4 rows selected (0.245 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from e
else        end         end-exec    escape      except      exception   
exec        execute     exists      external    extract     
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp_bak;
+----------------+----------------+--------------+
| emp_bak.empno  | emp_bak.ename  | emp_bak.job  |
+----------------+----------------+--------------+
| 123            | donald         | lawer        |
| 456            | rain           | teacher      |
| 789            | jamel          | cleaner      |
+----------------+----------------+--------------+
3 rows selected (0.327 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```


## 删除表

删除表emp_bak

```
0: jdbc:hive2://pseduoDisHadoop:10000> drop table if exists emp_bak;
No rows affected (4.845 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> show tables;
+-----------+
| tab_name  |
+-----------+
| emp       |
| emp2      |
| emp3      |
+-----------+
3 rows selected (0.23 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```
### 清空表中所有数据
```
0: jdbc:hive2://pseduoDisHadoop:10000> truncate table emp3;
No rows affected (0.521 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp3;
+-------------+-------------+-----------+-----------+----------------+-----------+------------+--------------+
| emp3.empno  | emp3.ename  | emp3.job  | emp3.mgr  | emp3.hiredate  | emp3.sal  | emp3.comm  | emp3.deptno  |
+-------------+-------------+-----------+-----------+----------------+-----------+------------+--------------+
+-------------+-------------+-----------+-----------+----------------+-----------+------------+--------------+
No rows selected (0.246 seconds)
```

# DML
这部分主要是对表的操作.

## 内部表和外部表
HIVE提供了两种模式的表，我们分别来看一下。
### 内部表
创建一张内部表 emp_managed
```
0: jdbc:hive2://pseduoDisHadoop:10000> create table emp_managed as select * from emp;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (65.511 seconds)

``

查看表emp_managed在hdfs中是否存在(/user/hive/warehouse/test.db/emp_managed)
```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -ls /user/hive/warehouse/test.db
Found 4 items
drwxrwxrwx   - donaldhan supergroup          0 2020-02-26 21:43 /user/hive/warehouse/test.db/emp
drwxrwxrwx   - donaldhan supergroup          0 2020-02-26 21:50 /user/hive/warehouse/test.db/emp2
drwxrwxrwx   - donaldhan supergroup          0 2020-02-26 22:50 /user/hive/warehouse/test.db/emp3
drwxrwxrwx   - donaldhan supergroup          0 2020-02-26 23:09 /user/hive/warehouse/test.db/emp_managed
donaldhan@pseduoDisHadoop
```
存在

登录mysql（TBLS中存放了hive中的所有表）数据库，查看hive的元数据信息
```
mysql> use single_hive_db;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from tbls;
+--------+-------------+-------+------------------+-----------+-----------+-------+-------------+---------------+--------------------+--------------------+--------------------+
| TBL_ID | CREATE_TIME | DB_ID | LAST_ACCESS_TIME | OWNER     | RETENTION | SD_ID | TBL_NAME    | TBL_TYPE      | VIEW_EXPANDED_TEXT | VIEW_ORIGINAL_TEXT | IS_REWRITE_ENABLED |
+--------+-------------+-------+------------------+-----------+-----------+-------+-------------+---------------+--------------------+--------------------+--------------------+
|      3 |  1582640961 |     6 |                0 | donaldhan |         0 |     3 | emp         | MANAGED_TABLE | NULL               | NULL               |                    |
|      6 |  1582725013 |     6 |                0 | donaldhan |         0 |     6 | emp2        | MANAGED_TABLE | NULL               | NULL               |                    |
|      7 |  1582725780 |     6 |                0 | donaldhan |         0 |     7 | emp3        | MANAGED_TABLE | NULL               | NULL               |                    |
|      9 |  1582729774 |     6 |                0 | donaldhan |         0 |     9 | emp_managed | MANAGED_TABLE | NULL               | NULL               |                    |
+--------+-------------+-------+------------------+-----------+-----------+-------+-------------+---------------+--------------------+--------------------+--------------------+
4 rows in set (0.00 sec)

mysql> 

``
emp_managed表元数据存在

删除表emp_managed
```
0: jdbc:hive2://pseduoDisHadoop:10000> drop table if exists emp_managed;
No rows affected (0.875 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

``

mysql 中hive的元数据被删除
```
mysql> mysql> select * from tbls;
+--------+-------------+-------+------------------+-----------+-----------+-------+----------+---------------+--------------------+--------------------+--------------------+
| TBL_ID | CREATE_TIME | DB_ID | LAST_ACCESS_TIME | OWNER     | RETENTION | SD_ID | TBL_NAME | TBL_TYPE      | VIEW_EXPANDED_TEXT | VIEW_ORIGINAL_TEXT | IS_REWRITE_ENABLED |
+--------+-------------+-------+------------------+-----------+-----------+-------+----------+---------------+--------------------+--------------------+--------------------+
|      3 |  1582640961 |     6 |                0 | donaldhan |         0 |     3 | emp      | MANAGED_TABLE | NULL               | NULL               |                    |
|      6 |  1582725013 |     6 |                0 | donaldhan |         0 |     6 | emp2     | MANAGED_TABLE | NULL               | NULL               |                    |
|      7 |  1582725780 |     6 |                0 | donaldhan |         0 |     7 | emp3     | MANAGED_TABLE | NULL               | NULL               |                    |
+--------+-------------+-------+------------------+-----------+-----------+-------+----------+---------------+--------------------+--------------------+--------------------+
3 rows in set (0.01 sec)

mysql> 

``

表emp_managed 对应hdfs 中的文件夹被删除
```
ehouse/test.db
Found 3 items
drwxrwxrwx   - donaldhan supergroup          0 2020-02-26 21:43 /user/hive/warehouse/test.db/emp
drwxrwxrwx   - donaldhan supergroup          0 2020-02-26 21:50 /user/hive/warehouse/test.db/emp2
drwxrwxrwx   - donaldhan supergroup          0 2020-02-26 22:50 /user/hive/warehouse/test.db/emp3
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ 

``


### 外部表

创建一张外部表
```
0: jdbc:hive2://pseduoDisHadoop:10000> create external table `emp_external`(
. . . . . . . . . . . . . . . . . . .> empno int,
. . . . . . . . . . . . . . . . . . .> ename string,
. . . . . . . . . . . . . . . . . . .> job string,
. . . . . . . . . . . . . . . . . . .> mgr int,
. . . . . . . . . . . . . . . . . . .> hiredate string,
. . . . . . . . . . . . . . . . . . .> sal double,
. . . . . . . . . . . . . . . . . . .> comm double,
. . . . . . . . . . . . . . . . . . .> deptno int
. . . . . . . . . . . . . . . . . . .> )
. . . . . . . . . . . . . . . . . . .> row format delimited fields terminated by '\t'
. . . . . . . . . . . . . . . . . . .> location '/user/hive/warehouse/external/emp';
No rows affected (1.055 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 


``
查看mysql中hive的元数据
```
mysql> select * from tbls;
+--------+-------------+-------+------------------+-----------+-----------+-------+--------------+----------------+--------------------+--------------------+--------------------+
| TBL_ID | CREATE_TIME | DB_ID | LAST_ACCESS_TIME | OWNER     | RETENTION | SD_ID | TBL_NAME     | TBL_TYPE       | VIEW_EXPANDED_TEXT | VIEW_ORIGINAL_TEXT | IS_REWRITE_ENABLED |
+--------+-------------+-------+------------------+-----------+-----------+-------+--------------+----------------+--------------------+--------------------+--------------------+
|      3 |  1582640961 |     6 |                0 | donaldhan |         0 |     3 | emp          | MANAGED_TABLE  | NULL               | NULL               |                    |
|      6 |  1582725013 |     6 |                0 | donaldhan |         0 |     6 | emp2         | MANAGED_TABLE  | NULL               | NULL               |                    |
|      7 |  1582725780 |     6 |                0 | donaldhan |         0 |     7 | emp3         | MANAGED_TABLE  | NULL               | NULL               |                    |
|     10 |  1582730557 |     6 |                0 | donaldhan |         0 |    10 | emp_external | EXTERNAL_TABLE | NULL               | NULL               |                    |
+--------+-------------+-------+------------------+-----------+-----------+-------+--------------+----------------+--------------------+--------------------+--------------------+
4 rows in set (0.00 sec)

mysql> 

``

外部表类型为：EXTERNAL_TABLE。

新创建的表emp_external中是没有数据的，我们将emp.txt文件上传到hdfs的/user/hive/warehouse/external/emp目录下
```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs  -put /bdp/hive/hiveLocalTables/emp.txt /user/hive/warehouse/external/emp
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -ls /user/hive/warehouse/external/emp
Found 1 items
-rw-r--r--   1 donaldhan supergroup        151 2020-02-26 23:29 /user/hive/warehouse/external/emp/emp.txt

``

上传完成后，表emp_external就有数据了，使用sql查看
```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp_external;
+---------------------+---------------------+-------------------+-------------------+------------------------+-------------------+--------------------+----------------------+
| emp_external.empno  | emp_external.ename  | emp_external.job  | emp_external.mgr  | emp_external.hiredate  | emp_external.sal  | emp_external.comm  | emp_external.deptno  |
+---------------------+---------------------+-------------------+-------------------+------------------------+-------------------+--------------------+----------------------+
| 123                 | donald              | lawer             | 4568              | 2019-02-26             | 8000.0            | 3000.0             | 7896                 |
| 456                 | rain                | teacher           | 1314              | 2018-01-25             | 5000.1            | 2000.2             | 4654                 |
| 789                 | jamel               | cleaner           | NULL              | 20170609               | 3000.0            | 1000.3             | 7895                 |
+---------------------+---------------------+-------------------+-------------------+------------------------+-------------------+--------------------+----------------------+
3 rows selected (0.45 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

``
然后我们来删除这张表，它是一张外部表，注意和内部表有什么区别

hive 中 表emp_external被删除
```
0: jdbc:hive2://pseduoDisHadoop:10000> show tables;
+-----------+
| tab_name  |
+-----------+
| emp       |
| emp2      |
| emp3      |
+-----------+
3 rows selected (0.22 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000>
``
mysql 中 元数据被删除
```
mysql> select * from tbls;
+--------+-------------+-------+------------------+-----------+-----------+-------+----------+---------------+--------------------+--------------------+--------------------+
| TBL_ID | CREATE_TIME | DB_ID | LAST_ACCESS_TIME | OWNER     | RETENTION | SD_ID | TBL_NAME | TBL_TYPE      | VIEW_EXPANDED_TEXT | VIEW_ORIGINAL_TEXT | IS_REWRITE_ENABLED |
+--------+-------------+-------+------------------+-----------+-----------+-------+----------+---------------+--------------------+--------------------+--------------------+
|      3 |  1582640961 |     6 |                0 | donaldhan |         0 |     3 | emp      | MANAGED_TABLE | NULL               | NULL               |                    |
|      6 |  1582725013 |     6 |                0 | donaldhan |         0 |     6 | emp2     | MANAGED_TABLE | NULL               | NULL               |                    |
|      7 |  1582725780 |     6 |                0 | donaldhan |         0 |     7 | emp3     | MANAGED_TABLE | NULL               | NULL               |                    |
+--------+-------------+-------+------------------+-----------+-----------+-------+----------+---------------+--------------------+--------------------+--------------------+
3 rows in set (0.01 sec)

``
hdfs 中的文件并不会被删除
```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -ls /user/hive/warehouse/external/emp
Found 1 items
-rw-r--r--   1 donaldhan supergroup        151 2020-02-26 23:29 /user/hive/warehouse/external/emp/emp.txt

``

**小节**
内部表与外部表的区别 
如果是内部表，在删除时，MySQL中的元数据和HDFS中的数据都会被删除 
如果是外部表，在删除时，MySQL中的元数据会被删除，HDFS中的数据不会被删除

## 表数据加载load

## 插入数据

## 查询数据

## 导出数据


## 分区表
    
# 复杂数据类型操作
## Array
## Map
## Struct
## Array

DML（Table） 操作
```
0: jdbc:hive2://pseduoDisHadoop:10000> !quit
Closing: 0: jdbc:hive2://pseduoDisHadoop:10000

```


### 
```
``

## 总结
由于我们创建数据库时没有指定对应的数仓存储路径，默认为HDFS下的数仓目录user/hive/warehouse+数据库名+.db对应的文件夹。

如果数据库中有0或多个表时，不能直接删除，需要先删除表再删除数据库；如果想要删除含有表的数据库，在删除时加上cascade，可以级联删除（慎用）。


总结：内部表与外部表的区别 
如果是内部表，在删除时，MySQL中的元数据和HDFS中的数据都会被删除 
如果是外部表，在删除时，MySQL中的元数据会被删除，HDFS中的数据不会被删除

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

vim-set命令使用:<https://www.jianshu.com/p/97d34b62d40d>  