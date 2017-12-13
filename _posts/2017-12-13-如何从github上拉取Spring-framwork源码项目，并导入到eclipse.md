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

[groovy]()
[git cn][git-cn-url]

[groovy-offical]:
[git-cn-url]: https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000 "git-cn-url"


查看当前分支
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

注意不要上传 *.classpath* 和 *.project* 文件及 *.settings* 文件夹 ，应为这是eclipse产生的项目本地文件，因为每个人用过开发工具可能有所不同
