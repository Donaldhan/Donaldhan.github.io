---
layout: page
title: Super-Diamond配置管理服务器
subtitle: Super-Diamond配置管理服务器
date: 2018-04-20 19：08：36
author: donaldhan
catalog: true
category:  Super-Diamond
categories:
    -  Super-Diamond
tags:
    -  Super-Diamond
---

# 引言

在我们配置服务器属性时候，一般常使用方法，将配置属性放在在一个配置文件中，当应用上线时，需要修改配置文件，这样容易导致手动修改配置文件出错；
自从maven的出现使我们可以，我们可以将不同环境的配置，写到不同的文件中，比如（dev，test ，exp，prod）等环境，在项目上线时，我们只需要根据
Profile属性，打包相应的属性文件，这样避免的手动修改配置引起的认为问题，但无法解决修改配置文件需要重新启动或重新打包的问题。历史的车轮，终是向前推动的。
一些配置管理服务顺应出现，比如Spring的[spring-cloud-config][], 淘宝的淘宝[diamond][],另外还有一种轻量级的配置管理服务器[ Super-Diamond][]。Spring家族的
spring-cloud-config的文档，分支版本管理比较规范，毕竟是专业滴。淘宝的diamond，不知现在淘宝现在，还在不在用，不过，在github上，搜不到淘宝的diamond，但是有个
diamond的分支，不过文档很少，这也是阿里开源产品通病。今天我们来看轻量级的配置管理服务器[Super-Diamond][]，从源码的版权声明来看是苏州科大国创信息技术有限公司的产品。



[spring-cloud-config]:https://github.com/Donaldhan/spring-cloud-config "spring-cloud-config"  
[diamond]:https://github.com/takeseem/diamond "diamond"  
[Super-Diamond]:https://github.com/Donaldhan/super-diamond "Super-Diamond"

# 目录
* [Super-Diamond架构设计](super-diamond架构设计)
    * [Super-Diamond数据库设计](#super-diamond数据库设计)
    * [Super-Diamond服务端](#super-diamond服务端)
    * [Super-Diamond客户端](#super-diamond客户端)
* [总结](#总结)

# Super-Diamond架构设计

![Super-Diamond架构设计](/image/super-diamond/framework.png)

从上图中，我们可以看到Super-Diamond主要包括两部分，一个是配置中心服务端，一个配置中心客户端。在服务中心配置，有一个配置中心后台界面我们可以管理项目和项目配置项，同时有一个
配置监听服务器。当项目配置改变时，配置监听服务器将配置更改信息以Json字符串的形式，推送到客户端。客户端启动时，开启一个定时任务以一定的间隔从服务端拉去配置信息。
由于所有的配置信息，实际上是放在数据库中，下面我们来看Super-Diamond数据库设计。

## Super-Diamond数据库设计
Super-Diamond其实是将配置信息放在数据库表中，客户端拉去时，服务端从数据库中，以Json字符串的形式，发送给客户端。主要表结构项目表，项目配置表，项目模块表，用户表。
这里我们就不给出数据库表设计图了，只简单给不表创建语句。具体如下：

### 项目表

```sql
CREATE TABLE `CONF_PROJECT` (
  `ID` int(11) NOT NULL,
  `PROJ_CODE` varchar(32) DEFAULT NULL,
  `PROJ_NAME` varchar(32) DEFAULT NULL,
  `OWNER_ID` int(11) DEFAULT NULL COMMENT '拥有者id',
  `DEVELOPMENT_VERSION` INT(11) DEFAULT 0  NULL,
  `PRODUCTION_VERSION` INT(11) DEFAULT 0  NULL,
  `TEST_VERSION` INT(11) DEFAULT 0  NULL,
  `DELETE_FLAG` int(1) DEFAULT '0',
  `CREATE_TIME` datetime DEFAULT NULL,
  `UPDATE_TIME` datetime DEFAULT NULL,
  PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 项目配置表

```sql
CREATE TABLE `CONF_PROJECT_CONFIG` (
  `CONFIG_ID` INT(11) NOT NULL,
  `CONFIG_KEY` VARCHAR(64) NOT NULL,
  `CONFIG_VALUE` VARCHAR(256) NOT NULL,
  `CONFIG_DESC` VARCHAR(256) DEFAULT NULL,
  `PROJECT_ID` INT(11) NOT NULL,
  `MODULE_ID` INT(11) NOT NULL,
  `DELETE_FLAG` INT(1) DEFAULT '0',
  `OPT_USER` VARCHAR(32) DEFAULT NULL,
  `OPT_TIME` DATETIME DEFAULT NULL,
  `PRODUCTION_VALUE` VARCHAR(256) NOT NULL,
  `PRODUCTION_USER` VARCHAR(32) DEFAULT NULL,
  `PRODUCTION_TIME` DATETIME DEFAULT NULL,
  `TEST_VALUE` VARCHAR(256) NOT NULL,
  `TEST_USER` VARCHAR(32) DEFAULT NULL,
  `TEST_TIME` DATETIME DEFAULT NULL,
  `BUILD_VALUE` VARCHAR(256) NOT NULL,
  `BUILD_USER` VARCHAR(32) DEFAULT NULL,
  `BUILD_TIME` DATETIME DEFAULT NULL,
  PRIMARY KEY (`CONFIG_ID`)
) ENGINE=INNODB DEFAULT CHARSET=utf8;
```
从上面可以看出，项目环境主要有测试，BUILD，和生产环境。

### 项目模块表

```sql
CREATE TABLE `CONF_PROJECT_MODULE` (
  `MODULE_ID` int(11) NOT NULL,
  `PROJ_ID` int(11) NOT NULL,
  `MODULE_NAME` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`MODULE_ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 配置用户表

```sql
CREATE TABLE `CONF_USER` (
  `ID` int(11) NOT NULL,
  `USER_CODE` varchar(32) DEFAULT NULL,
  `USER_NAME` varchar(32) NOT NULL,
  `PASSWORD` varchar(32) NOT NULL,
  `DELETE_FLAG` int(1) DEFAULT '0',
  `CREATE_TIME` datetime DEFAULT NULL,
  `UPDATE_TIME` datetime DEFAULT NULL,
  PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 项目用户关系表，项目用户角色权限表
```sql

CREATE TABLE `CONF_PROJECT_USER` (
  `PROJ_ID` int(11) NOT NULL,
  `USER_ID` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`PROJ_ID`,`USER_ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `CONF_PROJECT_USER_ROLE` (
  `PROJ_ID` int(11) NOT NULL,
  `USER_ID` int(11) NOT NULL,
  `ROLE_CODE` varchar(32) NOT NULL,
  PRIMARY KEY (`PROJ_ID`,`USER_ID`,`ROLE_CODE`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
从数据库的表设计可以看出，配置管理中心，关键点主要是项目和项目配置（test，build，product)

## Super-Diamond服务端

服务端两中部署方式，一种放在tomcat中，一个是放到Jetty中。我们先来看Tomcat方式：
配置监听服务器主要由Spring的应用上下文来管理，在bean工厂初始化的时候，初始化配置监听服务器。
具体的配置在applicationContext.xml配文件中：

```xml
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate" >
    	<constructor-arg index="0" ref="dataSource" />
    </bean>

    <!-- 配置事务管理器 -->
	<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">   
  		<property name="dataSource" ref="dataSource" />
 	</bean>

	<bean id="transactionTemplate" class="org.springframework.transaction.support.TransactionTemplate">
 		<property name="transactionManager" ref="transactionManager" />
 	</bean>

 	<tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="true" />

	<context:component-scan base-package="com.github.diamond.web"/>

	<bean id="diamondServerHandler" class="com.github.diamond.netty.DiamondServerHandler" />
	<bean class="com.github.diamond.netty.DiamondServer">
		<property name="port" value="${netty.server.port}" />
		<property name="serverHandler" ref="diamondServerHandler" />
	</bean>
```

从上面配置来看，主要配合中心服务端的Dao使用的是JdbcTemplate。



## Super-Diamond客户端

# 总结
