---
layout: page
title: 转账到合约及降级策略 
subtitle: 转账到合约及降级策略 
date: 2022-09-05 19:53:39
author: ravitn
catalog: true
category: solidity
categories:
    - solidity
tags:
    - solidity
---

# 引言
在以太坊上，有两类账户，一种外部账户，一种为合约账户，外部账号间可以直接转账，外部账户和合约的转账如何转账，转账的方式，以及回退机制；
今天，我们来一起探究一下；


# 目录
* [转账到合约方式](#转账到合约方式) 
    * [send/transfer方式](#send/transfer方式) 
    * [call与delegatecall区别](#call与delegatecall区别) 
    * [小节](#小节) 
* [测试合约](#测试合约) 
    * [非payable的Fallback合约](#非payable的Fallback合约) 
    * [接受方法测试合约](#接受方法测试合约) 
    * [call转账回退测试合约](#call转账回退测试合约) 
    * [transfer转账回退测试合约](#transfer转账回退测试合约) 
    * [转账测试合约](#转账测试合约) 
* [总结](#总结)     
* [附](#附) 

# 转账到合约方式
以太坊中send/transfer/call都能进行转ETH的操作，但这几个关键字的不同对solidity开发者来说非常重要，容易产生安全问题。delegatecall很容易与call混淆，这里也拿出来区分。

我们先来看下send/transfer方式：

## send/transfer方式
1. send原型

```
<address>.send(uint256 amount) returns (bool)
```

简介:向address发送amount数量的Wei（注意单位），如果执行失败返回false。发送的同时传输2300gas，gas数量不可调整



2. transfer原型
```
<address>.transfer(uint256 amount)
```

简介:向address发送amount数量的Wei（注意单位），如果执行失败则throw。发送的同时传输2300gas，gas数量不可调整



> send与transfer对比简析
* 相同之处
1. 均是向address发送ETH（以Wei做单位）
2. 发送的同时传输2300gas（gas数量很少，只允许接收方合约执行最简单的操作，2300gas只能够发送一个事件，有其他操作，则会抛出gas不足

* 不同之处

1. send执行失败返回false，transfer执行失败则会throw。这也就意味着使用send时一定要判断是否执行成功。

> 推荐:默认情况下最好使用transfer（因为内置了执行失败的处理）

再来看一下call和delegatecall

## call与delegatecall区别
1. call原型
```
<address>.call(...) returns (bool)
```
简介:以address（被调用合约）的身份调用address内的函数，默认情况下将所有可用的gas传输过去，gas传输量可调。执行失败时返回false

> 实例

```solidity
//call的函数调用
nameReg.call("register", "MyName");
nameReg.call(bytes4(keccak256("fun(uint256)")), a);
//设置调用时的gas和传输的eth value
nameReg.call.gas(1000000).value(1 ether)("register", "MyName");
```

2. delegatecall原型
```
<address>.delegatecall(...) returns (bool)
```
简介:以调用合约的身份调用address内的函数，默认情况下将所有可用的gas传输过去，gas传输量可调。执行失败时返回false。本函数目的在于让合约能够在不传输自身状态(如balance、storage)的情况下使用其他合约的代码。

> 实例

```
nameReg.delagatecall.gas(1000000)("register", "MyName");
//delegatecall不支持.value,eth
```

> call与delegatecall对比简析
* 相同之处

1. 调用时会将本合约所有可用的gas传输过去
2. 执行失败均返回false

* 不同之处

1. call可以使用.value传ETH给被调用合约
2. 假设在contract_test合约中分别有nameReg.call("somefunction")以及nameReg.delegatecall("somefunction")

> nameReg.call以nameReg合约的身份在nameReg中执行somefunction   
> nameReg.delegatecall以contract_test合约的身份在nameReg中执行somefunction   

3. delegatecall的目的就是让合约在不用传输自身状态(如balance、storage)的情况下可以使用其他合约的代码


## 小节
向合约发送eth的方式有3中，分别为：
1. send；
2. transfer；
3. address(test).call{value: 2 ether}("")；


3中转账的方式中，合约使用什么方法来接受，默认使用默认使用receive（payable），如果没有，则调用fallback（payable），如果fallback非payalbe则将失败；
另外说明一下 address(test).call{value: 2 ether}("")；方式，calldata为空，默认先调用receive，没有再调fallback；当call的calldata不为空时（address(test).call{value: 1}(abi.encodeWithSignature("nonExistingFunction()"))），则调用的为fallback函数。具体如下：

```
/*
Which function is called, fallback() or receive()?
           send Ether
               |
         msg.data is empty?
              / \
            yes  no
            /     \
receive() exists?  fallback()
         /   \
        yes   no
        /      \
    receive()   fallback()
*/
```

另外说明一下fallback函数：

如果call的方法签名，没有其他函数与给定的函数签名匹配，则在调用合约时执行回退函数。fallback 函数总是接收数据，但为了也接收 Ether，它必须被标记payable。
一个合约最多可以有一个fallback函数，使用 or 声明 （都没有关键字）。此功能必须具有可见性。回退函数可以是虚拟的，
可以覆盖并且可以具有修饰符。比如

1. fallback () external [payable]
2. fallback (bytes calldata input) external [payable] returns (bytes memory output) 

如果使用带参数的版本，input将包含发送到合约的完整数据（等于msg.data），并且可以在 中返回数据output。返回的数据不会经过 ABI 编码。

在最坏的情况下，如果还使用支付回退函数代替接收函数，它只能依赖 2300 gas 可用（ 有关此含义的简要描述，请参阅接收以太函数）。
与任何函数一样，只要有足够的 gas 传递给它，fallback 函数就可以执行复杂的操作


如果要解码输入数据，可以检查函数选择器的前四个字节，然后可以abi.decode与数组切片语法一起使用来解码 ABI 编码的数据：
请注意，这只能作为最后的手段使用，并且应该使用适当的功能。(c, d) = abi.decode(input[4:], (uint256, uint256));

eip1884后，需要相关操作码的gas成本，在gas成本不变（在2300gas以内）的情况下，推荐使用transfer，超限的情况下，建议使用call空calldata的方式，进行安全转账。


具体概括如下：

* 在Gas成本不变的假设下，推荐transfer()是有道理的。
* 但Gas成本不是不变的。 智能合约应该有力地应对这一事实。
* Solidity的 transfer() 和 send() 使用一个硬编码的Gas 成本。这些方法应避免使用。使用.call.value(...)("")代替。这就存在着重入的风险。 一定要使用现有的一种强大的方法来防止重入漏洞。Vyper的send()也有同样的问题。

简单说：2300 是gas 津贴CALL的数量，如果转移的以太币数量不为零，则将其添加到明确传递给 a 的 gas数量中。transfer()如果转移了非零数量的以太币，
 Solidity会将气体参数设置为 0。当与气体津贴相结合时，结果是总共 2300 气体。如果传输零以太币，Solidity 会明确将 gas 参数设置为 2300，以便在两种情况下都转发 2300 gas。↩︎
 

# 测试合约


## 非payable的Fallback合约
```solidity
import "hardhat/console.sol";
pragma solidity ^0.8.0;

contract TestErrorPayable {
    event FallbackPayError(address indexed from, uint256 x);
    uint x;
    fallback() external { 
        x = 1; 
        console.log("TestErrorPayable:FallbackPayError msg.sender" ,msg.sender);
        emit FallbackPayError(msg.sender,x);
    }
}
```


## 接受方法测试合约
```solidity
import "hardhat/console.sol";
pragma solidity ^0.8.0;

contract TestPayable {
    event FallbackPay(address indexed from, uint256 amout);
    event ReceivePay(address indexed from, uint256 amout);
    uint256 x;
    uint256 y;
    fallback() external payable {
        x = 1;
        y = msg.value;
        console.log("TestPayable: FallbackPay msg.sender" ,msg.sender);
        console.log("TestPayable: FallbackPay msg.value" ,msg.value);
        emit FallbackPay(msg.sender, msg.value);
    }
    receive() external payable {
        x = 2;
        y = msg.value;
        console.log("TestPayable: ReceivePay msg.sender" ,msg.sender);
        console.log("TestPayable: ReceivePay msg.value" ,msg.value);
        emit ReceivePay(msg.sender, msg.value);
    }
}

```


## call非空data测试合约
```solidity
import "hardhat/console.sol";
pragma solidity ^0.8.0;

contract TestPayableV2 {
    event FallbackPayV2(address indexed from, uint256 amout);
    event ReceivePayV2(address indexed from, uint256 amout);
    uint256 x;
    uint256 y;

    fallback() external payable {
        x = 1;
        y = msg.value;
        console.log("TestPayableV2: FallbackPayV2 msg.sender" ,msg.sender);
        console.log("TestPayableV2: FallbackPayV2 msg.value" ,msg.value);
        emit FallbackPayV2(msg.sender, msg.value);
    }
    receive() external payable {
        // console.log("TestPayableV2: ReceivePayV2 msg.sender" ,msg.sender);
        console.log("TestPayableV2: ReceivePayV2 msg.value" ,msg.value);
        //2300gas， 只够发一个事件
        emit ReceivePayV2(msg.sender, msg.value);
    }
}

```

## call转账回退测试合约


```solidity
import "hardhat/console.sol";
pragma solidity ^0.8.0;

contract TestPayableV3 {
    event FallbackPayV3(address indexed from, uint256 amout);
    event ReceivePayV3(address indexed from, uint256 amout);
    uint256 x;
    uint256 y;

    fallback() external payable {
        x = 1;
        y = msg.value;
        console.log("TestPayableV3: FallbackPayV3 msg.sender" ,msg.sender);
        console.log("TestPayableV3: FallbackPayV3 msg.value" ,msg.value);
        emit FallbackPayV3(msg.sender, msg.value);
    }
}

```


## transfer转账回退测试合约

```solidity
import "hardhat/console.sol";
pragma solidity ^0.8.0;

contract TestPayableV4 {
    event FallbackPayV4(address indexed from, uint256 amout);
    event ReceivePayV4(address indexed from, uint256 amout);
    uint256 x;
    uint256 y;

    fallback() external payable {
        // x = 1;
        // y = msg.value;
        console.log("TestPayableV4: FallbackPayV4 msg.sender" ,msg.sender);
        // console.log("TestPayableV4: FallbackPayV4 msg.value" ,msg.value);
        // emit FallbackPayV4(msg.sender, msg.value);
    }
}

```


## 转账测试合约
```solidity
import "./TestErrorPayable.sol";
import "./TestPayable.sol";
import "./TestPayableV2.sol";
import "./TestPayableV3.sol";
import "./TestPayableV4.sol";
import "hardhat/console.sol";
pragma solidity ^0.8.0;
contract CallerPayable {
    uint64 private  blockTimestamp;
    function setBlockTimestamp() external payable {
        blockTimestamp = uint64(block.timestamp);
    }
    /**
     * 非payable fallback测试，send将会失败
     */
    function callTestError(TestErrorPayable test) public returns (bool) {
        //调用成功fackback方法
        (bool success,) = address(test).call(abi.encodeWithSignature("nonExistingFunction()"));
        require(success);
        console.log("CallerPayable callTestError call with no signer function:",success);
        address payable testPayable = payable(address(test));
        return testPayable.send(2 ether);
    }
    /**
     * 不存在方法调用，将会调用fallback， call空data，将会调用receive函数
     */
  function callTestPayable(TestPayable test) public returns (bool) {
        //无对应的签名函数，回退到fallback函数
        (bool success,) = address(test).call(abi.encodeWithSignature("nonExistingFunction()"));
        require(success);
       console.log("CallerPayable callTestPayable call with no signer function:",success);
        //无对应的签名函数，回退到fallback函数
        (success,) = address(test).call{value: 1}(abi.encodeWithSignature("nonExistingFunction()"));
          // results in test.x becoming == 1 and test.y becoming 1.
        require(success);
        console.log("CallerPayable callTestPayable call with no signer function and send 1 eth:",success);
        // 如果任何发送eth到合约，receive方法将会被调用。由于TestPayable的receive需要写存储，消耗将会
        (success,) = address(test).call{value: 2 ether}("");
        require(success);
        console.log("CallerPayable callTestPayable call with no data and send 2 eth:",success);
        return true;
    }
    /**
     * 如果任何发送eth到合约，receive方法将会被调用;send方法测试，固定2300s
     */
    function callTestPayableBySend(TestPayableV2 test) public returns (bool) {
       address payable testPayable = payable(address(test));
       console.log("CallerPayable callTestPayableBySend send====");
        return testPayable.send(2 ether);
    }
    /**
     * 如果任何发送eth到合约，receive方法将会被调用，transfer方法测试，固定2300
     * 如果Gas成本是可以变化的，那么智能合约就不能依赖于任何特定的Gas成本。
      任何使用transfer()或send()的智能合约，都是通过转发固定数量的Gas来而产生2300Gas成本的硬性依赖。
      因此建议停止在代码中使用transfer()和send()，而改用call()。
      https://learnblockchain.cn/article/2191
     */
    function callTestPayableByTransfer(TestPayableV2 test) public {
        address payable testPayable = payable(address(test));
        console.log("CallerPayable callTestPayableByTransfer transfer====");
        return testPayable.transfer(2 ether);
    }
    /**
     * 空函数体，回退到receive函数，没有receive， 则回退到fallback函数
     */
    function callTestPayableV3(TestPayableV3 test) public returns (bool) {
        ( bool success,) = address(test).call{value: 2 ether}("");
          // results in test.x becoming == 2 and test.y becoming 2 ether.
        require(success);
        console.log("CallerPayable callTestPayableV3 call with no data and send 2 eth:",success);
        return true;
    }
    /**
     * send，transfef，没有recevie，回退到fallback
     */
    function callTestPayableV4(TestPayableV4 test) public {
        address payable testPayable = payable(address(test));
        console.log("CallerPayable callTestPayableV4 transfer====");
        // is a contract and gas costs change.
        return testPayable.transfer(2 ether);
    }
}
```

 # 总结

向合约发送eth的方式有3中，分别为send, transfer和call 空calldata；3中转账的方式中，合约使用什么方法来接受，默认使用默认使用receive（payable），如果没有，则调用fallback（payable），如果fallback非payalbe则将失败；当call的calldata不为空时（address(test).call{value: 1}(abi.encodeWithSignature("nonExistingFunction()"))），则调用的为fallback函数；

在gas成本不变（在2300gas以内）的情况下，推荐使用transfer，超限的情况下，建议使用call空calldata的方式，进行安全转账，不过要控制好重入问题。




# 附

[细究以太坊中send/transfer/call/delegatecall](https://zhuanlan.zhihu.com/p/35292014)  
[fallback-function](https://docs.soliditylang.org/en/latest/contracts.html?highlight=fallback-function#fallback-function)    
[停止使用Solidity的transfer()](https://learnblockchain.cn/article/2191)      
[sending-ether](https://solidity-by-example.org/sending-ether/)     
[Three methods to send ether by means of Solidity](https://medium.com/daox/three-methods-to-transfer-funds-in-ethereum-by-means-of-solidity-5719944ed6e9)     
[Stop Using Solidity's transfer() Now](https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/)    
[EIP-1285: Increase Gcallstipend in the CALL OPCODE #1285](https://github.com/ethereum/EIPs/issues/1285)     
[EIP 1884: 对 trie-size-dependent 操作码调整gas消耗](https://learnblockchain.cn/docs/eips/eip-1884.html)  
[停止使用Solidity的transfer](https://zhuanlan.zhihu.com/p/370035663)  
[vyper](https://github.com/vyperlang/vyper)    
