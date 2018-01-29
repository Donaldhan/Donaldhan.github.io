---
layout: page
title: Memory Analyzer Tool的使用
subtitle: sub title
date: 2018-01-29 22:14:29
author: donaldhan
catalog: true
category: java
categories:
    - java
tags:
    - JVM
---

# 引言
最近又遇到了Java堆内存溢出异常，以前在[JVisualVM与MemoryAnalyzer分析堆内存过程][]，这篇文章中，使用VIsualVM与[MemoryAnalyzer][]分析堆内存，最近应用中出现又出现Java堆内存溢出异常。应用背景，调用WebService接口，查询信息。

## 目录
* [分析堆内存泄漏问题](#分析堆内存泄漏问题)
    * [导出堆内存](#导出堆内存)
    * [Overview](#overview)
    * [Leak_Suspects](#leak_suspects)
    * [Histogram](#histogram)
    * [Dominator_tree](#dominator_tree)
    * [Top-consumer](#top-consumer)
    * [Oql](#oql)
## 分析堆内存泄漏问题
分析堆内存，首先要将虚拟机内存导出到文件，然后使用Memory Analyzer Tool去分析堆内存文件，一般我们使用Memory Analyzer Tool的提供的Overview，Leak_Suspects，Histogram，Dominator_tree，Top-consumer,Oql

### 导出堆内存
启动应用，可以使用命令
```
jmap -dump:live,format=b,file=heap.bin 29136  
```
导出堆内存。具体可以参考[java虚拟机内存查看相关命令][]。
我们也可以使用JVisualVM监控应用，然后将堆内存导出，具体如下图：
![](/image/memory-analyzer/Java-VisualVm.png)

导出后我们在使用Memory Analyzer Tool分析堆内存文件，注意使用Memory Analyzer Tool打开堆内存文件，可能需要一些的时间，
同时要保证Memory Analyzer Tool的虚拟机内存要大于需要分析的堆内存文件。

### Overview
当Memory Analyzer Tool分析完堆内存文件后，将会有一个堆内存的概括性界面为Overview，具体如下如：
![](/image/memory-analyzer/overview.png)
在概括性的界面中我们可以，可到jvm可回收的的内存资源和内存泄漏报告，对象热力柱状图等。

### Leak_Suspects
Leak_Suspects为堆内存泄漏报告，从报告中，我们可以直接看出，那些对象占用的虚拟机的大部分资源。内存泄漏报告具体如下：
![Leak-Suspects](/image/memory-analyzer/Leak-Suspects.png)

![Leak-Suspects0](/image/memory-analyzer/Leak-Suspects0.png)

从上面内存泄漏报告中，我们看到以供有3个问题：
1. *2,890 instances of "org.apache.axis2.description.AxisService", loaded by "org.apache.catalina.loader.WebappClassLoader @ 0x86a3fe0" occupy 270,799,024 (31.26%) bytes. These instances are referenced from one instance of "java.util.Hashtable$Entry[]", loaded by "<system class loader>"  
Keywords
java.util.Hashtable$Entry[]
org.apache.catalina.loader.WebappClassLoader @ 0x86a3fe0
org.apache.axis2.description.AxisService*

2. *80,920 instances of "org.apache.axiom.om.impl.llom.OMElementImpl", loaded by "org.apache.catalina.loader.WebappClassLoader @ 0x86a3fe0" occupy 127,876,240 (14.76%) bytes. These instances are referenced from one instance of "java.util.Hashtable$Entry[]", loaded by "<system class loader>"   
Keywords
java.util.Hashtable$Entry[]
org.apache.catalina.loader.WebappClassLoader @ 0x86a3fe0   
org.apache.axiom.om.impl.llom.OMElementImpl*

3. *2,890 instances of "org.apache.axis2.engine.AxisConfiguration", loaded by "org.apache.catalina.loader.WebappClassLoader @ 0x86a3fe0" occupy 96,063,600 (11.09%) bytes. These instances are referenced from one instance of "java.util.Hashtable$Entry[]", loaded by "<system class loader>"  
Keywords
org.apache.axis2.engine.AxisConfiguration
java.util.Hashtable$Entry[]
org.apache.catalina.loader.WebappClassLoader @ 0x86a3fe0*

从上面可以看出，主要几个存在内存泄漏的对象为AxisService，OMElementImpl，AxisConfiguration。
点击详情泄漏问题详情，可以进一步查看对象存在内存泄漏的原因，如下图：
![](/image/memory-analyzer/Leak-Suspects1.png)



### Histogram
在Overview概括性视图中，点击Histogram，可以查看堆内存中对象的实例数的热力柱状图，具体如下图：
![](/image/memory-analyzer/Histogram-group-class.png)

默认热力柱状图是按class分类计算的，我们可以使用包来分类计算，具体如下：
![](/image/memory-analyzer/Histogram-group-package.png)

点击org包，展示如下：
![](/image/memory-analyzer/Histogram-group-package1.png)

可以看出，org.apache.axis2和org.apache.axiom两个包实例数较多，可能存在内存泄漏，这个热力柱状图的结果的内存泄漏报告结果吻合。
热力柱状图有两个重要的指标，分别为Shallow和Retained Heap。简单来说,
Shallow:就是对象本身占用内存的大小，不包含其引用的对象.
Retained:该对象被GC之后所能回收到内存的总和.
具体可以参考：
[Shallow and retained sizes][]

[Shallow and retained sizes]:https://www.yourkit.com/docs/java/help/sizes.jsp "Shallow and retained sizes"

我们可以使用热力柱状图的Inspector，查看对象的Shallow，Retained sizes，及GC Root等信息，
具体如下图：
![](/image/memory-analyzer/Histogram-inspector.png)  

![](/image/memory-analyzer/Histogram-inspector1.png)

### Dominator_tree
在Overview概括性视图中，点击Actions下的Dominator_tree，可以查看堆内存中大对象信息，具体如下：
![](/image/memory-analyzer/dominator-tree.png)

默认情况下以class来分类，我们可以选择按包来分,分后结果如下；
![](/image/memory-analyzer/dominator-tree-group-package.png)
从上图可以看出，org.apache.axis2和org.apache.axiom两个包实例数较多，可能存在内存泄漏，与内存泄漏报告结果吻合。

### Top-consumer
在Overview概括性视图中，点击Actions下的Top-consumer，可以查看堆内存中大对象信息，具体如下：
![](/image/memory-analyzer/top-consumer1.png)

![](/image/memory-analyzer/top-consumer2.png)


![](/image/memory-analyzer/top-consumer3.png)

![](/image/memory-analyzer/top-consumer4.png)

从Top-consumer饼状图和包top消耗图来看，消耗最多的包为org.apache.axis2和org.apache.axiom
两个包，最大的对象为AxisService，OMElementImpl，AxisConfiguration。与内存泄漏报告结果吻合。
### Oql
另外我们可以使用虚拟机对象查询语言，查询对象相关的信息，如下图：
![](/image/memory-analyzer/oql.png)
更多OQL的使用参考：[Querying Java heap with OQL][]


[Querying Java heap with OQL]:https://blogs.oracle.com/sundararajan/querying-java-heap-with-oql "Querying Java heap with OQL"

## 总结
从内存泄漏报告，对象实例热力柱状图，大对象树等可以看出，内存泄漏是由，org.apache.axis2和org.apache.axiom两个包下的AxisService，OMElementImpl，AxisConfiguration类引起的，
查了Axis是否存在内存泄露的情况，确实存在这样的问题，应用确实使用的Axis的存根客户端，升级Axis或使用CXF解决问题。

[JVisualVM与MemoryAnalyzer分析堆内存过程]:http://donald-draper.iteye.com/blog/2359052 "JVisualVM与MemoryAnalyzer分析堆内存过程"
[java虚拟机内存查看相关命令]:http://donald-draper.iteye.com/blog/2358771 "java虚拟机内存查看相关命令"
[MemoryAnalyzer]:http://wiki.eclipse.org/MemoryAnalyzer#HPROF_dumps_from_Sun_Virtual_Machines "Memory Analyzer Tool"
