---
layout: page
title: DAO服务MolochV3
subtitle: DAO服务MolochV3
date: 2022-09-20 23:22:00
author: Ravitn
catalog: true
category: DAO
categories:
    - DAO
tags:
    - Moloch
---


# 引言
基于Moloch的DAO，有初始成员组成，同时限制DAO使用的ERC20 Token；任何成员如果想要成为DAO成员，只有发起提案，需要超过提案通过，方可加入DAO组织；DAO的公会银行资金，有提案者，贡献献给公会；任何怒退和被怒踢的人都可取回公会资金池相应份额的token。MolochV2只是基于怒退机制的Grant DAO，MolochV3增加了可插拔，可升级的模块化功能，
今天，我们将会扯掉MolochV3情趣内衣（~!~）。

# 目录
* [V2及V3概述](#v2及v3概述) 
    * [MolochV2](#molochv2) 
    * [MolochV3](#molochv3) 
* [V3整体架构](#V3整体架构) 
* [资金流模型](#资金流模型)
* [总结](#总结)     
* [附](#附) 

# V2及V3概述

## MolochV2
Moloch v2是MolochDAO的升级版本，它使DAO可以获取并使用多个不同的代币，而不仅仅是一种。它引入了公会剔（Guild Kick）提案类型，该类型允许成员强行删除另一个成员（他们的资产会被全额退还）。它还允许以战利品的形式发行无投票权的股份。最后，v2修复了最初的Nomic Labs审核中提出的“不安全批准”问题。

### V2设计原则

在开发Moloch v2的过程中，molochDAO坚持其无情的极简主义，在大幅提高实用性的同时，尽可能少地偏离原作。因此，许多功能被再次跳过，该设计代表了一个最小可行的营利性DAO，但它足够灵活，可以支持各种使用去中心化的情况，包括风险基金、对冲基金、投资银行和孵化器。


## MolochV3
Moloch v3的灵感来自于六边形的架构模式，除了将主合约分解成小合约外，还可以提供额外的安全层。有了这个松散耦合的模块/合约，就创造了更容易审计的模块/合约，并且可以很容易地连接到DAO。V3架构主要由3种主要类型的组件组成： 核心模块

* 核心模块跟踪DAO的状态变化。
* 每个核心模块通过接口定义，实现，并在DAO创建时注册到DAO注册表模块。
* 核心模块名称注册表跟踪所有注册的核心模块，因此在调用执行时可以验证它们。
* 只有适配器或其他核心模块才允许调用核心模块的功能。
* 核心模块不直接与外部世界通信，它们需要通过一个适配器。
* 每个核心模块都是一个智能合约，在其功能上应用了 "onlyAdapter "和/或 "onlyModule "修改器，它不应该以公开的方式暴露其功能（外部或公共修改器不应该被添加到核心模块的功能中，除了只读的功能）。

### 适配器

从外部世界调用的公共/外部可访问函数。

* 适配器不跟踪DAO的状态，他们可能会使用存储，但理想的情况是，任何DAO相关的状态变化都会传播到核心模块中。
* 适配器只是执行智能合约逻辑，通过调用核心模块来改变DAO的状态，它们也可以组成复杂的调用，与外部世界互动，拉动/推送额外的数据。
* 每个适配器都是一个非常专业的智能合约，旨在很好地做一件事。

适配器可以有公共访问权，也可以只对DAO的成员进行访问（onlyMembers修改器）

### 外部世界

RPC客户端负责调用适配器的公共/外部函数，与DAO核心模块进行交互。

主要的想法是根据每一层来限制对合约的访问。外部世界（例如：RPC客户端）只能通过适配器访问核心模块，决不能直接访问。
每个适配器将包含所有必要的逻辑和数据，在调用过程中提供给核心模块，而核心模块将跟踪DAO中的状态变化。
这个想法的初稿在融资适配器中实现，它允许个人提交grant的请求。信息总是从外部世界流向核心模块，而不是反过来。
如果一个核心模块需要外部信息，应该通过一个输出适配器提供，而不是直接调用外部世界信息。

# V3整体架构
![MolochV3-framework](/image/molochv3/MolochV3-framework..png)

整体架构主要有基础层，核心层，适配器层，及工厂层。


## 基础层

基础层主要Guard模块和 注册器Registry  

* Guard模块
主要包含重入ReentrancyGuard、模块ModuleGuard、适配器AdapterGuard的调用控制

* 注册器Registry

注册器，主要负责核心模块的管理，比如银行Bank、成员Member、提案Proposal

## 核心层
core模块， 主要包括银行Bank、成员Member、提案Proposal，以dao为维护，进行管理


成员Member：赞助ETH加入，并获取响应的shares份额

## 适配器层

应用层服务，提供DAO维度的管理，比如成员的加入Onboarding，提案赞助经费申请Financing，模块升级变更Managing，投票Voting，怒退Ragequit；

## 工厂层

工厂层主要是DaoFactory，负责创建创建dao



V3主要将MolochV2的提案，投票，怒退，银行管理等模块化；在V2中，每通过MolochSummoner部署一个Moloch，实际上就是一个DAO，DAO之间没有管理，自己管理自己的DAO；同时，投票，成员，银行功能无法升级，V3将这些功能模块化，同时支持模块功能的定制和升级，不过这个需要提案来完成；并将所有的DAO通过核心层管理起来；另外
V3中不再有loot的概念，统一为投票份额shares。




# 资金流模型


![molochv2-guild-bank-cash-model](/image/molochv2/molochv2-guild-bank-cash-model.png)


* 发起提案
1. 提案用户转账贡献tributeToken到DAO合约地址；
2. 划账贡献token到公会托管池ESCROW，同时添加公会总池TOTAL；

* 赞助提案
1. 赞助用户转账质押depositToken到DAO合约地址；
2. 划账质押depositToken到公会托管池ESCROW，同时添加公会总池； 

* 处理提案

提案通过的情况

1. 从托管池ESCROW，将贡献tributeToken，划转到公会池GUILD；
2. 从公会池GUILD，划转支付paymentToken，给公会用户；
3. 从托管池ESCROW，划转奖励的质押token，给公会处理提案账号；
4. 从托管池ESCROW，将剩余的质押token，退回到公会赞助者账号；

提案不通过的情况

1. 从托管池ESCROW，退回贡献tributeToken，到公会提案用户账户；
2. 从托管池ESCROW，划转奖励的质押token，给公会处理提案账号；
3. 从托管池ESCROW，将剩余的质押token，退回到公会赞助者账号；


* 取消提案
1. 从托管池ESCROW，将贡献tributeToken，划转到公会提案用户账户；


* 怒退
怒退的公会成员，可以取回给定share和loot份额的公会资产token(所有提案者贡献给公会的token)；

token份额计算公式为：
（sharesToBurn）/（totalShares* token(GUILD)

1. 从公会池GUILD，划转到用户的公会账户；


* 怒踢
怒踢和怒退的区别是，share份额被废除；

token份额计算公式为：
（成员shares）/（totalShares）* token(GUILD)

1. 从公会池GUILD，划转到用户的公会账户；


* 退款
1. DAO合约发起Token转账给用户账户；


更多参见

[MolochV2 DAO解读](https://donaldhan.github.io/solidity/2022/05/09/molochv2-design-framwork.html) 



# 总结
V3主要将MolochV2的提案，投票，怒退，银行管理等模块化；在V2中，每通MolochSummoner部署一个Moloch，实际上就是一个DAO，DAO之间没有管理，自己管理自己的DAO；同时，投票，成员，银行功能无法升级，V3将这些功能模块化，同时支持模块功能的定制和升级，不过这个需要提案来完成；并将所有的DAO通过核心层管理起来；另外
V3中不再有loot的概念，统一为投票份额shares。




# 附
[引介 | Moloch DAO：瓦解公司制](https://zhuanlan.zhihu.com/p/65605851)   
[热门区块链治理项目 Moloch DAO 究竟是什么？为何如此重要？](https://zhuanlan.zhihu.com/p/75036887)     
[DAO 框架构建者 Moloch 在 ETHDenver 发布 V3](https://chainoe.com/8461.html)   
[MolochV3预览](https://www.panewslab.com/zh/articledetails/D64612532.html)     
[金色前哨｜MolochDAO发布V3版本](https://zhuanlan.zhihu.com/p/469235625)   

[Molochv2.1](https://github.com/Moloch-Mystics/Molochv2.1)   
[moloch-v3](https://github.com/Moloch-Mystics/moloch-v3)   


[​DAOrayaki研究｜MolochDAO v1、v2、v3](https://media.daorayaki.org/daorayakiyan-jiu-molochdao-v1-v2-v3/)   
[六边形架构入门与实践](https://www.jianshu.com/p/c2a361c2406c) 


## 六边形架构
六边形架构又称“端口和适配器模式”，是Alistair Cockburn提出的一种具有对称性特征的架构风格。在这种架构中，系统通过适配器的方式与外部交互，将应用服务于领域服务封装在系统内部。