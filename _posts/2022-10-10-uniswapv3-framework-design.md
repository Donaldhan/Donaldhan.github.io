---
layout: page
title: 以太坊的黑丝袜UniswapV3
subtitle: 以太坊的黑丝袜UniswapV3
date: 2022-10-10 23:37:00
author: Ravitn
catalog: true
category: solidity
categories:
    - solidity
tags:
    - Uniswap
---


# 引言
Uniswap协议是一个用来在以太坊区块链上交易加密货币（ERC-20代币）的点对合约系统。这个协议通过一个持久化、不可更改的智能合约集合来实现，旨在优先考虑抗审查性、安全性、自我监管，
以及在没有任何可能有选择地限制访问的可信中介的情况下运行。简单点说就是通过智能合约实现了一个去中心化的ERC-20代币的自动交易系统。V2版本基于恒定乘积公式的自动做市商（AMM）交易机制，
由于v2所有的交易pair全部放在一个交易池中进行管理，不便于精细化的流动性管理，同时基于恒定乘积公式的自动做市商（AMM）交易机制存在资金利用效率低的问题，uniswap团队，2020年5月推出
v2之后的一年后，推出的v3，提供集中流动性机制，提供资金的利用效率，同时增强的oralce，用户不用根据历史区块的价格信息自己计算价格，使用合约的oracle功能，就可以获取基于TWAP的几何算法（v2为算术平均数）价格，今天我们一起来看一下v3新特性及相应合约功能架构。


# 目录
* [新特性](#新特性) 
* [架构变更](#架构变更) 
    * [ORACLE UPGRADES](#oracle-upgrades) 
* [uniswap体验](#uniswap体验) 
* [集中流动性实现](#集中流动性实现) 
* [v3合约架构](#v3合约架构) 
    * [v3-core](#v3-core) 
    * [v3-periphery](#v3-periphery) 
* [总结](#总结)     
* [附](#附) 


# 新特性
* 集中流动性：集中流动性提高资金的利用效率，提高流程，避免类似V2资金池中的交易对，出现极端的情况，导致池流动性见底；
指定报价区间，及区间步长；流动性提供者可以提供一个武断的价格区间；
* 灵活的费用：0.05%, 0.30%, and 1%. 流动池创建者，可以指定费用，同时UNI governance可以添加其他的费用到费用集；
* Improved Price Oracle：提供用户查询最近价格，不依赖与TWAP（ time-weighted average price (TWAP)）的checkpoint值；
* Liquidity Oracle:提供基于时间的平均流动性预言机合约；


# 架构变更
* 多流动池模式，每个交易对可以一个池，也可以多个交易对多个池；V2所有的交易对都在一个池；
* 交易token不在单单支持ERC20， 拓展至非同质化token； 在Uniswap periphery创建交易池，可以对ERC20进行包装；
* governance：每个池有一个owner，可以支持tick space，同时可以设置没有个tick的fee，一旦设置不可改变；
* price oracle升级；

## ORACLE UPGRADES
v2用户如果需要计算某个period的TWAP，需要追踪从period开始到结束的checkpoint，V3中不在需要追踪检查点，
可以获取最近period的TWAP，V3会log这些price的checkpoint，允许用户计算TWAP算术平均值；

Oracle Observations：在v2只会保存最近区块的价格检查点，用户需要机制拉取以前区块的价格点进行统计。v3中，所有的价格点将会保存一个环形价格检查点中，
，环形中最大可以容纳65,536 checkpoints，当新的检查点产生时，环形中的检查点没有slot，则老的slot的将会被覆盖；

Geometric Mean Price Oracle：V2中，交易对的token是单独跟踪的，没有关联性，V3将会根据交易对的在tick中随时间的token price ratio，计算相应的TWAP；

Liquidity Orace：v3同时每个区块基于秒级的seconds-weighted accumulator, 用于分配流动性奖励；


# uniswap体验
交易池，我们以ETH和USDT为例，进入交易池界面
![uniswap3-pool](/image/uniswapv3/uniswap3-pool.png)

点击添加流动性，将会跳到添加流动性页面，点击交易进入swap操作界面

* 添加流动性
![uniswap3-pool-liquidty-add](/image/uniswapv3/uniswap3-pool-liquidty-add.png)
选择交易池关联token， 我们选择为eth和swap，并选择流动性价格区间（兑换率）


* 交易操作（swap）
![uniswapv3-swap](/image/uniswapv3/uniswapv3-swap.png)
输入给定的eth,根据当前交易池价格swap出相应的USDT。兑换时，前端将会拉取当前交易池的最优交易价格（oracle的TWAP），进行swap操作。


* 交易列表
![uniswap3-pool-swap-transactions](/image/uniswapv3/uniswap3-pool-swap-transactions.png)


# 集中流动性实现

我们先来看一下v2的AMM DEX曲线，为下图中的绿色部分。
![uniswap3-real-reservers](/image/uniswapv3/uniswap3-real-reservers.png)

在uniswap v2，流动性价格可以无限大，当价格趋向于曲线两端的无穷大是，流动性将会减少，没有人愿意使用超过的价格，swap出token，这将导致这个
交易池的资金利用率不高，为了改善这种情况，v3，将允许用户添加给定价格区间的流动性，如上图中的黄线部分。这样整个交易池的流动流动性分布将会
如图：

![uniswap3-liquidity-distributions](/image/uniswapv3/uniswap3-liquidity-distributions.png)

提供不同深度的流动性，提供资金的利用效率；用户自己可以任意价格区间的流动性，将会存在精度问题：

> 价格精度问题
因为用户可以在任意 [P0,P1] 价格区间内提供流动性，Uniswap v3 需要保存每一个用户提供流动性的边界价格，即 P0 和 P1。这样就引入了一个新的问题，假设两个用户提供的流动性价格下限分别是 5.00000001 和 5.00000002，那么 Uniswap 需要标记价格为 5.00000001 和 5.00000002 的对应的流动性大小。同时当交易发生时，需要将 [5.00000001,5.00000002] 作为一个单独的价格区间进行计算。这样会导致：

几乎很难有两个流动性设置相同的价格边界，这样会导致消耗大量合约存储空间保存这些状态，
当进行交易计算时，价格变化被切分成很多个小的范围区间，需要逐一分段进行计算，这会消耗大量的 gas，并且如果范围的价差太小，可能会引发计算精度的问题，
Uniswap v3 解决这个问题的方式是，将 [Pmin,Pmax] 这一段连续的价格范围为，分割成有限个离散的价格点。每一个价格对应一个 tick，用户在设置流动性的价格区间时，只能选择这些离散的价格点中的某一个作为流动性的边界价格。

Uniswap v3 采用了等比数列的形式确定价格数列，公比为 1.0001。即下一个价格点为当前价格点的 100.01%;如此一来 Uniswap v3 可以提供比较细粒度的价格选择范围（每个可选价格之间的差值为 0.01%），同时又可以将计算的复杂度控制在一定范围内。


> v3核心思路

1. 将流动池的tick化，每个tick粒度，默认为0.01；pool将会追踪没有tick的每秒的sqrt价格；在初始化时，tick没有暂用的情况下，可以初始化；
2. pool初始化时，会设置tickSpacing，只有tickSpacing允许的范围内，才能添加到pool中，比如如果tickSpacing设为2,则(...-4, -2, 0, 2, 4...)形式的tick才可以初始化，用户添加流动性，
只能添加tick规则允许的价格区间；
3. 为了确保正确数量的流动性的加入和退出，pool合约将会追踪pool的全局状态，每个tick及每个位置的状态；

##  Global State


| Type | Variable Name | Notation |  
|  ----  | ----  |   
|  uint128 |  liquidity | 𝐿|   
|  uint160 |  sqrtPriceX96|  sqrt(𝑃) |   
|  int24 | tick |  𝑖𝑐 |  
|  uint256 |  feeGrowthGlobal0X128|  𝑓𝑔,0 |   
|  uint256 | feeGrowthGlobal1X128 |  𝑓𝑔,1 |   
|  uint128 | protocolFees.token0  | 𝑓𝑝,0 |  
|  uint128 | protocolFees.token1 |  𝑓𝑝,1 |   

```
pair(token x， token y) :

L=sqrt(xy);     
sqrt(p)=sqrt(y/x);    


x=L/sqrt(p);    
y=L/sqrt(p);    


tick(ic)= log(sqrt[basePrice]^sqrt[p]);
 ```

我们使用L和sqrt(p)跟踪流动性，主要是因为在每个tick，任何时间swap的产生，将会导致sqrt(p)的变动；

[𝑓𝑔,0/𝑓𝑔,1]为swap的全局token0, token1费用, 收费时，以(0.0001%)为基点进行收费；


[𝑓𝑝,0/𝑓𝑝,1]为协议token0, token1费用，具体到交易池；




## Tick-Indexed State

| Type | Variable Name | Notation |  
|  ----  | ----  |   
| int128 | liquidityNet |  Δ𝐿 |   
| uint128 | liquidityGross | 𝐿𝑔 |  
| uint256 | feeGrowthOutside0X128 | 𝑓𝑜,0 |  
| uint256 | feeGrowthOutside1X128 | 𝑓𝑜,1 |  
| uint256 | secondsOutside | 𝑠𝑜 |  
| uint256 | tickCumulativeOutside | 𝑖𝑜 |  
| uint256 | secondsPerLiquidityOutsideX128 | 𝑠𝑙o |  

liquidityNet(Δ𝐿):每个tick内的流动性；
liquidityGross(𝐿𝑔):用于判断当流动性不在给定的范围内时，是否需要更新ticks bitMap;
 [𝑓𝑜,0/𝑓𝑜,1]：用于追踪token0, token1在给你定范围外的fee；
secondsOutside, tickCumulativeOutside,secondsPerLiquidityOutsideX128：用于计算合约外部的更细粒度的收益；


##  Position-Indexed State

| Type | Variable Name | Notation |  
|  ----  | ----  |     
| uint128  | liquidity | 𝑙 |   
| uint256  | feeGrowthInside0LastX128 | 𝑓𝑟,0 (𝑡0) |   
| uint256  | feeGrowthInside1LastX128  | 𝑓𝑟,1 (𝑡0) |   

liquidity (𝑙): 用于表示上次位置点的虚拟流动性；
 [𝑓𝑟,0 (𝑡0) / 𝑓𝑟,1 (𝑡0)]：用于计算token0, token1的uncollected fees；



解决集中流动性，涉及每个tick内的交易fee，已经跨tick的交易费用，比如可能大于tick的上限，也可能小于tick的下限，或者不在整个tick的bit map范围之内，需要计算相应的费用；
针对虚拟流动性，有些流动性不能反映从合约创建时的fee，我们称为uncollected fees，
我们通过Position-Indexed State可以计算相应的uncollected fees。



# v3合约架构

![uniswap-v3-contract-framework](/image/uniswapv3/uniswap-v3-contract-framework.png)

v3合约作最重要的两个模块交易池核心v3-core和外围的swap路由模块v3-periphery

## v3-core
提供交易池的创建，交易相关的核心操作，比如添加流动mint，swap操作等，管理价格tick map，用户的位置信息，交易池状态，及oralce， 主要有以下几个合约：

* IUniswapV3PoolImmutables：交易池常量，比如：工厂地址、交易pair， token0，token1的地址，及tickspacing和每个池的最大流动性；
* IUniswapV3PoolState：用户记录交易池的状态，比如tick的费用信息，以及tick的oracle追踪信息（observations）；
* IUniswapV3PoolActions:主要用于初始化交易池，提供与交易核心操作，比如mint，burn，swap，flash等操作以及记录交易池快照信息increaseObservationCardinalityNext；
* IUniswapV3PoolOwnerActions：提供设置交易协议费用和收集协议费用
* IUniswapV3Factory：创建交易池（交易池token pair，及tick交易费用）；
* UniswapV3Pool:提供mint，burn，swap，flash，及collect(销毁流动性之后提取自己资产及fee收益)等核心实现，另外还有oralce的观察点记录；
Oracle提供的价格信息，可以体用第三方DEX，进行清算；

> UniswapV3Pool维护者交易者池的状态Slot0, 当前流动性，token0和token1的当前累计费用，tick信息，tick bitMap信息，用户流动性位置信息，及oralce观察点信息；

1. 交易者池的状态Slot0：主要有当前价格sqrt(x*y)，tick，最近观察点索引，当前存储观察点数量，下次需要存储的观察点索引，协议费用以及交易池是否锁住；
2. tick信息: 维护每个tick的信息，主要有当前tick流动性，流动性网格数量，tick范围外的token0和token1的fee，相对于当前tick外的每个流动性单元运行时间seconds，当前tick外的花费总时间seconds；
3. tick bitMap信息：每个tick的状态等信息；
4. 用户流动性位置信息：用户在tick上线限之间的流动，token0和token1收益， 及流动性fee


![uniswap-v3-pool-liquity-tick-position](/image/uniswapv3/uniswap-v3-pool-liquity-tick-position.png)


每个交易池根据token0和token1的地址及交易费fee来创建，相同token0和token1，fee不同的，则会重建一个新的交易池；在同一个交易池，不同的用户可以添加自己的流动性价格区间位置，每个交易池
会将所有的用户位置价格区间分别以tick进行分割，交易池的流动所有的ticks使用TickBitMap进行管理；用户swap时将会用户的限制价格和交易池slot0状态价格，从交易池的TickBitMap中筛选出最优的tick的流动区间进行swap，如果tick的流动性区间属于某个用户，则将交易费直接给相应的用户，否则将费用平均分配给覆盖tick的位置的用户（???），并更新用户的位置费用信息。

如上图所示在同一个交易池内，用户A添加价格区间位置流动性Pab，流动性为400；用户B添加添加价格区间位置流动性Pcd，流动性为300；
在tick1到tick4（Pa-Pc）区间流动为用户A添加的流动性为400，在tick4到tick5（Pc-Pb）区间流动性为用户A，用户B流动性的叠加为700，在tick5-tick7（Pb-Pd）区间流动性为用户B提供的流动性为300；



## v3-periphery
提供swap相关的路由操作，并在没有swap相应的基于用户位置信息的NFT，如果mint操作的交易池不存在，则创建相应的交易池；主要有几个合约

* NonfungiblePositionManager:非同质位置管理器，提供流动mint，增加，减少操作，以及收益费用的收集；在mint流动性时，同时会创建基于位置的NFT；
* NonfungibleTokenPositionDescriptor:提供生成位置NFT描述URL；
* SwapRouter: swap操作：根据输入token，swap出尽可能大的另一个token；swap评估操作：根据输入的token数量，需要输入的最小另外一个token的数量，并支付相应的费用到收费账户；

针对用户swap的tokenIn和tokenOut交易池不存在时，uniswap前端将会生成相应的路径，委托给SwapRouter进行swap操作；
多路径的的情况编码为：token0+fee01+token1+fee12+token2+fee23+token3+fee34+token4+..., 这种是针对swap是如果没有对应的交易对pair，则从不同的
交易池进行swap， 比如使用token0，想swap token3，整个swap的路径为（token0+fee01+token1，token1+fee12+token2，token2+fee23+token3），使用token0从
pool01中swap出token1，使用swap出的token1从pool12中swap出token2， 使用swap出的token2从pool23中swap出token3； 具体路径编码如下：

![uniswap-v3-swap-path-encode](/image/uniswapv3/uniswap-v3-swap-path-encode.png)


* V3Migrator: 从V2流动性，迁移到V3，可以指定迁移流动性百分比，没有迁移的将会退回给；

> 合约工具库
1. Path：交易池路径工具，路径实际编码为token0地址+fee+token1地址  ；
2. PoolAddress:交易池地址工具，根据交易池key，包含token0地址，token1地址，fee，生成交易池地址；
3. PeripheryPaymentsWithFee：提供带续费费用的支付，支持eth和token；
4. PeripheryPayments：提供支付操作，支持eth和token；
5. Multicall：批量调用代理合约；


# 总结

uniswapV3主要是解决uniswapV2基于常量的AMM极端情况下的流动性不足的问题，提出基于tick的集中流程性，同时加入的交易池的概念；v2中所有的交易对在一个池中，v3可以自己使用交易pair，创建交易池池，自己设置交易费用，及流动性tick区间；并改善的oracle，用户不用自己
计算基于TWAP（ time-weighted average price (TWAP)），使用合约获取最近的period的TWAP，并允许用户计算TWAP算术平均值；


解决集中流动性，涉及每个tick内的交易fee，已经跨tick的交易费用，比如可能大于tick的上限，也可能小于tick的下限，或者不在整个tick的bit map范围之内，需要计算相应的费用。
V3通过 Global State和Tick-Indexed State来解决这些问题；

针对虚拟流动性，有些流动性不能反映从合约创建时的fee，我们称为uncollected fees，
V3通过Position-Indexed State可以计算相应的uncollected fees。

v3-core提供交易池的创建，交易相关的核心操作，比如添加流动mint，swap操作等，管理价格tick map，用户的位置信息，交易池状态，及oralce；

v3-periphery提供swap相关的路由操作，并在没有swap相应的基于用户位置信息的NFT，如果mint操作的交易池不存在，则创建相应的交易池。
UniswapV3Pool维护者交易者池的状态Slot0, 当前流动性，token0和token1的当前累计费用，tick信息，tick bitMap信息，用户流动性位置信息，及oralce观察点信息；
1. 交易者池的状态Slot0：主要有当前价格sqrt(x*y)，tick，最近观察点索引，当前存储观察点数量，下次需要存储的观察点索引，协议费用以及交易池是否锁住；
2. tick信息: 维护每个tick的信息，主要有当前tick流动性，流动性网格数量，tick范围外的token0和token1的fee，相对于当前tick外的每个流动性单元运行时间seconds，当前tick外的花费总时间seconds；
3. tick bitMap信息：每个tick的状态等信息；
4. 用户流动性位置信息：用户在tick上线限之间的流动，token0和token1收益， 及流动性fee

每个交易池根据token0和token1的地址及交易费fee来创建，相同token0和token1，fee不同的，则会重建一个新的交易池；在同一个交易池，不同的用户可以添加自己的流动性价格区间位置，每个交易池
会将所有的用户位置价格区间分别以tick进行分割，交易池的流动所有的ticks使用TickBitMap进行管理；用户swap时将会用户的限制价格和交易池slot0状态价格，从交易池的TickBitMap中筛选出最优的tick的流动区间进行swap，如果tick的流动性区间属于某个用户，则将交易费直接给相应的用户，否则将费用平均分配给覆盖tick的位置的用户（???），并更新用户的位置费用信息。


针对用户swap的tokenIn和tokenOut交易池不存在时，uniswap前端将会生成相应的路径，委托给SwapRouter进行swap操作；

多路径的的情况编码为：token0+fee01+token1+fee12+token2+fee23+token3+fee34+token4+..., 这种是针对swap是如果没有对应的交易池，则从不同的
交易池进行swap， 比如使用token0，想swap token3，整个swap的路径为（token0+fee01+token1，token1+fee12+token2，token2+fee23+token3），使用token0从
pool01中swap出token1，使用swap出的token1从pool12中swap出token2， 使用swap出的token2从pool23中swap出token3；





# 附
[uniswap pools](https://info.uniswap.org/#/pools)   
[Uniswap v2-core](https://github.com/Donaldhan/v2-core)     
[Uniswap v2-periphery](https://github.com/Donaldhan/v2-periphery)  
[Uniswap lib](https://github.com/Donaldhan/solidity-lib)    
[一文看懂Uniswap和Sushiswap](https://zhuanlan.zhihu.com/p/226085593)   
[Uniswap深度科普](https://zhuanlan.zhihu.com/p/380749685)    
[去中心化交易所：Uniswap v2白皮书中文版](https://zhuanlan.zhihu.com/p/255190320)   
[Uniswap v3 设计详解](https://zhuanlan.zhihu.com/p/448382469)   
[Uniswap V3 到底是什么鬼？一文带你了解V3新特性](https://zhuanlan.zhihu.com/p/359732262)  
[Uniswap v3 详解（一）：设计原理](https://liaoph.com/uniswap-v3-1/) 
[uniswap - V3源代码导读](https://learnblockchain.cn/article/2371) 
[Uniswap V3 白皮书](https://uniswap.org/whitepaper-v3.pdf) 
[uniswap-v3 blog](https://uniswap.org/blog/uniswap-v3/) 
[jit-liquidity](https://uniswap.org/blog/jit-liquidity)  
[graphical-guide-for-understanding-uniswap](https://docs.ethhub.io/guides/graphical-guide-for-understanding-uniswap/)    
[什么是DeFi中的闪电贷？](https://academy.binance.com/zh/articles/what-are-flash-loans-in-defi)    

