---
layout: page
title: Seata分布式事务AT模式初体验
subtitle: Seata分布式事务AT模式初体验
date: 2021-01-19 20:33:00
author: valuewithTime
catalog: true
category: Seata
categories:
    - Seata
tags:
    - Seata
---

# 引言
随着应用体量的增加，微服务作为上帝之手为超级单体打开了屏障；同时使用整个应用服务显得更加清晰。服务寄托于数据，如何解决分布式事务的ACID，成为分布式事务亟待解决的问题。比较有名的分布式事务规范有XA（2PC），TCC(3PC), SAGA, 基于BASE理论的本地事务表重试达到最终一致性解决方案；基于MQ的2PC+补偿机制（事务回查）解决方案。Seata 作为一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。

# 目录
* [建库建表](#建库建表)
* [业务实现](#业务实现)
    * [定义服务](#定义服务)
    * [业务实现](#业务实现)
    * [业务测试接口](#业务测试接口)
* [服务配置](#服务配置)
  * [仓储账户服务配置](#仓储账户服务配置)
  * [订单服务配置](#订单服务配置)
  * [TC配置](#TC配置)
* [事务验证](#事务验证)
    * [正常事务]
    * [异常事务]
* [总结](#总结)
* [附](#附)





我们来模拟用户购买商品的业务逻辑。整个业务逻辑由3个微服务提供支持：

* 仓储服务：对给定的商品扣除仓储数量。   
* 订单服务：根据采购需求创建订单。  
* 帐户服务：从用户帐户中扣除余额。  

具体模型查看seata快速开始doc。

**两个应用**  

comosus-boot: 订单服务；
comosus-ravitn：仓储服务；帐户服务；
服务间通过dubbo进行通信。

配置是基于spring-cloud-starter-alibaba-seata， 在application.properties可以配置seata的相关事务配置

**角色**

* TC服务端；
* TM：comosus-boot
* RM（Client端）：comosus-boot；comosus-ravitn


# 建库建表

## comosus-ravitn数据库
comosus-ravitn数据库用于存储，商品库存，和账户金额管理；建表SQL包含undo_log表如下：

```sql
/*
Navicat MySQL Data Transfer

Source Server         : localhost
Source Server Version : 50639
Source Host           : localhost:3306
Source Database       : comosus-ravitn

Target Server Type    : MYSQL
Target Server Version : 50639
File Encoding         : 65001

Date: 2021-01-18 10:39:09
*/

SET FOREIGN_KEY_CHECKS=0;

-- ----------------------------
-- Table structure for ravitn_account
-- ----------------------------
DROP TABLE IF EXISTS `ravitn_account`;
CREATE TABLE `ravitn_account` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(64) DEFAULT NULL,
  `money` decimal(32,5) DEFAULT '0.00000',
  `is_delete` tinyint(4) DEFAULT '0' COMMENT '1:删除；0:正常',
  `remark` varchar(256) DEFAULT NULL COMMENT '备注',
  `version` bigint(20) DEFAULT NULL COMMENT '更新版本',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `create_time` datetime NOT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of ravitn_account
-- ----------------------------
INSERT INTO `ravitn_account` VALUES ('1', 'CU00000001', '1000.00000', '0', null, '1', '2021-01-05 14:37:45', '2021-01-05 14:37:42');

-- ----------------------------
-- Table structure for ravitn_storage
-- ----------------------------
DROP TABLE IF EXISTS `ravitn_storage`;
CREATE TABLE `ravitn_storage` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `commodity_code` varchar(32) DEFAULT NULL COMMENT '商品编号',
  `count` bigint(20) DEFAULT '0' COMMENT '库存数量',
  `is_delete` tinyint(4) DEFAULT '0' COMMENT '1:删除；0:正常',
  `remark` varchar(256) DEFAULT NULL COMMENT '备注',
  `version` bigint(20) DEFAULT NULL COMMENT '更新版本',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `create_time` datetime NOT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `commodity_code` (`commodity_code`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of ravitn_storage
-- ----------------------------
INSERT INTO `ravitn_storage` VALUES ('1', 'CGD0001', '1000', '0', null, '1', '2021-01-05 14:38:38', '2021-01-05 14:38:36');

-- ----------------------------
-- Table structure for undo_log
-- ----------------------------
DROP TABLE IF EXISTS `undo_log`;
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of undo_log
-- ----------------------------

```

## comosus-boot数据库
comosus-boot数据库用户存储订单数据，包括undo日志表；具体sql如下：
```sql
/*
Navicat MySQL Data Transfer

Source Server         : localhost
Source Server Version : 50639
Source Host           : localhost:3306
Source Database       : master0

Target Server Type    : MYSQL
Target Server Version : 50639
File Encoding         : 65001

Date: 2021-01-04 16:37:57
*/

SET FOREIGN_KEY_CHECKS=0;

-- ----------------------------
-- Table structure for seate_order
-- ----------------------------
DROP TABLE IF EXISTS `seate_order`;
CREATE TABLE `seate_order` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`user_id` varchar(64) DEFAULT NULL,
`commodity_code` varchar(32) DEFAULT NULL COMMENT '商品编号',
`count` int(11) DEFAULT '0',
`money` decimal(32,5) DEFAULT '0.00000',
`version` bigint(20) DEFAULT NULL COMMENT '更新版本',
`update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
`create_time` datetime NOT NULL COMMENT '创建时间',
`is_delete` tinyint(1) DEFAULT '0' COMMENT '[0:normal: 正常;1:deleted: 删除]',
`remark` varchar(32) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
-- ----------------------------
-- Table structure for undo_log
-- ----------------------------
DROP TABLE IF EXISTS `undo_log`;
CREATE TABLE `undo_log` (
`id` bigint(20) NOT NULL AUTO_INCREMENT,
`branch_id` bigint(20) NOT NULL,
`xid` varchar(100) NOT NULL,
`context` varchar(128) NOT NULL,
`rollback_info` longblob NOT NULL,
`log_status` int(11) NOT NULL,
`log_created` datetime NOT NULL,
`log_modified` datetime NOT NULL,
`ext` varchar(100) DEFAULT NULL,
PRIMARY KEY (`id`),
UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)

```


## TC数据库
TC提供了本地文件，db和redis三种方式存储事务日志，我们选择db方式，以便观察对应的事务过程。具体sql如下：

```sql
/*
Navicat MySQL Data Transfer

Source Server         : localhost
Source Server Version : 50639
Source Host           : localhost:3306
Source Database       : seata

Target Server Type    : MYSQL
Target Server Version : 50639
File Encoding         : 65001

Date: 2021-01-12 19:35:23
*/

SET FOREIGN_KEY_CHECKS=0;

-- ----------------------------
-- Table structure for branch_table
-- ----------------------------
DROP TABLE IF EXISTS `branch_table`;
CREATE TABLE `branch_table` (
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(128) NOT NULL,
  `transaction_id` bigint(20) DEFAULT NULL,
  `resource_group_id` varchar(32) DEFAULT NULL,
  `resource_id` varchar(256) DEFAULT NULL,
  `lock_key` varchar(128) DEFAULT NULL,
  `branch_type` varchar(8) DEFAULT NULL,
  `status` tinyint(4) DEFAULT NULL,
  `client_id` varchar(64) DEFAULT NULL,
  `application_data` varchar(2000) DEFAULT NULL,
  `gmt_create` datetime DEFAULT NULL,
  `gmt_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`branch_id`),
  KEY `idx_xid` (`xid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ----------------------------
-- Table structure for global_table
-- ----------------------------
DROP TABLE IF EXISTS `global_table`;
CREATE TABLE `global_table` (
  `xid` varchar(128) NOT NULL,
  `transaction_id` bigint(20) DEFAULT NULL,
  `status` tinyint(4) NOT NULL,
  `application_id` varchar(32) DEFAULT NULL,
  `transaction_service_group` varchar(32) DEFAULT NULL,
  `transaction_name` varchar(128) DEFAULT NULL,
  `timeout` int(11) DEFAULT NULL,
  `begin_time` bigint(20) DEFAULT NULL,
  `application_data` varchar(2000) DEFAULT NULL,
  `gmt_create` datetime DEFAULT NULL,
  `gmt_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`xid`),
  KEY `idx_gmt_modified_status` (`gmt_modified`,`status`),
  KEY `idx_transaction_id` (`transaction_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ----------------------------
-- Table structure for lock_table
-- ----------------------------
DROP TABLE IF EXISTS `lock_table`;
CREATE TABLE `lock_table` (
  `row_key` varchar(128) NOT NULL,
  `xid` varchar(96) DEFAULT NULL,
  `transaction_id` mediumtext,
  `branch_id` mediumtext,
  `resource_id` varchar(256) DEFAULT NULL,
  `table_name` varchar(32) DEFAULT NULL,
  `pk` varchar(36) DEFAULT NULL,
  `gmt_create` datetime DEFAULT NULL,
  `gmt_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`row_key`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ----------------------------
-- Table structure for undo_log
-- ----------------------------
DROP TABLE IF EXISTS `undo_log`;
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```
建库建表已经准备好了，我们来实现业务。

# 业务实现 
引用需要引入seata的相关包, 几版本信息如下：

```xml
  <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <springfox.swagger2.version>2.9.2</springfox.swagger2.version>
        <spring.boot.version>2.1.1.RELEASE</spring.boot.version>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <metrics.version>2.0.1</metrics.version>
        <alibaba.boot.dubbo.springboot.starter>0.2.1.RELEASE</alibaba.boot.dubbo.springboot.starter>
        <sharding-sphere.version>4.1.0</sharding-sphere.version>
        <seata-spring-boot-starter.version>1.4.1</seata-spring-boot-starter.version>
        <dubbo.version>2.6.5</dubbo.version>
        <comosus-ravitn-api.version>1.0.0-SNAPSHOT</comosus-ravitn-api.version>
    </properties>
```

使用的dubbo版本为2.6.5， dubbo-boot-starter的版本为0.2.1.RELEASE。 Seata版本为为1.4.1。spring-cloud-starter-alibaba-seata为2.2.1.RELEASE。


```xml
<!-- seata start-->
        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring-boot-starter</artifactId>
            <version>${seata-spring-boot-starter.version}</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
            <version>2.2.1.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>io.seata</groupId>
                    <artifactId>seata-spring-boot-starter</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!--seata end-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
            <version>2.1.1.RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>org.bouncycastle</groupId>
                    <artifactId>bcpkix-jdk15on</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
```

注意spring-cloud-starter-alibaba-seata需要依赖于hystrix。

先来定义服务
## 定义服务

在comosus-ravitn中定义仓储服务；帐户服务；

### 仓储服务
```java
package com.story.home.ravitn.api.domain.storage;

/**
 * @ClassName: StorageRemoteService
 * @Description:
 * @Author: VT
 * @Date: 2021-01-05 10:56
 */
public interface StorageRemoteService {
    /**
     * 扣除存储数量
     */
    Boolean deduct(String commodityCode, int count);
}

```

### 帐户服务
```java
package com.story.home.ravitn.api.domain.account.service;


import java.math.BigDecimal;

/**
 * @ClassName: AccountRemoteService
 * @Description:
 * @Author: Donaldhan
 * @Date: 2018-09-02 15:06
 */
public interface AccountRemoteService {
    /**
     * 从用户账户中借出
     */
    Boolean debit(String userId, BigDecimal money);
}

```

具体业务的实现，我们就不说，基于Mybatis的CRUD Mapper 操作。

我们在账户业务中模拟账户不存在的事务异常，具体如下：

```java
package com.story.home.ravitn.account.service.biz.impl;

import com.story.home.ravitn.account.entity.Account;
import com.story.home.ravitn.account.service.base.AccountDbService;
import com.story.home.ravitn.account.service.biz.AccountBizService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.util.ObjectUtils;

import javax.annotation.Resource;
import java.math.BigDecimal;

/**
 * @ClassName: AccountBizServiceImpl
 * @Description:
 * @Author: VT
 * @Date: 2021-01-04 16:31
 */
@Slf4j
@Service
public class AccountBizServiceImpl implements AccountBizService {
    @Resource
    AccountDbService accountDbService;

    @Override
    public Boolean debit(String userId, BigDecimal money) {
        Account account = accountDbService.findAccountByUserId(userId);
        if(ObjectUtils.isEmpty(account)){
            throw new IllegalStateException("account is null userId:"+userId);
        }
        account.setMoney(account.getMoney().subtract(money));
        return accountDbService.updateAccount(account);
    }
}

```


### 订单服务
在comosus-boot提供订单服务，这里我们忽略，直接做业务了


## 业务实现

```java
package org.home.seata.order.service.facade.impl;

import com.alibaba.dubbo.config.annotation.Reference;
import com.story.home.ravitn.api.domain.account.service.AccountRemoteService;
import com.story.home.ravitn.api.domain.storage.StorageRemoteService;
import io.seata.spring.annotation.GlobalTransactional;
import lombok.extern.slf4j.Slf4j;
import org.home.seata.order.service.biz.OrderBizService;
import org.home.seata.order.service.facade.OrderFacadeService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;

/**
 * @ClassName: OrderFacadeServiceImpl
 * @Description:
 * @Author: VT
 * @Date: 2021-01-05 14:06
 */
@Slf4j
@Service
public class OrderFacadeServiceImpl implements OrderFacadeService {
    @Autowired
    OrderBizService orderBizService;
    @Reference(validation = "true",check = false, version = "1.0.0")
    StorageRemoteService storageRemoteService;
    @Reference(validation = "true",check = false, version = "1.0.0")
    AccountRemoteService accountRemoteService;

    @GlobalTransactional(rollbackFor = Exception.class,  timeoutMills = 30000, name = "purchase-tx")
    @Override
    public Boolean purchase(String userId, String commodityCode, int orderCount) {
        try {
            Boolean result =  storageRemoteService.deduct(commodityCode,orderCount);
            if(!result){
                log.error("OrderFacadeService purchase storage deduct  error");
            }
            BigDecimal money =  new BigDecimal("16.8");
            result = orderBizService.create(userId,commodityCode,orderCount,money);
            if(!result){
                log.error("OrderFacadeService purchase order create  error");
            }
            result =  accountRemoteService.debit(userId,money);
            if(!result){
                log.error("OrderFacadeService purchase  account debit error");
            }
            return result;
        } catch (Exception e) {
            log.error("OrderFacadeService purchase error");
            throw new IllegalStateException(e);
        }
    }
}

```
order 相关业务操作，基于Mappder的Insert操作，不在赘述。

我们使用接口测试业务

## 业务测试接口

```java
package org.home.web.seata;

import io.swagger.annotations.Api;
import io.swagger.annotations.ApiOperation;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.home.seata.order.service.facade.OrderFacadeService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Profile;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @ClassName: TestSeateController
 * @Description:
 * @Author: VT
 * @Date: 2021-01-05 11:02
 */
@Slf4j
@Profile("seata")
@RestController
@RequestMapping("/api/test/seata")
@Api(tags = {"seata"}, description = "seata test")
public class TestSeataController {
    @Autowired
    OrderFacadeService orderFacadeService;
    /**
     * @return
     */
    @PostMapping("/purchase")
    @ApiOperation("下单")
    public String purchase(@RequestBody PurchaseForm purchaseForm)  {
        orderFacadeService.purchase(purchaseForm.getUserId(),purchaseForm.getCommodityCode(), purchaseForm.getOrderCount());
        return "ok";
    }
    @Data
    static class PurchaseForm {
        private String userId;
        private String commodityCode;
        private int orderCount;
    }

}
```
业务实现完，我们来配置应用
# 服务配置


## 仓储账户服务配置

```properties
# seata

seata.enabled=true
# SeataProperties
seata.application-id=comosus-ravitn
seata.tx-service-group=comosus-ravitn-tx
# SpringCloudAlibabaConfiguration
#spring.cloud.alibaba.seata.application-id=comosus-ravitn
#spring.cloud.alibaba.seata.tx-service-group=comosus-ravitn-tx
seata.data-source-proxy-mode=AT
seata.enable-auto-data-source-proxy=true
seata.use-jdk-proxy=false

## seata service
seata.service.disable-global-transaction=false
seata.service.enable-degrade=false
seata.service.vgroup-mapping.comosus-ravitn-tx=tc-cluster
#only support when registry.type=file, please don't set multiple addresses
seata.service.grouplist.tc-cluster=127.0.0.1:8091

## registry zk
#seata.registry.type=file
seata.registry.zk.server-addr=127.0.0.1:2181
seata.registry.zk.session-timeout=6000
seata.registry.zk.connect-timeout=2000
#seata.registry.zk.username=
#seata.registry.zk.password=
seata.registry.load-balance=RoundRobinLoadBalance
seata.registry.load-balance-virtual-nodes=10


## seata config

### file config
#seata.config.type=file
#seata.config.type=springCloudConfig
#seata.config.file.name=file.conf

### nacos config
#seata.config.type=nacos
#seata.config.nacos.group=comosus-ravitn-seata-group
#seata.config.nacos.server-addr=localhost
#seata.config.nacos.username=
#seata.config.nacos.password=
#seata.config.nacos.namespace=



## seata client
### tm
seata.client.tm.commit-retry-count=5
seata.client.tm.rollback-retry-count=5
seata.client.tm.default-global-transaction-timeout=60000
seata.client.tm.degrade-check-allow-times=10
seata.client.tm.degrade-check=false
seata.client.tm.degrade-check-period=2000

### rm
seata.client.rm.lock.retry-times=30
seata.client.rm.async-commit-buffer-limit=10000
seata.client.rm.lock.retry-interval=10
seata.client.rm.lock.retry-policy-branch-rollback-on-conflict=true
seata.client.rm.table-meta-check-enable=false
seata.client.rm.report-retry-count=5
seata.client.rm.report-success-enable=false
seata.client.rm.saga-json-parser=fastjson
seata.client.rm.saga-branch-register-enable=false

### undo log
seata.client.undo.log-table=undo_log
seata.client.undo.data-validation=true
seata.client.undo.log-serialization=fastjson
seata.client.undo.only-care-update-columns=true


## seata transport
seata.transport.server=nio
seata.transport.type=tcp
seata.transport.heartbeat=true
seata.transport.serialization=seata
seata.transport.enable-client-batch-send-request=true
seata.transport.shutdown.wait=3
# zip
seata.transport.compressor=none
seata.transport.thread-factory.boss-thread-prefix=comosus-ravitn
seata.transport.thread-factory.boss-thread-size=1
seata.transport.thread-factory.server-executor-thread-prefix=NettyServerBizHandler
seata.transport.thread-factory.share-boss-worker=false
seata.transport.thread-factory.worker-thread-prefix=NettyServerNIOWorker
seata.transport.thread-factory.worker-thread-size=Default
seata.transport.thread-factory.client-worker-thread-prefix=NettyClientWorkerThread
seata.transport.thread-factory.client-selector-thread-prefix=NettyClientSelector
seata.transport.thread-factory.client-selector-thread-size=1
```

注意seata.service.disable-global-transaction=false配置，默认为false，如果为true，则全局事务无效。

## 订单服务配置

```properties
# seata

seata.enabled=true
# SeataProperties
seata.application-id=comosus-boot
seata.tx-service-group=comosus-boot-tx
# SpringCloudAlibabaConfiguration
#spring.cloud.alibaba.seata.application-id=comosus-boot
#spring.cloud.alibaba.seata.tx-service-group=comosus-boot-tx
seata.data-source-proxy-mode=AT
seata.enable-auto-data-source-proxy=true
seata.use-jdk-proxy=false

## seata service
seata.service.disable-global-transaction=false
seata.service.enable-degrade=false
seata.service.vgroup-mapping.comosus-boot-tx=tc-cluster
#only support when registry.type=file, please don't set multiple addresses
seata.service.grouplist.tc-cluster=127.0.0.1:8091

## registry zk
#seata.registry.type=file
seata.registry.zk.server-addr=127.0.0.1:2181
seata.registry.zk.session-timeout=6000
seata.registry.zk.connect-timeout=2000
#seata.registry.zk.username=
#seata.registry.zk.password=
seata.registry.load-balance=RoundRobinLoadBalance
seata.registry.load-balance-virtual-nodes=10


## seata config

### file config
#seata.config.type=file
#seata.config.type=springCloudConfig
#seata.config.file.name=file.conf

### nacos config
#seata.config.type=nacos
#seata.config.nacos.group=comosus-boot-seata-group
#seata.config.nacos.server-addr=localhost
#seata.config.nacos.username=
#seata.config.nacos.password=
#seata.config.nacos.namespace=



## seata client
### tm
seata.client.tm.commit-retry-count=5
seata.client.tm.rollback-retry-count=5
seata.client.tm.default-global-transaction-timeout=60000
seata.client.tm.degrade-check-allow-times=10
seata.client.tm.degrade-check=false
seata.client.tm.degrade-check-period=2000

### rm
seata.client.rm.lock.retry-times=30
seata.client.rm.async-commit-buffer-limit=10000
seata.client.rm.lock.retry-interval=10
seata.client.rm.lock.retry-policy-branch-rollback-on-conflict=true
seata.client.rm.table-meta-check-enable=false
seata.client.rm.report-retry-count=5
seata.client.rm.report-success-enable=false
seata.client.rm.saga-json-parser=fastjson
seata.client.rm.saga-branch-register-enable=false

### undo log
seata.client.undo.log-table=undo_log
seata.client.undo.data-validation=true
seata.client.undo.log-serialization=fastjson
seata.client.undo.only-care-update-columns=true


## seata transport
seata.transport.server=nio
seata.transport.type=tcp
seata.transport.heartbeat=true
seata.transport.serialization=seata
seata.transport.enable-client-batch-send-request=true
seata.transport.shutdown.wait=3
# zip
seata.transport.compressor=none
seata.transport.thread-factory.boss-thread-prefix=comosus-boot
seata.transport.thread-factory.boss-thread-size=1
seata.transport.thread-factory.server-executor-thread-prefix=NettyServerBizHandler
seata.transport.thread-factory.share-boss-worker=false
seata.transport.thread-factory.worker-thread-prefix=NettyServerNIOWorker
seata.transport.thread-factory.worker-thread-size=Default
seata.transport.thread-factory.client-worker-thread-prefix=NettyClientWorkerThread
seata.transport.thread-factory.client-selector-thread-prefix=NettyClientSelector
seata.transport.thread-factory.client-selector-thread-size=1
```

## TC配置
配置server， 从seata的github仓库下载相应的tc server；配置文件在conf目录下的file.conf，我们使用的是db方式具体配置如下：
```conf
## transaction log store, only used in seata-server
store {
  ## store mode: file、db、redis
  mode = "db"

  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }

  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    driverClassName = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata"
    user = "root"
    password = "123456"
    minConn = 5
    maxConn = 100
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }

  ## redis store property
  redis {
    host = "127.0.0.1"
    port = "6379"
    password = ""
    database = "0"
    minConn = 1
    maxConn = 10
    maxTotal = 100
    queryLimit = 100
  }

}
lock {
  ## the lock store mode: local、remote
  mode = "remote"

  local {
    ## store locks in user's database
  }

  remote {
    ## store locks in the seata's server
  }
}
recovery {
  #schedule committing retry period in milliseconds
  committing-retry-period = 1000
  #schedule asyn committing retry period in milliseconds
  asyn-committing-retry-period = 1000
  #schedule rollbacking retry period in milliseconds
  rollbacking-retry-period = 1000
  #schedule timeout retry period in milliseconds
  timeout-retry-period = 1000
}
transaction {
  undo.data.validation = true
  undo.log.serialization = "jackson"
  undo.log.save.days = 7
  #schedule delete expired undo_log in milliseconds
  undo.log.delete.period = 86400000
  undo.log.table = "undo_log"
}

## metrics settings
metrics {
  enabled = false
  registry-type = "compact"
  # multi exporters use comma divided
  exporter-list = "prometheus"
  exporter-prometheus-port = 9898
}

support {
  ## spring
  spring {
    # auto proxy the DataSource bean
    datasource.autoproxy = false
  }
}
```

接下来我们来验证事务。


# 事务验证
先启动TC服务
 ```cmd
  seata-server.bat -p 8091 -h 127.0.0.1 -m db
```

然后分别启动
两个应用
comosus-boot: 订单服务；
comosus-ravitn：仓储服务；帐户服务；

TC控制台日志如下：
```log
20:23:09.996  INFO --- [rverHandlerThread_1_1_500] i.s.c.r.processor.server.RegRmProcessor  : RM register success,message:RegisterRMRequest{resourceIds='jdbc:mysql://localhost:3306/comosus-boot', applicationId='comosus-boot', transactionServiceGroup='comosus-boot-tx'},channel:[id: 0x2241d2d0, L:/127.0.0.1:8091 - R:/127.0.0.1:56709],client version:1.4.1
20:23:10.060  INFO --- [rverHandlerThread_1_2_500] i.s.c.r.processor.server.RegRmProcessor  : RM register success,message:RegisterRMRequest{resourceIds='jdbc:mysql://localhost:3306/comosus-boot', applicationId='comosus-boot', transactionServiceGroup='comosus-boot-tx'},channel:[id: 0x2241d2d0, L:/127.0.0.1:8091 - R:/127.0.0.1:56709],client version:1.4.1
20:23:28.886  INFO --- [rverHandlerThread_1_3_500] i.s.c.r.processor.server.RegRmProcessor  : RM register success,message:RegisterRMRequest{resourceIds='jdbc:mysql://localhost:3306/comosus-ravitn', applicationId='comosus-ravitn', transactionServiceGroup='comosus-ravitn-tx'},channel:[id: 0x9f8692e2, L:/127.0.0.1:8091 - R:/127.0.0.1:56780],client version:1.4.1
20:23:28.954  INFO --- [rverHandlerThread_1_4_500] i.s.c.r.processor.server.RegRmProcessor  : RM register success,message:RegisterRMRequest{resourceIds='jdbc:mysql://localhost:3306/comosus-ravitn', applicationId='comosus-ravitn', transactionServiceGroup='comosus-ravitn-tx'},channel:[id: 0x9f8692e2, L:/127.0.0.1:8091 - R:/127.0.0.1:56780],client version:1.4.1
20:24:01.842  INFO --- [ettyServerNIOWorker_1_3_8] i.s.c.r.processor.server.RegTmProcessor  : TM register success,message:RegisterTMRequest{applicationId='comosus-boot', transactionServiceGroup='comosus-boot-tx'},channel:[id: 0x4b144e0e, L:/127.0.0.1:8091 - R:/127.0.0.1:56840],client version:1.4.1
20:24:20.302  INFO --- [ettyServerNIOWorker_1_4_8] i.s.c.r.processor.server.RegTmProcessor  : TM register success,message:RegisterTMRequest{applicationId='comosus-ravitn', transactionServiceGroup='comosus-ravitn-tx'},channel:[id: 0x199abf63, L:/127.0.0.1:8091 - R:/127.0.0.1:56861],client version:1.4.1
```
从上面日志可以看出，comosus-ravitn（TM、RM），comosus-boot（TM、RM）注册到TC服务中心。

接下来我们验证正常的业务，为了便于观察我们这个过程，我们在业务中添加断点,具体为

```java
result =  accountRemoteService.debit(userId,money);
```

## 正常事务

先来看一下初始数据（comosus-ravitn：仓储服务；帐户服务）：

```
mysql> select * from ravitn_account;
+----+------------+------------+-----------+--------+---------+---------------------+---------------------+
| id | user_id    | money      | is_delete | remark | version | update_time         | create_time         |
+----+------------+------------+-----------+--------+---------+---------------------+---------------------+
|  1 | CU00000001 | 1000.00000 |         0 | NULL   |       1 | 2021-01-05 14:37:45 | 2021-01-05 14:37:42 |
+----+------------+------------+-----------+--------+---------+---------------------+---------------------+
1 row in set

mysql> select * from ravitn_storage;
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
| id | commodity_code | count | is_delete | remark | version | update_time         | create_time         |
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
|  1 | CGD0001        |  1000 |         0 | NULL   |       1 | 2021-01-05 14:38:38 | 2021-01-05 14:38:36 |
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
1 row in set

mysql> 
```


开启正常事务，调用接口参数如下：

```json
{
  "commodityCode": "CGD0001",
  "orderCount": 1,
  "userId": "CU00000001"
}
```


进入断点，观察一下相关表信息：

comosus-boot: 订单服务；
comosus-ravitn：仓储服务；帐户服务；
TC:seata-sever


1. comosus-ravitn
```sql
mysql> select * from undo_log;
+----+-------------------+---------------------------------------+---------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
| id | branch_id         | xid                                   | context             | rollback_info                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | log_status | log_created         | log_modified        | ext  |
+----+-------------------+---------------------------------------+---------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
|  1 | 94531698438029312 | 10.242.214.156:8091:94531694608629760 | serializer=fastjson | {"@type":"io.seata.rm.datasource.undo.BranchUndoLog","branchId":94531698438029312,"sqlUndoLogs":[{"afterImage":{"rows":[{"fields":[{"keyType":"PRIMARY_KEY","name":"id","type":-5,"value":1L},{"keyType":"NULL","name":"commodity_code","type":12,"value":"CGD0001"},{"keyType":"NULL","name":"count","type":-5,"value":999L},{"keyType":"NULL","name":"version","type":-5,"value":2L}]}],"tableName":"ravitn_storage"},"beforeImage":{"rows":[{"fields":[{"keyType":"PRIMARY_KEY","name":"id","type":-5,"value":1L},{"keyType":"NULL","name":"commodity_code","type":12,"value":"CGD0001"},{"keyType":"NULL","name":"count","type":-5,"value":1000L},{"keyType":"NULL","name":"version","type":-5,"value":1L}]}],"tableName":"ravitn_storage"},"sqlType":"UPDATE","tableName":"ravitn_storage"}],"xid":"10.242.214.156:8091:94531694608629760"} |          0 | 2021-01-18 20:35:13 | 2021-01-18 20:35:13 | NULL |
+----+-------------------+---------------------------------------+---------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
1 row in set
```

2. comosus-boot
```sql
+----+-------------------+---------------------------------------+---------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
|  1 | 94531700921057281 | 10.242.214.156:8091:94531694608629760 | serializer=fastjson | {"@type":"io.seata.rm.datasource.undo.BranchUndoLog","branchId":94531700921057281,"sqlUndoLogs":[{"afterImage":{"rows":[{"fields":[{"keyType":"PRIMARY_KEY","name":"id","type":4,"value":1},{"keyType":"NULL","name":"user_id","type":12,"value":"CU00000001"},{"keyType":"NULL","name":"commodity_code","type":12,"value":"CGD0001"},{"keyType":"NULL","name":"count","type":4,"value":1},{"keyType":"NULL","name":"money","type":3,"value":16.80000},{"keyType":"NULL","name":"version","type":-5,"value":1L},{"keyType":"NULL","name":"update_time","type":93,"value":"2021-01-18 20:35:13"},{"keyType":"NULL","name":"create_time","type":93,"value":"2021-01-18 20:35:13"},{"keyType":"NULL","name":"is_delete","type":-7,"value":false},{"keyType":"NULL","name":"remark","type":12}]}],"tableName":"seate_order"},"beforeImage":{"@type":"io.seata.rm.datasource.sql.struct.TableRecords$EmptyTableRecords","rows":[],"tableName":"seate_order"},"sqlType":"INSERT","tableName":"seate_order"}],"xid":"10.242.214.156:8091:94531694608629760"} |          0 | 2021-01-18 20:35:14 | 2021-01-18 20:35:14 | NULL |
+----+-------------------+---------------------------------------+---------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
1 row in set

mysql> 
```

3. seata-sever
```sql
mysql> select * from global_table;
+---------------------------------------+-------------------+--------+----------------+---------------------------+------------------+---------+---------------+------------------+---------------------+---------------------+
| xid                                   | transaction_id    | status | application_id | transaction_service_group | transaction_name | timeout | begin_time    | application_data | gmt_create          | gmt_modified        |
+---------------------------------------+-------------------+--------+----------------+---------------------------+------------------+---------+---------------+------------------+---------------------+---------------------+
| 10.242.214.156:8091:94531694608629760 | 94531694608629760 |      6 | comosus-boot   | comosus-boot-tx           | purchase-tx      |   30000 | 1610973312308 | NULL             | 2021-01-18 20:35:12 | 2021-01-18 20:35:42 |
+---------------------------------------+-------------------+--------+----------------+---------------------------+------------------+---------+---------------+------------------+---------------------+---------------------+
1 row in set

mysql> select * from branch table;
1064 - You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'table' at line 1
mysql> select * from branch_table;
+-------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------+-------------+--------+--------------------------------+------------------+---------------------+---------------------+
| branch_id         | xid                                   | transaction_id    | resource_group_id | resource_id                                | lock_key | branch_type | status | client_id                      | application_data | gmt_create          | gmt_modified        |
+-------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------+-------------+--------+--------------------------------+------------------+---------------------+---------------------+
| 94531698438029312 | 10.242.214.156:8091:94531694608629760 | 94531694608629760 | NULL              | jdbc:mysql://localhost:3306/comosus-ravitn | NULL     | AT          |      0 | comosus-ravitn:127.0.0.1:56780 | NULL             | 2021-01-18 20:35:13 | 2021-01-18 20:35:13 |
| 94531700921057281 | 10.242.214.156:8091:94531694608629760 | 94531694608629760 | NULL              | jdbc:mysql://localhost:3306/comosus-boot   | NULL     | AT          |      0 | comosus-boot:127.0.0.1:56709   | NULL             | 2021-01-18 20:35:14 | 2021-01-18 20:35:14 |
+-------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------+-------------+--------+--------------------------------+------------------+---------------------+---------------------+
2 rows in set

mysql> select * from lock_table;
+-----------------------------------------------------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------------+----+---------------------+---------------------+
| row_key                                                         | xid                                   | transaction_id    | branch_id         | resource_id                                | table_name     | pk | gmt_create          | gmt_modified        |
+-----------------------------------------------------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------------+----+---------------------+---------------------+
| jdbc:mysql://localhost:3306/comosus-boot^^^seate_order^^^1      | 10.242.214.156:8091:94531694608629760 | 94531694608629760 | 94531700921057281 | jdbc:mysql://localhost:3306/comosus-boot   | seate_order    | 1  | 2021-01-18 20:35:13 | 2021-01-18 20:35:13 |
| jdbc:mysql://localhost:3306/comosus-ravitn^^^ravitn_storage^^^1 | 10.242.214.156:8091:94531694608629760 | 94531694608629760 | 94531698438029312 | jdbc:mysql://localhost:3306/comosus-ravitn | ravitn_storage | 1  | 2021-01-18 20:35:13 | 2021-01-18 20:35:13 |
+-----------------------------------------------------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------------+----+---------------------+---------------------+
2 rows in set

```
从上面可以看出，有comosus-boot开启了一个全局事务，TM为comosus-boot， 两个事务分支，分别为jdbc:mysql://localhost:3306/comosus-ravitn和 jdbc:mysql://localhost:3306/comosus-boot，以及相关的表行锁。comosus-boot和comosus-ravitn分别记录相应的undo_log.

在没有断点的情况，正常情况，上面的全局事务信息，和事务分支，table锁及undo-log都是中间过程，如果全局事务完成，则将会删除相关的中间终态。

正常的完成事务后，相关数据为

仓单和账户数据如下：

```sql
mysql> select * from ravitn_storage;
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
| id | commodity_code | count | is_delete | remark | version | update_time         | create_time         |
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
|  1 | CGD0001        |   999 |         0 | NULL   |       2 | 2021-01-18 20:40:09 | 2021-01-05 14:38:36 |
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
1 row in set

mysql> select * from ravitn_account;
+----+------------+-----------+-----------+--------+---------+---------------------+---------------------+
| id | user_id    | money     | is_delete | remark | version | update_time         | create_time         |
+----+------------+-----------+-----------+--------+---------+---------------------+---------------------+
|  1 | CU00000001 | 983.20000 |         0 | NULL   |       2 | 2021-01-18 20:40:09 | 2021-01-05 14:37:42 |
+----+------------+-----------+-----------+--------+---------+---------------------+---------------------+
1 row in set

mysql> 
```


订单数据如下：
```sql
mysql> select * from seate_order;
+----+------------+----------------+-------+----------+---------+---------------------+---------------------+-----------+--------+
| id | user_id    | commodity_code | count | money    | version | update_time         | create_time         | is_delete | remark |
+----+------------+----------------+-------+----------+---------+---------------------+---------------------+-----------+--------+
|  2 | CU00000001 | CGD0001        |     1 | 16.80000 |       1 | 2021-01-18 20:40:09 | 2021-01-18 20:40:09 |         0 | NULL   |
+----+------------+----------------+-------+----------+---------+---------------------+---------------------+-----------+--------+
1 row in set

mysql> 
```

## 异常事务

全局事务我们，我们需要引入断点，观察整个过程, 准备数据如下：
```json
{
  "commodityCode": "CGD0001",
  "orderCount": 1,
  "userId": "CU00000002"
}
```


进入断点，观察一下相关表信息：

comosus-boot: 订单服务；
comosus-ravitn：仓储服务；帐户服务；
TC:seata-sever


订单数据
```sql
mysql> select * from seate_order;
+----+------------+----------------+-------+----------+---------+---------------------+---------------------+-----------+--------+
| id | user_id    | commodity_code | count | money    | version | update_time         | create_time         | is_delete | remark |
+----+------------+----------------+-------+----------+---------+---------------------+---------------------+-----------+--------+
|  2 | CU00000001 | CGD0001        |     1 | 16.80000 |       1 | 2021-01-18 20:40:09 | 2021-01-18 20:40:09 |         0 | NULL   |
|  3 | CU00000002 | CGD0001        |     1 | 16.80000 |       1 | 2021-01-18 20:55:55 | 2021-01-18 20:55:55 |         0 | NULL   |
+----+------------+----------------+-------+----------+---------+---------------------+---------------------+-----------+--------+
2 rows in set

mysql> 
```

库存数据

```sql
mysql> select * from ravitn_storage;
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
| id | commodity_code | count | is_delete | remark | version | update_time         | create_time         |
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
|  1 | CGD0001        |   998 |         0 | NULL   |       3 | 2021-01-18 20:55:54 | 2021-01-05 14:38:36 |
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
1 row in set

```



异常发生，事务回滚后

库存数据
```sql
mysql> select * from seate_order;
+----+------------+----------------+-------+----------+---------+---------------------+---------------------+-----------+--------+
| id | user_id    | commodity_code | count | money    | version | update_time         | create_time         | is_delete | remark |
+----+------------+----------------+-------+----------+---------+---------------------+---------------------+-----------+--------+
|  2 | CU00000001 | CGD0001        |     1 | 16.80000 |       1 | 2021-01-18 20:40:09 | 2021-01-18 20:40:09 |         0 | NULL   |
+----+------------+----------------+-------+----------+---------+---------------------+---------------------+-----------+--------+
1 row in set
```

```sql
mysql> select * from ravitn_storage;
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
| id | commodity_code | count | is_delete | remark | version | update_time         | create_time         |
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
|  1 | CGD0001        |   999 |         0 | NULL   |       2 | 2021-01-18 20:57:02 | 2021-01-05 14:38:36 |
+----+----------------+-------+-----------+--------+---------+---------------------+---------------------+
1 row in set
```
从上面可以看出，在账户扣款失败之前，已经减库存，并生成订单，在异常发生，回滚事务后，则根据undo_log回滚相关数据。

在这个过程中， 全局事务信息，和事务分支，table锁及undo-log都存在相应的事务相关数据，如果全局事务完成包括异常，则将会删除相关的中间状。

我们把断点放到if语句来看一下分支表和lock表
```java
result =  accountRemoteService.debit(userId,money);
if(!result){
       log.error("OrderFacadeService purchase  account debit error");
}
```

具体数据如下：

```sql
mysql> select * from global_table;
+---------------------------------------+-------------------+--------+----------------+---------------------------+------------------+---------+---------------+------------------+---------------------+---------------------+
| xid                                   | transaction_id    | status | application_id | transaction_service_group | transaction_name | timeout | begin_time    | application_data | gmt_create          | gmt_modified        |
+---------------------------------------+-------------------+--------+----------------+---------------------------+------------------+---------+---------------+------------------+---------------------+---------------------+
| 10.242.214.156:8091:94744863792807936 | 94744863792807936 |      6 | comosus-boot   | comosus-boot-tx           | purchase-tx      |   30000 | 1611024135802 | NULL             | 2021-01-19 10:42:15 | 2021-01-19 10:42:46 |
+---------------------------------------+-------------------+--------+----------------+---------------------------+------------------+---------+---------------+------------------+---------------------+---------------------+
1 row in set

mysql> select * from branch_table;
+-------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------+-------------+--------+--------------------------------+------------------+---------------------+---------------------+
| branch_id         | xid                                   | transaction_id    | resource_group_id | resource_id                                | lock_key | branch_type | status | client_id                      | application_data | gmt_create          | gmt_modified        |
+-------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------+-------------+--------+--------------------------------+------------------+---------------------+---------------------+
| 94744867513155584 | 10.242.214.156:8091:94744863792807936 | 94744863792807936 | NULL              | jdbc:mysql://localhost:3306/comosus-ravitn | NULL     | AT          |      0 | comosus-ravitn:127.0.0.1:62241 | NULL             | 2021-01-19 10:42:17 | 2021-01-19 10:42:17 |
| 94744870000377857 | 10.242.214.156:8091:94744863792807936 | 94744863792807936 | NULL              | jdbc:mysql://localhost:3306/comosus-boot   | NULL     | AT          |      0 | comosus-boot:127.0.0.1:62233   | NULL             | 2021-01-19 10:42:17 | 2021-01-19 10:42:17 |
| 94744870734381057 | 10.242.214.156:8091:94744863792807936 | 94744863792807936 | NULL              | jdbc:mysql://localhost:3306/comosus-ravitn | NULL     | AT          |      0 | comosus-ravitn:127.0.0.1:62241 | NULL             | 2021-01-19 10:42:17 | 2021-01-19 10:42:17 |
+-------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------+-------------+--------+--------------------------------+------------------+---------------------+---------------------+
3 rows in set

mysql> select * from lock_table;
+-----------------------------------------------------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------------+----+---------------------+---------------------+
| row_key                                                         | xid                                   | transaction_id    | branch_id         | resource_id                                | table_name     | pk | gmt_create          | gmt_modified        |
+-----------------------------------------------------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------------+----+---------------------+---------------------+
| jdbc:mysql://localhost:3306/comosus-boot^^^seate_order^^^3      | 10.242.214.156:8091:94744863792807936 | 94744863792807936 | 94744870000377857 | jdbc:mysql://localhost:3306/comosus-boot   | seate_order    | 3  | 2021-01-19 10:42:17 | 2021-01-19 10:42:17 |
| jdbc:mysql://localhost:3306/comosus-ravitn^^^ravitn_account^^^1 | 10.242.214.156:8091:94744863792807936 | 94744863792807936 | 94744870734381057 | jdbc:mysql://localhost:3306/comosus-ravitn | ravitn_account | 1  | 2021-01-19 10:42:17 | 2021-01-19 10:42:17 |
| jdbc:mysql://localhost:3306/comosus-ravitn^^^ravitn_storage^^^1 | 10.242.214.156:8091:94744863792807936 | 94744863792807936 | 94744867513155584 | jdbc:mysql://localhost:3306/comosus-ravitn | ravitn_storage | 1  | 2021-01-19 10:42:16 | 2021-01-19 10:42:16 |
+-----------------------------------------------------------------+---------------------------------------+-------------------+-------------------+--------------------------------------------+----------------+----+---------------------+---------------------+
3 rows in set

```


从上面可以看出， TM开启一个全局事务xid，每个服务RM对DB的操作，实际为一个事物分支；通过行记录锁控制数据的并发访问。

再看看一下undo日志


订单undo日志

```sql
mysql> select * from undo_log;
+----+-------------------+---------------------------------------+---------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
| id | branch_id         | xid                                   | context             | rollback_info                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | log_status | log_created         | log_modified        | ext  |
+----+-------------------+---------------------------------------+---------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
|  3 | 94746347146166273 | 10.242.214.156:8091:94746346839982080 | serializer=fastjson | {"@type":"io.seata.rm.datasource.undo.BranchUndoLog","branchId":94746347146166273,"sqlUndoLogs":[{"afterImage":{"rows":[{"fields":[{"keyType":"PRIMARY_KEY","name":"id","type":4,"value":5},{"keyType":"NULL","name":"user_id","type":12,"value":"CU00000001"},{"keyType":"NULL","name":"commodity_code","type":12,"value":"CGD0001"},{"keyType":"NULL","name":"count","type":4,"value":1},{"keyType":"NULL","name":"money","type":3,"value":16.80000},{"keyType":"NULL","name":"version","type":-5,"value":1L},{"keyType":"NULL","name":"update_time","type":93,"value":"2021-01-19 10:48:09"},{"keyType":"NULL","name":"create_time","type":93,"value":"2021-01-19 10:48:09"},{"keyType":"NULL","name":"is_delete","type":-7,"value":false},{"keyType":"NULL","name":"remark","type":12}]}],"tableName":"seate_order"},"beforeImage":{"@type":"io.seata.rm.datasource.sql.struct.TableRecords$EmptyTableRecords","rows":[],"tableName":"seate_order"},"sqlType":"INSERT","tableName":"seate_order"}],"xid":"10.242.214.156:8091:94746346839982080"} |          0 | 2021-01-19 10:48:09 | 2021-01-19 10:48:09 | NULL |
+----+-------------------+---------------------------------------+---------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
1 row in set
```

具体信息如下：

context:serializer=fastjson  
rollback_info:
```json
{
"@type":"io.seata.rm.datasource.undo.BranchUndoLog",
"branchId":94746347146166273,"
sqlUndoLogs":[
{
"afterImage":
{"rows":[{"fields":[{"keyType":"PRIMARY_KEY","name":"id","type":4,"value":5},{"keyType":"NULL","name":"user_id","type":12,"value":"CU00000001"},{"keyType":"NULL","name":"commodity_code","type":12,"value":"CGD0001"},{"keyType":"NULL","name":"count","type":4,"value":1},{"keyType":"NULL","name":"money","type":3,"value":16.80000},{"keyType":"NULL","name":"version","type":-5,"value":1L},{"keyType":"NULL","name":"update_time","type":93,"value":"2021-01-19 10:48:09"},{"keyType":"NULL","name":"create_time","type":93,"value":"2021-01-19 10:48:09"},{"keyType":"NULL","name":"is_delete","type":-7,"value":false},{"keyType":"NULL","name":"remark","type":12}]}],
"tableName":"seate_order"},
"beforeImage":{"@type":"io.seata.rm.datasource.sql.struct.TableRecords$EmptyTableRecords","rows":[],"tableName":"seate_order"},"sqlType":"INSERT","tableName":"seate_order"
}],
"xid":"10.242.214.156:8091:94746346839982080"
}
```


库存undo日志


```sql
mysql> select * from undo_log;
+----+-------------------+---------------------------------------+---------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
| id | branch_id         | xid                                   | context             | rollback_info                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | log_status | log_created         | log_modified        | ext  |
+----+-------------------+---------------------------------------+---------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
|  6 | 94748420306747393 | 10.242.214.156:8091:94748420164141056 | serializer=fastjson | {"@type":"io.seata.rm.datasource.undo.BranchUndoLog","branchId":94748420306747393,"sqlUndoLogs":[{"afterImage":{"rows":[{"fields":[{"keyType":"PRIMARY_KEY","name":"id","type":-5,"value":1L},{"keyType":"NULL","name":"commodity_code","type":12,"value":"CGD0001"},{"keyType":"NULL","name":"count","type":-5,"value":998L},{"keyType":"NULL","name":"version","type":-5,"value":3L}]}],"tableName":"ravitn_storage"},"beforeImage":{"rows":[{"fields":[{"keyType":"PRIMARY_KEY","name":"id","type":-5,"value":1L},{"keyType":"NULL","name":"commodity_code","type":12,"value":"CGD0001"},{"keyType":"NULL","name":"count","type":-5,"value":999L},{"keyType":"NULL","name":"version","type":-5,"value":2L}]}],"tableName":"ravitn_storage"},"sqlType":"UPDATE","tableName":"ravitn_storage"}],"xid":"10.242.214.156:8091:94748420164141056"} |          0 | 2021-01-19 10:56:24 | 2021-01-19 10:56:24 | NULL |
+----+-------------------+---------------------------------------+---------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+------------+---------------------+---------------------+------+
1 row in set

mysql> 
```

具体信息如下：
context: serializer=fastjson   
rollback_info:
```json
{"@type":"io.seata.rm.datasource.undo.BranchUndoLog",
"branchId":94748420306747393,
"sqlUndoLogs":[
{"afterImage":
{"rows":[
{"fields":
[{"keyType":"PRIMARY_KEY","name":"id","type":-5,"value":1L},{"keyType":"NULL","name":"commodity_code","type":12,"value":"CGD0001"},{"keyType":"NULL","name":"count","type":-5,"value":998L},{"keyType":"NULL","name":"version","type":-5,"value":3L}]}],
"tableName":"ravitn_storage"}
,"beforeImage":
{"rows":[
{"fields":[
{"keyType":"PRIMARY_KEY","name":"id","type":-5,"value":1L},{"keyType":"NULL","name":"commodity_code","type":12,"value":"CGD0001"},{"keyType":"NULL","name":"count","type":-5,"value":999L},{"keyType":"NULL","name":"version","type":-5,"value":2L}]}],
"tableName":"ravitn_storage"}
,"sqlType":"UPDATE",
"tableName":"ravitn_storage"}],
"xid":"10.242.214.156:8091:94748420164141056"}
```
从上，可以看出每个undo日志会记录包括全局事务id，分支id，日志类型，sql undo日志；sql undo日志由数据表行记录的操作前镜像image（表信息，行记录，行字段类型及数据，操作类型[插入、更新、删除]）和操作后的镜像组成，如果事务回滚，则恢复操作前的镜像。
# 总结
TC事务中心管理所有的TM和RM。在服务（RM、TM）启动时，将会注册服务（RM、TM）到TC事务中心。发起的全局业务方作为TM， 具体业务为事务分支RM。开启一个全局事务会在
TC注册中心生成的全局事务数据（global_table），业务方注册事物分支（branch_table），使用行锁，锁住相关的事务表行记录。并在个业务方，记录业务的undo log。如果分支事务本地都执行提交成功，则删除
全局事务、事务分支，锁和undo 日志。如果有一个分支事务本地事务执行失败，则根据跟业务方的undo日志，恢复数据，并删除全局事务、事务分支，锁和undo 日志。SEATA的一个全局事务，如果处理时间长的话，
在全局事务执行的过程中，会存在中间业务数据。SEATA的AT事务模式，使用的补偿操作，完成整个事务数据的回滚。TM开启一个全局事务xid，每个服务RM对DB的操作，实际为一个事物分支；通过行记录锁控制数据的并发访问。
每个undo日志会记录包括全局事务id，分支id，日志类型，sql undo日志；sql undo日志由数据表行记录的操作前镜像image（表信息，行记录，行字段类型及数据，操作类型[DML ，DDL]）和操作后的镜像组成，如果事务回滚，则恢复操作前的镜像。

# 附
[seata cn](https://seata.io/zh-cn/)   
[seata github](https://github.com/Donaldhan/seata)   
[seata-samples](https://github.com/Donaldhan/seata-samples)   
[分布式事务 Seata Saga 模式首秀以及三种模式详解 | Meetup#3 回顾分布式事务 Seata Saga 模式首秀以及三种模式详解 | Meetup#3 回顾](https://juejin.cn/post/6844903913691283469)   
[分布式事务：Saga模式](https://www.jianshu.com/p/e4b662407c66)    

## usage
[SpringBoot+Dubbo+MybatisPlus整合Seata分布式事务](https://seata.io/zh-cn/blog/springboot-dubbo-mybatisplus-seata.html)   
[SEATA 快速开始](https://seata.io/zh-cn/docs/user/quickstart.html)  
[Seata新手部署指南(1.4.0版本)](https://seata.io/zh-cn/docs/ops/deploy-guide-beginner.html) 
