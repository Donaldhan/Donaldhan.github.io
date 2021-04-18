---
layout: page
title: 如何高效的阅读源代码
subtitle: 如何高效的阅读源代码
date: 2021-02-15 22:30:19
author: ravitn
catalog: true
category: life
categories:
    - essay
tags:
    - essay
---

从NIO、JUC，从MINA到NETTY，从SpringFramework到SpringBoot，从jdbc到mybatis，从activemq到rokcetMq，TOMCAT，配置中心SuperDiamond，轻量级调度框架XXL-JOB及TSchedule改进版, RPC框架Dubbo，P2P框架incubator-gossip，Zookeeper，再到分布式框架Seata；阅读源码是一个探索的过程，也是一个和设计者交流的一个过程，如何高效阅读源码，如何确认，你懂得框架设计，是每个Coder需要关注的事情。

高效阅读源码的过程，可以分为以下几步：
1. 了解相关知识，包括官方文档，尝试运行Demo；
2. 顺藤摸瓜，通读，记录关键入口，在关键方法上把流程记录下来;
3. 在第二部的基础之上，画出框架图及关键流程图，如果需要，还有相关的时序图；
4. 思考框架使用场景，优缺点。
5. 形成文章，发布到个人博客，公总号，欢迎网友点评；
6. 如果有条件可以，开个技术分享会，与大家面对面的交流；

具体可以参考一下我的这篇文档，
[Seata分布式事务框架设计](https://donaldhan.github.io/seata/2021/02/08/seata-framework-design.html), 当前个人最高源码解读水准（~）。


PS：不要拘泥细节，关注整体框架和关键流程