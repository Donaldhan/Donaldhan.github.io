---
layout: page
title: 合约库模式实战分析 
subtitle: 合约库模式实战分析 
date: 2022-09-06 21:59:39
author: ravitn
catalog: true
category: solidity
categories:
    - solidity
tags:
    - solidity
---


#  引言
库的两种方式：1.单独一个合约逻辑合约引用；2.在合约中包含库；库是一个特殊的合约；
在部署合约的时候，需要链到对应的库合约，有文章说使用库可以节省gas，真的这样吗，今天我们就来看一下：

# 目录
* [库使用模式](#库使用模式) 
    * [导入库模式合约](#导入库模式合约) 
    * [包含库模式合约](#包含库模式合约) 
* [实战](#实战) 
    * [导入模式测试](#导入模式测试) 
    * [包含模式测试](#包含模式测试) 
    * [实战分析](#实战分析) 
* [总结](#总结)     
* [附](#附) 


# 库使用模式

## 导入库模式合约
* 库合约

```solidity
pragma solidity ^0.8.0;
library Adder{

    function add(uint[] memory seeds) public pure returns(uint)
    {    
        uint sum = 0;
        for(uint i = 0; i<seeds.length; i++)
        {
            sum += seeds[i];
        }
        return sum;
    }
    
}
```

* 逻辑合约

```solidity
import "./Adder.sol";
pragma solidity ^0.8.0;

contract AdderImport{
    uint256 private total;
    //function to access the library
    function sum(uint[] memory data)external pure returns(uint)
    {
       uint sum;
       sum = Adder.add(data);
       return sum;
    }
     function totalSum(uint[] memory data) external returns(uint)
    {
       total = Adder.add(data);
       return total;
    }
    
}


```

## 包含库模式合约


```solidity

pragma solidity ^0.8.0;
library AdderX{

    function add(uint[] memory seeds) public pure returns(uint)
    {    
        uint sum = 0;
        for(uint i = 0; i<seeds.length; i++)
        {
            sum += seeds[i];
        }
        return sum;
    }
    
}
contract AdderInclude{
    uint256 private total;
    //function to access the library
    function sum(uint[] memory data)external pure returns(uint)
    {
       uint sum;
       sum = AdderX.add(data);
       return sum;
    }
       function totalSum(uint[] memory data) external returns(uint)
    {
       total = AdderX.add(data);
       return total;
    }
}

```


# 实战


## 导入模式测试


```
 describe("Library-test", function() {
    it("AdderImport Test error", async function () {
        const [owner, ravitn] = await ethers.getSigners();
        console.log("owner:", owner.address);
        const Adder = await ethers.getContractFactory("Adder");
        const adder = await Adder.deploy();
        await adder.deployed();
        console.log("adder address:", adder.address);
        const AdderImport = await ethers.getContractFactory("AdderImport", {
            libraries: {
                Adder: adder.address,
            },
          });
        const adderImport = await AdderImport.deploy();
        await adderImport.deployed();
        console.log("adderImport address:", adderImport.address);
        const txSum = await adderImport.sum([1,2]);
        console.log("txSum:", txSum);
        const total = await adderImport.totalSum([1,2]);
        // console.log("total:", total);
        expect(await adderImport.sum([1,2])).to.equal(3);
    });
});
```

## 包含模式测试


```
describe("Library-test", function() {
    it("AdderInclude Test error", async function () {
        const [owner, ravitn] = await ethers.getSigners();
        console.log("owner:", owner.address);
        const AdderX = await ethers.getContractFactory("AdderX");
        const adderX = await AdderX.deploy();
        await adderX.deployed();
        console.log("adderX address:", adderX.address);
        const AdderInclude = await ethers.getContractFactory("AdderInclude", {
            libraries: {
                AdderX: adderX.address,
            },
        });
        const adderInclude = await AdderInclude.deploy();
        await adderInclude.deployed();
        console.log("adderInclude address:", adderInclude.address);
        const txSum = await adderInclude.sum([1,2]);
        console.log("txSum:", txSum);
        const total = await adderInclude.totalSum([1,2]);
        // console.log("total:", total);
        expect(await adderInclude.sum([1,2])).to.equal(3);
    });
});
```

## 实战分析

使用hardhat，测试结果无论如何，库都要单独部署，链到对应的合约中。


我们来看一下AdderImport的MetaData JSON文件


```json
{
  "_format": "hh-sol-artifact-1",
  "contractName": "AdderImport",
  "sourceName": "contracts/library/AdderImport.sol",
  "abi": [
    {
      "inputs": [
        {
          "internalType": "uint256[]",
          "name": "data",
          "type": "uint256[]"
        }
      ],
      "name": "sum",
      "outputs": [
        {
          "internalType": "uint256",
          "name": "",
          "type": "uint256"
        }
      ],
      "stateMutability": "pure",
      "type": "function"
    }
  ],
  "bytecode": "0x608060405234801561001057600080fd5b5061024e806100206000396000f3fe608060405234801561001057600080fd5b506004361061002b5760003560e01c80630194db8e14610030575b600080fd5b61004361003e3660046100e6565b610055565b60405190815260200160405180910390f35b60008073__$fd5d5809563fd152781d739c4c1c9489bf$__6310b0b5d5846040518263ffffffff1660e01b815260040161008f91906101be565b60206040518083038186803b1580156100a757600080fd5b505af41580156100bb573d6000803e3d6000fd5b505050506040513d601f19601f820116820180604052508101906100df91906101a6565b9392505050565b600060208083850312156100f8578182fd5b823567ffffffffffffffff8082111561010f578384fd5b818501915085601f830112610122578384fd5b81358181111561013457610134610202565b8060051b604051601f19603f8301168101818110858211171561015957610159610202565b604052828152858101935084860182860187018a1015610177578788fd5b8795505b8386101561019957803585526001959095019493860193860161017b565b5098975050505050505050565b6000602082840312156101b7578081fd5b5051919050565b6020808252825182820181905260009190848201906040850190845b818110156101f6578351835292840192918401916001016101da565b50909695505050505050565b634e487b7160e01b600052604160045260246000fdfea2646970667358221220b3bfe6bda8a933689f9df80b6ebdf2a9601df4c5c02913c70c11204f1554f96664736f6c63430008040033",
  "deployedBytecode": "0x608060405234801561001057600080fd5b506004361061002b5760003560e01c80630194db8e14610030575b600080fd5b61004361003e3660046100e6565b610055565b60405190815260200160405180910390f35b60008073__$fd5d5809563fd152781d739c4c1c9489bf$__6310b0b5d5846040518263ffffffff1660e01b815260040161008f91906101be565b60206040518083038186803b1580156100a757600080fd5b505af41580156100bb573d6000803e3d6000fd5b505050506040513d601f19601f820116820180604052508101906100df91906101a6565b9392505050565b600060208083850312156100f8578182fd5b823567ffffffffffffffff8082111561010f578384fd5b818501915085601f830112610122578384fd5b81358181111561013457610134610202565b8060051b604051601f19603f8301168101818110858211171561015957610159610202565b604052828152858101935084860182860187018a1015610177578788fd5b8795505b8386101561019957803585526001959095019493860193860161017b565b5098975050505050505050565b6000602082840312156101b7578081fd5b5051919050565b6020808252825182820181905260009190848201906040850190845b818110156101f6578351835292840192918401916001016101da565b50909695505050505050565b634e487b7160e01b600052604160045260246000fdfea2646970667358221220b3bfe6bda8a933689f9df80b6ebdf2a9601df4c5c02913c70c11204f1554f96664736f6c63430008040033",
  "linkReferences": {
    "contracts/library/Adder.sol": {
      "Adder": [
        {
          "length": 20,
          "start": 122
        }
      ]
    }
  },
  "deployedLinkReferences": {
    "contracts/library/Adder.sol": {
      "Adder": [
        {
          "length": 20,
          "start": 90
        }
      ]
    }
  }
}

```

在deployedBytecode和bytecode有一段占位符：

```
05180910390f35b60008073__$fd5d5809563fd152781d739c4c1c9489bf$__6310b0b
```


字节码中的值 __$fd5d5809563fd152781d739c4c1c9489bf$__ 只是库地址的占位符。在您的情况下，占位符将介于 __$ 和 $__ 之间。它有点隐蔽——所以搜索 __$（下划线下划线美元符号）。
来自Hardhat的元数据 JSON 告诉Hardhat用给定地址替换占位符。，库的地址是在部署时注入的——所以在这个阶段用实际地址替换占位符。





库合合约的本质是什么？



出于这个目的，我们对linkReferences元素和object元素感兴趣。


```
linkReferences：（第一个元素）描述了合约使用的库。bytecode是编译后的合约（字节码）。这就是部署并保存到区块链上的内容。在此示例中，字节码中的值 __$83229fb62534ab89035722de277194ff6d$__ 只是库地址的占位符。
，占位符将介于 __$ 和 $__ 之间。它有点隐蔽——所以搜索 __$（下划线下划线美元符号）。

部署字节码deployedBytecode包含同样的占位符；
```


下面是使用remix测试的文章gas节省说明


```
库的地址是在部署时注入的——所以在这个阶段用实际地址替换占位符。如果我们只是将该库用作普通合约并在编译时将其导入，
我们将使用合约代码部署其代码——因此不会节省 gas。


通过挖掘 metadata.json 文件并更新其设置，我们可以使用库来减小合约的大小并享受节省燃料的乐趣。
来自 Remix IDE 的元数据 JSON 告诉 Remix 用给定地址替换占位符。
```


经测试发现两种模式的gas消耗是一样的，具体如下

* 导入模式消耗的Gas


```
----------------------------|---------------------------|-------------|-----------------------------·
|    Solc version: 0.8.4     ·  Optimizer enabled: true  ·  Runs: 200  ·  Block limit: 30000000 gas  │
·····························|···························|·············|······························
|  Methods                                                                                           │
················|············|·············|·············|·············|···············|··············
|  Contract     ·  Method    ·  Min        ·  Max        ·  Avg        ·  # calls      ·  chf (avg)  │
················|············|·············|·············|·············|···············|··············
|  AdderImport  ·  totalSum  ·          -  ·          -  ·      49055  ·            1  ·          -  │
················|············|·············|·············|·············|···············|··············
|  Deployments               ·                                         ·  % of limit   ·             │
·····························|·············|·············|·············|···············|··············
|  Adder                     ·          -  ·          -  ·     167482  ·        0.6 %  ·          -  │
·····························|·············|·············|·············|···············|··············
|  AdderImport               ·          -  ·          -  ·     219374  ·        0.7 %  ·          -  │
·----------------------------|-------------|-------------|-------------|---------------|-------------·
```

* 包含方式消耗的gas   


```
·-----------------------------|---------------------------|-------------|-----------------------------·
|     Solc version: 0.8.4     ·  Optimizer enabled: true  ·  Runs: 200  ·  Block limit: 30000000 gas  │
······························|···························|·············|······························
|  Methods                                                                                            │
·················|············|·············|·············|·············|···············|··············
|  Contract      ·  Method    ·  Min        ·  Max        ·  Avg        ·  # calls      ·  chf (avg)  │
·················|············|·············|·············|·············|···············|··············
|  AdderInclude  ·  totalSum  ·          -  ·          -  ·      49055  ·            1  ·          -  │
·················|············|·············|·············|·············|···············|··············
|  Deployments                ·                                         ·  % of limit   ·             │
······························|·············|·············|·············|···············|··············
|  AdderInclude               ·          -  ·          -  ·     219374  ·        0.7 %  ·          -  │
······························|·············|·············|·············|···············|··············
|  AdderX                     ·          -  ·          -  ·     167482  ·        0.6 %  ·          -  │
·-----------------------------|-------------|-------------|-------------|---------------|-------------·
```



从上面来看，两种方式没有太大的区别Gas消耗是一样的；



同时可以看出

> 逻辑合约调用库合约的方式:
* 针对修改逻辑合约存储的使用DELEGATECALL方式（Homestead之前是用CALLCODE），只能是这种方式，才能修改逻辑合约的存储。


我们使用remix部署包含库模式测试合约AdderInclude

```
creation of library tests/sad.sol:AdderX pending...
[vm]from: 0x5B3...eddC4to: AdderX.(constructor)value: 0 weidata: 0x610...50033logs: 0hash: 0xc1e...e5226
status	true Transaction mined and execution succeed
transaction hash	0xc1eed194c1fbef1f5b3ac9b7c9e8d066c72fae4c1d445ac25dd72af7dc6e5226
from	0x5B38Da6a701c568545dCfcB03FcB875f56beddC4
to	AdderX.(constructor)
gas	318992 gas
transaction cost	277384 gas 
execution cost	277384 gas 
input	0x610...50033
decoded input	{}
decoded output	 - 
logs	[]
val	0 wei
creation of AdderInclude pending...
[vm]from: 0x5B3...eddC4to: AdderInclude.(constructor)value: 0 weidata: 0x608...50033logs: 0hash: 0x064...5df47
status	true Transaction mined and execution succeed
transaction hash	0x0649bd15a7bf20f600523e2ca018987facec5f84aa2533522e03a4cfb2d5df47
from	0x5B38Da6a701c568545dCfcB03FcB875f56beddC4
to	AdderInclude.(constructor)
gas	393947 gas
transaction cost	342562 gas 
execution cost	342562 gas 
input	0x608...50033
decoded input	{}
decoded output	 - 
logs	[]
val	0 wei
```

发现部署合约发起了两笔交易，一个是部署库，一个是逻辑合约；


JAVA contract，同理；


使用导入模式是，remix测试同样发起两笔交易

测试发现；逻辑合约totalSum修改存储的方法，gas一致，没有什么区别；pure或view的sum计算方法，不收取gas费用。

#  总结

导入库合约和包含库合约之间不存，同样方法逻辑的调用gas费用一致，另外库调用形式，修改逻辑合约存储的使用DELEGATECALL方式（Homestead之前是用CALLCODE）。


# 附
[面向开发人员的 Solidity：Solidity 中的库](https://learnblockchain.cn/article/3699)   
[Solidity For Developers: Libraries In Solidity](https://medium.com/coinmonks/solidity-for-developers-libraries-in-solidity-f8c7e348dc24)    
[Deploying with Libraries on Remix-IDE](https://medium.com/remix-ide/deploying-with-libraries-on-remix-ide-24f5f7423b60)  
[library-linking](https://hardhat.org/hardhat-runner/plugins/nomiclabs-hardhat-ethers#library-linking)   
[SOLIDITY之LIBRARY 用法（一），以及USING A FOR B的特性（一个特殊的CONTRACT合约）](https://www.freesion.com/article/3476824750/)    
[SOLIDITY之LIBRARY 用法（二）库的核心用法总结（一个特殊的CONTRACT合约）](https://www.freesion.com/article/3809829977/)     
[solidity系列教程<九>库(library)的使用](https://www.jianshu.com/p/ab7665691a60)    
[库(Libraries)](https://www.tryblockchain.org/solidity-libraries-%E5%BA%93.html)  

[阅读以太坊EVM的一些笔记](https://stevenbai.top/ethereum/%E4%BB%A5%E5%A4%AA%E5%9D%8Aevm%E7%AC%94%E8%AE%B0/)   
[solidity智能合约[4]-pure与view剖析](https://dreamerjonson.com/2018/11/09/solidity-4/index.html)    