---
layout: page
title: 以太坊的黑丝袜UniswapV2
subtitle: 以太坊的黑丝袜UniswapV2
date: 2022-03-31 22:37:00
author: Ravitn
catalog: true
category: BlockChain
categories:
    - BlockChain
tags:
    - Uniswap
---



# 引言
Uniswap协议是一个用来在以太坊区块链上交易加密货币（ERC-20代币）的点对合约系统。这个协议通过一个持久化、不可更改的智能合约集合来实现，旨在优先考虑抗审查性、安全性、自我监管，以及在没有任何可能有选择地限制访问的可信中介的情况下运行。简单点说就是通过智能合约实现了一个去中心化的ERC-20代币的自动交易系统。本文不打算详细介绍自动做市商（AMM）的原理和经济激励机制， 相关介绍文章好多，大家从附录索引文章中了解。我们从合约实现角度，看一下uniswap的核心实现。


# 目录
* [核心机制](#核心机制) 
    * [自动做市商（AMM）的原理](#自动做市商（AMM）的原理) 
    * [经济激励机制](#经济激励机制) 
* [合约实现](#合约实现) 
    * [v2-core](#v2-core) 
    * [v2-periphery](#v2-periphery) 
    * [lib](#lib) 
* [总结](#总结)     
* [附](#附) 

# 核心机制
先简单介绍一下核心机制



## 自动做市商（AMM）的原理


Uniswap并不使用订单簿模式来决定代币的价格，相反，代币的价格（汇率）会在用户交易的过程中连续且自动地根据供需量和恒定乘积公式计算得出。恒定乘积公式就是：X*Y=K。其中X和Y分别代表交易池中两种代币的可用数量。配合恒定乘积公式，一个交易对（也即一个流动性兑换池）中的一种代币的价格，根据池中的供给量和交易者的需求量得出，价格会在根据该公式画出的一条曲线上变动。

X*Y=K，或者说Y=K/X，就是我们初中学过的反比例函数。当K不变时，X增大，Y必然减小，反之亦然。在DEX里，用户向兑换池中充入token X，X的数量增多了，Y的数量必然减小，而这减少的Y的数量就被用户取出，完成兑换。


## 经济激励机制

Uniswap给流动性提供者的奖励就是交易所的手续费。流动性提供者可以得到所在流动性池中代币交易的手续费作为奖励，手续费率为0.3%，流动性提供者之间依据存入资金的份额按比例分配。

Uniswap会根据流动性提供者存入资金的数额发给他们一定数量的LP token，我们可以理解为存款奖状或者收据，它是LP获得交易所手续费奖励的凭证。

注意：LP token不是Uniswap的项目代币或者说平台币，对于每一种代币交易对有不同的LP token。


我们再来看一下合约实现
# 合约实现
整体架构如下

![uniswap-v2-framwork](/image/uniswap/uniswap-v2-framwork.png)


* uniswap-v2-core ：核心合约的实现； 
* uniswap-v2-periphery ：提供了和 UniswapV2 进行交互的外围合约，主要是路由合约；
* uniswap-lib ：封装了一些工具合约


## v2-core
* UniswapV2ERC20：提供ERC20接口的功能  
* UniswapV2Factory：创建pair及手续费账户管理 
* UniswapV2Pair：提供token pair的流动性管理功能，包括挖取 
销毁、及交换功能；

## v2-periphery
* UniswapV2Router02：提供token之间，及token与ETH之间的swap功能，同时支持带付gas模式；移除流动性LP获取响应pair代币，及签名免授权gas模式；
* UniswapV2Library： token资产输入输出数量工具类，支持多pair path；

## lib

* TransferHelper: 转账交易工具类



我们再来看一些有趣的合约代码

先来看pair中的**lock**

UniswapV2Pair
```solidity
modifier lock() {
    require(unlocked == 1, 'UniswapV2: LOCKED');
    unlocked = 0;
    _;
    unlocked = 1;
}
```
同时进入lock修饰的方法，竞态是否存在？？？



**pair地址的创建**


UniswapV2Factory
```solidity
// 内联汇编
assembly {
    // 通过create2方法布置合约，并且加salt，返回合约的地址是固定的，可预测的
    pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
}
```
部署Pair合约使用的是create2方法，使用该方法部署合约可以固定这个合约的地址，使这个合约的地址可预测，这样便于Router合约不进行任何调用，就可以计算得到Pair合约的地址。

create2用 mem[p...(p + s)) 中的代码，在地址 keccak256(<address> . n . keccak256(mem[p...(p + s))) 上 创建新合约、发送 v wei 并返回新地址 




**eip712的实现**

UniswapV2ERC20
```
//https://eips.ethereum.org/EIPS/eip-712
//https://zhuanlan.zhihu.com/p/40596830
//定义DOMAIN_SEPARATOR方法，这个方法会返回[EIP712](EIP-712: Ethereum typedstructured data hashing and signing)所规定的DOMAIN_SEPARATOR值
DOMAIN_SEPARATOR = keccak256(
    abi.encode(
        keccak256('EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)'),
        keccak256(bytes(name)),
        keccak256(bytes('1')),
        chainId,
        address(this)
    )
);
```
 定义DOMAIN_SEPARATOR方法，这个方法会返回[EIP712](EIP-712: Ethereum typed structured data hashing and signing)所规定的DOMAIN_SEPARATOR值






**eip2612permit实现** 

UniswapV2ERC20
```
 // permit授权方法 该方法的参数具体含义可以查询[EIP2612](EIP-2612: permit 712-signed approvals)中的定义。
// 零gas以太坊交易实现原理及源码:https://zhuanlan.zhihu.com/p/269226515
// https://github.com/Donaldhan/ERC20Permit
//通过链下签名授权实现更少 Gas 的 ERC20代币:https://zhuanlan.zhihu.com/p268699937
//https://eips.ethereum.org/EIPS/eip-2612
// https://github.com/makerdao/dss/blob/master/src/dai.sol
// 用户线下签名，授权代理服务商线上授权（服务商需要线上数据验证，用户需要使用相同方进行线上验证）
function permit(address owner, address spender, uint value, uint deadline,uint8 v, bytes32 r, bytes32 s) external {
    //大于当前时间戳
    require(deadline >= block.timestamp, 'UniswapV2: EXPIRED');
    //abi.encodePacked(...) returns (bytes)：对给定参数执行 紧打包编码
    //
    bytes32 digest = keccak256(
        abi.encodePacked(
            '\x19\x01',
            DOMAIN_SEPARATOR,
            keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, nonces[owner]++, deadline))
        )
    );
    //ecrecover(bytes32 hash, uint8 v, bytes32 r, bytes32 s) returns (address)：基于椭圆曲线签名找回与指定公钥关联的地址，发生错误的时候返回 0
    address recoveredAddress = ecrecover(digest, v, r, s);
    require(recoveredAddress != address(0) && recoveredAddress == owner, 'UniswapV2: INVALID_SIGNATURE');
    _approve(owner, spender, value);
}
```
permit授权方法 该方法的参数具体含义可以查询[EIP2612](EIP-2612: permit – 712-signed approvals)中的定义。用户线下签名，授权代理服务商线上授权（服务商需要线上数据验证，用户需要使用相同方法进行线上验证）, 用户可以零gas交易，实际转嫁给服务商；


以上是一些个人感觉有趣和有意思的代码段。


# 总结
本文从简单介绍了uniswap2的核心机制，及合约架构，并摘取了一些有趣的代码片段，功能品读。MD，只能写这么多了，说说最近的情况下，最近一年多很少出文章了，自动转到blockchain core tech group， 有一些敏感的信息，不便于分享，同时去年贼累，当DOG usage， haha。最近在学习uniswap，敏感度不高，分享给大家。


# 附

[Uniswap v2-core](https://github.com/Donaldhan/v2-core)     
[Uniswap v2-periphery](https://github.com/Donaldhan/v2-periphery)  
[Uniswap lib](https://github.com/Donaldhan/solidity-lib)    
[一文看懂Uniswap和Sushiswap](https://zhuanlan.zhihu.com/p/226085593)   
[Uniswap深度科普](https://zhuanlan.zhihu.com/p/380749685)    
[快速了解 Uniswap-v2](https://zhuanlan.zhihu.com/p/463229465)   
[去中心化交易所：Uniswap v2白皮书中文版](https://zhuanlan.zhihu.com/p/255190320)   



