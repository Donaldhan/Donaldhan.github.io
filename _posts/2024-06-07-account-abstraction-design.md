
# 引言

Account abstraction一致是以太坊开发社区的梦，同时也是以太坊路线图的一部分；它具有支持**多签和社交恢复，签名支持更高效和更简单的签名策略比如Schnorr, BLS， 以及后量子安全签名算法（Lamport, Winternitz），同时可升级**。虽然智能合约钱包在理论上可以执行多种操作，如存储数字资产、执行智能合约、管理多重签名钱包等，但在以太坊的实际应用中，存在一些限制。以太坊协议要求所有的操作都必须通过一个由ECDSA（椭圆曲线数字签名算法）保护的外部账户（EOA）发起的交易来进行包装。这意味着，即使操作是在智能合约钱包内部进行的，也需要通过EOA账户来触发交易。每次用户操作都需要通过一个EOA发起的交易来包装，这会产生额外的开销，即所谓的“gas”。在以太坊中，gas是用于支付交易执行成本的单位。每次这样的操作至少需要支付21000单位的gas，这构成了额外的开销。用户需要有一个单独的EOA账户来支付gas费用，并且需要管理两个账户（一个用于智能合约钱包，一个用于支付gas）的余额。为了简化这一过程，一些用户可能依赖于中继系统。然而，这些中继系统通常是中心化的，这与区块链的去中心化原则相悖。

EIP 2938 是一个以太坊改进提案（Ethereum Improvement Proposal），它建议对以太坊协议进行一些更改，以允许顶级以太坊交易从智能合约开始，而不是从外部拥有账户（EOA）开始。这意味着，智能合约本身将包含验证和费用支付逻辑，矿工在处理交易时会检查这些逻辑。然而，实施 EIP 2938 需要对以太坊协议进行显著的更改，而这在目前阶段对协议开发者来说可能是一个挑战，因为他们当时正集中精力于以太坊主网与信标链（Beacon Chain）的合并（即“The Merge”）以及网络的可扩展性问题。在这种情况下，你们的新提案（ERC 4337）提供了一个替代方案，它旨在实现与 EIP 2938 类似的收益，但不需要对共识层协议进行更改。这意味着，你们的方法可能更加灵活和易于实施，因为它不需要对以太坊网络的核心部分进行大规模的修改。


---
layout: page
title: Web3加速器Account abstraction
subtitle: Web3加速器Account abstraction
date: 2024-06-07 23:11:00
author: 0xTTEPX
catalog: true
category: Blockchain
categories:
    - Blockchain
tags:
    - Account-abstraction
---

# 目录
* [Account abstraction工作流程](#Account-abstraction工作流程) 
    * [概念](#概念) 
    * [核心流程](#核心流程) 
* [合约](#合约) 
* [总结](#总结)     
* [附](#附) 


# Account abstraction工作流程

![account-abstraction-framework](/image/account-abstraction/account-abstraction-framework.png)     


## 概念
* UserOperation: 用户发送的操作：主要包含 “sender”, “to”, “calldata”, “maxFeePerGas”, “maxPriorityFee”, “signature”, “nonce”，“signature”等， 签名协议不定义，有具体的account实现定义，比如EOA用户AA账户签名一般为EDSA，多签AA账户签名，具体以多签机制实现为准，比如BLS等；
* Sender（Account Contract）：发送OP的AA账户，可以为EOA绑定的AA账户，也可以是多签账户；
* AccountFactory：创建AA账户的工厂合约
* UserOperation  MemPool：higher-level内存池系统，用户提交的操作，先发送到内存池；
* EntryPoint：执行bundler捆绑的UserOperation；Bundlers/Clients白名单支持EntryPoint；
* Bundler：为处理UserOperation的节点（区块 builder），创建一个有效的EntryPoint.handleOps() 交易，并在他有效之前添加到区块；实现的途径可以两种形式，一种为Bundler自己是区块builder，另外一种是bundler使用区块构建基础设施，比如mev-boost或者 PBS (proposer-builder separation)；
* PayMaster：赞助交易gas合约；
* Aggregator：聚合签名合约，Bundlers/Clients白名单支持aggregators；
* EntryPointSimulations：在Bunlers打包UserOperation和处理UserOperations之前，模拟校验UserOperation和模拟处理UserOperations


## 核心流程
1. 发送UserOperation两种场景：
* 1.1 针对初始态，EOA账户需要先发送UserOperation，创建Account Contract， 这里的gas先由PayMaster代付，但需要用户先支付gas某个代理支付合约（具体是否需要根据平台或者业务方来定）；
* 1.2 针对已经创建Account Contract，可以直接由Account Contract发送UserOperation，gas可以选择PayMaster代付，也可以自己支付；

**Account Contract可以为EOA控制账户，也可以是多个EOA控制的多签账户；**

2. Bundler打包捆绑UserOperation
* 2.1 Bundler打包捆绑之前，会调用EntryPointSimulations提前验证UserOperation；
* 2.2 针对多签账户发起的UserOperation，需要使用签名聚合器Aggregator，进行聚合签名；
* 2.3 如果所有UserOperation验证没问题，则调用EntryPointSimulations模拟执行UserOperations；
* 2.3.1 模拟验证没有问题，则调用EntryPoint处理UserOperations；
* 2.3.2 针对多签账户发送UserOperation，则调用EntryPoint的多签处理UserOperations服务；

**为便于支付gas，AccountContact，Paymaster在发送UserOperation前，需要保证deposit原生币到EntryPoint；**
**为防止Paymaster引起的Dos，Paymaster，AccountFactory，Aggregator需要先质押（addStake）原生币到EntryPoint**

3. 处理UserOperations
* 3.1 针对UserOperation的initcode不为空的情况，需要调用AccountFactory，创建Account Contract;
* 3.2.1 针对使用Paymaster代付的gas，需要确保预付金足够；
* 3.2.2 校验账户签名及nonce，如果deposit的gas不足，需要支付预付金；
* 3.2.2 如果使用了签名聚合器Aggregator，需要验证聚合签名；
* 3.3.1 执行UserOperation的合约调用call；
* 3.3.1 如果是可执行账户AccountExecute, 直接执行UserOperation；
* 3.4 如果在实际执行过程中，执行结果非EntryPoint内部错误PostOpMode.postOpReverte（opSucceeded：执行成功，opReverted：实际执行revert），需要返还支付预付金；
* 3.5 待处理完所有UserOperation，补偿消耗的gas费给Bundler；



# 合约

![account-abstraction-framework](/image/account-abstraction/account-abstraction-contract.png)   

* BaseAccount：基础账户，提供校验OP；提供两种实现SimpleAccount和BLSAccount，在EntryPoint校验OP的过程中，分别通过EP通过SenderCreator，调用SimpleAccountFactory和BLSAccountFactory生成；
* Aggregator：聚合签名和验证签名服务，提供基于BLS的签名聚合器实现BLSSignatureAggregator；**为什么需要聚合签名？，目的是减少验证的gas，不用将打包的UserOperations签名全部验证一遍，只需要验证最终的聚合签名即可**
* Paymaster：校验op，depoist及质押，解押，提现等操作；提供基于Token的gas支付PayMaster实现TokenPaymaster等；

> TokenPaymaster：基于oracle和swap实现；关键功能如下：
1. validatePaymasterUserOp：将所需要支付的gas等价的token从OP发送者转账到paymaster；
2. postOp：更新token价格，如果由于gas预付资金不足，发送者需要补足gas预发金，充盈的情况下，退还多余预付金；最后如果Paymaster预付金不足，则需要将token转换原生币，deposit到EntryPoint；


* NonceManager：账户nonce管理；
* StakeManager：账户deposit（pay gas）和stake（Paymaster、AccountFactory、Aggregator）管理；
* EntryPoint：提供核心handleOps和handleAggregatedOps操作；**handleAggregatedOps执行的核心逻辑和handleOps一样，只不过，增加了校验聚合签名的过程；**
* EntryPointSimulations：提供模拟校验UserOperation和模拟处理UserOperations操作，在Bunlers打包UserOperation和处理UserOperations之前；在模拟校验UserOperation同时会验证Paymaster、AccountFactory、Aggregator的质押信息；



# 总结
Account abstraction使合约可以直接发起交易，支持多种高效和简单的签名，同时可升级，另外Account abstraction也可以保护隐私，进一步保护账户的安全。个人感觉更重要的是可以支持赞助gas，这样在当前Layer2生态一片繁荣的情况下，可以进一步降低用户进入web3的门槛，打开地球村民可以web3新世界里自由漫步的大门，Account abstraction基础设施完善的时刻，也是web3进入实质生产的高原期的阶段，让我们热烈地迎接即将到来的这一时刻。


# Refer
[safe wallet](https://safe.global/wallet) 
[smart-account-overview](https://docs.safe.global/advanced/smart-account-overview)   
[safe-smart-account](https://github.com/Donaldhan/safe-smart-account)  
[account-abstraction](https://github.com/Donaldhan/account-abstraction) 
[bundler](https://github.com/Donaldhan/bundler) 
[bundler-spec-tests](https://github.com/Donaldhan/bundler-spec-tests) 



[ERC 4337: account abstraction without Ethereum protocol changes](https://medium.com/infinitism/erc-4337-account-abstraction-without-ethereum-protocol-changes-d75c9d94dc4a)    
[eip-4337](https://github.com/ethereum/EIPs/blob/e4519f1e182e5ec49d99022532b54369e8b293e9/EIPS/eip-4337.md)      
[eip-4337](https://eips.ethereum.org/EIPS/eip-4337)    
[EIP-2938: Account Abstraction ](https://eips.ethereum.org/EIPS/eip-2938)     
[EIP-3074: AUTH and AUTHCALL opcodes ](https://eips.ethereum.org/EIPS/eip-3074) 


[ERC 4337 | Account Abstraction 中文詳解](https://medium.com/@alan890104/erc-4337-account-abstraction-37535ff5fe24)  
[分析EIP-4337：以太坊最强账户抽象提案](https://learnblockchain.cn/article/5768)   
[理解账户抽象#1 - 从头设计智能合约钱包](https://learnblockchain.cn/article/5426)   
[理解账户抽象 #2：使用Paymaster赞助交易](https://learnblockchain.cn/article/5432)     
[理解账户抽象 #3 - 钱包创建](https://learnblockchain.cn/article/5442)    
[理解账户抽象 #4：聚合签名](https://learnblockchain.cn/article/5483)   
[解读 EigenLayer 中使用的 BLS 聚合签名](https://learnblockchain.cn/article/7855)    
[BLS签名实现阈值签名的流程](https://learnblockchain.cn/2019/08/29/bls)     
[Schnorr 和 BLS 算法详解丨区块链技术课程 #4](https://learnblockchain.cn/article/8364)     


[EIP-4337](https://www.notion.so/plancker/EIP-4337-0baad80755eb498c81d4651ccb527eb2)       
[深入剖析 Ownbit 和 Gnosis 多签](https://learnblockchain.cn/article/1902)      
[多签钱包的工作原理与使用方式](https://learnblockchain.cn/article/4077)    
[深入探讨Vitalik及各种路线图对以太坊治理的影响](https://www.theblockbeats.info/news/53643)      
[]()    
[]()    


