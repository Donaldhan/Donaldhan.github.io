---
layout: page
title: MolochV2 DAO解读
subtitle: MolochV2 DAO解读
date: 2022-05-10 23:22:00
author: Ravitn
catalog: true
category: DAO
categories:
    - DAO
tags:
    - Moloch
---


# 引言
Minty的Patka认为“MolochMystics”（包括使用Moloch的各种DAO的成员）组装了Baal的详细信息。该集体
包括DAOHaus、Minty、ReflexerFinance、LexDAO的成员以及来自RaidGuild和MetaCartel的贡献者。

MolochDAO 提供了一个带有 v2 智能合约的开源 DAO 框架。Tribute 和 DAOhaus 都从 Moloch 的工作中受益。 

Moloch DAO主要运营实体分为DAO成员委员会和与非成员协调合作的公会，因此在Moloch DAO v2智能合约标准
里会根据以下几个特性来实现治理目标：

* MolochDAO的会员资格是一个准入的过程，在链上提交新的成员提案并进行投票之前，
候选人必须首先得到DAO现有成员的支持，并接受内部成员驱动的评估，其中考虑其成员的各个方面：
文化契合度、专业知识等。

* 会员资格是社区监管的：成为MolochDAO的成员，该人必须获得MolochDAO经济上大多数成员的同意。
任何成员都可以随时提议将另一名成员从DAO中除名，如果该提议得到足够多的其他成员的批准，
则该人将失去其DAO股份和治理权，同时（通过RageQuit机制）接收代表他们MolochDAO
 财产的百分比的付款代币。


* Moloch DAO v2智能合约体现了“选择加入”和自主选择的朋克设计原则，
因此退出DAO 变得尽可能容易、无需信任和经济上无风险。
因此，每个DAO成员都可以随时就其全部或任何部分股份进行RageQuit，并接收支付代币和索赔代币，
这些代币代表该成员凭借股份数量有权获得的DAO财产百分比。

# 目录
* [合约架构](#合约架构) 
* [基于MolochDAO的运行机制](#基于molochdao的运行机制) 
    * [提案流程](#提案流程) 
    * [资金流转模型y](#资金流转模型) 
* [总结](#总结)     
* [附](#附) 


接下来，我们将从DAO合约的角度，探索DAO的运行机制。

# 合约架构

![molochv2-contract-framework](/image/molochv2/molochv2-contract-framework.png)

CloneFactory：基于EIP-1167合约clone工厂
Moloch：提案管理，包括创建，投票，处理，怒退，怒踢等流程；
MolochSummoner：创建DAO，注册DAO


# 基于MolochDAO的运行机制

部署DAO合约Moloch，需要提供的信息如下：
* 初始化提案发起间隔
* 提案投票期限
* 提案公示缓冲期（此期间对结果不满意的人都可以怒退）
* 初始化成员及投票份额share
* 允许使用的token（DAO组织允许使用的token）
* 处理提案奖励ETH数量（每个处理提案的人，将会获得的奖励）
* 投票稀释率（控制提案share与loot份额占公会总share与loot份额的比率, 超过比率，则无效）；



## 提案流程
1. 发起提案；
2. 赞助者赞助提案，则提案放到投票的提案队列；
3. 成员在投票期间投票；
4. 在投票结束后的缓存期（Grace Period）内，不满意的股东可以怒退；
5. 执行提案（处理提案）；

提案类型：正常提案；白名单提案；踢出提案；
股份类型：投票份额share，loot份额（无投票权限，怒退，被怒踢时，可以获得相应公会token的占比份额）


提案主要包含的信息
* 请求投票share份额（提案成功，将会奖励投票份额）
* 请求的loot份额（提案成功，奖励loot份额；怒退，或被怒踢时，可以获取loot加share相应比率的公会token金额）；
* 贡献公会的token（托管的公会托管池，提案成功时，会划转到公会银行）
* 公会需要支付的token（提案通过时，公会需要支付的token）

提案主要包含4个状态：
是否被赞助：是否执行：是否取消：是否提案通过



提案被链上执行后，将会将赞助者质押的token，退还给赞助者；
提案不通过，将会把贡献的token返回给提议者；

所有提议者贡献的，赞助者赞助的token将会放到公会银行；


具体我们再来结合资金流转模型看一下。

## 资金流转模型

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
（sharesToBurn+lootToBurn）/（totalShares+totalLoot）* token(GUILD)

1. 从公会池GUILD，划转到用户的公会账户；


* 怒踢
怒踢和怒退的区别是，share份额被废除；

token份额计算公式为：
（成员loot）/（totalShares+totalLoot）* token(GUILD)

1. 从公会池GUILD，划转到用户的公会账户；


* 退款
1. DAO合约发起Token转账给用户账户；


# 总结

基于Moloch的DAO，有初始成员组成，同时限制DAO使用的ERC20 Token；任何成员如果想要成为DAO成员，只有
发起提案，需要超过提案通过，方可加入DAO组织；DAO的公会银行资金，有提案者，贡献献给公会；任何怒退和被怒踢的
人都可取回公会资金池相应份额的token。

成员通过提案加入公会的成本是否太高，能否用多签方式去解决？？MolochV2只是基于怒退机制的Grant DAO，
后面我们将会扯掉MolochV3情趣内衣（~!~）。





# 附
[引介 | Moloch DAO：瓦解公司制](https://zhuanlan.zhihu.com/p/65605851)   
[热门区块链治理项目 Moloch DAO 究竟是什么？为何如此重要？](https://zhuanlan.zhihu.com/p/75036887)     
[DAO 框架构建者 Moloch 在 ETHDenver 发布 V3](https://chainoe.com/8461.html)   
[MolochV3预览](https://www.panewslab.com/zh/articledetails/D64612532.html)     
[金色前哨｜MolochDAO发布V3版本](https://zhuanlan.zhihu.com/p/469235625)  

[深入了解智能合约的最小代理“EIP-1167”](https://blog.csdn.net/chinadefi/article/details/121631038)  
[以太坊使用最小Gas克隆合约-合约工厂](https://zhuanlan.zhihu.com/p/252341880)   


 