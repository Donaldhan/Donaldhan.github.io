---
layout: page
title: 去中心借贷Compound
subtitle: 去中心借贷Compound
date: 2022-10-17 23:37:00
author: Ravitn
catalog: true
category: DeFi
categories:
    - DeFi
tags:
    - Compound
---


# 引言

当前中心交易所借贷，面临黑客攻击，携款跑路，导致你的账号资金都是虚拟的，无法在链上使用，同时个人进行借贷需要管理各种条款和投机的风险；去中心化的借贷平台Compound
简化了借贷流程，同时避免了用户管理各种条款、投机的风险，主要有一个优势：

1. 不需要填写任何订单及线下操作就可以完成借贷；
2. 用户可以使用他的现有投资组合，借出ETH进行项目的ICO;
3. 交易者，可以借出代币，并在交易所抛售，进而获利（做空）；

在Compound上，用户利用空闲资产，赚钱收益，同时为借贷需求方，提供支持。

Compound III 最核心的改动在于放弃了其首创的「全池风险模型」（pooled-risk model），改为根据基础资产的不同将各个资产池隔离开来。这里的「基础资产」是一个相对于「抵押资产」的概念，与旧版本（Compound v2）不同，Compound III 在新的协议中对这两个概念做了很清楚的划分。

具体来说，在 Compound v2 之中，由于协议允许用户自由存入（抵押）或借出所有已支持的资产，因此这些资产既可被视为「基础资产」，也可以被视为「抵押资产」。如下图所示，以太坊主网上的 Compound v2 现已支持 ETH、COMP、USDC、USDT 、DAI 等 17 种资产，用户可以在这 17 种资产内自由选择存入（抵押）某种代币，再自由借出另一种代币，具体借出哪种代币不受限制，甚至还可以将借出的代币再抵押一遍去借出其他代币……总而言之，这些不同的资产池在借贷规则内是打通的。


整体来看，Compound III 几乎是对整个协议的运转机制做了一次再设计。Leshner 称新版本更强调安全性、资本效率和用户体验。前两点其实非常清晰，Compound III 的资产池隔离模型有效阻隔了系统性风险的蔓延，同时独立设置各池参数的新机制也赋予了协议更灵活、更高效的资产利用可能性，至于用户体验这一点，操作层面 Compound III 其实并没有太大改变，熟悉 DeFi 借贷的用户基本看一眼就能上手。



# 目录
* [借贷协议](#借贷协议) 
    * [Interest Rate Model](#Interest-rate-model) 
    * [协议实现](#v协议实现) 
        * [CToken Contracts](#ctoken-contracts) 
        * [Interest Rate Mechanics](#interest-rate-mechanics) 
        * [Borrower Dynamics](#borrower-dynamics) 
        * [Borrowing](#borrowing) 
        * [Liquidation](#liquidation) 
        * [Price Feeds](#price-feeds) 
        * [Comptroller](#comptroller) 
        * [Governance](#governance) 
        * [Summary](#summary) 
* [治理](#治理) 
* [Compound体验](#compound体验) 
* [协议实现](#v协议实现) 
* [合约架构](#合约架构) 
    * [借贷合约](#借贷合约) 
    * [治理合约](#治理合约) 
    * [open-oracle](#open-oracle) 
* [Compound借贷资金及治理代币流转模型](#Compound借贷资金及治理代币流转模型) 
* [总结](#总结)     
* [附](#附) 


# 借贷协议

提供者提供资金，将会放入资产总池，同时会获得相应的ERC20 cToken；cToken可以作为收益的分配和提供者的体现依据；

借贷者抵押需要的资产进行借贷；高流动性，高价值的借贷抵押利率高，反之，贷抵押利率；

借贷方，可以借出不超过其资产的贷款，此举为了降低协议面临的风险；

borrowing capacity（借贷容量）：

collateral factors:（抵押因子）
liquidation discount：（清算折扣）

当前用户借出了超其资产的贷款时，将会面临系统清算，以市场的liquidation discount（清算折扣）和用户的cToken进行清算，超额的借贷；降低用户和协议的风险；
任何EOA账户都可以发起清算；

close factor：超额借贷时，需要偿还的贷款因子；

## Interest Rate Model

借贷需求越高，则利率越高，否则越低；

使用率：
U·a = Borrow·sa / (Cash·a + Borrow·sa)


Borrow·sa：借出的贷款
Cash·a：流动池中的现金

借贷率
Borrowing Interest Rate·a = 2.5% + U·a * 20%

协议不通过流动性保证，只通过借贷利率来保证，在极端情况下，高额的借贷利率将会激励提供现金供应，同时抑制借贷需求；


## 协议实现
所有借贷的实现都是基于ERC20的cToken合约

### CToken Contracts
用户提供的任何资产价格将会转换为基于ERC20的cToken；用户可以使用mint(uint
amountUnderlying) ，通过现金资产到市场获取cToken，使用 cToken的redeem(uint amount)赎回操作，获取自己的现金资产；


汇率exchangeRate:

exchangeRate = （underlyingBalance +totalBorrowBalance·a − reserves·a）/cTokenSupply·a
随着市场借款金额的增加totalBorrowBalance·a， 汇率将会在底层资产underlyingBalance和cTokenSupply之间进行增加；


> 合约abi

1. mint(uint256 amountUnderlying) ：转移底层资产到市场，获取相应的cToken；
2. redeem(uint256 amount)/redeemUnderlying(uint256 amountUnderlying):从市场赎回底层资产，并减少用户的cToken数量；
3. borrow(uint amount) :从市场借出抵押物允许的底层资产，并更新账户借贷余额；
4. repayBorrow(uint amount)/repayBorrowBehalf(address account, uint amount):还贷底层资产到时长，减少账户的借款金额；
5. liquidate(address borrower, address collateralAsset, uint closeAmount)：清算底层资产到市场，，减少账户的借款金额，转移借贷者相应的cToken到清算发起者；



### Interest Rate Mechanics 
利率随着借贷需求的变化而变化，
Interest Rate Index将会管理基于时间历史的利率，每次计算利率，将会影响资产的供应，借贷，偿还，赎回和清算；

每次交易发生， Interest Rate Index将会更新Compound的每个间隔索引的利率；


区块利率：
Index.a,n = Index.a,(n−1) *(1 + r *t)

市场借贷总额：

totalBorrowBalance.a,n = totalBorrowBalance.a,(n−1) *(1 + r *t)

借贷产生的利息reserves.a 将会作为储备金使用，具体计算如下：

reserves.a = reserves.a,(n−1)+ totalBorrowBalance.a,(n−1) *(r * t * reserveFactor)

reserveFactor:储备系数


### Borrower Dynamics
随着利率的变化，用户的可借贷金额随之变化，同时以tuple <uint256 balance, uint256 interestIndex> 进行维护；


### Borrowing
用户借贷将会检查用户的抵押物是否充足，另外放贷后，将会触发市场利率的更新；同时用户可以在任何适合偿还贷款；


### Liquidation
当抵押品市场价值降低，或者用户的借贷超额，将会触发清算，清算者将会获取清算借贷额度相应的cToken；


### Price Feeds
Compound协议将会代理委员会使用Price Oracle维护每个资产的汇率

### Comptroller
Compound协议针对特定的资产默认不支持，只有白名单中的才支持， 用户可以使用管理功能supportMarket(address market,address interest rate model)，添加
资产到市场，为了能够借出资产，资产必须从Price Oracle可以拉出对应有效价格；同时为了能够抵押，必须有一个有效的价格和抵押因子；

每个方法将会通过策略层Comptroller校验，Comptroller在用户的每个动作之前，将会校验抵押品和流动性
## Governance
Compound将会先使用集中化的管理，控制利率的模型等，在将来将会交给社区和权益股东；管理员可以拥有一下权利：
1. list新的cToken市场；
2. 更新每个市场的利率模型；
3. 更新Oracle地址；
4. 从储备的cToken体现；
5. 任命新的admin，比如社区控制的DAO；

## Summary

* Compound提供了ETH资产的市场化流动；
* 每个市场的利率有供需关系决定，借出资产，利率上升，赎回资产，利率上升，激励额外的流动性；
* 用户可以在依赖于中心化组织的情况，从市场上获取收益；
* 用户可以使用从Compound中借出来的资产进行使用，售卖或者再借贷。

# 治理
Compound 计划发行 1000 万个 COMP 治理代币，其中 42%（即 423 万个 COMP）分配给 Compound 用户；其余分配给投资人、开发团队和待定用途。
面向用户的 423 万个 COMP 将进行免费分发，只要大家使用 Compound 进行存款或借贷就能获得一定比例的 COMP，存贷金额越大，获得的 COMP 越多。

> 具体分配规则:  

423 万个 COMP 被放置在「蓄水池」智能合约中，并且将以每个以太坊区块释放 0.5 个 COMP 的速度发行 （每天约 2880 个 COMP），这也就意味着需要 4 年的时间才会全部分发完；
COMP 将被分配至 Compound 的每个借贷市场中（ETH、USDC、DAI 等），以各市场的利息确定配比，这也就意味着分配比例会随时变化；
在每个市场中，50% 的 COMP 会分配给存款人，50% 的 COMP 分配给借款人，用户可以根据自己资产在所在市场内的占比获得相应比例的 COMP

![compoud-comp-distribute](/image/compound/compoud-comp-distribute.png) 

# Compound体验

进入app，可以看到市场资产的供应和借贷情况
![compound-app](/image/compound/compound-app.png) 

我们选择以USDC作为底层资产的借贷市场，抵押和借贷情况
![compound-market](/image/compound/compound-market.png) 


同时可以进行相应的供应或者借贷操作
![compound-supply-and-usdc](/image/compound/compound-supply-and-usdc.png) 


# 合约

![compound-contract-framework](/image/compound/compound-contract-framework.png) 

Compound整体包括2大部分协议合约和开放Oracle；协议合约包括借贷合约和社区治理合约。

## 借贷合约

* InterestRateModel（利率模型）：提供借贷率和供应率的计算；具体有WhitePaperInterestRateModel，JumpRateModelV2，DAIInterestRateModelV3利率模型；
* CToken：提供挖取、借贷，偿还，赎回，清算等核心操作， 每个核心操作发生时，都会重新计算利率；计算利率时，并借贷产生的利率会算到新的借贷总额和储备新上；
清算时，会将借贷的抵押资产，扣押一部分给清算者，一部分作为新的储备金，同时CToken总供应量减少抵押扣留的Token数量。除核心操作之外，提供利率模型的设置，
管理变更, 控制器设置，储备因子的设置，管理员增加减少现金储备等管理操作；

1. CEther：ETH的CToken
2. CErc20：EIP20的CToken

* CErc20Delegator：CErc20代理合约，通过delegateCall，调用底层CErc20合约实现implementation；
* PriceOracle（价格预言机）：提供资产价格的查询；SimplePriceOracle,UniswapAnchoredView;
* Comptroller：提供mint，借贷，偿还，清算，扣押相关检查操作及治理代币的Comp分配和claim（转给用户Comp），同时提供用户的市场准入和退出，核心操作的紧急暂停，监管者和管理员可以暂停，恢复只能是管理员；在mint和借贷及偿还操作时会根据当前的供应和借贷飞轮索引，分配相应的COMP。扣押会根据抵押资产的供应索引，给借贷者和清算者分配COMP；



## 治理合约

Compound有2个版本的治理合约，分别是Alpha和Bravo。治理合约和时间锁timelock配合使用，以提升治理的安全性。治理代币是Comp，用户持有Comp代币即拥有投票权，也可以将投票权委托给别人。


* Comp（治理代币）：类ERC20投票代币，并提供授权委托投票操作；
* GovernorBravoDelegate：提供发起提案，投票等提案治理核心操作；
* Timelock:治理提案时间锁，管理投票成功的提案，提供添加提案到队列，取消，执行提案核心操作；

整个提案治理模型如下：

![Compound-dao-model](/image/compound/Compound-dao-model.png) 

提案成功后将会，添加到Timelock的提案队列中，在eta时间到达时，并且未到eta+14d时，才可以执行提案，使用Timelock主要是项目方放弃控制权为社区提供更多保障。



## open-oracle
在open-oracle仓库，提供了基于UniswapV2的价格预言机：

* OpenOraclePriceData：维护各种代币价格；
* UniswapAnchoredView：提供基于UniswapV2的价格抓取等操作；



# Compound借贷资金及治理代币流转模型
![compound-market-model](/image/compound/compound-market-model.png) 

底层资产：现金池，借贷池，储备池；  
CToken：借贷市场；  
COMP：治理；

> 借贷核心操作：

1. 挖取：用户供应底层资产，获取CToken，增加供应量，转给底层资产给CToken合约，获取供应相应的COMP；
2. 借贷：使用用户抵押资产，获取资产资产贷款，从CToken合约转给借贷者，获取借贷相应的COMP；
3. 赎回：使用CToken赎回对应的底层资产，减少CToken供应量，转换底层资产给赎回者；
4. 偿还：从偿还者，转账底层资产到CToken合约，获取借贷相应的COMP；
5. 清算：首先给借贷者偿还贷款，从清算者，转账底层资产到CToken合约，转移借贷者相应的CToken给用户，并扣押借贷一部分的底层资产作为储备金，减少相应的CToken供应量，清算者，借贷者获取相应供应COMP；


# 总结
Compound基于ERC20的cToken为用户提供的去中心化的借贷及存款，主要包括提供资产，借贷，偿还，赎回，清算关键操作。

提供者提供资金，将会放入资产总池，同时会获得相应的ERC20 cToken；cToken可以作为收益的分配和提供者的体现依据；

用户借贷将会检查用户的抵押物是否充足，另外放贷后，将会触发市场利率的更新；同时用户可以在任何适合偿还贷款；

随着利率的变化，用户的可借贷金额随之变化，同时以tuple <uint256 balance, uint256 interestIndex> 进行维护；

当抵押品市场价值降低，或者用户的借贷超额，将会触发清算，清算者将会以市场的liquidation discount（清算折扣）进行清算，获取清算借贷额度相应的cToken；
以降低用户和协议的风险，任何EOA账户都可以发起清算；


Compound协议针对特定的资产默认不支持，只有白名单中的才支持， 用户可以使用管理功能supportMarket(address market,address interest rate model)，添加
资产到市场，为了能够借出资产，资产必须从Price Oracle可以拉出对应有效价格；同时为了能够抵押，必须有一个有效的价格和抵押因子；

Compound的作用及关键特性、机制：

* Compound提供了ETH资产的市场化流动；
* 每个市场的利率有供需关系决定，借出资产，利率上升，赎回资产，利率上升，激励额外的流动性；
* 用户可以在依赖于中心化组织的情况，从市场上获取收益；
* 用户可以使用从Compound中借出来的资产进行使用，售卖或者再借贷。

Compound III 最核心的改动在于放弃了其首创的「全池风险模型」（pooled-risk model），改为根据基础资产的不同将各个资产池隔离开来。这里的「基础资产」是一个相对于「抵押资产」的概念，与旧版本（Compound v2）不同，Compound III 在新的协议中对这两个概念做了很清楚的划分。

具体来说，在 Compound v2 之中，由于协议允许用户自由存入（抵押）或借出所有已支持的资产，因此这些资产既可被视为「基础资产」，也可以被视为「抵押资产」。如下图所示，以太坊主网上的 Compound v2 现已支持 ETH、COMP、USDC、USDT 、DAI 等 17 种资产，用户可以在这 17 种资产内自由选择存入（抵押）某种代币，再自由借出另一种代币，具体借出哪种代币不受限制，甚至还可以将借出的代币再抵押一遍去借出其他代币……总而言之，这些不同的资产池在借贷规则内是打通的。


整体来看，Compound III 几乎是对整个协议的运转机制做了一次再设计。Leshner 称新版本更强调安全性、资本效率和用户体验。前两点其实非常清晰，Compound III 的资产池隔离模型有效阻隔了系统性风险的蔓延，同时独立设置各池参数的新机制也赋予了协议更灵活、更高效的资产利用可能性，至于用户体验这一点，操作层面 Compound III 其实并没有太大改变，熟悉 DeFi 借贷的用户基本看一眼就能上手。

Compound有2个版本的治理合约，分别是Alpha和Bravo。治理合约和时间锁timelock配合使用，以提升治理的安全性。治理代币是Comp，用户持有Comp代币即拥有投票权，也可以将投票权委托给别人。



# 附
[compound](https://compound.finance/) 
[compound-protocol](https://github.com/compound-finance/compound-protocol) 
[Compound Whitepaper](https://compound.finance/documents/Compound.Whitepaper.pdf)   
[CompoundProtocol](https://github.com/compound-finance/compound-protocol/blob/master/docs/CompoundProtocol.pdf)      
[DeFi 中的借贷及 Aave、Compound 的比较](https://www.tuoluo.cn/article/detail-10035172.html)   
[DeFi借贷平台两大龙头项目Compound和AAVE全面解析](https://www.panewslab.com/zh/articledetails/1606214769415477.html)  
[【DeFi技术解析】去中心化算法银行Compound技术解析之概述篇](https://zhuanlan.zhihu.com/p/114319666)   
[【DeFi技术解析】去中心化算法银行Compound技术解析之利率模型篇](https://zhuanlan.zhihu.com/p/126503548)   
[【DeFi 的世界】Compound 完全解析-利率模型篇](https://medium.com/steaker-com/defi-%E7%9A%84%E4%B8%96%E7%95%8C-compound-%E5%AE%8C%E5%85%A8%E8%A7%A3%E6%9E%90-%E5%88%A9%E7%8E%87%E6%A8%A1%E5%9E%8B%E7%AF%87-95e9b303c284)        
[详解 Compound 运作原理](https://learnblockchain.cn/article/1015)       
[什么是DeFi 中的Compound Finance？](https://academy.binance.com/zt/articles/what-is-compound-finance-in-defi)    
[Compound应用架构](https://learnblockchain.cn/article/4158)  
[剖析DeFi借贷产品之Compound：概述篇](https://mp.weixin.qq.com/s/6W81W9mz5hYCoTvJ9S5-9Q)          
[剖析DeFi借贷产品之Compound：合约篇](https://juejin.cn/post/6974005248947929124)     
[剖析DeFi借贷产品之Compound：Subgraph篇](https://learnblockchain.cn/article/2632)  
[剖析DeFi借贷产品之Compound：清算篇](https://mirror.xyz/0x546086AfA3D285aCD2c84783c2dCf8F2C23b6433/qdqHZGPih7gXdderdPtZaqloeedTVpgCUdpgUGMGGTk)      
[剖析DeFi借贷产品之Compound：延伸篇](https://mirror.xyz/0x546086AfA3D285aCD2c84783c2dCf8F2C23b6433/yYi562kzBNUSgcuZKbN0M_hGXtNpb_Su0X6kDuAC8kY)    
[Compound III 上线，有哪些升级和改动？](https://foresightnews.pro/article/detail/12592)   
[如何使用Compound领取COMP币？免费领取COMP币](https://www.112btc.com/baike/wakuang/4621.html)     
[如何免费领取 Compound 治理代币：COMP](https://zhuanlan.zhihu.com/p/148789070)    
[Expanding Compound Governance](https://medium.com/compound-finance/expanding-compound-governance-ce13fcd4fe36)    
[Compound学习——DAO治理](https://mirror.xyz/rbtree.eth/OQEHK-Zrmz2RZZ47KE--UjyjLVetE4yNzH1jQ6KWvKM)     
[Compound governance](https://docs.compound.finance/v2/governance/)  

