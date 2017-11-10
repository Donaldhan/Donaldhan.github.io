[![Build Status](https://travis-ci.org/pages-themes/architect.svg?branch=master)](https://travis-ci.org/pages-themes/architect) [![Gem Version](https://badge.fury.io/rb/jekyll-theme-architect.svg)](https://badge.fury.io/rb/jekyll-theme-architect)

[![Thumbnail of architect](thumbnail.png)](http://pages-themes.github.io/architect)

目录  
* [简介](#简介)
* [快速启动](#快速启动)
* [从零开始](#从零开始)


# 简介

*此静态 [blog][]用于记录个人的学习笔记及所思所想。博客使用的主题是[jekyll-theme-architect][]，如果你不想关心博客是如何搭建的，你可以直接fork我的blog，
修改相应的配置文件即可，具体see[快速开始](#quick-start)。*

[blog]: https://donaldhan.github.io/ "Donald Blog"
[jekyll-theme-architect]: http://pages-themes.github.io/architect "jekyll-theme-architect"
## 快速启动

基于当前仓库，创建博客 :
1. 首先fork仓库到自己的账号，然后修改仓库名，如下
    Donaldhan.github.io -> jamel.github.io
注意jamel是你的github 账号名。
2. 修改配置文件`_config.yml`，这个文件是博客站点的全局配置文件，主要是博客名称，主题和github仓库配置，及分页信息，具体见配置文件；
修改个人信息配置`profile.yml` ，这个文件主要是个人生活城市，工作，email信息配置；当然还有一个重要的文件就是`gitalk.yml`，用于配置
文件评论的应用账户信息具体见 [gitalk][]，注意第一个配置文件在根目录下，后两个在\_data目录下。具体需要修改的配置如下：

`_config.yml`

```
# 具体会在_include/header.html中引用
title: Donaldhan Blog
description: Just do it, deeply...
show_downloads: false
google_analytics:
theme: jekyll-theme-architect
# head github repository
github.repository_url: Donaldhan.github.io.git
repository: Donaldhan/Donaldhan.github.io.git

# 个人账号链接信息，在_include/aside.html中使用
#be care for variable name cant't include .
github:
    username: Donaldhan
iteye:
    username: donald-draper
gitee:
    username: Donaldhans
jianshu:
    username: Donaldhan
Donaldhan:
    email: shaoqinghan@aliyun.com
```

 `profile.yml`

```
# 个人生活城市，工作，email信息配置，在 about/index.html and _include/aside.html有引用
cityInfo: Live in GuangZhou.
workInfo: Serving on the  budget center of CPB.
jobInfo: Now, job on the GuangZhou budget center of chinese people bank.
motto: Everything is possible, just do it, deeply...
```

 `gitalk.yml`

```
#这个配置文件，主要用于文件评论用的插件，这个插件是基于github issue实现的，下面的配置主要是github应用账户信息，自己可以申请，
#这些要修改成自己的使用我的，没有什么作用的，这些变量在 _/layout/page.html引用
clientID: 0939abcd9b2a3251b2d6
clientSecret: 047e01d125fe0e6f362f2318193eec0806a2dcb4
repo: Donaldhan.github.io
owner: Donaldhan
admin: Donaldhan
language: en
mode: false

```
具体配置的作用，可以参见 [gitalk][]；

3. 安装jekyll,由于jekyll是ruby驱动的，在安装之前要安装ruby及Ruby包管理工具,
[Linux安装][linux-jekyll],[Windows安装][windows-jekyll]。
4. 克隆仓库到本地；
5. 进入本地仓库根目录；
6. 执行`bundle exec jekyll serve`启动服务器；
5. 访问[`localhost:4000`](http://localhost:4000)，即可浏览你的博客。  
如果想要写文章，在\_post文件夹下即可，主要文件格式为:
```
YYYY-MM-DD-title.md
```
日期加文章标题。

[gitalk]: https://github.com/gitalk/gitalk
[linux-jekyll]: https://jekyllrb.com/docs/installation/ "Runnig Jekyll on Linux"
[windows-jekyll]: http://www.madhur.co.in/blog/2011/09/01/runningjekyllwindows.html "Running Jekyll on Windows"

## 从零开始

如果你喜欢折腾的话，可以参见我搭建博客的笔记[notes][notes_url] 。
在创建博客之前，最好先看一下相关的概念。比如jekyll是什么？个人理解，jekyll是基于Ruby的将资源文件转换为站点的工具。另外对于jekyll，必须知道的一点是，模板引擎[Liuqid][]，Liuqid对于jekyll，犹如jsp和freemaker对于html一样，但更像freemaker。可以引用网站的数据，同时拥有逻辑判断和循环语法。在创建jekyll博客时，你可能会用两种文件格式，分别为 [yaml][] 和 [markdown][]，`yaml`是一个友好的数据序列序列编程语言，类似json，但比json更简单。markdown是一种易读易写的纯文本标记语言，不想xml和html那样繁琐，每个元素必须有开始和结束name标记，markdown精选了一些特殊的符号做为标记，markdown同时兼容html语言。浏览器解析markdown文件时，实际解析为html语言。如果你想编辑markdown语言，你可以使用[Atom][] , [MarkdownPad][]两个编辑软件，`atom`是github官方markdown编辑软件，有linux和window版本，
MarkdownPad也不错，但只适用windows系统，建议使用atom。  
在了解上面的知识后，创建一个可以发布文章的博客，已经没有什么问题了，当文章发布量增大时，如何快速查看以前的文章了，这就用到的全文搜索了，这个我们可以用基于
liquid的[tipue search[tipue-search]。有了搜索，还有一点，不能发布过文章，就没有事，如果与读者交流才是最重要的，这个我们可以使用基于github issue的评论插件
[gitalk][]。到这里，一个完整的博客该有的功能已基本实现，*Just play， hava fun， enjoy the game...*。

[notes_url]: https://gitee.com/Donaldhans/draft/blob/master/git-page-blog.md
[yaml]: http://www.yaml.org/ "YAML"
[markdown]: https://daringfireball.net/projects/markdown/syntax "Markdown"
[Liuqid]: https://help.shopify.com/themes/liquid/basics "Liuqid"
[tipue-search]: https://github.com/jekylltools/jekyll-tipue-search "Jekyll Tipue Search based liquid"
