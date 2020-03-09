---
layout: page
title: my blog
subtitle: sub title
date: 2020-11-04 15:17:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - spring-context
---

# 引言

[Flume][]属于大数据体系中的一个分布式的、可靠的、高可用的海量日志采集、聚合和传输的系统。基于Source-Channel-Sink事件数据流模型，同时通过事务机制保证消息传递的可靠性， 内置丰富插件，轻松与其他系统集成。底层Java实现，同时具备优秀的系统框架设计，模块分明，易于开发。



[Flume]:http://flume.apache.org/ "Flume" 

# 目录
* [Flume数据流模型](#flume数据流模型)
* [Flume实战](#flume实战)
    * [单agent单channel、sink场景](#单agent单channel、sink场景)
    * [多agent多channel、sink场景](#多agent多channel、sink场景)
    * [单agent多channel、sink场景](#单agent多channel、sink场景)
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

原始数据

http://flume.apache.org/releases/content/1.9.0/FlumeUserGuide.html#logging-raw-data

### 多agent多channel、sink场景

### 单agent多channel、sink场景


## 总结
[Flume][]属于大数据体系中的一个分布式的、可靠的、高可用的海量日志采集、聚合和传输的系统。基于Source-Channel-Sink事件数据流模型，同时通过事务机制保证消息传递的可靠性， 内置丰富插件，轻松与其他系统集成。底层Java实现，同时具备优秀的系统框架设计，模块分明，易于开发。



# 附

## 参考文献
flume:<http://flume.apache.org/>    
FlumeUserGuide:<http://flume.apache.org/releases/content/1.9.0/FlumeUserGuide.html>   
Flume构建日志采集系统:<https://www.jianshu.com/p/1183139ed3a0> 
Flume:<https://blog.csdn.net/qq_35078688/article/details/83552451>   
Flume(一):<https://www.cnblogs.com/xuziyu/p/11004103.html>  
Flume:<https://www.jianshu.com/p/323859671420>  
ELK日志收集系统搭建:<https://www.iteye.com/blog/donald-draper-2302224>   