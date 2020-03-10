---
layout: page
title: my blog
subtitle: sub title
date: 2020-11-04 15:17:19
author: donaldhan
catalog: true
category: bdp
categories:
    - bdp
tags:
    - flume
---

# 引言

[Flume][]属于大数据体系中的一个分布式的、可靠的、高可用的海量日志采集、聚合和传输的系统。基于Source-Channel-Sink事件数据流模型，同时通过事务机制保证消息传递的可靠性， 内置丰富插件，轻松与其他系统集成。底层Java实现，同时具备优秀的系统框架设计，模块分明，易于开发。



[Flume]:http://flume.apache.org/ "Flume" 

# 目录
* [Flume数据流模型](#flume数据流模型)
   * [网络流](#网络流) 
   * [sink](#sink) 
   * [Channel组件](#Channel组件) 
* [Flume实战](#flume实战)
    * [单agent单channel、sink场景](#单agent单channel、sink场景)
    * [Source组件Spooling-Directory-Source](#source组件spooling-directory-source)
    * [Source组件Taildir_source](#source组件taildir_source)
    * [Sink组件-HDFS-Sink](#sink组件-hdfs-sink)
    * [Sink组件_Kafka_Sink](#sink组件_kafka_sink)
* [总结](#总结)
* [附](#附)


# Flume数据流模型

Flume基本组件主要有一下5个。
* Event：消息的基本单位，有header和body组成
* Agent：JVM进程，负责将一端外部来源产生的消息转 发到另一端外部的目的地
* Source：从外部来源读入event，并写入channel
* Channel：event暂存组件，source写入后，event将会 一直保存,
* Sink：从channel读入event，并写入目的地

根据需求, 我们可以根据agent、channel和sink设计不同的数据流传输模型。

## 单agent单channel和sink场景
![flow_dataflow](/image/flume/flow_dataflow.png)
典型的单一场景， 主要使用与单体应用。

## 多agent多channel和sink场景
![dataflow_multi_agent_and_channel](/image/flume/dataflow_multi_agent_and_channel.webp)

一般适用于分布式场景数据收集场景。

## 单agent多channel和sink场景
![dataflow_multi_channel_and_sink](/image/flume/dataflow_multi_channel_and_sink.webp)

适用于单体应用，多业务数据分析的场景。

## 网络流
flume支持从以下几个网络流中读取数据

* Avro
* Thrift
* Syslog
* Netcat


## sink
flume支持一下sink；
file， logger， hdfs，hbase， kafka，hive等。

## Channel组件
Channel被设计为event中转暂存区，存储Source 收集并且没有被Sink消费的event ，为了平衡Source收集 和Sink读取数据的速度，可视为Flume内部的消息队列。Channel是线程安全的并且具有事务性，支持source写失 败重复写和sink读失败重复读等操作

常用的Channel类型有：Memory Channel、File Channel、Kafka Channel、JDBC Channel等

### Memory Channel
Memory Channel使用内存作为Channel，Memory Channel读写速度 快，但是存储数据量小，Flume进程挂掉、服务器停机或者重启都会 导致数据丢失。部署Flume Agent的线上服务器内存资源充足、不关 心数据丢失的场景下可以使用

### Channel组件- File Channel
* File Channel将event写入到磁盘文件中，与Memory Channel相比存 储容量大，无数据丢失风险。
* File Channle数据存储路径可以配置多磁盘文件路径，提高写入文件性能
* Flume将Event顺序写入到File Channel文件的末尾，在配置文件中通过设置maxFileSize参数设置数据文件大小上限
* 当一个已关闭的只读数据文件中的Event被完全读取完成，并且Sink已经提交读取完成的事务，则Flume将删除存储该数据文件
* 通过设置检查点和备份检查点在Agent重启之后能够快速将File Channle中的数据按顺序回放到内存中

### Kafka Channel
Kafka Channel将分布式消息队列kafka作为channel相对于Memory Channel和File Channel存储容量更大、 容错能力更强，弥补了其他两种Channel的短板，如果合理利用Kafka的性能，能够达到事半功倍的效果。

### SSL/TLS 
flume支持SSL/TLS的组件如下：
```
Avro    Source	    server
Avro    Sink	    client
Thrift  Source	    server
Thrift  Sink	    client
Kafka   Source	    client
Kafka   Channel	    client
Kafka   Sink	    client
HTTP    Source	    server
JMS     Source	    client
Syslog TCP Source	server
Multiport Syslog TCP Source	server
```

## Flume实战
首先下载flume，将其解压到我们本地，我使用的版本是, apache-flume-1.8.0, 注意flume是基于Java的，需要安装JDK，这里就略过。

解压压缩包

```
donaldhan@pseduoDisHadoop:/bdp/flume$ ls
apache-flume-1.8.0-bin.tar.gz
donaldhan@pseduoDisHadoop:/bdp/flume$ tar -zxvf apache-flume-1.8.0-bin.tar.gz
donaldhan@pseduoDisHadoop:/bdp/flume$ ls
apache-flume-1.8.0-bin  apache-flume-1.8.0-bin.tar.gz
```

配置环境变量
```
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin$ tail -f ~/.bashrc 
export ZOOKEEPER_HOME=/bdp/zookeeper/zookeeper-3.4.12
export HADOOP_MAPRED_HOME=${HADOOP_HOME}
export HADOOP_COMMON_HOME=${HADOOP_HOME}
export HADOOP_HDFS_HOME=${HADOOP_HOME}
export HADOOP_COMMON_LIB_NATIVE_DIR=${HADOOP}/lib/native
export YARN_HOME=${HADOOP_HOME}
export HADOOP_OPT="-Djava.library.path=${HADOOP_HOME}/lib/native"
export HIVE_HOME=/bdp/hive/apache-hive-2.3.4-bin
export FLUME_HOME=/bdp/flume/apache-flume-1.8.0-bin
export PATH=${JAVA_HOME}/bin:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:${ZOOKEEPER_HOME}/bin:${HBASE_HOME}/bin:${HIVE_HOME}/bin:${FLUME_HOME}/bin:${PATH}
^C
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin$ 

```

配置flume环境

首先拷贝配置属性文件和环境变量文件
```
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin$ cd conf/
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin/conf$ ls
flume-conf.properties.template  flume-env.sh.template
flume-env.ps1.template          log4j.properties
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin/conf$ cp flume-conf.properties.template flume-conf.properties
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin/conf$ view flume-conf.properties
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin/conf$ ls
flume-conf.properties           flume-env.ps1.template  log4j.properties
flume-conf.properties.template  flume-env.sh.template
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin/conf$ cp flume-env.sh.template flume-env.sh
```

配置flume java环境
```
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin/conf$ tail -f flume-env.sh
# Let Flume write raw event data and configuration information to its log files for debugging
# purposes. Enabling these flags is not recommended in production,
# as it may result in logging sensitive user information or encryption secrets.
# export JAVA_OPTS="$JAVA_OPTS -Dorg.apache.flume.log.rawdata=true -Dorg.apache.flume.log.printconfig=true "

# Note that the Flume conf directory is always included in the classpath.
#FLUME_CLASSPATH=""
export JAVA_HOME=/usr/share/jdk1.8.0_191
export JAVA_OPTS="$JAVA_OPTS -Dorg.apache.flume.log.rawdata=true -Dorg.apache.flume.log.printconfig=true"

```

修改日志文件路径
```
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin/conf$ vim log4j.properties 
```

修改如下属性
```
flume.log.dir=/bdp/flume/logs
```




我们先看一下简单的场景。
### 单agent单channel、sink场景

#### netcat数据源控制台日志实例
创建配置文件
```
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin/conf$ vim example.conf
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin/conf$ cat example.conf 
# example.conf: A single-node Flume configuration

# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# Describe the sink
a1.sinks.k1.type = logger

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1

```

上面首先定了对flume组件进行命名，分别定义source，sink，channel的属性；然后通过channel将source和sink绑定。

上面定义了一个类型为netcat的网络监听数据源，sink为控制台日志。

我们来启动
```
flume-ng agent --conf conf --conf-file example.conf --name a1 -Dflume.root.logger=INFO,console
```
```
donaldhan@pseduoDisHadoop:/bdp/flume/apache-flume-1.8.0-bin/conf$ flume-ng agent --conf conf --conf-file example.conf --name a1 -Dflume.root.logger=INFO,console
Info: Including Hadoop libraries found via (/bdp/hadoop/hadoop-2.7.1/bin/hadoop) for HDFS access
Info: Including Hive libraries found via (/bdp/hive/apache-hive-2.3.4-bin) for Hive access
+ exec /usr/share/jdk1.8.0_191/bin/java -Xmx20m -Dflume.root.logger=INFO,console -cp 'conf:/bdp/flume/apache-flume-1.8.0-bin/lib/*:/bdp/hadoop/hadoop-2.7.1/etc/hadoop:/bdp/hadoop/hadoop-2.7.1/share/hadoop/common/lib/*:/bdp/hadoop/hadoop-2.7.1/share/hadoop/common/*:/bdp/hadoop/hadoop-2.7.1/share/hadoop/hdfs:/bdp/hadoop/hadoop-2.7.1/share/hadoop/hdfs/lib/*:/bdp/hadoop/hadoop-2.7.1/share/hadoop/hdfs/*:/bdp/hadoop/hadoop-2.7.1/share/hadoop/yarn/lib/*:/bdp/hadoop/hadoop-2.7.1/share/hadoop/yarn/*:/bdp/hadoop/hadoop-2.7.1/share/hadoop/mapreduce/lib/*:/bdp/hadoop/hadoop-2.7.1/share/hadoop/mapreduce/*:/bdp/hadoop/hadoop-2.7.1/contrib/capacity-scheduler/*.jar:/bdp/hive/apache-hive-2.3.4-bin/lib/*' -Djava.library.path=:/bdp/hadoop/hadoop-2.7.1//lib/native org.apache.flume.node.Application --conf-file example.conf --name a1
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/bdp/flume/apache-flume-1.8.0-bin/lib/slf4j-log4j12-1.6.1.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/bdp/hadoop/hadoop-2.7.1/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/bdp/hive/apache-hive-2.3.4-bin/lib/log4j-slf4j-impl-2.6.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
20/03/09 22:34:34 INFO node.PollingPropertiesFileConfigurationProvider: Configuration provider starting
20/03/09 22:34:34 INFO node.PollingPropertiesFileConfigurationProvider: Reloading configuration file:example.conf
20/03/09 22:34:35 INFO conf.FlumeConfiguration: Added sinks: k1 Agent: a1
20/03/09 22:34:35 INFO conf.FlumeConfiguration: Processing:k1
20/03/09 22:34:35 INFO conf.FlumeConfiguration: Processing:k1
20/03/09 22:34:35 INFO conf.FlumeConfiguration: Post-validation flume configuration contains configuration for agents: [a1]
20/03/09 22:34:35 INFO node.AbstractConfigurationProvider: Creating channels
20/03/09 22:34:35 INFO channel.DefaultChannelFactory: Creating instance of channel c1 type memory
20/03/09 22:34:35 INFO node.AbstractConfigurationProvider: Created channel c1
20/03/09 22:34:35 INFO source.DefaultSourceFactory: Creating instance of source r1, type netcat
20/03/09 22:34:35 INFO sink.DefaultSinkFactory: Creating instance of sink: k1, type: logger
20/03/09 22:34:35 INFO node.AbstractConfigurationProvider: Channel c1 connected to [r1, k1]
20/03/09 22:34:35 INFO node.Application: Starting new configuration:{ sourceRunners:{r1=EventDrivenSourceRunner: { source:org.apache.flume.source.NetcatSource{name:r1,state:IDLE} }} sinkRunners:{k1=SinkRunner: { policy:org.apache.flume.sink.DefaultSinkProcessor@6b3bbad4 counterGroup:{ name:null counters:{} } }} channels:{c1=org.apache.flume.channel.MemoryChannel{name: c1}} }
20/03/09 22:34:35 INFO node.Application: Starting Channel c1
20/03/09 22:34:36 INFO instrumentation.MonitoredCounterGroup: Monitored counter group for type: CHANNEL, name: c1: Successfully registered new MBean.
20/03/09 22:34:36 INFO instrumentation.MonitoredCounterGroup: Component type: CHANNEL, name: c1 started
20/03/09 22:34:36 INFO node.Application: Starting Sink k1
20/03/09 22:34:36 INFO node.Application: Starting Source r1
20/03/09 22:34:36 INFO source.NetcatSource: Source starting
20/03/09 22:34:36 INFO source.NetcatSource: Created serverSocket:sun.nio.ch.ServerSocketChannelImpl[/127.0.0.1:44444]
```

现在我们telnet 44444端口，发送一个事件
```
donaldhan@pseduoDisHadoop:~$ telnet localhost 44444
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello flume
OK

```

这使我们在flume的控制台日志多了一条日志
```
20/03/09 22:36:34 INFO sink.LoggerSink: Event: { headers:{} body: 68 65 6C 6C 6F 20 66 6C 75 6D 65 0D             hello flume. }
```

从上面可以看出，flume从source接收到的数据流单位，为事件包括头部和body。

flume可以将环境变量放到配置文件中, 比如如下配置
```
a1.sources = r1
a1.sources.r1.type = netcat
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = ${NC_PORT}
a1.sources.r1.channels = c1
```

使用java属性实现的方式，调用代理比如：
```
flume-ng agent –conf conf –conf-file example.conf –name a1 -Dflume.root.logger=INFO,console -DpropertiesImplementation=org.apache.flume.node.EnvVarResolverProperties
```
环境变量也可以通过conf/flume-env.sh文件去配置。

#### 开启日志或者输出原始数据
如果我们对数据有什么问题，我们可以开启日志或者输出原始数据, 来调试，这个一般在生产环境中，不建议使用。

具体命令如下：

```
flume-ng agent --conf conf --conf-file example.conf --name a1 -Dflume.root.logger=DEBUG,console -Dorg.apache.flume.log.printconfig=true -Dorg.apache.flume.log.rawdata=true
```

主要是两个参数
```
-Dorg.apache.flume.log.printconfig=true -Dorg.apache.flume.log.rawdata=true
```

我们可以将配置文件放到zookeeper上去，在启动时，只需要将zookeeper地址加上即可。
同样Flume支持插件化，可以使用任务第三方sources, channels, sinks, serializers插件。


#### kafka source
Flume支持kafka source配置如下:
```
tier1.sources.source1.type = org.apache.flume.source.kafka.KafkaSource
tier1.sources.source1.channels = channel1
tier1.sources.source1.batchSize = 5000
tier1.sources.source1.batchDurationMillis = 2000
tier1.sources.source1.kafka.bootstrap.servers = localhost:9092
tier1.sources.source1.kafka.topics = test1, test2
tier1.sources.source1.kafka.consumer.group.id = custom.g.id
```

对于topic可以使用正则匹配模式，具体如下：
```
tier1.sources.source1.type = org.apache.flume.source.kafka.KafkaSource
tier1.sources.source1.channels = channel1
tier1.sources.source1.kafka.bootstrap.servers = localhost:9092
tier1.sources.source1.kafka.topics.regex = ^topic[0-9]$
# the default kafka.consumer.group.id=flume is used
```


#### Source组件Spooling-Directory-Source
配置一个Spooling Directory Source ,spooldirsource.conf 配置文件内容如下：
```
a1.sources = r1
a1.channels = c1
a1.sinks = k1
a1.sources.r1.type = spooldir
a1.sources.r1.channels = c1
a1.sources.r1.spoolDir = /bdp/flume/spoolDir
a1.sources.r1.fileHeader = true
a1.channels.c1.type = memory
a1.channels.c1.capacity = 10000
a1.channels.c1.transactionCapacity = 1000
a1.sinks.k1.type = logger
a1.sinks.k1.channel = c1
```
/bdp/flume/spoolDir 必须已经创建且具有用户读写权限
```
donaldhan@pseduoDisHadoop:/bdp/flume$ mkdir spoolDir
donaldhan@pseduoDisHadoop:/bdp/flume$ chmod -R 777 spoolDir/
donaldhan@pseduoDisHadoop:/bdp/flume$ ll -al
total 57336
drwxr-xr-x  4 donaldhan donaldhan     4096 Mar 10 21:53 ./
drwxr-xr-x 11 donaldhan donaldhan     4096 Mar 10 21:25 ../
drwxrwxr-x  7 donaldhan donaldhan     4096 Mar  9 22:12 apache-flume-1.8.0-bin/
-rw-rw-r--  1 donaldhan donaldhan 58688757 Dec 22  2018 apache-flume-1.8.0-bin.tar.gz
drwxrwxrwx  2 donaldhan donaldhan     4096 Mar 10 21:53 spoolDir/
donaldhan@pseduoDisHadoop:/bdp/flume$ 

```

启动 SpoolDirsourceAgent
```
flume-ng agent --conf conf --conf-file spooldirsource.conf  --name a1 -Dflume.root.logger=INFO,console
```

在spoolDir文件夹下创建文件并写入文件内容，
```
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ vim log
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ cat log.COMPLETED 
hello flume
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ ls -al
total 16
drwxrwxrwx 3 donaldhan donaldhan 4096 Mar 10 21:56 .
drwxr-xr-x 4 donaldhan donaldhan 4096 Mar 10 21:53 ..
drwxrwxr-x 2 donaldhan donaldhan 4096 Mar 10 21:56 .flumespool
-rw-rw-r-- 1 donaldhan donaldhan   12 Mar 10 21:56 log.COMPLETED
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ 
```

观察控制台消息：
```
20/03/10 21:55:39 INFO node.Application: Starting Sink k1
20/03/10 21:55:39 INFO node.Application: Starting Source r1
20/03/10 21:55:39 INFO source.SpoolDirectorySource: SpoolDirectorySource source starting with directory: /bdp/flume/spoolDir
20/03/10 21:55:39 INFO instrumentation.MonitoredCounterGroup: Monitored counter group for type: SOURCE, name: r1: Successfully registered new MBean.
20/03/10 21:55:39 INFO instrumentation.MonitoredCounterGroup: Component type: SOURCE, name: r1 started
20/03/10 21:56:10 INFO avro.ReliableSpoolingFileEventReader: Last read took us just up to a file boundary. Rolling to the next file, if there is one.
20/03/10 21:56:10 INFO avro.ReliableSpoolingFileEventReader: Preparing to move file /bdp/flume/spoolDir/log to /bdp/flume/spoolDir/log.COMPLETED
20/03/10 21:56:10 INFO sink.LoggerSink: Event: { headers:{file=/bdp/flume/spoolDir/log} body: 68 65 6C 6C 6F 20 66 6C 75 6D 65                hello flume }

```

此时监测到SpoolDirSourceAgent 可以监控到文件变化。

**注意：Spooling Directory Source Agent 并不能监听子级文件夹的文件变化,也不支持已存在的文件更新数据变化.**


创建二级目录日志：
```
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ echo "bye bye" > log2
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ mkdir logdir
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ echo "bye bye" > logdir/log3
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ 
```
控制台只有一条消息
```
20/03/10 22:02:25 INFO sink.LoggerSink: Event: { headers:{file=/bdp/flume/spoolDir/log2} body: 62 79 65 20 62 79 65                            bye bye }
20/03/10 22:02:25 INFO avro.ReliableSpoolingFileEventReader: Last read took us just up to a file boundary. Rolling to the next file, if there is one.
20/03/10 22:02:25 INFO avro.ReliableSpoolingFileEventReader: Preparing to move file /bdp/flume/spoolDir/log2 to /bdp/flume/spoolDir/log2.COMPLETED

```

### Source组件Taildir_source


<!-- TODO -->


#### Sink组件-HDFS-Sink
* HDFS Sink将Event写入到HDFS中持久化存储
* HDFS Sink提供了强大的时间戳转义功能，根据Event头信息中的
* timestamp时间戳信息转义成日期格式，在HDFS中以日期目录分层存储

关键参数信息说明如下：
```
type：Sink类型为hdfs。
hdfs.path：HDFS存储路径，支持按日期时间分区。
hdfs.filePrefix：Event输出到HDFS的文件名前缀，默认前缀FlumeData
hdfs.fileSuffix：Event输出到HDFS的文件名后缀
hdfs.inUsePrefix：临时文件名前缀
hdfs.inUseSuffix：临时文件名后缀，默认值.tmp
hdfs.rollInterval：HDFS文件滚动生成时间间隔，默认值30秒，该值设置 为0表示文件不根据时间滚动生成
```

配置一个hdfsink.conf文件，配置内容如下：

```
a1.sources = r1
a1.channels = c1
a1.sinks = k1
a1.sources.r1.type = spooldir
a1.sources.r1.spoolDir = /bdp/flume/spoolDir
a1.sources.r1.fileHeader = true
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type = timestamp
a1.sources.r1.interceptors.i1.preserveExisting = false
a1.sources.r1.channels = c1
a1.channels.c1.type = memory
a1.channels.c1.capacity = 10000 
a1.channels.c1.transactionCapacity = 1000
a1.sinks.k1.type = hdfs
a1.sinks.k1.channel = c1
a1.sinks.k1.hdfs.path = /data/flume/%Y%m%d
a1.sinks.k1.hdfs.filePrefix = hdfssink
a1.sinks.k1.hdfs.fileType = DataStream
a1.sinks.k1.hdfs.writeFormat = Text
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.roundValue = 1
a1.sinks.k1.hdfs.roundUnit = minute
a1.sinks.k1.hdfs.callTimeout = 60000

```
启动一个hdfssink agent，命令如下：

```
flume-ng agent --conf conf --conf-file hdfsink.conf --name a1 -Dflume.root.logger=INFO,console
```

在spoolDir文件夹下创建文件并写入文件内容，
```
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ echo "hello hdfs" > log3
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ 

```

此时控制台打印，在HDFS文件系统生成一个临时文件
```
20/03/10 22:19:24 INFO instrumentation.MonitoredCounterGroup: Component type: SINK, name: k1 started
20/03/10 22:19:24 INFO source.SpoolDirectorySource: SpoolDirectorySource source starting with directory: /bdp/flume/spoolDir
20/03/10 22:19:24 INFO instrumentation.MonitoredCounterGroup: Monitored counter group for type: SOURCE, name: r1: Successfully registered new MBean.
20/03/10 22:19:24 INFO instrumentation.MonitoredCounterGroup: Component type: SOURCE, name: r1 started
20/03/10 22:20:02 INFO avro.ReliableSpoolingFileEventReader: Last read took us just up to a file boundary. Rolling to the next file, if there is one.
20/03/10 22:20:02 INFO avro.ReliableSpoolingFileEventReader: Preparing to move file /bdp/flume/spoolDir/log3 to /bdp/flume/spoolDir/log3.COMPLETED
20/03/10 22:20:02 INFO hdfs.HDFSDataStream: Serializer = TEXT, UseRawLocalFileSystem = false
20/03/10 22:20:03 INFO hdfs.BucketWriter: Creating /data/flume/20200310/hdfssink.1583850002738.tmp
20/03/10 22:20:37 INFO hdfs.BucketWriter: Closing /data/flume/20200310/hdfssink.1583850002738.tmp
20/03/10 22:20:37 INFO hdfs.BucketWriter: Renaming /data/flume/20200310/hdfssink.1583850002738.tmp to /data/flume/20200310/hdfssink.1583850002738

```

查看hdfs文件：
```
donaldhan@pseduoDisHadoop:~$ hdfs dfs -ls /data/flume
Found 1 items
drwxr-xr-x   - donaldhan supergroup          0 2020-03-10 22:20 /data/flume/20200310
donaldhan@pseduoDisHadoop:~$ hdfs dfs -ls /data/flume/20200310
Found 1 items
-rw-r--r--   1 donaldhan supergroup         11 2020-03-10 22:20 /data/flume/20200310/hdfssink.1583850002738
donaldhan@pseduoDisHadoop:~$ hdfs dfs -cat /data/flume/20200310/hdfssink.1583850002738
hello hdfs
donaldhan@pseduoDisHadoop:~$ 

```

**注意：请使用hadoop用户来执行agent的创建和消息的发送，避免因权限导致HDFS文件无法写入**

#### Sink组件_Kafka_Sink
Flume通过KafkaSink将Event写入到Kafka指定的主题中
主要参数说明如下
```
 type：Sink类型，值为KafkaSink类路径  org.apache.flume.sink.kafka.KafkaSink。
 kafka.bootstrap.servers：Broker列表，定义格式host:port，多个Broker之间用逗号隔开，可以配置一个也可以配置多个，用于Producer发现集群中的Broker，建议配置多个，防止当个Broker出现问题连接 失败。
 kafka.topic：Kafka中Topic主题名称，默认值flume-topic。
 flumeBatchSize：Producer端单次批量发送的消息条数，该值应该根据实际环境适当调整，增大批量发送消息的条数能够在一定程度上提高性能，但是同时也增加了延迟和Producer端数据丢失的风险。 默认值100。
 kafka.producer.acks：设置Producer端发送消息到Borker是否等待接收Broker返回成功送达信号。0表示Producer发送消息到Broker之后不需要等待Broker返回成功送达的信号，这种方式吞吐量高，但是存 在数据丢失的风险。1表示Broker接收到消息成功写入本地log文件后向Producer返回成功接收的信号，不需要等待所有的Follower全部同步完消息后再做回应，这种方式在数据丢失风险和吞吐量之间做了平衡。all（或者-1）表示Broker接收到Producer的消息成功写入本 地log并且等待所有的Follower成功写入本地log后向Producer返回成功接收的信号，这种方式能够保证消息不丢失，但是性能最差。默 认值1。
 useFlumeEventFormat：默认值false，Kafka Sink只会将Event body内 容发送到Kafka Topic中。如果设置为true，Producer发送到KafkaTopic中的Event将能够保留Producer端头信息
```
配置一个kafkasink.conf,具体配置内容如下：

```
a1.sources = r1
a1.channels = c1
a1.sinks = k1
a1.sources.r1.type = spooldir
a1.sources.r1.spoolDir = /bdp/flume/spoolDir
a1.sources.r1.fileHeader = true
a1.sources.r1.channels = c1
a1.channels.c1.type = memory
a1.channels.c1.capacity = 10000 
a1.channels.c1.transactionCapacity = 1000
a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.k1.channel = c1
a1.sinks.k1.kafka.topic = FlumeKafkaSinkTopic1
a1.sinks.k1.kafka.bootstrap.servers = 192.168.5.139:9092
a1.sinks.k1.kafka.flumeBatchSize = 100
a1.sinks.k1.kafka.producer.acks = 1
```
kafka单机模式:<https://www.iteye.com/blog/donald-draper-2397170> 

启动服务器,由于kafka需要使用Zookeeper，所以你需要先启动Zookeeper，如果没有Zookeeper，可以使用kafka内部自带的Zookeeper。

```
donaldhan@pseduoDisHadoop:/kafka/kafka_defalut/bin$ ./zookeeper-server-start.sh ../config/zookeeper.properties &  
```
启动kafka服务器
```
donaldhan@pseduoDisHadoop:/kafka/kafka_defalut/bin$ ./kafka-server-start.sh ../config/server.properties  &  

```
按配置文件创建主题信息
```
donaldhan@pseduoDisHadoop:/kafka/kafka_defalut/bin$ ./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic FlumeKafkaSinkTopic1 
Created topic "FlumeKafkaSinkTopic1".
donaldhan@pseduoDisHadoop:/kafka/kafka_defalut/bin$ 

```

启动一个kafkasink agent，启动命令如下：

```
flume-ng agent --conf conf --conf-file kafkasink.conf --name a1 -Dflume.root.logger=INFO,console
```

在spoolDir文件夹下创建文件并写入文件内容，
```
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ echo "hello kafka" > log5
donaldhan@pseduoDisHadoop:/bdp/flume/spoolDir$ 

```
此时控制台打印
```

20/03/10 22:51:15 INFO instrumentation.MonitoredCounterGroup: Component type: CHANNEL, name: c1 started
20/03/10 22:51:15 INFO node.Application: Starting Sink k1
20/03/10 22:51:15 INFO node.Application: Starting Source r1
20/03/10 22:51:15 INFO source.SpoolDirectorySource: SpoolDirectorySource source starting with directory: /bdp/flume/spoolDir
20/03/10 22:51:15 INFO producer.ProducerConfig: ProducerConfig values: 
	compression.type = none
	metric.reporters = []
	metadata.max.age.ms = 300000
	metadata.fetch.timeout.ms = 60000
	reconnect.backoff.ms = 50
	sasl.kerberos.ticket.renew.window.factor = 0.8
	bootstrap.servers = [192.168.5.139:9092]
	retry.backoff.ms = 100
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	buffer.memory = 33554432
	timeout.ms = 30000
	key.serializer = class org.apache.kafka.common.serialization.StringSerializer
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	ssl.keystore.type = JKS
	ssl.trustmanager.algorithm = PKIX
	block.on.buffer.full = false
	ssl.key.password = null
	max.block.ms = 60000
	sasl.kerberos.min.time.before.relogin = 60000
	connections.max.idle.ms = 540000
	ssl.truststore.password = null
	max.in.flight.requests.per.connection = 5
	metrics.num.samples = 2
	client.id = 
	ssl.endpoint.identification.algorithm = null
	ssl.protocol = TLS
	request.timeout.ms = 30000
	ssl.provider = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	acks = 1
	batch.size = 16384
	ssl.keystore.location = null
	receive.buffer.bytes = 32768
	ssl.cipher.suites = null
	ssl.truststore.type = JKS
	security.protocol = PLAINTEXT
	retries = 0
	max.request.size = 1048576
	value.serializer = class org.apache.kafka.common.serialization.ByteArraySerializer
	ssl.truststore.location = null
	ssl.keystore.password = null
	ssl.keymanager.algorithm = SunX509
	metrics.sample.window.ms = 30000
	partitioner.class = class org.apache.kafka.clients.producer.internals.DefaultPartitioner
	send.buffer.bytes = 131072
	linger.ms = 0

20/03/10 22:51:15 INFO instrumentation.MonitoredCounterGroup: Monitored counter group for type: SOURCE, name: r1: Successfully registered new MBean.
20/03/10 22:51:15 INFO instrumentation.MonitoredCounterGroup: Component type: SOURCE, name: r1 started
20/03/10 22:51:15 INFO utils.AppInfoParser: Kafka version : 0.9.0.1
20/03/10 22:51:15 INFO utils.AppInfoParser: Kafka commitId : 23c69d62a0cabf06
20/03/10 22:51:15 INFO instrumentation.MonitoredCounterGroup: Monitored counter group for type: SINK, name: k1: Successfully registered new MBean.
20/03/10 22:51:15 INFO instrumentation.MonitoredCounterGroup: Component type: SINK, name: k1 started
20/03/10 22:51:33 INFO avro.ReliableSpoolingFileEventReader: Last read took us just up to a file boundary. Rolling to the next file, if there is one.
20/03/10 22:51:33 INFO avro.ReliableSpoolingFileEventReader: Preparing to move file /bdp/flume/spoolDir/log5 to /bdp/flume/spoolDir/log5.COMPLETED



```
启动kafka消费客户端
```
donaldhan@pseduoDisHadoop:/kafka/kafka_defalut/bin$ ./kafka-console-consumer.s--bootstrap-server 192.168.5.139:9092 --topic FlumeKafkaSinkTopic1 --from-beginning  

hello kafka

```
从上面可以看出，flume已经文件日志，发送到kafka的topic中。


## 总结
[Flume][]属于大数据体系中的一个分布式的、可靠的、高可用的海量日志采集、聚合和传输的系统。基于Source-Channel-Sink事件数据流模型，同时通过事务机制保证消息传递的可靠性， 内置丰富插件，轻松与其他系统集成。底层Java实现，同时具备优秀的系统框架设计，模块分明，易于开发。



tailfile TODO

source:tailfile 
channel:kafka
sink:hdfs



source:tailfile 
channel:kafka
sink:kafka

拦截器，sink处理器

TODO


# 附



## 参考文献
flume:<http://flume.apache.org/>    
FlumeUserGuide:<http://flume.apache.org/releases/content/1.9.0/FlumeUserGuide.html>   
Flume构建日志采集系统:<https://www.jianshu.com/p/1183139ed3a0> 
Flume:<https://blog.csdn.net/qq_35078688/article/details/83552451>   
Flume(一):<https://www.cnblogs.com/xuziyu/p/11004103.html>  
Flume:<https://www.jianshu.com/p/323859671420>  
ELK日志收集系统搭建:<https://www.iteye.com/blog/donald-draper-2302224>   

## 错误

```
 ERROR [KafkaServer id=0] Fatal error during KafkaServer startup. Prepare to shutdown (kafka.server.KafkaServer)
org.apache.kafka.common.KafkaException: Socket server failed to bind to 192.168.5.128:9092: Cannot assign requested address.
	at kafka.network.Acceptor.openServerSocket(SocketServer.scala:442)
	at kafka.network.Acceptor.<init>(SocketServer.scala:332)
	at kafka.network.SocketServer.$anonfun$createAcceptorAndProcessors$
```
问题原因，kafka无法绑定host，检查server的地址绑定配置。

##

```
RROR source.SpoolDirectorySource: FATAL: Spool Directory source r1: { spoolDir: /bdp/flume/spoolDir }: Uncaught exception in SpoolDirectorySource thread. Restart or reconfigure Flume to continue processing.
java.lang.IllegalStateException: File name has been re-used with different files. Spooling assumptions violated for /bdp/flume/spoolDir/log4.COMPLETED
```

问题原因：文件名重复导致。我是测试用的，直接删除文件夹，如果是线上，可能要考虑一下，行不行。



