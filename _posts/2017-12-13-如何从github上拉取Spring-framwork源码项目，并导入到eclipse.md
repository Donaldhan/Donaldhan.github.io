---
layout: page
title: 如何从github上拉取Spring-framwork源码项目，导入到eclipse中
subtitle: spring源码构建
date: 2017-12-13 09:17:19
author: donaldhan
catalog: true
category: spring-framework
categories:
    - spring-framework
tags:
    - gradle
---
# 引言
* [下载源码](#下载源码)
* [配置gradle环境](#配置gradle环境)
* [导入到eclipse中](#导入到eclipse中)
* [提交修改](#提交修改)
* [注意事项](#注意事项)   

在工作两年的时间内，一直再用spring框架做开发，在空闲时间搞过hadoop，hbase，学习了netty和mina，java nio和juc的源码，其他以下项目redis，activemq等都是通过反编译去看源码，反编译会丢失很多java doc注释，可读性不高。在之前也反编译过spring框架的源码，也是知道个大概，上次有人问我spring的几个特点，我居然一时打不上来，着实可恨，所以下定决心研究一下spring框架的源码。

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

[buildship-url]:https://github.com/eclipse/buildship/blob/master/docs/user/Installation.md "buildship"

如果在线安装失败的话，可以在eclipse的 *Marketplace* 软件库中安装指定的buildship插件即可。 安装完插件到Windows-peference中配置gradle的安装目录和gradle用户目录 *D:.gradle* 文件夹，这个和maven的 *.m2* 文件夹 的作用相同。gradle项目的jar包一般在用户目录的 *caches* 下，我的是在 *D:.gradle\caches\modules-2\files-2.1* 目录下。
在配置gradle和groove的时候，如果找不到对应的命令，看看是不是gradle和groove的用户变量和系统变量，及PATH声明错误。
下面是我的gradle(3.1)和groovy(2.4.7) 版本信息：

```
$ gradle -v

------------------------------------------------------------
Gradle 3.1
------------------------------------------------------------

Build time:   2016-09-19 10:53:53 UTC
Revision:     13f38ba699afd86d7cdc4ed8fd7dd3960c0b1f97

Groovy:       2.4.7
Ant:          Apache Ant(TM) version 1.9.6 compiled on June 29 2015
JVM:          1.8.0_131 (Oracle Corporation 25.131-b11)
OS:           Windows 10 10.0 amd64



donald@donaldHP MINGW64 /f/github.io/Donaldhan.github.io (master)
$ groovy -v
Groovy Version: 2.4.7 JVM: 1.8.0_131 Vendor: Oracle Corporation OS: Windows 10

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

针对构建后出现缺失 *spring-cglib-repack* 和 *spring-objenesis-repack* 这两个依赖问题,可以直接执行任务相应的任务重新打包。
搜 *build.gradle* 文件可以找到如下片段：
```
task cglibRepackJar(type: Jar) { repackJar ->
    repackJar.baseName = "spring-cglib-repack"
    repackJar.version = cglibVersion

    doLast() {
        project.ant {
            taskdef name: "jarjar", classname: "com.tonicsystems.jarjar.JarJarTask",
                classpath: configurations.jarjar.asPath
            jarjar(destfile: repackJar.archivePath) {
                configurations.cglib.each { originalJar ->
                    zipfileset(src: originalJar)
                }
                // Repackage net.sf.cglib => org.springframework.cglib
                rule(pattern: "net.sf.cglib.**", result: "org.springframework.cglib.@1")
                // As mentioned above, transform cglib"s internal asm dependencies from
                // org.objectweb.asm => org.springframework.asm. Doing this counts on the
                // the fact that Spring and cglib depend on the same version of asm!
                rule(pattern: "org.objectweb.asm.**", result: "org.springframework.asm.@1")
            }
        }
    }
}

task objenesisRepackJar(type: Jar) { repackJar ->
    repackJar.baseName = "spring-objenesis-repack"
    repackJar.version = objenesisVersion

    doLast() {
        project.ant {
            taskdef name: "jarjar", classname: "com.tonicsystems.jarjar.JarJarTask",
                classpath: configurations.jarjar.asPath
            jarjar(destfile: repackJar.archivePath) {
                configurations.objenesis.each { originalJar ->
                    zipfileset(src: originalJar)
                }
                // Repackage org.objenesis => org.springframework.objenesis
                rule(pattern: "org.objenesis.**", result: "org.springframework.objenesis.@1")
            }
        }
    }
}
```
从代码片段中可以看出，cglibRepackJar任务是将 *org.objectweb.asm.*** 重新打包成 *org.springframework.asm*，
*net.sf.cglib.***重新打包成 *org.springframework.cglib* ，*org.objenesis.*** 重新打包成 *org.springframework.objenesis*
,从spring3.2以后，spring集成了cglib，不在需要这些包，而使将其放在spring框架内部使用，具体说明如下：

for this dynamic subclassing to work, the class that the spring container will subclass cannot be final, and the method to be overridden cannot be final either. also, testing a class that has an abstract method requires you to subclass the class yourself and to supply a stub implementation of the abstract method. finally, objects that have been the target of method injection cannot be serialized. as of spring 3.2 it is no longer necessary to add cglib to your classpath, because cglib classes are repackaged under org.springframework and distributed within the spring-core jar. this is done both for convenience as well as to avoid potential conflicts with other projects that use differing versions of cglib

到spring-framwork源码目录下，执行相应任务：
```
donald@donaldHP MINGW64 /f/github/spring-framework ((35298201cc...))
$ gradle -q cglibRepackJar

donald@donaldHP MINGW64 /f/github/spring-framework ((35298201cc...))
$ gradle -q objenesisRepackJar

donald@donaldHP MINGW64 /f/github/spring-framework ((35298201cc...))
$
```
然后 *Refresh Gradle Project* 即可。

如果出现构建错误 *Gradle sync failed: Cause: org/gradle/listener/ActionBroadcast* ， 则将sonarqube的版本从1.1 升级到2.5即可。
```
plugins {
	id "org.sonarqube" version "1.1"
}
```
改为
```
plugins {
	id "org.sonarqube" version "2.5"
}
```
执行 *Refresh Gradle Project* 即可。

## 提交修改
修改代码提交主要有两种一种是在主干上直接修改，另一种在分支上修改，下面我们分别来看这两种方式。
### 修改远端主干
刚从github下载到本地时，本地的master的分支对应远端的origin/master,
```
donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working tree clean
```
修改相关代码，提交远端origin/master，注意在push之前，要先pull
```
donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git add .

donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git commit -m "perfect jcenter"
[master afbd1f640c] perfect jcenter
 1 file changed, 1 insertion(+), 1 deletion(-)

donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git pull
Already up-to-date.

donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git push origin master
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 290 bytes | 0 bytes/s, done.
Total 3 (delta 2), reused 0 (delta 0)
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To https://github.com/Donaldhan/spring-framework.git
   3b26f7f51f..afbd1f640c  master -> master

donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working tree clean

donald@donaldHP MINGW64 /f/github/spring-framework (master)
$
```
### 修改远端分支
如果要修远端的分支可以是先切换到远端的对应分支，我们以origin/4.3.x为例。  

首先切换到origin/4.3.x远端分支
```
donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working tree clean

donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git branch
* master
donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git checkout origin/4.3.x
```

创建origin/4.3.x的本地分支4.3.x
```
donald@donaldHP MINGW64 /f/github/spring-framework ((7dfdb67674...))
$ git status
HEAD detached at origin/4.3.x
nothing to commit, working tree clean

donald@donaldHP MINGW64 /f/github/spring-framework ((7dfdb67674...))
$ git branch
* (HEAD detached at origin/4.3.x)
  master

donald@donaldHP MINGW64 /f/github/spring-framework ((7dfdb67674...))
$ git checkout -b 4.3.x
Switched to a new branch '4.3.x'
```
修改相关代码，提交到本地分支4.3.x
```
donald@donaldHP MINGW64 /f/github/spring-framework (4.3.x)
$ git add .

donald@donaldHP MINGW64 /f/github/spring-framework (4.3.x)
$ git commit -m "perferct the maven repository url"
[4.3.x 98dfd2591c] perferct the maven repository url
 1 file changed, 3 insertions(+), 3 deletions(-)

donald@donaldHP MINGW64 /f/github/spring-framework (4.3.x)
$ git status
On branch 4.3.x
nothing to commit, working tree clean
```
切换到远端origin/4.3.x，合并本地4.3.x分支到远端origin/4.3.x分支
```
donald@donaldHP MINGW64 /f/github/spring-framework (4.3.x)
$ git checkout origin/4.3.x
Note: checking out 'origin/4.3.x'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at 7dfdb67674... add repository in cn to growth load jar

donald@donaldHP MINGW64 /f/github/spring-framework ((7dfdb67674...))
$ git merge 4.3.x
Updating 7dfdb67674..98dfd2591c
Fast-forward
 build.gradle | 6 +++---
 1 file changed, 3 insertions(+), 3 deletions(-)

donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git pull origin 4.3.x
From https://github.com/Donaldhan/spring-framework
 * branch                  4.3.x      -> FETCH_HEAD
Already up-to-date.

donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git push origin HEAD:4.3.x
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 310 bytes | 0 bytes/s, done.
Total 3 (delta 2), reused 0 (delta 0)
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To https://github.com/Donaldhan/spring-framework.git
   7dfdb67674..98dfd2591c  HEAD -> 4.3.x

```
删除本地4.3.x分支

```
donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git branch
* (HEAD detached at origin/4.3.x)
  4.3.x
  master

donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git branch -d 4.3.x
Deleted branch 4.3.x (was 98dfd2591c).

donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git checkout master
Checking out files: 100% (5741/5741), done.
Previous HEAD position was 98dfd2591c... perferct the maven repository url
Switched to branch 'master'
Your branch is up-to-date with 'origin/master'.

donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working tree clean

donald@donaldHP MINGW64 /f/github/spring-framework (master)
$ git branch
* master

```
到此，分支修改，这种方式我们时通过创建远程分支的本地分支，然后将本地分支合并到远端分支，强烈建议使用分支开发这种方式。   

***

当然还有可以直接在分支上修改、提交,具体如下：
```
donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git status
HEAD detached at origin/4.3.x
nothing to commit, working tree clean
```
修改相关代码，并直接提交到远端分支。
```
donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git status
HEAD detached at origin/4.3.x
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   build.gradle

no changes added to commit (use "git add" and/or "git commit -a")

donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git diff .
diff --git a/build.gradle b/build.gradle
index 9bbc5f2e5e..a7205dafc2 100644
--- a/build.gradle
+++ b/build.gradle
@@ -2,9 +2,9 @@ buildscript {
        repositories {
            jcenter()
                mavenCentral()
-               maven { url "http://maven.aliyun.com/nexus/content/groups/public/"}
-               maven { url 'http://maven.oschina.net/content/groups/public/'}
-               maven { url "https://repo.spring.io/plugins-release"}
+               maven { url "http://maven.aliyun.com/nexus/content/groups/public/" }
+               maven { url 'http://maven.oschina.net/content/groups/public/' }
+               maven { url "https://repo.spring.io/plugins-release" }
        }
        dependencies {
                classpath("org.springframework.build.gradle:propdeps-plugin:0.0.7")

donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git add .


donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git commit -m "formate mavne reposity url"
[detached HEAD 35298201cc] formate mavne reposity url
 1 file changed, 3 insertions(+), 3 deletions(-)

donald@donaldHP MINGW64 /f/github/spring-framework ((35298201cc...))
$ git pull origin 4.3.x
From https://github.com/Donaldhan/spring-framework
 * branch                  4.3.x      -> FETCH_HEAD
Already up-to-date.

donald@donaldHP MINGW64 /f/github/spring-framework ((35298201cc...))
$ git push origin 4.3.x
error: src refspec 4.3.x does not match any.
error: failed to push some refs to 'https://github.com/Donaldhan/spring-framework.git'

donald@donaldHP MINGW64 /f/github/spring-framework ((35298201cc...))
$ git status
HEAD detached from origin/4.3.x
nothing to commit, working tree clean

donald@donaldHP MINGW64 /f/github/spring-framework ((35298201cc...))
$ git push origin HEAD:4.3.x
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 327 bytes | 0 bytes/s, done.
Total 3 (delta 2), reused 0 (delta 0)
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To https://github.com/Donaldhan/spring-framework.git
   98dfd2591c..35298201cc  HEAD -> 4.3.x

donald@donaldHP MINGW64 /f/github/spring-framework ((35298201cc...))
```


## 注意事项
在我们阅读源码修改的过程中，注意不要上传 *.classpath* 和 *.project* 文件及 *.settings* 文件夹 ，应为这是eclipse产生的项目本地文件，因为每个人用过开发工具可能有所不同，在提交之前最好使用如下命令查看修改:  
```
donald@donaldHP MINGW64 /f/github/spring-framework ((98dfd2591c...))
$ git diff .
```
另外需要注意的一点时，在push之前，要先pull相应分支或主干远端代码。

## 附
如果出现构建错误，*Caused by: groovy.lang.GroovyRuntimeException: Could not find matching constructor for: org.gradle.plugins.ide.eclipse.model.ProjectDependency(org.codehaus.groovy.runtime.GStringImpl, java.lang.String)*
这是由于gradle的版本导致到，我是用gradle-3.1构建的，用gradle-4.4构建就会出现如上错误：
到gradle-4.4的安装目录源码下查看ProjectDependency源码：

```java
package org.gradle.plugins.ide.eclipse.model;

import com.google.common.base.Preconditions;
import groovy.util.Node;

/**
 * A classpath entry representing a project dependency.
 */
public class ProjectDependency extends AbstractClasspathEntry {

    public ProjectDependency(Node node) {
        super(node);
        assertPathIsValid();
    }

    /**
     * Create a dependency on another Eclipse project.
     * @param path The path to the Eclipse project, which is the name of the eclipse project preceded by "/".
     */
    public ProjectDependency(String path) {
        super(path);
        assertPathIsValid();
    }

    private void assertPathIsValid() {
        Preconditions.checkArgument(path.startsWith("/"));
    }

    @Override
    public String getKind() {
        return "src";
    }

    @Override
    public String toString() {
        return "ProjectDependency" + super.toString();
    }
}
```
没有对应的构造，只有一个path参数构造，
```java
public ProjectDependency(String path) {
    super(path);
    assertPathIsValid();
}
```
到gradle-3.1的的安装目录源码下查看ProjectDependency源码：

```java
/*
 * Copyright 2016 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.gradle.plugins.ide.eclipse.model;

import com.google.common.base.Preconditions;
import groovy.util.Node;
import org.gradle.util.DeprecationLogger;

/**
 * A classpath entry representing a project dependency.
 */
public class ProjectDependency extends AbstractClasspathEntry {

    private String gradlePath;

    public ProjectDependency(Node node) {
        super(node);
        assertPathIsValid();
    }

    /**
     * Create a dependency on another Eclipse project.
     * @param path The path to the Eclipse project, which is the name of the eclipse project preceded by "/".
     */
    public ProjectDependency(String path) {
        super(path);
        assertPathIsValid();
    }

    /**
     * Create a dependency on another Eclipse project.
     * @deprecated Use {@link #ProjectDependency(String)} instead
     */
    @Deprecated
    public ProjectDependency(String path, String gradlePath) {
        this(path);
        DeprecationLogger.nagUserOfDiscontinuedMethod("ProjectDependency(String path, String gradlePath)", "Please use ProjectDependency(String path) instead.");
        this.gradlePath = gradlePath;
    }

    @Deprecated
    public String getGradlePath() {
        DeprecationLogger.nagUserOfDiscontinuedMethod("ProjectDependency.getGradlePath()");
        return gradlePath;
    }

    @Deprecated
    public void setGradlePath(String gradlePath) {
        DeprecationLogger.nagUserOfDiscontinuedMethod("ProjectDependency.setGradlePath(String)");
        this.gradlePath = gradlePath;
    }

    private void assertPathIsValid() {
        Preconditions.checkArgument(path.startsWith("/"));
    }

    @Override
    public String getKind() {
        return "src";
    }

    @Override
    public String toString() {
        return "ProjectDependency" + super.toString();
    }
}
```
存在相应的构造函数
```java
/**
 * Create a dependency on another Eclipse project.
 * @deprecated Use {@link #ProjectDependency(String)} instead
 */
@Deprecated
public ProjectDependency(String path, String gradlePath) {
    this(path);
    DeprecationLogger.nagUserOfDiscontinuedMethod("ProjectDependency(String path, String gradlePath)", "Please use ProjectDependency(String path) instead.");
    this.gradlePath = gradlePath;
}
```
