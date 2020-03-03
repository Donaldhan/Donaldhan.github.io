---
layout: page
title: HIVE DDL&DML简介
subtitle: HIVE DDL&DML简介
date: 2020-03-03 23:04:00
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


[HIVE单机环境搭建]:https://donaldhan.github.io/bigdata/2020/11/04/HIVE%E5%8D%95%E6%9C%BA%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA.html  
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
    * [Array](#array)
    * [Map](#map)
    * [Struct](#struct)
    * [Array](#array)


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
create table `emp`(
empno int,
ename string,
job string,
mgr int,
hiredate string,
sal double,
comm double,
deptno int
)
row format delimited fields terminated by '\t';
```


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

```

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

```

emp_managed表元数据存在

删除表emp_managed

```
0: jdbc:hive2://pseduoDisHadoop:10000> drop table if exists emp_managed;
No rows affected (0.875 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

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

```

表emp_managed 对应hdfs 中的文件夹被删除

```
ehouse/test.db
Found 3 items
drwxrwxrwx   - donaldhan supergroup          0 2020-02-26 21:43 /user/hive/warehouse/test.db/emp
drwxrwxrwx   - donaldhan supergroup          0 2020-02-26 21:50 /user/hive/warehouse/test.db/emp2
drwxrwxrwx   - donaldhan supergroup          0 2020-02-26 22:50 /user/hive/warehouse/test.db/emp3
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ 

```


### 外部表

创建一张外部表

```
create external table `emp_external`(
 empno int,
 ename string,
 job string,
 mgr int,
 hiredate string,
 sal double,
 comm double,
 deptno int
 )
 row format delimited fields terminated by '\t'
 location '/user/hive/warehouse/external/emp';
```



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


```

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

```


外部表类型为：EXTERNAL_TABLE。

新创建的表emp_external中是没有数据的，我们将emp.txt文件上传到hdfs的/user/hive/warehouse/external/emp目录下

```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs  -put /bdp/hive/hiveLocalTables/emp.txt /user/hive/warehouse/external/emp
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -ls /user/hive/warehouse/external/emp
Found 1 items
-rw-r--r--   1 donaldhan supergroup        151 2020-02-26 23:29 /user/hive/warehouse/external/emp/emp.txt

```

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

```

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
```

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

```

hdfs 中的文件并不会被删除

```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -ls /user/hive/warehouse/external/emp
Found 1 items
-rw-r--r--   1 donaldhan supergroup        151 2020-02-26 23:29 /user/hive/warehouse/external/emp/emp.txt

```


**小节**
内部表与外部表的区别 
如果是内部表，在删除时，MySQL中的元数据和HDFS中的数据都会被删除 
如果是外部表，在删除时，MySQL中的元数据会被删除，HDFS中的数据不会被删除

## 表数据加载load

具体语法如下：

```
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)]
```

LOCAL：如果加上表示从本地加载数据;默认不加，从hdfs中加载数据.
OVERWRITE：加上表示覆盖表中数据


加载数据到文件有两种方式，一种是从本地文件加载，一种从hdfs文件加载；我们先看第一种。

### 本地文件加载数据

从本地文件emp.txt加载数据到emp表中

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
0: jdbc:hive2://pseduoDisHadoop:10000>   load data local inpath '/bdp/hive/hiveLocalTables/emp.txt' into table emp;
No rows affected (4.266 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp;
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| 123        | donald     | lawer    | 4568     | 2019-02-26    | 8000.0   | 3000.0    | 7896        |
| 456        | rain       | teacher  | 1314     | 2018-01-25    | 5000.1   | 2000.2    | 4654        |
| 789        | jamel      | cleaner  | NULL     | 20170609      | 3000.0   | 1000.3    | 7895        |
| 123        | donald     | lawer    | 4568     | 2019-02-26    | 8000.0   | 3000.0    | 7896        |
| 456        | rain       | teacher  | 1314     | 2018-01-25    | 5000.1   | 2000.2    | 4654        |
| 789        | jamel      | cleaner  | NULL     | 20170609      | 3000.0   | 1000.3    | 7895        |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
6 rows selected (4.651 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> load data local inpath '/bdp/hive/hiveLocalTables/emp.txt' overwrite into table emp;
No rows affected (4.046 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp;
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| 123        | donald     | lawer    | 4568     | 2019-02-26    | 8000.0   | 3000.0    | 7896        |
| 456        | rain       | teacher  | 1314     | 2018-01-25    | 5000.1   | 2000.2    | 4654        |
| 789        | jamel      | cleaner  | NULL     | 20170609      | 3000.0   | 1000.3    | 7895        |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
3 rows selected (0.444 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

从上面可以看出，使用overwrite方式，在加载数据前删除原有数据。再来看从hdfs文件加载。

### hdfs文件加载数据

首先将文件上传到HDFS中

```
donaldhan@pseduoDisHadoop:~$ hdfs dfs -mkdir -p /user/hive/data
donaldhan@pseduoDisHadoop:~$ hdfs dfs -put /bdp/hive/hiveLocalTables/emp.txt /user/hive/data
donaldhan@pseduoDisHadoop:~$ hdfs dfs -ls /user/hive/data
Found 1 items
-rw-r--r--   1 donaldhan supergroup        151 2020-02-27 22:07 /user/hive/data/emp.txt
donaldhan@pseduoDisHadoop:~$ 
```


加载数据到表emp中

```
0: jdbc:hive2://pseduoDisHadoop:10000> load data inpath '/user/hive/data/emp.txt' overwrite into table emp;
No rows affected (1.28 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp;
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| 123        | donald     | lawer    | 4568     | 2019-02-26    | 8000.0   | 3000.0    | 7896        |
| 456        | rain       | teacher  | 1314     | 2018-01-25    | 5000.1   | 2000.2    | 4654        |
| 789        | jamel      | cleaner  | NULL     | 20170609      | 3000.0   | 1000.3    | 7895        |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
3 rows selected (0.383 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> load data inpath '/user/hive/data/emp.txt'  into table emp;
Error: Error while compiling statement: FAILED: SemanticException Line 1:17 Invalid path ''/user/hive/data/emp.txt'': No files matching path hdfs://pseduoDisHadoop:9000/user/hive/data/emp.txt (state=42000,code=40000)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

从hdfs方式加载完数据，需要注意hdfs上的文件将会被删除，一刀hdfs的垃圾箱中。



## 插入数据

向表 emp 中插入数据

```
 insert into emp(empno,ename,job) values(1001,'TOM','MANAGER');
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (110.63 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp;
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| 1001       | TOM        | MANAGER  | NULL     | NULL          | NULL     | NULL      | NULL        |
| 123        | donald     | lawer    | 4568     | 2019-02-26    | 8000.0   | 3000.0    | 7896        |
| 456        | rain       | teacher  | 1314     | 2018-01-25    | 5000.1   | 2000.2    | 4654        |
| 789        | jamel      | cleaner  | NULL     | 20170609      | 3000.0   | 1000.3    | 7895        |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
4 rows selected (0.327 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```


插入数据实际为一个MR任务。


## 查询数据

查询部门编号为10的员工信息

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp where deptno=10;
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
No rows selected (0.884 seconds)

```

查询姓名为SMITH的员工

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp where ename='SMITH';
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
No rows selected (0.31 seconds)

```

查询员工编号小于等于7766的员工

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp where empno <= 7766;
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| 1001       | TOM        | MANAGER  | NULL     | NULL          | NULL     | NULL      | NULL        |
| 123        | donald     | lawer    | 4568     | 2019-02-26    | 8000.0   | 3000.0    | 7896        |
| 456        | rain       | teacher  | 1314     | 2018-01-25    | 5000.1   | 2000.2    | 4654        |
| 789        | jamel      | cleaner  | NULL     | 20170609      | 3000.0   | 1000.3    | 7895        |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
4 rows selected (0.287 seconds)
```

查询员工工资大于1000小于1500的员工

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp where sal between 1000 and 1500;
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
No rows selected (0.297 seconds)


```

查询前5条记录

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp limit 5;
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| 1001       | TOM        | MANAGER  | NULL     | NULL          | NULL     | NULL      | NULL        |
| 123        | donald     | lawer    | 4568     | 2019-02-26    | 8000.0   | 3000.0    | 7896        |
| 456        | rain       | teacher  | 1314     | 2018-01-25    | 5000.1   | 2000.2    | 4654        |
| 789        | jamel      | cleaner  | NULL     | 20170609      | 3000.0   | 1000.3    | 7895        |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
4 rows selected (0.315 seconds)
```

查询姓名为SCOTT或MARTIN的员工

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp where ename in ('SCOTT','MARTIN');
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
No rows selected (0.266 seconds)
```

查询有津贴的员工

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from emp where comm is not null; 
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| emp.empno  | emp.ename  | emp.job  | emp.mgr  | emp.hiredate  | emp.sal  | emp.comm  | emp.deptno  |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
| 123        | donald     | lawer    | 4568     | 2019-02-26    | 8000.0   | 3000.0    | 7896        |
| 456        | rain       | teacher  | 1314     | 2018-01-25    | 5000.1   | 2000.2    | 4654        |
| 789        | jamel      | cleaner  | NULL     | 20170609      | 3000.0   | 1000.3    | 7895        |
+------------+------------+----------+----------+---------------+----------+-----------+-------------+
3 rows selected (0.357 seconds)
```

统计部门10下共有多少员工

```
0: jdbc:hive2://pseduoDisHadoop:10000> select count(*) from emp where deptno=10; 
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
+------+
| _c0  |
+------+
| 0    |
+------+
1 row selected (72.948 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```

COUNT为MR任务。

查询员工的最大、最小、平均工资及所有工资的和

```
0: jdbc:hive2://pseduoDisHadoop:10000> select max(sal),min(sal),avg(sal),sum(sal) from emp;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
+---------+---------+--------------------+----------+
|   _c0   |   _c1   |        _c2         |   _c3    |
+---------+---------+--------------------+----------+
| 8000.0  | 3000.0  | 5333.366666666667  | 16000.1  |
+---------+---------+--------------------+----------+
1 row selected (65.214 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```


查询平均工资大于2000的部门

```
0: jdbc:hive2://pseduoDisHadoop:10000> select deptno,avg(sal) from emp group by deptno having avg(sal) > 2000;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
+---------+---------+
| deptno  |   _c1   |
+---------+---------+
| 4654    | 5000.1  |
| 7895    | 3000.0  |
| 7896    | 8000.0  |
+---------+---------+
3 rows selected (60.831 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

查询员工的姓名和工资等级，按如下规则显示

```
select ename, sal, 
case 
when sal < 1000 then 'lower'
when sal > 1000 and sal <= 2000 then 'middle'
when sal > 2000 and sal <= 4000 then 'high'
else 'highest' end
from emp;
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> select ename, sal, 
. . . . . . . . . . . . . . . . . . .> case 
. . . . . . . . . . . . . . . . . . .> when sal < 1000 then 'lower'
. . . . . . . . . . . . . . . . . . .> when sal > 1000 and sal <= 2000 then 'middle'
. . . . . . . . . . . . . . . . . . .> when sal > 2000 and sal <= 4000 then 'high'
. . . . . . . . . . . . . . . . . . .> else 'highest' end
. . . . . . . . . . . . . . . . . . .> from emp;
+---------+---------+----------+
|  ename  |   sal   |   _c2    |
+---------+---------+----------+
| TOM     | NULL    | highest  |
| donald  | 8000.0  | highest  |
| rain    | 5000.1  | highest  |
| jamel   | 3000.0  | high     |
+---------+---------+----------+
4 rows selected (0.263 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```

从上面可以看出，聚合类的操作，都需要运行MR任务。

## 导出数据

语法：

```
INSERT OVERWRITE [LOCAL] DIRECTORY directory1
  [ROW FORMAT row_format] [STORED AS file_format] (Note: Only available starting with Hive 0.11.0)
  SELECT ... FROM ...
```

导出数据与加载数据LOCAL使用基本一致。如果加上LOCAL表示导出到本地默认不加，导出到hdfs，OVERWRITE关键字为不可选，即先删除，在导出。

### 导出数据到本地
将表emp中的数据导出到本地目录/bdp/hive/data/tmp下

```
0: jdbc:hive2://pseduoDisHadoop:10000> insert overwrite local directory '/bdp/hive/data/tmp' row format delimited fields terminated by '\t' select * from emp;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (50.128 seconds)

```

查看文件：

```
donaldhan@pseduoDisHadoop:~$ cd /bdp/hive/data/tmp/
donaldhan@pseduoDisHadoop:/bdp/hive/data/tmp$ ll
total 16
drwxrwxr-x 2 donaldhan donaldhan 4096 Feb 27 22:41 ./
drwxrwxr-x 3 donaldhan donaldhan 4096 Feb 27 22:41 ../
-rw-r--r-- 1 donaldhan donaldhan  185 Feb 27 22:41 000000_0
-rw-r--r-- 1 donaldhan donaldhan   12 Feb 27 22:41 .000000_0.crc
donaldhan@pseduoDisHadoop:/bdp/hive/data/tmp$ cat 000000_0 
1001	TOM	MANAGER	\N	\N	\N	\N	\N
123	donald	lawer	4568	2019-02-26	8000.0	3000.0	7896
456	rain	teacher	1314	2018-01-25	5000.1	2000.2	4654
789	jamel	cleaner	\N	20170609	3000.0	1000.3	7895
donaldhan@pseduoDisHadoop:/bdp/hive/data/tmp$ 

```

重新执行一遍

```
0: jdbc:hive2://pseduoDisHadoop:10000> insert overwrite local directory '/bdp/hive/data/tmp' row format delimited fields terminated by '\t' select * from emp;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (46.835 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

```
donaldhan@pseduoDisHadoop:/bdp/hive/data/tmp$ ll
total 16
drwxrwxr-x 2 donaldhan donaldhan 4096 Feb 27 22:45 ./
drwxrwxr-x 3 donaldhan donaldhan 4096 Feb 27 22:41 ../
-rw-r--r-- 1 donaldhan donaldhan  185 Feb 27 22:45 000000_0
-rw-r--r-- 1 donaldhan donaldhan   12 Feb 27 22:45 .000000_0.crc
donaldhan@pseduoDisHadoop:/bdp/hive/data/tmp$ cat 000000_0 
1001	TOM	MANAGER	\N	\N	\N	\N	\N
123	donald	lawer	4568	2019-02-26	8000.0	3000.0	7896
456	rain	teacher	1314	2018-01-25	5000.1	2000.2	4654
789	jamel	cleaner	\N	20170609	3000.0	1000.3	7895
donaldhan@pseduoDisHadoop:/bdp/hive/data/tmp$ 

```

文件内容被覆盖。

### 导出数据到HDFS
将表emp中的数据导出到hdfs的/user/hive/data目录下

```
0: jdbc:hive2://pseduoDisHadoop:10000> insert overwrite directory '/user/hive/data' row format delimited fields terminated by '\t' select * from emp;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (54.917 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

在hdfs中查看文件数据

```
donaldhan@pseduoDisHadoop:~$ hdfs dfs -ls /user/hive/data
Found 1 items
-rwxr-xr-x   1 donaldhan supergroup        185 2020-02-27 22:48 /user/hive/data/000000_0
donaldhan@pseduoDisHadoop:~$ hdfs dfs -cat /user/hive/data/000000_0

1001	TOM	MANAGER	\N	\N	\N	\N	\N
123	donald	lawer	4568	2019-02-26	8000.0	3000.0	7896
456	rain	teacher	1314	2018-01-25	5000.1	2000.2	4654
789	jamel	cleaner	\N	20170609	3000.0	1000.3	7895
donaldhan@pseduoDisHadoop:~$ 
donaldhan@pseduoDisHadoop:~$ 

```

重试一遍

```
0: jdbc:hive2://pseduoDisHadoop:10000> insert overwrite directory '/user/hive/data' row format delimited fields terminated by '\t' select * from emp;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (55.794 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

```
donaldhan@pseduoDisHadoop:~$ hdfs dfs -ls /user/hive/data
Found 1 items
-rwxr-xr-x   1 donaldhan supergroup        185 2020-02-27 22:51 /user/hive/data/000000_0
donaldhan@pseduoDisHadoop:~$ hdfs dfs -cat /user/hive/data/000000_0
1001	TOM	MANAGER	\N	\N	\N	\N	\N
123	donald	lawer	4568	2019-02-26	8000.0	3000.0	7896
456	rain	teacher	1314	2018-01-25	5000.1	2000.2	4654
789	jamel	cleaner	\N	20170609	3000.0	1000.3	7895
donaldhan@pseduoDisHadoop:~$ 
```

数据被覆盖。

## 分区表
Hive 可以创建分区表，主要用于解决由于单个数据表数据量过大进而导致的性能问题 Hive 中的分区表分为两种：静态分区和动态分区

### 静态分区

静态分区由分为两种：单级分区和多级分区。我们分别来看这两种方式
1. 单级分区

创建一种订单分区表 

```
create table `order_partition`(
 order_number string,
 event_time string
 )
 partitioned by (event_month string)
 row format delimited fields terminated by '\t';
```


```
0: jdbc:hive2://pseduoDisHadoop:10000> create table `order_partition`(
. . . . . . . . . . . . . . . . . . .> order_number string,
. . . . . . . . . . . . . . . . . . .> event_time string
. . . . . . . . . . . . . . . . . . .> )
. . . . . . . . . . . . . . . . . . .> partitioned by (event_month string)
. . . . . . . . . . . . . . . . . . .> row format delimited fields terminated by '\t';
No rows affected (1.356 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
0: jdbc:hive2://pseduoDisHadoop:10000> show tables;
+------------------------+
|        tab_name        |
+------------------------+
| emp                    |
| emp2                   |
| emp3                   |
| order_partition        |
| values__tmp__table__1  |
+------------------------+
5 rows selected (0.251 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

我们先来看，从本地文件加载数据到分区表。

* 加载本地文件数据到分区表
将order.txt 文件中的数据加载到order_partition表中

```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ vim order.txt
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ pwd
/bdp/hive/hiveLocalTables
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ cat order.txt 
1	2020-02-27 22:59:23
2	2020-02-27 22:59:24
3	2020-02-27 22:59:25

```

```
0: jdbc:hive2://pseduoDisHadoop:10000> load data local inpath '/bdp/hive/hiveLocalTables/order.txt' overwrite into table order_partition partition (event_month='2020-02');
No rows affected (2.185 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from order_partition;
+-------------------------------+-----------------------------+------------------------------+
| order_partition.order_number  | order_partition.event_time  | order_partition.event_month  |
+-------------------------------+-----------------------------+------------------------------+
| 1                             | 2020-02-27 22:59:23         | 2020-02                      |
| 2                             | 2020-02-27 22:59:24         | 2020-02                      |
| 3                             | 2020-02-27 22:59:25         | 2020-02                      |
+-------------------------------+-----------------------------+------------------------------+
3 rows selected (0.34 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```

查看hdfs上的文件夹order_partition下多一个分区字段+分区Value的文件加载（event_month=2020-02） ，他下面的文件就是我们本地对应的order.txt文件。

```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -ls /user/hive/warehouse/test.db/order_partition
Found 1 items
drwxrwxrwx   - donaldhan supergroup          0 2020-02-27 23:02 /user/hive/warehouse/test.db/order_partition/event_month=2020-02
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -ls /user/hive/warehouse/test.db/order_partition/event_month=2020-02

Found 1 items
-rwxrwxrwx   1 donaldhan supergroup         66 2020-02-27 23:02 /user/hive/warehouse/test.db/order_partition/event_month=2020-02/order.txt
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -cat /user/hive/warehouse/test.db/order_partition/event_month=2020-02/order.txt
1	2020-02-27 22:59:23
2	2020-02-27 22:59:24
3	2020-02-27 22:59:25
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ 
```

* 使用hadoop shell 加载数据

donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -rm -r /user/hive/warehouse/test.db/order_partition


```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -rm -r /user/hive/warehouse/test.db/order_partition
20/02/27 23:19:26 INFO fs.TrashPolicyDefault: Namenode trash configuration: Deletion interval = 0 minutes, Emptier interval = 0 minutes.
Deleted /user/hive/warehouse/test.db/order_partition
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -ls /user/hive/warehouse/test.db/order_partition
ls: `/user/hive/warehouse/test.db/order_partition': No such file or directory
```

一不小心把分区表的文件夹给删了，不过没关系，这正是我们要做的，其实删除表文件下的分区问价夹即可。

查询数据，这是表中已经没有数据量

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from order_partition;
+-------------------------------+-----------------------------+------------------------------+
| order_partition.order_number  | order_partition.event_time  | order_partition.event_month  |
+-------------------------------+-----------------------------+------------------------------+
+-------------------------------+-----------------------------+------------------------------+
No rows selected (0.32 seconds)

```

创建分区分区文件夹，并将本地数据文件order.txt，上传到对应的分区目录下 /user/hive/warehouse/test.db/order_partition/event_month=2020-02：

```

donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -mkdir /user/hive/warehouse/test.db/order_partition
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -mkdir /user/hive/warehouse/test.db/order_partition/event_month=2020-02
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -put /bdp/hive/hiveLocalTables/order.txt /user/hive/warehouse/test.db/order_partition/event_month=2020-02
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -ls /user/hive/warehouse/test.db/order_partition/event_month=2020-02
Found 1 items
-rw-r--r--   1 donaldhan supergroup         66 2020-02-27 23:22 /user/hive/warehouse/test.db/order_partition/event_month=2020-02/order.txt
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ 
```

查看数据

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from order_partition;
+-------------------------------+-----------------------------+------------------------------+
| order_partition.order_number  | order_partition.event_time  | order_partition.event_month  |
+-------------------------------+-----------------------------+------------------------------+
| 1                             | 2020-02-27 22:59:23         | 2020-02                      |
| 2                             | 2020-02-27 22:59:24         | 2020-02                      |
| 3                             | 2020-02-27 22:59:25         | 2020-02                      |
+-------------------------------+-----------------------------+------------------------------+
3 rows selected (0.306 seconds)

```

当前数据已经有了。如果没有数据，我们可以使用如下命令进行修复

```
msck repair table order_partition;
```

我们再在HIVE数仓目录下，创建一个分区目录event_month=2020-01，并将数据文件order.txt上传上去；

```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -mkdir /user/hive/warehouse/test.db/order_partition/event_month=2020-01
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ hdfs dfs -put /bdp/hive/hiveLocalTables/order.txt /user/hive/warehouse/test.db/order_partition/event_month=2020-01
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ 
```

这次我们再次查询没有将数据加载表中，我们修复分区数据后，再次查询，可以看到了。

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from order_partition;
+-------------------------------+-----------------------------+------------------------------+
| order_partition.order_number  | order_partition.event_time  | order_partition.event_month  |
+-------------------------------+-----------------------------+------------------------------+
| 1                             | 2020-02-27 22:59:23         | 2020-02                      |
| 2                             | 2020-02-27 22:59:24         | 2020-02                      |
| 3                             | 2020-02-27 22:59:25         | 2020-02                      |
+-------------------------------+-----------------------------+------------------------------+
3 rows selected (0.299 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> msck repair table order_partition;
No rows affected (0.776 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from order_partition;
+-------------------------------+-----------------------------+------------------------------+
| order_partition.order_number  | order_partition.event_time  | order_partition.event_month  |
+-------------------------------+-----------------------------+------------------------------+
| 1                             | 2020-02-27 22:59:23         | 2020-01                      |
| 2                             | 2020-02-27 22:59:24         | 2020-01                      |
| 3                             | 2020-02-27 22:59:25         | 2020-01                      |
| 1                             | 2020-02-27 22:59:23         | 2020-02                      |
| 2                             | 2020-02-27 22:59:24         | 2020-02                      |
| 3                             | 2020-02-27 22:59:25         | 2020-02                      |
+-------------------------------+-----------------------------+------------------------------+
6 rows selected (0.329 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

对于分区表，不建议直接使用select * 查询，性能低，建议查询时加上条件，如果加上条件后它会直接从指定的分区中查找数据

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from order_partition where event_month='2020-02';
+-------------------------------+-----------------------------+------------------------------+
| order_partition.order_number  | order_partition.event_time  | order_partition.event_month  |
+-------------------------------+-----------------------------+------------------------------+
| 1                             | 2020-02-27 22:59:23         | 2020-02                      |
| 2                             | 2020-02-27 22:59:24         | 2020-02                      |
| 3                             | 2020-02-27 22:59:25         | 2020-02                      |
+-------------------------------+-----------------------------+------------------------------+
3 rows selected (0.483 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

```
mysql> select * from partitions;
+---------+-------------+------------------+---------------------+-------+--------+
| PART_ID | CREATE_TIME | LAST_ACCESS_TIME | PART_NAME           | SD_ID | TBL_ID |
+---------+-------------+------------------+---------------------+-------+--------+
|       1 |  1582815779 |                0 | event_month=2020-02 |    17 |     16 |
|       2 |  1582817419 |                0 | event_month=2020-01 |    18 |     16 |
+---------+-------------+------------------+---------------------+-------+--------+
2 rows in set
mysql> select * from partition_keys;
+--------+--------------+-------------+-----------+-------------+
| TBL_ID | PKEY_COMMENT | PKEY_NAME   | PKEY_TYPE | INTEGER_IDX |
+--------+--------------+-------------+-----------+-------------+
|     16 | NULL         | event_month | string    |           0 |
+--------+--------------+-------------+-----------+-------------+
1 row in set

```

2. 多级分区

创建表order_multi_partition

```
create table `order_multi_partition`(
order_number string,
event_time string
)
partitioned by (event_month string, step string)
row format delimited fields terminated by '\t';
```

加载数据到表order_multi_partition

```
0: jdbc:hive2://pseduoDisHadoop:10000> create table `order_multi_partition`(
. . . . . . . . . . . . . . . . . . .> order_number string,
. . . . . . . . . . . . . . . . . . .> event_time string
. . . . . . . . . . . . . . . . . . .> )
. . . . . . . . . . . . . . . . . . .> partitioned by (event_month string, step string)
. . . . . . . . . . . . . . . . . . .> row format delimited fields terminated by '\t';
No rows affected (1.521 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> load data local inpath '/bdp/hive/hiveLocalTables/order.txt' overwrite into table order_multi_partition partition (event_month='2020-02',step=1);
No rows affected (4.165 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from order_multi_partition;
+-------------------------------------+-----------------------------------+------------------------------------+-----------------------------+
| order_multi_partition.order_number  | order_multi_partition.event_time  | order_multi_partition.event_month  | order_multi_partition.step  |
+-------------------------------------+-----------------------------------+------------------------------------+-----------------------------+
| 1                                   | 2020-02-27 22:59:23               | 2020-02                            | 1                           |
| 2                                   | 2020-02-27 22:59:24               | 2020-02                            | 1                           |
| 3                                   | 2020-02-27 22:59:25               | 2020-02                            | 1                           |
+-------------------------------------+-----------------------------------+------------------------------------+-----------------------------+
3 rows selected (4.09 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

把step修改为2，再次加载数据

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from order_multi_partition;
+-------------------------------------+-----------------------------------+------------------------------------+-----------------------------+
| order_multi_partition.order_number  | order_multi_partition.event_time  | order_multi_partition.event_month  | order_multi_partition.step  |
+-------------------------------------+-----------------------------------+------------------------------------+-----------------------------+
| 1                                   | 2020-02-27 22:59:23               | 2020-02                            | 1                           |
| 2                                   | 2020-02-27 22:59:24               | 2020-02                            | 1                           |
| 3                                   | 2020-02-27 22:59:25               | 2020-02                            | 1                           |
| 1                                   | 2020-02-27 22:59:23               | 2020-02                            | 2                           |
| 2                                   | 2020-02-27 22:59:24               | 2020-02                            | 2                           |
| 3                                   | 2020-02-27 22:59:25               | 2020-02                            | 2                           |
+-------------------------------------+-----------------------------------+------------------------------------+-----------------------------+
6 rows selected (0.51 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

查看hdfs中的目录结构

```
donaldhan@pseduoDisHadoop:~$ hdfs dfs -ls /user/hive/warehouse/test.db/order_multi_partition/event_month=2020-02
Found 2 items
drwxrwxrwx   - donaldhan supergroup          0 2020-03-02 22:34 /user/hive/warehouse/test.db/order_multi_partition/event_month=2020-02/step=1
drwxrwxrwx   - donaldhan supergroup          0 2020-03-02 22:36 /user/hive/warehouse/test.db/order_multi_partition/event_month=2020-02/step=2
donaldhan@pseduoDisHadoop:~$ 

```

从上面可以看出，单级分区和多级分区唯一的区别就是多级分区在hdfs中的目录为多级。

### 动态分区

hive 中默认是静态分区，想要使用动态分区，需要设置如下参数，笔者使用的是临时设置，你也可以写在配置文件（hive-site.xml）里，永久生效。临时配置如下

开启动态分区（默认为false，不开启）

```
set hive.exec.dynamic.partition=true;
```

指定动态分区模式，默认为strict，即必须指定至少一个分区为静态分区，nonstrict模式表示允许所有的分区字段都可以使用动态分区。

```
set hive.exec.dynamic.partition.mode=nonstrict;
```

创建表student

```
create table `student`(
id int,
name string,
tel string,
age int
)
row format delimited fields terminated by '\t';
```

student.txt文件内容如下
```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ vim student.txt
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ cat student.txt 
1	donald	15965839766	23
2	rain	13697082376	18
3	jamel	15778566988	36
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ 

```

将文件student.txt中的内容加载到student表中

```
load data local inpath '/bdp/hive/hiveLocalTables/student.txt' overwrite into table student;
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> load data local inpath '/bdp/hive/hiveLocalTables/student.txt' overwrite into table student;
No rows affected (1.649 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from student;
+-------------+---------------+--------------+--------------+
| student.id  | student.name  | student.tel  | student.age  |
+-------------+---------------+--------------+--------------+
| 1           | donald        | 15965839766  | 23           |
| 2           | rain          | 13697082376  | 18           |
| 3           | jamel         | 15778566988  | 36           |
+-------------+---------------+--------------+--------------+
3 rows selected (0.414 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

创建分区表stu_age_partition

```
create table `stu_age_partition`(
id int,
name string,
tel string
)
partitioned by (age int)
row format delimited fields terminated by '\t';
```

将student表的数据以age为分区插入到stu_age_partition表，试想如果student表中的数据很多，使用insert一条一条插入数据，很不方便，所以这个时候可以使用hive的动态分区来实现

```
insert into table stu_age_partition partition (age) select id,name,tel,age from student;
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> insert into table stu_age_partition partition (age) select id,name,tel,age from student;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (104.21 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from stu_age_partition;
+-----------------------+-------------------------+------------------------+------------------------+
| stu_age_partition.id  | stu_age_partition.name  | stu_age_partition.tel  | stu_age_partition.age  |
+-----------------------+-------------------------+------------------------+------------------------+
| 2                     | rain                    | 13697082376            | 18                     |
| 1                     | donald                  | 15965839766            | 23                     |
| 3                     | jamel                   | 15778566988            | 36                     |
+-----------------------+-------------------------+------------------------+------------------------+
3 rows selected (0.473 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```
查询时，建议加上分区条件，性能高

```
select * from stu_age_partition;
select * from stu_age_partition where age > 20;
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from stu_age_partition where age > 20;
+-----------------------+-------------------------+------------------------+------------------------+
| stu_age_partition.id  | stu_age_partition.name  | stu_age_partition.tel  | stu_age_partition.age  |
+-----------------------+-------------------------+------------------------+------------------------+
| 1                     | donald                  | 15965839766            | 23                     |
| 3                     | jamel                   | 15778566988            | 36                     |
+-----------------------+-------------------------+------------------------+------------------------+
2 rows selected (0.998 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from stu_age_partition;
+-----------------------+-------------------------+------------------------+------------------------+
| stu_age_partition.id  | stu_age_partition.name  | stu_age_partition.tel  | stu_age_partition.age  |
+-----------------------+-------------------------+------------------------+------------------------+
| 2                     | rain                    | 13697082376            | 18                     |
| 1                     | donald                  | 15965839766            | 23                     |
| 3                     | jamel                   | 15778566988            | 36                     |
+-----------------------+-------------------------+------------------------+------------------------+
3 rows selected (0.337 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

# 复杂数据类型操作
这部分建议使用HIVE2（beeline），hive2中解决了hive1中的单点故障，可以搭建高可用的hive集群，hive2界面显示格式要比hive1直观、好看（类似于mysql中的shell界面）
## Array
创建一张带有数组的表tb_array
```
create table `tb_array`(
name string,
work_locations array<string>
)
row format delimited fields terminated by '\t'
collection items terminated by ',';

```

```
0: jdbc:hive2://pseduoDisHadoop:10000> desc tb_array;
+-----------------+----------------+----------+
|    col_name     |   data_type    | comment  |
+-----------------+----------------+----------+
| name            | string         |          |
| work_locations  | array<string>  |          |
+-----------------+----------------+----------+
2 rows selected (0.399 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```
加载数据到表tb_array
hive_array.txt文件内容如下
```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ vim hive_array.txt
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ cat hive_array.txt 
jamel	guangzhou,hangzhou
rain	fuyang,tongling,nanchang
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ 

```

命令如下：
```
load data local inpath '/bdp/hive/hiveLocalTables/hive_array.txt' overwrite into table tb_array;
```

表数据：
```
0: jdbc:hive2://pseduoDisHadoop:10000> select  * from tb_array;
+----------------+-----------------------------------+
| tb_array.name  |      tb_array.work_locations      |
+----------------+-----------------------------------+
| jamel          | ["guangzhou","hangzhou"]          |
| rain           | ["fuyang","tongling","nanchang"]  |
+----------------+-----------------------------------+
2 rows selected (4.112 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```
查询jamel的第一个工作地点（数组下标从0开始）
```
select name,work_locations[0] from tb_array where name='jamel';
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> select name,work_locations[0] from tb_array where name='jamel';
+--------+------------+
|  name  |    _c1     |
+--------+------------+
| jamel  | guangzhou  |
+--------+------------+
1 row selected (1.883 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```
查询每个人工作地点的数量（size）
```
select name,size(work_locations) from tb_array;
```
```
0: jdbc:hive2://pseduoDisHadoop:10000> select name,size(work_locations) from tb_array;
+--------+------+
|  name  | _c1  |
+--------+------+
| jamel  | 2    |
| rain   | 3    |
+--------+------+
2 rows selected (0.434 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```

## Map
创建一个带有map类型的表tb_map
```
create table `tb_map`(
name string,
scores map<string,int>
)
row format delimited fields terminated by '\t'
collection items terminated by ','
map keys terminated by ':';

```

```
0: jdbc:hive2://pseduoDisHadoop:10000> desc tb_map;;
+-----------+------------------+----------+
| col_name  |    data_type     | comment  |
+-----------+------------------+----------+
| name      | string           |          |
| scores    | map<string,int>  |          |
+-----------+------------------+----------+
2 rows selected (0.348 seconds)

```

加载数据到表tb_map
hive_map.txt文件内容如下
```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ vim hive_map.txt
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ cat hive_map.txt 
jamel	math:100,chinese:88,english:96
rain	math:89,chinese:68,english:78

```
命令如下：
```
load data local inpath '/bdp/hive/hiveLocalTables/hive_map.txt' overwrite into table tb_map;
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> load data local inpath '/bdp/hive/hiveLocalTables/hive_map.txt' overwrite into table tb_map;
No rows affected (2.021 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> select * from tb_map;
+--------------+-----------------------------------------+
| tb_map.name  |              tb_map.scores              |
+--------------+-----------------------------------------+
| jamel        | {"math":100,"chinese":88,"english":96}  |
| rain         | {"math":89,"chinese":68,"english":78}   |
+--------------+-----------------------------------------+
2 rows selected (0.364 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```
查询所有学生的英语成绩
```
select name,scores['english'] from tb_map;
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> select name,scores['english'] from tb_map;
+--------+------+
|  name  | _c1  |
+--------+------+
| jamel  | 96   |
| rain   | 78   |
+--------+------+
2 rows selected (0.344 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 
```
查询所有学生的英语和数学成绩

```
select name,scores['english'],scores['math'] from tb_map;
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> select name,scores['english'],scores['math'] from tb_map;
+--------+------+------+
|  name  | _c1  | _c2  |
+--------+------+------+
| jamel  | 96   | 100  |
| rain   | 78   | 89   |
+--------+------+------+
2 rows selected (0.323 seconds)
0: jdbc:hive2://pseduoDisHadoop:10000> 

```
## Struct

创建一张带有结构体的表
```
0: jdbc:hive2://pseduoDisHadoop:10000> desc tb_struct;
+-----------+------------------------------+----------+
| col_name  |          data_type           | comment  |
+-----------+------------------------------+----------+
| ip        | string                       |          |
| userinfo  | struct<name:string,age:int>  |          |
+-----------+------------------------------+----------+
2 rows selected (0.354 seconds)

```
加载文件hive_struct.txt中的数据到表tb_struct
hive_struct.txt文件内容如下:


```
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ vim hive_struct.txt
donaldhan@pseduoDisHadoop:/bdp/hive/hiveLocalTables$ cat hive_struct.txt 
192.168.1.1#zhangsan:40
192.168.1.2#lisi:50
192.168.1.3#wangwu:60
192.168.1.4#zhaoliu:70

```

命令如下：
```
load data local inpath '/bdp/hive/hiveLocalTables/hive_struct.txt' overwrite into table tb_struct;
```

表数据
```
0: jdbc:hive2://pseduoDisHadoop:10000> select * from tb_struct;
+---------------+-------------------------------+
| tb_struct.ip  |      tb_struct.userinfo       |
+---------------+-------------------------------+
| 192.168.1.1   | {"name":"zhangsan","age":40}  |
| 192.168.1.2   | {"name":"lisi","age":50}      |
| 192.168.1.3   | {"name":"wangwu","age":60}    |
| 192.168.1.4   | {"name":"zhaoliu","age":70}   |
+---------------+-------------------------------+
4 rows selected (0.27 seconds)

```

查询姓名及年龄
```
select userinfo.name,userinfo.age from tb_struct;
```

```
0: jdbc:hive2://pseduoDisHadoop:10000> select userinfo.name,userinfo.age from tb_struct;
+-----------+------+
|   name    | age  |
+-----------+------+
| zhangsan  | 40   |
| lisi      | 50   |
| wangwu    | 60   |
| zhaoliu   | 70   |
+-----------+------+
4 rows selected (0.375 seconds)

```

退出beeline

```
0: jdbc:hive2://pseduoDisHadoop:10000> !quit
Closing: 0: jdbc:hive2://pseduoDisHadoop:10000

```
至此，我们将hive的复查结构数据表讲完。

## 总结
由于我们创建数据库时没有指定对应的数仓存储路径，默认为HDFS下的数仓目录user/hive/warehouse+数据库名+.db对应的文件夹。

如果数据库中有0或多个表时，不能直接删除，需要先删除表再删除数据库；如果想要删除含有表的数据库，在删除时加上cascade，可以级联删除（慎用）。


内部表与外部表的区别 ：  
如果是内部表，在删除时，MySQL中的元数据和HDFS中的数据都会被删除 
如果是外部表，在删除时，MySQL中的元数据会被删除，HDFS中的数据不会被删除

加载文件数据到Hive表有两种方式，一种是从本地文件加载，一种从hdfs文件加载。
如果加上LOCAL表示从本地加载数据，默认不加，从hdfs中加载数据，添加
OVERWRITE关键字，将会覆盖表中数据，及先删除，在加载。

从hdfs方式加载完数据，需要注意hdfs上的文件将会被删除，移动hdfs的垃圾箱中。


插入数据实际为一个MR任务。聚合类的操作（max，min，avg，count），都需要运行MR任务。

导出数据与加载数据LOCAL和OVERWRITE使用基本一致。如果加上LOCAL表示导出到本地默认不加，导出到hdfs，如果加OVERWRITE关键字，将会覆盖原文件中的数据，及先删除，在导出。

从本地加载文件到分区表时，实际上是，将本地文件放到hdfs上的数据库分区表文件夹（order_partition）下的分区字段+分区Value（event_month=2020-02）文价夹。

单级分区和多级分区唯一的区别就是多级分区在hdfs中的目录为多级。

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

## Dynamic partition strict mode requires at least one static partition column
```
0: jdbc:hive2://pseduoDisHadoop:10000> insert into table stu_age_partition partition (age) select id,name,tel,age from student;
Error: Error while compiling statement: FAILED: SemanticException [Error 10096]: Dynamic partition strict mode requires at least one static partition column. To turn this off set hive.exec.dynamic.partition.mode=nonstrict (state=42000,code=10096)
0: jdbc:hive2://pseduoDisHadoop:10000> insert into table stu_age_partition partition (age) select id,name,tel,age from student;
Error: Error while compiling statement: FAILED: SemanticException [Error 10096]: Dynamic partition strict mode requires at least one static partition column. To turn this off set hive.exec.dynamic.partition.mode=nonstrict (state=42000,code=10096)

```

### 解决方式

主要是因为，hive 中默认是静态分区，想要使用动态分区，需要设置如下参数，笔者使用的是临时设置，你也可以写在配置文件（hive-site.xml）里，永久生效。临时配置如下

开启动态分区（默认为false，不开启）

```
set hive.exec.dynamic.partition=true;
```

指定动态分区模式，默认为strict，即必须指定至少一个分区为静态分区，nonstrict模式表示允许所有的分区字段都可以使用动态分区。

```
set hive.exec.dynamic.partition.mode=nonstrict;
```