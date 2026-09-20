---
layout: page
title: RWA 行业深度分析：Ondo 与 XStock 两大链上资产路线对比
subtitle: RWA 行业深度分析：Ondo 与 XStock 两大链上资产路线对比
date: 2026-09-20 15:17:19
author: 0xTTEPX
catalog: true
category: web3
categories:
    - web3
tags:
    - RWA
---

## 一、引言

2024-2026 年，RWA（Real World Assets，现实世界资产代币化）成为加密行业最具实质增长意义的赛道之一。从 BlackRock 推出 BUIDL 基金，到 Ondo 的 OUSG/USDY 突破数十亿美元 TVL，再到 xStocks 将 715 支美股/ETF 搬上公链——RWA 正在从"概念验证"走向"金融基础设施"。

但"RWA"本身是一个极其宽泛的概念。国债、股票、房地产、私募信贷、大宗商品都可以被代币化，而不同资产类别对法律结构、托管模式、发行机制、交易架构、合规要求的需求截然不同。这也导致市场上出现了多条截然不同的技术路线。

本文聚焦当前 RWA 赛道最具代表性的两个项目——**Ondo** 与 **XStocks**——从产品定位、技术架构、发行机制、法律结构、交易基础设施等维度进行深入对比，试图回答一个核心问题：

> 在"把现实资产搬上链"这件事上，Ondo 和 XStocks 到底在做两件怎样不同的事？它们各自的技术选择和架构取舍背后，反映了 RWA 行业怎样的分化方向？

---

### 目录
* [一、引言](#一引言)
* [二、行业概览](#二行业概览)
    * [2.1 RWA 市场规模与结构](#21-rwa-市场规模与结构)
    * [2.2 RWA 全产业地图](#22-rwa-全产业地图)
    * [2.3 RWA 的演进路径](#23-rwa-的演进路径)
* [三、Ondo 与 XStock 深度对比](#三ondo-与-xstock-深度对比)
    * [3.1 项目定位：两条不同的 RWA 路线](#31-项目定位两条不同的-rwa-路线)
    * [3.2 产品演进路径](#32-产品演进路径)
    * [3.3 技术架构对比](#33-技术架构对比)
    * [3.4 发行与赎回机制对比](#34-发行与赎回机制对比)
    * [3.5 法律结构对比](#35-法律结构对比)
    * [3.6 交易与流动性架构对比](#36-交易与流动性架构对比)
    * [3.7 多链与分发策略对比](#37-多链与分发策略对比)
    * [3.8 Corporate Action 处理对比](#38-corporate-action-处理对比)
* [四、核心差异总结](#四核心差异总结)
    * [4.1 一张表看清两个项目](#41-一张表看清两个项目)
    * [4.2 两个项目反映的 RWA 行业分化](#42-两个项目反映的-rwa-行业分化)
    * [4.3 从交易所后端视角看两个项目](#43-从交易所后端视角看两个项目)
* [五、总结](#五总结)
    * [5.1 核心结论](#51-核心结论)
    * [5.2 行业展望](#52-行业展望)


## 二、行业概览

### 2.1 RWA 市场规模与结构

截至 2026 年，RWA 赛道呈现以下格局：

- **链上分布式资产总值（Distributed）**：约 $38.3B
- **链下被代表的资产总值（Represented）**：约 $345B
- **Tokenized Treasury** 单项规模超过 $14B
- **Tokenized Stocks/ETFs** 快速扩张，头部项目已覆盖 700+ 支资产

RWA 的应用可以拆分为 8 个主要方向：

```
RWA 应用光谱
    │
    ├── Tokenized Treasury（国债代币化）
    ├── Tokenized Stocks/ETFs（股票/ETF 代币化）
    ├── Tokenized Real Estate（房地产代币化）
    ├── Private Credit / Private Debt（私募信贷）
    ├── Tokenized Commodities（大宗商品代币化）
    ├── Tokenized Carbon Credits（碳信用代币化）
    ├── Tokenized Receivables（应收账款代币化）
    └── Structured Products / Derivatives（结构化产品/衍生品）
```

其中，**Tokenized Treasury** 和 **Tokenized Stocks/ETFs** 是当前规模最大、基础设施最成熟的两条主线。Ondo 和 XStocks 分别代表了这两条主线的头部项目。

### 2.2 RWA 全产业地图

RWA 行业并非只有"发一个 Token"那么简单。从产业链角度，它至少包含 8 个层次：

```
                    RWA 全产业地图
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
 第1层               第2层               第3层
 资产端               法律端               托管端
    │                    │                    │
 Asset Manager       SPV / Legal          Custodian
 BlackRock           Jersey/BVI           Bank/Broker
 Franklin            Cayman               Regulated
    │                    │                    │
    └────────────────────┼────────────────────┘
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
 第4层               第5层               第6层
 身份/合规            代币化               区块链
    │                    │                    │
 KYC/AML            Tokenization          Ethereum
 Whitelist           Engine               Solana
 Jurisdiction        Smart Contract        BSC/TON
    │                    │                    │
    └────────────────────┼────────────────────┘
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
 第7层               第8层
 分销/交易             金融应用
    │                    │
 CEX / DEX            DeFi / Lending
 Wallet               Collateral
 Broker/RIA           Payment / Settlement
```

一个关键认知是：**Token 只是最下面的一层**。真正决定一个 RWA 项目能否成立的，是上面的法律结构、托管安排、合规框架、资产管理和分销能力。这也是 Ondo 和 XStocks 投入最多资源的地方。

### 2.3 RWA 的演进路径

从行业整体看，RWA 正在沿以下路径演进：

```
Asset → Token → Trading → DeFi → Collateral → Stablecoin → Payment → Settlement
  │        │        │        │          │            │           │          │
  资产    代币化    交易     金融应用    抵押品       稳定币      支付      结算
```

Ondo 和 XStocks 在这条路径上的位置不同，选择的重心也不同。

---

## 三、Ondo 与 XStock 深度对比

### 3.1 项目定位：两条不同的 RWA 路线

| 维度 | Ondo | XStocks |
|------|------|---------|
| **核心方向** | RWA 金融产品（Treasury → 公共证券 → 交易基础设施） | Tokenized Equities（股票/ETF 代币化基础设施） |
| **起步产品** | Tokenized Treasury（OUSG, 2021） | Tokenized Stocks（2025年6月上线） |
| **代表资产** | OUSG（短期国债）、USDY（收益型票据）、Ondo Stocks | AAPLx、TSLAx、NVDAx、SPYx 等 715 支股票/ETF |
| **TVL/规模** | TVL > $2.5B（Treasury + Stocks） | $40B+ 交易量，810 产品 |
| **发行主体** | Ondo Finance / Ondo Generator | Backed Assets (JE) Limited（Jersey SPV） |
| **目标定位** | Institutional RWA + Onchain Trading Infrastructure | Onchain Capital Markets（链上资本市场） |

**核心区别**：Ondo 从"收益率产品"（国债）起步，逐步扩展到股票和交易基础设施；XStocks 从一开始就聚焦"股票代币化"，做的是把传统证券市场的资产直接搬上链。

### 3.2 产品演进路径

**Ondo 的演进**是一条"从简单到复杂"的产品线扩展路径：

```
2021-2023                2024                  2025-2026
    │                      │                      │
Tokenized Treasury    Public Securities       Ondo Stocks
    │                      │                      │
  OUSG               USDY（零售）          440+ 股票/ETF
  （机构）                                      │
    │                      │                      │
    └──────────────────────┼──────────────────────┘
                           │
                     Ondo Network（2026）
                           │
                  Verifiable Execution Layer
                  TEE + Attestors + Settlement
```

Ondo 的产品逻辑是：先用国债验证"链上合规金融产品"的可行性 → 扩展到零售产品 → 扩展到股票 → 最终构建自己的交易基础设施（Ondo Network）。

**XStocks 的演进**则是一条"从产品到基础设施"的路径：

```
2025.05              2025.06              2025.11-12
    │                    │                    │
  公布项目           正式上线            xPort + TON
  55+ 支股票         60+ 支股票          机构代币化引擎
    │                    │                    │
    └────────────────────┼────────────────────┘
                           │
                        2026
                           │
              715 支资产 / $40B+ 交易量
              多链 / 国际股票市场（香港首发）
              API / Developer Platform
```

XStocks 的逻辑是：先把股票代币化做通 → 建立 CEX/DEX 分销网络 → 扩展到机构级代币化基础设施（xPort）→ 开放 API 平台。

### 3.3 技术架构对比

#### Ondo 架构：Attestation + Smart Contract + Ondo Network

Ondo 的技术架构核心是**"链下策略层 + 链上结算层"的分离**：

```
                  Wallet / Exchange / FinTech
                              │
                              ↓
                  ┌─────────────────────┐
                  │    Ondo REST API    │
                  │                     │
                  │ Price / Quote       │
                  │ Attestation         │
                  │ Limits / Status     │
                  │ Asset Metadata      │
                  └──────────┬──────────┘
                             │
                     Signed Attestation
                             │
                             ↓
                  ┌─────────────────────┐
                  │   Smart Contract    │
                  │                     │
                  │ Verify Signature    │
                  │ Transfer USDC       │
                  │ Mint / Burn Token   │
                  └──────────┬──────────┘
                             │
                             ↓
                     Ondo Stock Token
                             │
                             ↓
                  ┌─────────────────────┐
                  │ Real-world Asset    │
                  │ US Stock / ETF      │
                  │ Broker / Custodian  │
                  └─────────────────────┘
```

Ondo 的 API 体系已经非常接近一个**证券交易所后台**：

| API 类别 | 功能 | 类比交易所 |
|----------|------|-----------|
| Soft Quote | 非绑定报价 | Quote Engine |
| Attestation | 可执行交易授权 | Order Authorization |
| Trading Limits | 实时风控限额 | Risk Engine |
| Market/Asset Status | 市场/资产状态 | Market Status |
| OHLC / Price Streaming | 行情数据 | Market Data |
| gRPC Streaming | 低延迟推送 | Market Data Stream |

而 2026 年推出的 **Ondo Network** 更进一步，引入了 **TEE（可信执行环境）** 的概念：

```
        Ondo Network 架构
              │
    ┌─────────┼─────────┐
    │         │         │
  Enclave  Attestors  Public Chain
  (TEE)    (验证方)    (结算层)
    │         │         │
  执行交易   验证执行    最终结算
  包含敏感   是否正确    记录上链
  业务逻辑
```

Ondo Network 的核心哲学是：**重要的不是所有事情都发生在链上，而是所有事情都可以被验证**。

#### XStocks 架构：三层发行 + 多链分发

XStocks 的技术架构核心是**三条发行通道 + 多链分发网络**：

```
             XStocks Primary Market
                      │
         ┌────────────┼────────────┐
         │            │            │
    Market Flow     xChange       xPort
         │            │            │
    Stablecoin     Stablecoin    Real Stock
         │            │            │
         ▼            ▼            ▼
       Broker      RFQ / MM      Alpaca
         │            │            │
         └────────────┼────────────┘
                      │
                      ▼
               Issuer / SPV
                      │
                      ▼
                 xStock Token
                      │
            ┌─────────┼─────────┐
            ▼         ▼         ▼
        Ethereum   Solana     BSC/TON
            │         │         │
            ▼         ▼         ▼
          CEX       DEX       DeFi
```

三条发行通道分别解决不同问题：

| 发行通道 | 本质 | 输入 | 核心机制 | 主要用户 |
|----------|------|------|---------|---------|
| Market Flow | Cash-based | Stablecoin | Broker Market Order | 标准客户 |
| xChange | Atomic RFQ | Stablecoin | Atomic Swap (~60s 窗口) | MM / Trader / 套利者 |
| xPort | In-Kind | 真实股票 | Custody Transfer (Alpaca) | 机构 / MM |

**核心区别**：Ondo 的架构重心在**交易基础设施**（API + Attestation + TEE Network），试图构建一个可验证的链上交易执行环境；XStocks 的架构重心在**发行基础设施**（三种发行通道 + 多链 + CEX/DEX 分销），试图构建一个连接传统证券和链上市场的桥梁。

### 3.4 发行与赎回机制对比

这是两个项目差异最显著的地方之一。

**Ondo 的发行机制**：

```
用户
 │
 │ 请求 Attestation
 ↓
Ondo API 验证
 │
 ├── KYC / Eligibility
 ├── Trading Limits
 ├── Market Status
 ├── Real-time Pricing
 │
 ↓
生成签名 Attestation
 │
 ↓
用户拿签名调用 Smart Contract
 │
 ├── USDC → Ondo Stock（Mint）
 └── Ondo Stock → USDC（Redeem）
```

Ondo 的 Attestation 机制本质上是一个**"链下风控 + 链上结算"**的混合模型。API 层处理所有业务逻辑（KYC、限额、市场状态、定价），Smart Contract 层只做签名验证和资金结算。

**XStocks 的发行机制**（以 Market Flow 为例）：

```
User → $100,000 USDC
          ↓
    Issuance Address
          ↓
       Issuer
          ↓
    Broker Market Buy
          ↓
   Real AAPL Stock
          ↓
    Mint / Deliver
          ↓
    User 收到 AAPLx
```

XStocks 的关键设计是：**Issuer 不维护内部库存，而是通过 Broker 按市场价格执行 underlying equity 的买卖**。这意味着发行/赎回本身就是一个真实的股票交易行为，而不是简单的 Token Mint。

**核心区别**：

- Ondo 的 Mint/Redeem 更接近**"链上结算的金融产品申购赎回"**
- XStocks 的 Issuance/Redemption 更接近**"通过区块链执行的证券经纪业务"**

### 3.5 法律结构对比

| 维度 | Ondo | XStocks |
|------|------|---------|
| **Token 法律性质** | 基金份额 / 收益型票据 | Bearer Debt Instrument + Tracker Certificate |
| **发行主体** | Ondo Finance 系列实体 | Backed Assets (JE) Limited（Jersey SPV） |
| **资产持有** | 通过 SPV 持有底层资产 | 1:1 Segregated Collateral（按产品隔离） |
| **破产隔离** | Bankruptcy Remote SPV | 三方 Account Control Agreement |
| **持有人权利** | 基金收益权 | 经济敞口（无投票权） |
| **监管框架** | 美国 / 离岸 | Jersey JFSC + Liechtenstein FMA（EU/EEA） |
| **地域限制** | 美国限制（部分产品） | 美国 / U.S. Persons 明确禁止 |

**核心区别**：

Ondo 的法律结构更接近**传统基金/票据**的框架，用已有的证券法体系来包装链上产品。

XStocks 的法律结构则专门设计了一套**"Tracker Certificate + SPV + Security Agent"**的三层保护体系：

```
       xStock Holder
            │
       owns Token
            │
            ▼
   Blockchain Token
            │
      Legal Certificate
            │
            ▼
Backed Assets (JE) Limited
   Jersey SPV / Issuer
            │
   1:1 Segregated Collateral
            │
            ▼
Regulated Broker / Custodian
            │
            ▼
    Real Stock / ETF
```

旁边还有一条独立的安全线：

```
Issuer ←→ Account Control Agreement ←→ Independent Security Agent
                                              │
                                    Default → Take Control
                                              │
                                    Liquidate / Distribute
```

XStocks 特别强调 **"Token IS the Certificate"**——区块链上的 Token 本身就是法律权属凭证，引用 Swiss DLT Act 框架。这个设计在 RWA 行业里是比较前沿的。

### 3.6 交易与流动性架构对比

| 维度 | Ondo | XStocks |
|------|------|---------|
| **Primary Market** | Attestation-based Mint/Redeem | Market Flow / xChange / xPort |
| **Secondary Market** | DEX + 部分 CEX | CEX（Bybit、Kraken 等）+ DEX + DeFi |
| **交易时间** | 按 Market Session 设计 | Primary 24/5, Secondary potentially 24/7 |
| **价格锚定** | Instant Mint/Redeem（类似稳定币 Peg） | Arbitrage via Issuance/Redemption |
| **流动性模型** | API Quote + Smart Contract Settlement | 三层：Onchain AMM + Hybrid RFQ + xChange |
| **做市机制** | Ondo 提供报价 | Market Maker + Atomic RFQ |

Ondo 的价格锚定机制和稳定币的 Peg 机制非常类似：

```
Token 价格偏离 NAV
        ↓
Instant Mint（价格过高时）
或 Instant Redeem（价格过低时）
        ↓
套利者推动价格回归
```

XStocks 则更依赖传统证券市场的套利逻辑：

```
xStock 价格 ≠ Underlying 价格
        ↓
Primary Market Issuance/Redemption
        ↓
Broker 执行真实股票买卖
        ↓
价格锚定
```

### 3.7 多链与分发策略对比

| 维度 | Ondo | XStocks |
|------|------|---------|
| **Ethereum** | 是 | 是 |
| **Solana** | 是 | 是 |
| **BNB Chain** | 是 | 是 |
| **Arbitrum** | - | 是 |
| **Mantle** | - | 是 |
| **TON** | - | 是（+ Telegram Wallet） |
| **Ink** | - | 是 |
| **CEX 分销** | 有限 | 核心（Kraken、Bybit 等） |
| **DeFi 集成** | 重要 | 重要（Wrapped xStocks via ERC-4626） |
| **API 平台** | 完整 REST + gRPC | Public + Authenticated API |

XStocks 在多链分发上明显更激进，尤其是 TON + Telegram 的布局，直接把股票代币化推向了 Web3 社交入口。

Ondo 则更注重 API 基础设施的深度，提供了从 Soft Quote 到 gRPC Streaming 的完整交易后端能力。

### 3.8 Corporate Action 处理对比

这是一个容易被忽视但极其重要的技术难点。

**Ondo** 使用 **Shares Multiplier** 来处理股票拆股/反向拆股等事件：

```
TSLA 1 share → 1.0 TSLAon
       ↓
  Stock Split
       ↓
Shares Multiplier 变化
       ↓
Token/Underlying 比例调整
```

Ondo 专门提供了 Shares Multiplier History API，说明这不是一个简单的静态映射。

**XStocks** 使用 **Onchain Rebasing** 机制：

```
Stock Split（2:1）
    ↓
100 NVDAx → 200 NVDAx

Reverse Split
    ↓
100 NVDAx → 50 NVDAx

Dividend
    ↓
重新投资到 underlying shares
    ↓
Underlying backing increases
    ↓
Multiplier/Rebase → Holder balance increases
```

XStocks 的 Dividend 处理尤其值得注意：**分红不是直接发现金，而是重新投资到底层股票**，这意味着 Token Holder 不会收到现金分红，而是通过底层资产增加来间接获得分红收益。

两者都需要一个 **Corporate Action Engine**，但实现方式不同：Ondo 通过 API + Multiplier 映射，XStocks 通过 Onchain Rebasing 直接调整链上余额。

---

## 四、核心差异总结

### 4.1 一张表看清两个项目

| 对比维度 | Ondo | XStocks |
|----------|------|---------|
| **一句话定位** | RWA 金融产品 + 可验证交易基础设施 | 股票代币化 + 链上资本市场基础设施 |
| **起点** | Tokenized Treasury（2021） | Tokenized Stocks（2025） |
| **核心资产** | 国债 → 股票 → 更多 RWA | 股票/ETF（715+ 支） |
| **技术重心** | Attestation + TEE + API 交易后端 | 三种发行通道 + 多链 + Corporate Action |
| **法律结构** | 基金/票据框架 | Tracker Certificate + Jersey SPV |
| **发行机制** | API → Attestation → Smart Contract | Market Flow / xChange / xPort |
| **交易基础设施** | 完整（Quote → Risk → Settlement → Streaming） | 三层流动性（AMM + RFQ + TradFi） |
| **CEX 整合** | 相对弱 | 核心战略（Kraken、Bybit） |
| **DeFi 整合** | 重要 | 重要（ERC-4626 Wrapped） |
| **多链** | Ethereum、Solana、BNB Chain | Ethereum、Solana、BSC、Arbitrum、Mantle、TON、Ink |
| **独特创新** | Ondo Network（TEE 可验证执行） | xPort（In-Kind 代币化）、Token = Certificate |
| **目标用户** | 机构 + FinTech + DeFi | 机构 + CEX 用户 + DeFi + Telegram 用户 |

### 4.2 两个项目反映的 RWA 行业分化

Ondo 和 XStocks 的差异，本质上反映了 RWA 行业的**两条路线之争**：

**路线 A：Ondo 模式 —— "从金融产品到交易基础设施"**

```
收益率产品（Treasury）
        ↓
公共证券（Stocks）
        ↓
交易基础设施（API + Attestation + TEE Network）
        ↓
最终形态：链上证券交易后台
```

这条路线的核心逻辑是：**先做产品积累规模和用户，再构建基础设施**。Ondo 的终极目标不是一个 Token 项目，而是一个**可验证的链上证券交易执行环境**（Ondo Network）。

**路线 B：XStocks 模式 —— "从股票代币化到链上资本市场"**

```
股票代币化产品
        ↓
多链分发 + CEX/DEX 整合
        ↓
机构级代币化基础设施（xPort）
        ↓
最终形态：全球链上资本市场
```

这条路线的核心逻辑是：**先把传统证券和区块链之间的桥梁建好，再让资产自由流动**。XStocks 的终极目标是让**全球股票市场 24/7 在链上运行**。

### 4.3 从交易所后端视角看两个项目

如果你有过中心化交易所（CEX）资产系统/交易系统的经验，这两个项目可以这样理解：

**Ondo ≈ 把交易所的交易后台搬到了链上**

```
传统交易所：
  Trading Backend → Risk Engine → Matching → Settlement

Ondo：
  Ondo API → Attestation → Smart Contract → Settlement
  (Quote/Risk/Status)    (Authorization)      (On-chain)
```

Ondo 的 API 体系几乎可以一一对应交易所的各个模块：Soft Quote = Quote Engine，Attestation = Order Authorization，Trading Limits = Risk Engine，Market Status = Market Status，gRPC Streaming = Market Data Stream。

**XStocks ≈ 把交易所的资产上架/发行/清算系统搬到了链上**

```
传统交易所：
  Asset Listing → Deposit/Withdrawal → Settlement → Reconciliation

XStocks：
  Issuance → Mint/Burn → Multi-chain → Corporate Action → Proof of Reserves
  (Market Flow/xChange/xPort)                    (Rebasing)
```

XStocks 的三种发行通道，本质上就是传统交易所的"资产入库"流程的链上版本。

---

## 五、总结

### 5.1 核心结论

1. **Ondo 和 XStocks 不是竞品，而是 RWA 赛道两条平行路线的代表**。Ondo 从 Treasury 起步，向交易基础设施演进；XStocks 从股票代币化起步，向链上资本市场演进。两者的交集在于"把证券搬上链"，但起点、重心和终局愿景不同。

2. **技术架构的核心差异在于"链下 vs 链上"的边界划分**。Ondo 选择把大量业务逻辑放在链下 API 层（Attestation），链上只做结算，并进一步用 TEE 构建可验证执行环境；XStocks 选择把发行/赎回与传统证券经纪流程深度绑定，通过 Broker 执行真实股票交易，链上 Token 只是最终的表现形式。

3. **法律结构的设计反映了不同的监管策略**。Ondo 使用相对传统的基金/票据框架，灵活性更高；XStocks 专门设计了 Jersey SPV + Tracker Certificate + Security Agent 的三层结构，并引入 Swiss DLT Act 让"Token = Certificate"，法律工程更加精密。

4. **分发策略的差异决定了不同的用户触达路径**。XStocks 深度绑定 CEX（Kraken、Bybit）+ 多链 + Telegram，走的是"大规模零售分发"路线；Ondo 更依赖 API 集成和机构合作，走的是"基础设施赋能"路线。

5. **两个项目共同验证了 RWA 行业的一个核心判断**：Token 只是最下面的一层。真正决定成败的是上面的法律结构、托管安排、合规框架、发行机制、交易基础设施和分销网络。

### 5.2 行业展望

RWA 赛道正在从"能不能做"走向"怎么大规模做"。Ondo 和 XStocks 分别给出了自己的答案：

- **Ondo 的答案**：构建一个可验证的链上交易执行环境（Ondo Network），让传统金融机构可以在链上安全地运行证券交易。
- **XStocks 的答案**：构建一个连接传统证券和链上世界的完整桥梁（发行 + 多链 + CEX/DEX + API），让全球股票市场 24/7 在链上流通。

两条路线最终可能会在某个点交汇——当 RWA 基础设施足够成熟时，"链上证券交易"和"链上资本市场"将成为同一个东西。但在那之前，Ondo 和 XStocks 的技术选择和架构取舍，仍然是这个赛道最值得持续关注的样本。

---

