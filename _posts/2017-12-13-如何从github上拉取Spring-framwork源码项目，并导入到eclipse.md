---
layout: page
title: 如何从github上拉取Spring-framwork源码项目，导入到eclipse中
subtitle: spring源码构建
date: 2017-12-13 09:17:19
author: donaldhan
catalog: true
category: spring
categories:
    - spring
tags:
    - gradle
---
# 引言
* [下载源码](#下载源码)
* [配置gradle环境](#配置gradle环境)
* [导入到eclipse中](#导入到eclipse中)
* [注意事项](#注意事项)
在工作两年的时间内，一直再用spring框架做开发，在空闲时间搞过hadoop，hbase，学习了netty和mina，java nio和juc的源码，其他以下项目redis，activemq等都是通过反编译
去看源码，反编译会丢失很多java doc注释，可读性不高。在之前也反编译过spring框架的源码，也是知道个大概，上次有人问我spring的几个特点，我居然一时打不上来，
着实可恨，所以下定决心研究一下spring框架的源码。

## 下载源码
首先，我们要从github上，fork *spring-framework* 项目到自己的github仓库中，在fork之前你要有一个github账号。然后使用git
将fork的spring-framework项目到自己本地。关闭git的使用，可以参见git使用教程[git cn][git-cn-url]

[git-cn-url]: https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000 "git-cn-url"

## 配置gradle环境
已经将 *spring-framework* 源码拉倒本地，接下来将会 *spring-framework* 导入到eclipse中，在导入到eclipse中之前，我们需要对[gradle][gradle-offical]和
[groovy][groovy-offical]有一个了解。简单说gradle是一个项目管理构建工具，类似于Maven和Ant,对开发者十分友好，google的Android studio，默认就使用gradle管理
项，gradle所使用语言是groovy，是一个基于java的JVM语言。gradle的[中文][Gradle-cn]、[英文][Gradle-User-Guide]使用文档和groovy的[中文使用文档][Groovy-cn]  

[gradle-offical]: https://gradle.org/ "gradle 官方网站"
[Gradle-cn]:http://www.yiibai.com/gradle/  "gradle 中文使用文档"
[Gradle-User-Guide]: http://wiki.jikexueyuan.com/project/GradleUserGuide-Wiki/ "gradle 官方使用文档"

[groovy-offical]:http://www.groovy-lang.org/ "groovy 官方网站"
[Groovy-cn]:https://www.w3cschool.cn/groovy "groovy 中文教程"

在具备使用gradle基础之后，我将要装一个gradle eclipse 插件[buildship][buildship-url]

[[buildship-url](https://github.com/eclipse/buildship/blob/master/docs/user/Installation.md)  "buildship"

如果在线安装失败的话，可以在eclipse的 *Marketplace* 软件库中安装指定的buildship插件即可。 安装完插件到Windows-peference中配置gradle的安装目录和gradle用户目录 *D:.gradle* 文件夹，这个和maven的 *.m2* 文件夹 的作用相同。gradle项目的jar包一般在用户目录的 *caches* 下，我的是在 *D:.gradle\caches\modules-2\files-2.1* 目录下。
在配置gradle和groove的时候，如果找不到对应的命令，看看是不是gradle和groove的用户变量和系统变量，及PATH声明错误。
下面是我的gradle和groovy 版本信息：

```
donald@donaldHP MINGW64 /f/github.io/Donaldhan.github.io (master)
$ gradle -v

------------------------------------------------------------
Gradle 4.4
------------------------------------------------------------

Build time:   2016-09-19 10:53:53 UTC
Revision:     13f38ba699afd86d7cdc4ed8fd7dd3960c0b1f97

Groovy:       2.4.7
Ant:          Apache Ant(TM) version 1.9.6 compiled on June 29 2015
JVM:          1.8.0_131 (Oracle Corporation 25.131-b11)
OS:           Windows 10 10.0 amd64


donald@donaldHP MINGW64 /f/github.io/Donaldhan.github.io (master)
$ groovy -v
Groovy Version: 2.4.13 JVM: 1.8.0_131 Vendor: Oracle Corporation OS: Windows 10

donald@donaldHP MINGW64 /f/github.io/Donaldhan.github.io (master)
```
## 导入到eclipse中
现在gradle插件安装完及环境配置结束之后，切换 *spring-framework*  master分支到我们需要的分支，我的是4.3.x。


首先查看当前分支，在master分支上
```
donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git branch
* master
```
查看所有分支
```
donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git branch -a
* master
  remotes/origin/3.0.x
  remotes/origin/3.1.x
  remotes/origin/3.2.x
  remotes/origin/4.0.x
  remotes/origin/4.1.x
  remotes/origin/4.2.x
  remotes/origin/4.3.x
  remotes/origin/HEAD -> origin/master
  remotes/origin/beanbuilder
  remotes/origin/conversation
  remotes/origin/gh-pages
  remotes/origin/master
  remotes/origin/update-stomp-reactor-netty
```
切换到 origin/4.3.x分支
```
  donald@donaldHP MINGW64 /f/github/spring-framework (master)
  $ git checkout origin/4.3.x
  Checking out files: 100% (5741/5741), done.
  Note: checking out 'origin/4.3.x'.

  You are in 'detached HEAD' state. You can look around, make experimental
  changes and commit them, and you can discard any commits you make in this
  state without impacting any branches by performing another checkout.

  If you want to create a new branch to retain commits you create, you may
  do so (now or later) by using -b with the checkout command again. Example:

    git checkout -b <new-branch-name>

  HEAD is now at 4fe94dffc0... Fix regression in StompHeaderAccessor

  donald@donaldHP MINGW64 /f/github/spring-framework ((4fe94dffc0...))
  $
```
查看当前所在分支
```
donald@donaldHP MINGW64 /f/github/spring-framework ((4fe94dffc0...))
$ git branch
* (HEAD detached at origin/4.3.x)
  master
```
实际版本为4.3.14
由于gradle需要下载相关jar。所有我们需要添加一些国内的构建仓库，具体如下：

添加maven仓库
```
repositories {
    jcenter()
    mavenCentral()
    maven { url "http://maven.aliyun.com/nexus/content/groups/public/" }
    maven { url 'http://maven.oschina.net/content/groups/public/' }
    maven { url "https://repo.spring.io/plugins-release" }
}
```

然后将 *spring-framework* 项目导入的eclipse中即可，不过这个过程要等一会，这个取决于网络环境，我构建的时候大概用了15-20分钟。
## 注意事项
在我们阅读源码修改的过程中，注意不要上传 *.classpath* 和 *.project* 文件及 *.settings* 文件夹 ，应为这是eclipse产生的项目本地文件，因为每个人用过开发工具可能有所不同
