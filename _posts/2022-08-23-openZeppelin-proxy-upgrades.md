layout: page
title: OpenZeppelin代理升级原理分析  
subtitle: OpenZeppelin代理升级原理分析  
date: 2022-08-23 19:53:39
author: ravitn
catalog: true
category: solidity
categories:
    - solidity
tags:
    - OpenZeppelin
---
# 引言
>什么是智能合约升级？
智能合约升级是一种在保留存储和余额的同时，而又可以任意更改在地址中执行代码的操作。

以太坊上部署的合约一旦部署，将不可以改变，毫无疑问，不可变代码是可怕的。这就是为什么以太坊生态系统中的许多项目一直在通过代理合约推动可升级机制的想法
。除其他外，此类机制允许快速修复错误并在已部署的合约之上添加新功能。代理的一个非常有趣的特性是可以利用它们来实现链上代码的可重用性。在一个理想的世界里，已经构建了弹性的经过实战测试的代码的知名项目可以将他们的代码部署到区块链上，
并让每个人都将他们的代理指向可信的链上代码。然后，普通开发人员需要做的就是部署代理并让用户通过它与应用程序交互。
这样的代理将转而delegatecall使用其他人的可信和可验证代码的链上代码。

# 目录
* [合约升级模式](#合约升级模式) 
    * [路由模式](#路由模式) 
    * [业务数据分离模式](#业务数据分离模式) 
    * [DELEGATCALL模式](#delegatcall模式) 
* [OpenZeppelin代理升级](#openzeppelin代理升级) 
    * [Eip1967](#eip1967) 
    * [OpenZeppelin代理升级实战](#openzeppelin代理升级实战) 
        * [基于OpenZeppelin的hardhat-upgrades插件升级](#基于openzeppelin的hardhat-upgrades插件升级) 
        * [OpenZeppelin代理升级库手动升级](#openzeppelin代理升级库手动升级) 
        * [OpenZeppelin代理升级库合约架构](#openzeppelin代理升级库合约架构) 
* [总结](#总结)     
* [附](#附) 


# 合约升级模式


## 路由模式

我们可以通过路由模式，动态调用（call）对应的合约，这种模式，初始化合约时，部署合约并注册到路由，
当要升级时，重新部署，并重新注册到路由器。这种方式的缺点是，原始的合约数据，将会丢失，唯一的办法，
就是将老合约的数据迁移到新合约，这个成本比较高，很容易出错。



## 业务数据分离模式

将合约业务逻辑功能和数据分别放到两个合约中，升级时，只需要重新部署逻辑合约，将合约存储指向数据合约即可；
这样要管理逻辑合约和数据合约。


## DELEGATCALL模式

在讲DelegatCall之前，我们说一下其他几种调用模式：
CALL：这种模式一般是A合约调用B合约，修改的是B合约的存储，B合约拿到的msg.sender是A合约的地址；
CALLCODE: 这种模式下，A合约调用B合约，修改的是A合约的存储，B合约拿到的msg.sender是A合约的地址；
DELEGATCALL：这种模式是CALLCODE的bugfix模式，A合约调用B合约，修改的是A合约的存储，B合约拿到的msg.sender是原始发送者地址，tx.origin;
STATICCALL:静态调用模式，一般为合约的public constant或者library的工具合约方法的调用，一般为静态调用模式。

DelegatCall模式的经典模式实现，见OpenZeppelin，可升级代理模式；这种模式下，调用代理合约时，
EVM 将首先检查代理代码中是否有任何标识符匹配的函数签名（比如：0x42966c68），
如果没有，则fallback代理的功能将被执行并将调用委托给implementation合约。


以太坊中的所有函数调用都由有效载荷payload前4个字节来标识，称为“函数选择器”。
选择器是根据函数名称及其签名的哈希值计算得出的。然而，4字节不具有很多熵，
这意味着两个函数之间可能会发生冲突：具有不同名称的两个不同函数最终可能具有相同的选择器。
如果你偶然发现这种情况，Solidity编译器将足够聪明，可以让你知道，
并且拒绝编译具有两个不同函数名称，但具有相同4字节标识符（函数选择器）的合约。


但是，对于实现合约而言，完全有可能具有与代理的升级函数具有相同的4字节标识符的函数。
这可能会导致尝试调用实现合约时，管理员无意中将代理升级到随机地址
（注：因为实现合约合约与升级函数4字节标识符相同）。 
这个帖子由Patricio Palladino解释了该漏洞，然后Martin Abbatemarco说明如何将其用于做恶.

这个问题可以通过开发用于可升级智能合约的适当工具解决，也可以通过代理本身解决。
特别是，如果将代理设置为仅管理员能调用升级管理函数，而所有其他用户只能调用实现合约的函数，
则不可能发生冲突。


# OpenZeppelin代理升级
[Proxy Upgrade Pattern](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#transparent-proxies-and-function-clashes)

OpenZeppelin升级，针对升级存在的问题及其解决方案：


* 非结构化存储代理  

使用代理时很快出现的一个问题与变量在代理合约中的存储方式有关。
假设代理将逻辑合约的地址存储在其唯一的变量address public _implementation;中。
现在，假设逻辑合约是一个基本代币，其第一个变量是address public _owner。
这两个变量的大小都是 32 字节，据 EVM 所知，它们占据了代理调用的结果执行流的第一个槽。
当逻辑合约写入 时_owner，它是在代理状态范围内执行的，
实际上是写入_implementation。这个问题可以称为“存储冲突”。
有很多方法可以解决这个问题，OpenZeppelin Upgrades 实现的“非结构化存储”方法的工作原理如下。
它没有将_implementation地址存储在代理的第一个存储槽中，而是选择了一个伪随机槽。
这个槽是足够随机的，逻辑合约在同一槽中声明变量的概率可以忽略不计。
代理存储中随机化插槽位置的相同原则用于代理可能具有的任何其他变量，
例如管理员地址（允许更新 的值_implementation）等。


* 实现版本之间的存储冲突

如前所述，非结构化方法避免了逻辑合约和代理之间的存储冲突。
但是，不同版本的逻辑合约之间可能会发生存储冲突。在这种情况下，
假设逻辑合约的第一个实现存储address public _owner在第一个存储槽中，
升级后的逻辑合约存储address public _lastContributor在相同的第一个槽中。
当更新的逻辑合约尝试写入_lastContributor变量时，
它将使用与之前存储值相同的存储位置_owner，并覆盖它！

非结构化存储代理机制无法防范这种情况。用户可以让逻辑合约的**新版本扩展以前的版本**，
或者以其他方式保证存储层次结构始终被附加但不被修改。但是，OpenZeppelin Upgrades
 会检测到此类冲突并适当地警告开发人员。

* 构造函数警告
在 Solidity 中，构造函数内部的代码或全局变量声明的一部分不是已部署合约的运行时字节码的一部分。
此代码仅在部署合约实例时执行一次。因此，逻辑合约构造函数中的代码将永远不会在代理状态的上下文中执行。
换句话说，代理完全不知道构造函数的存在。就好像他们没有代理一样。

不过这个问题很容易解决。逻辑合约应将构造函数中的代码移动到常规的“初始化程序”函数，
并在代理链接到此逻辑合约时调用此函数。需要特别注意这个初始化函数，使其只能被调用一次，
这是一般编程中构造函数的属性之一。

这就是为什么当我们使用 OpenZeppelin Upgrades 创建代理时，您可以提供初始化函数的名称并传递参数。
为了确保该initialize函数只能被调用一次，使用了一个简单的修饰符。
OpenZeppelin Upgrades 通过可扩展的合约提供此功能。



* 透明代理和函数冲突

如前几节所述，可升级的合约实例（或代理）通过将所有调用委托给逻辑合约来工作。
但是，代理需要自己的一些功能，例如upgradeTo(address)升级到新的实现。
这引出了一个问题，如果逻辑合约也有一个名为 的函数upgradeTo(address)：
调用该函数时，调用者打算调用代理还是逻辑合约？
  

OpenZeppelin Upgrades 处理这个问题的方式是通过透明代理模式。
透明代理将根据调用者地址（即msg.sender）决定将哪些调用委托给底层逻辑合约：

1. 如果调用者是代理的管理员（有权升级代理的地址），
那么代理不会委托任何呼叫，而只会回答它理解的任何消息。

2. 如果调用者是任何其他地址，则代理将始终委托调用，无论它是否匹配代理的功能之一

具体如下：
TransparentUpgradeableProxy
```sol
/**
 * @dev Modifier used internally that will delegate the call to the implementation unless the sender is the admin.
 * 除非为管理员，否则，通过delegatecall调用逻辑目标合约
 */
modifier ifAdmin() {
    if (msg.sender == _getAdmin()) {//管理员
        _;
    } else {
        _fallback();//否则降级，委托给代理合约
    }
}
/**
 * @dev Upgrade the implementation of the proxy.
 *
 * NOTE: Only the admin can call this function. See {ProxyAdmin-upgrade}.
 */
function upgradeTo(address newImplementation) external ifAdmin {
    _upgradeToAndCall(newImplementation, bytes(""), false);
}

```



Proxy
```sol
/**
 * @dev This is a virtual function that should be overridden so it returns the address to which the fallback function
 * and {_fallback} should delegate.
 */
function _implementation() internal view virtual returns (address);

/**
 * @dev Delegates the current call to the address returned by `_implementation()`.
 * 代理当前逻辑合页实现的调用
 * This function does not return to its internal call site, it will return directly to the external caller.
 */
function _fallback() internal virtual {
    _beforeFallback();
    _delegate(_implementation());
}

 /**
 * @dev Delegates the current call to `implementation`.
 *
 * This function does not return to its internal call site, it will return directly to the external caller.
 */
function _delegate(address implementation) internal virtual {
    assembly {
        // Copy msg.data. We take full control of memory in this inline assembly
        // block because it will not return to Solidity code. We overwrite the
        // Solidity scratch pad at memory position 0.
        calldatacopy(0, 0, calldatasize())

        // Call the implementation.
        // out and outsize are 0 because we don't know the size yet.
        let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)

        // Copy the returned data.
        returndatacopy(0, 0, returndatasize())

        switch result
        // delegatecall returns 0 on error.
        case 0 {
            revert(0, returndatasize())
        }
        default {
            return(0, returndatasize())
        }
    }
}
```


我们再来看一下，OpenZeppelin代理架构中解决存储冲突的提案Eip1967
## Eip1967

该标准产生的背景是因为合约部署越来越多地采用路由合约跟逻辑合约分开部署的方式，
这种方式的好处是在升级逻辑合约的时候，只需要将路由合约中逻辑合约的地址更改，
就可以路由到新的逻辑合约上。

EIP-1967的目的是规定一个通用的存储插槽使用标准，用于在代理合约中的特定位置存放逻辑合约的地址。
其规定了如下特定的插槽： 


来看一下OpenZeppelin的ERC1967实现
```sol
abstract contract ERC1967Upgrade {
    // This is the keccak-256 hash of "eip1967.proxy.rollback" subtracted by 1 回滚代理插槽
    bytes32 private constant _ROLLBACK_SLOT = 0x4910fdfa16fed3260ed0e7147f7cc6da11a60208b5b9406d12a635614ffd9143;
    /**
     * @dev Storage slot with the address of the current implementation. 当前实现的地址存储槽
     * This is the keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1, and is
     * validated in the constructor.
     * bytes32(uint256(keccak256("eip1967.proxy.implementation") - 1)) 
     */
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    /**
     * @dev Storage slot with the admin of the contract.
     合约管理员存储槽
     * This is the keccak-256 hash of "eip1967.proxy.admin" subtracted by 1, and is
     * validated in the constructor.
     */
    bytes32 internal constant _ADMIN_SLOT = 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;
    /**
     * @dev The storage slot of the UpgradeableBeacon contract which defines the implementation for this proxy.
     * This is bytes32(uint256(keccak256('eip1967.proxy.beacon')) - 1)) and is validated in the constructor.
     基于信标链的升级插槽？？
     */
    bytes32 internal constant _BEACON_SLOT = 0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50;
    ...
} 
```

## OpenZeppelin代理升级实战
我们假设存在存在这么BoxUpgrades合约两个版本的合约，分别为BoxUpgradesV1，BoxUpgradesV2；

### 基于OpenZeppelin的hardhat-upgrades插件升级

1. 安装OpenZeppelin hardhat-upgrades升级插件

```
npm install --save-dev @openzeppelin/hardhat-upgrades
```

2.config中引用
```
require('@openzeppelin/hardhat-upgrades');
```

3. 编写升级脚本

```js
 const BoxUpgradesV1 = await ethers.getContractFactory("BoxUpgradesV1");
console.log("Deploying BoxUpgradesV1...");
// const boxUpgradesProxy = await upgrades.deployProxy(BoxUpgradesV1, 42);
// const boxUpgradesProxy = await upgrades.deployProxy(BoxUpgradesV1, [42]);
var boxUpgradesProxy = await upgrades.deployProxy(BoxUpgradesV1, [42], { initializer: 'init' });
await boxUpgradesProxy.deployed();
//0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9
const boxUpgradesProxyAddress = boxUpgradesProxy.address;
console.log("boxUpgradesProxy address:", boxUpgradesProxy.address);
console.log("boxUpgradesProxy retrieve:", await boxUpgradesProxy.retrieve());
await boxUpgradesProxy.store(1);
console.log("boxUpgradesProxy retrieve:", await boxUpgradesProxy.retrieve());

//升级到V2
const BoxUpgradesV2 = await ethers.getContractFactory("BoxUpgradesV2");
console.log("Upgrading BoxUpgradesV2...");
boxUpgradesProxy = await upgrades.upgradeProxy(boxUpgradesProxyAddress, BoxUpgradesV2);
console.log("boxUpgradesProxy upgraded:",boxUpgradesProxy.address);
//Box实例已经升级到了最新版本的代码，同时保持了它的状态和之前的地址。我们不需要在新的地址部署一个新的合约，也不需要手动将旧Box的value复制到新Box中。
console.log("boxUpgradesProxy retrieve:", await boxUpgradesProxy.retrieve());
await boxUpgradesProxy.increment();
console.log("boxUpgradesProxy retrieve:", await boxUpgradesProxy.retrieve());
```
执行测试脚本，我们可以发现，Box代理实现已经升级到了最新版本的代码，同时保持了它的状态和之前的地址，
不需要手动将BoxUpgradesV1的value复制到新BoxUpgradesV2中。

在整个过程中，我们使用 hardhat-upgrades插件进行升级，那，插件如何知道代理管理ProxyAdmin合约和
和透明代理合约，及实现合约的地址信息呢？


插件会追踪.openzeppelin目录下的文件，每个网络一个文件，文件内记录了ProxyAdmin合约地址，
透明代理Proxies合约和实现合约地址，文件的名称是{unknown-chainId}.json。
如果属于某个网络的json文件已存在，那么ProxyAdmin合约不再重新部署。
如果属于某个网络的json文件不存在，那么直接升级到新的实现合约会报错；

文件内容具体如下：

```json
 "admin": {
    "address": "0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512",
    "txHash": "0x97403ff2098937f07be29a875da02f2514d0346e0dd29e847f66a587b2e0e8b4"
  },
  "proxies": [
      {
          "address": "0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0",
          "txHash": "0xce22bed65b7d3ec3c6f1ccf7b70540fa9a895305a3fbf77644e5d2de666e004c",
          "kind": "transparent"
        },
    ...
    ]
  "impls": {
   "649d44be66ab4d38cb5690a054926d4ce8d535525749498f793e167fb9f31f50": {
        "address": "0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9",
        "txHash": "0xdceea4161e01e34bf01449cb1922f9db69d760ec20d7f2b35388f5c420c2a796",
        "layout": {
                "storage": [
                ...
                {
                    "label": "value",
                    "offset": 0,
                    "slot": "101",
                    "type": "t_uint256",
                    "contract": "BoxUpgradesV1",
                    "src": "contracts\\BoxUpgradesV1.sol:12"
                  }
                ]
                ...
                
        },
    "649d44be66ab4d38cb5690a054926d4ce8d535525749498f793e167fb9f31f50": {
    "address": "0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9",
    "txHash": "0xdceea4161e01e34bf01449cb1922f9db69d760ec20d7f2b35388f5c420c2a796",
    "layout": {
            "storage": [
            ...
            {
              "label": "value",
              "offset": 0,
              "slot": "101",
              "type": "t_uint256",
              "contract": "BoxUpgradesV2",
              "src": "contracts\\BoxUpgradesV2.sol:9"
            }
          ]
       ...    
    }
  }
```

### OpenZeppelin代理升级库手动升级脚本

使用openzeppelin代理库升级如下：
1. 部署实现合约初始版本V1；
2. 部署代理管理合约；
3. 部署透明代理合约，并初始化实现合约；
4. 部署实现合约版本V2
5. 使用代理管理合约升级实现合约为V2；

一句话：通过透明合约代理，对逻辑合约进行代理，管理员通过ProxyAdmin升级透明合约代理的逻辑合约。


### OpenZeppelin代理升级库合约架构

![OpenZeppelinProxyFramwork](/image/openzeppelin/OpenZeppelin-Proxy-Framwork.png) 


* ProxyAdmin:代理管理合约, 控制透明代理升级及管理员变更；
* TransparentUpgradeableProxy:解决函数冲突的透明合约，管理员走代理合约，其他调用者委托给逻辑目标合约。
* ERC1967Proxy:基于ERC1967的代理
* ERC1967Upgrade:基于ERC1967的安全插槽可升级合约
* Proxy:代理合约，将所有调用降级到_fallback实现调用，即通过DELEGATCALL的方式委托给，逻辑合约；




[ProxyAdmin](https://github.com/Donaldhan/openzeppelin-contracts/blob/master/contracts/proxy/transparent/ProxyAdmin.sol)  
[TransparentUpgradeableProxy](https://github.com/Donaldhan/openzeppelin-contracts/blob/master/contracts/proxy/transparent/TransparentUpgradeableProxy.sol)   
[ERC1967Proxy](https://github.com/Donaldhan/openzeppelin-contracts/blob/master/contracts/proxy/ERC1967/ERC1967Proxy.sol)  
[ERC1967Upgrade](https://github.com/Donaldhan/openzeppelin-contracts/blob/master/contracts/proxy/ERC1967/ERC1967Upgrade.sol)   
[Proxy](https://github.com/Donaldhan/openzeppelin-contracts/blob/master/contracts/proxy/Proxy.sol)   
[EIP1967](https://zhuanlan.zhihu.com/p/480217161) 



# 总结


OpenZeppelin Upgrades通过伪随机插槽（Eip1967）解决了存储冲突，针对合约内部变量的存储冲突，建议新版本扩展以前的版本，
不过OpenZeppelin Upgrades会检测到此类冲突并适当地警告开发人员。

为了解决逻辑合约构造函数中的代码将永远不会在代理状态的上下文中执行，在部署透明代理时，
提供初始化函数的名称并传递参数，为了确保该initialize函数只能被调用一次，使用了一个简单的修饰符。

通过透明代理模式，解决透明代理和函数冲突冲突的问题，
透明代理将根据调用者地址（即msg.sender）决定将哪些调用委托给底层逻辑合约：

可升级代理的本质为：通过透明合约代理，对逻辑合约进行代理，管理员通过ProxyAdmin升级透明合约代理的逻辑合约。



# 附 
[Upgrades Plugins](https://docs.openzeppelin.com/upgrades-plugins/1.x/)      
[Proxy Upgrade Pattern](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#transparent-proxies-and-function-clashes)     
[What does it mean for a contract to be upgrade safe?](https://docs.openzeppelin.com/upgrades-plugins/1.x/faq#what-does-it-mean-for-a-contract-to-be-upgrade-safe)  
[透明代理和函数冲突](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#transparent-proxies-and-function-clashes)        
[全面理解智能合约升级](https://learnblockchain.cn/article/1933)     
[升级智能合约(Hardhat)](https://learnblockchain.cn/article/1990)       
[手动部署OpenZeppelin可升级合约](https://learnblockchain.cn/article/2758)       
[手把手部署以太坊可升级智能合约](https://learnblockchain.cn/article/2920)      
[以太坊 - 深入浅出虚拟机](https://learnblockchain.cn/2019/04/09/easy-evm/)     
[当心代理：学习如何利用函数冲突](https://forum.openzeppelin.com/t/beware-of-the-proxy-learn-how-to-exploit-function-clashing/1070)    


