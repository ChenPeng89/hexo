---
title: Solr学习笔记：三、使用Solr管理员界面
date: 2016-06-24 10:14:30
tags: Solr
---
本节主要介绍的是Solr的管理员界面（Admin UI）使用方法。
## 简介
Solr提供了一个Web界面，让Solr管理员和程序员更直观的看到Solr的配置，运行查询和分析来微调Solr配置，访问在线文档等等。
![](http://i.imgur.com/E1IjbG0.png)

## 帮助信息
在页面最下面就可以找到帮助信息栏。

![](http://i.imgur.com/jVS9Seh.png)

它包括如下链接：
- Documentation ： 定向到Apache Solr的文档地址。https://lucene.apache.org/solr/。
- Issue Tracker ： 定向到Apache Solr项目的JIRA上。https://issues.apache.org/jira/browse/SOLR。
- IRC Channel ： 定向到描述怎么加入Solr IRC聊天室的Apache Wiki上面。https://wiki.apache.org/solr/IRCChannels。
- Community forum ： 定位到描述加入Solr用户协会的email地址列表的Wiki上。
- Solr Query Syntax : 定位到Solr的 “查询语法和解析” 文档章节。

## 日志
Logging界面显示了最近的Solr节点日志。
![](http://i.imgur.com/nzYtDIm.png)

在Logging下面有一个Level链接，点击后可以看到classpath 和classname的路径。黄色高亮的行说明class是可以打log的。点击高亮的行，会出现让你选择日志等级的菜单。粗体字符表示这个class不会被root日志等级的修改所影响。

![](http://i.imgur.com/2pr27kv.png)	


## Cloud 界面
Cloud界面显示了不同样式的Cloud的结构图。包含"Tree", "Graph", "Graph (Radial)" 和 "Dump"。

![](http://i.imgur.com/rzjXfBU.png)

上面的Graph显示了包含两个分片和两个备份的“gettingstarted”集群，还包含了一个分片的films集群。

![](http://i.imgur.com/qRTXptp.png)

Graph (Radial) 以另一种不同的试图展示了上面的集群结构。

![](http://i.imgur.com/7JRF8YO.png)

Tree 展示的是ZooKeeper上的数据结构目录，包含了 live_nodes 和 overseer 的状态，还有集群特定的信息，比如 state.json ， 当前分片的leader，配置文件等。

最后的 Dump 选项，可以返回包含所有节点的JSON文档。可以用来导出Solr保存在ZooKeeper上的所有快照数据同时也能够debug SolrCloud 。

## Collections / Core Admin
![](http://i.imgur.com/WbvOpPF.png)
Collections 界面提供了基本的操作Collections的功能。
如果使用的是single模式，则显示的是Core Admin。
主页面显示了一个在你集群中的集合列表。点击集合的姓名会显示一些基本的元数据，包括 集合的定义，当前分片和备份，增加和删除单个备份。

## Java Properties
![](http://i.imgur.com/RkRpv35.png)
Java Properties 界面可以看到Solr JVM的所有参数，比如 classpath ， 文件编码，JVM内存，操作系统等。

## Thread Dump
![](http://i.imgur.com/dFV9Gll.png)
Thread Dump 界面能让你检查当前活动的线程。每个线程都被列出来并可以查看堆栈状态。左边的图标显示了堆栈状态，例如，每个绿色的对勾图标显示线程在"RUNNABLE"状态。在线程名称右侧，展开向下的箭头可以看到线程的堆栈。

线程状态

- NEW -- 线程还没开始运行。
- RUNNABLE -- 线程在JVM中运行。
- BLOCKED -- 线程等待锁而阻塞。
- WAITING -- 线程无限期等待另一个线程的一个特定操作。
- TIMED_WAITING -- 线程在制定的时间等待另一个线程的一个特定操作。
- TERMINATED -- 线程结束。

## 针对集合的工具
在左边的菜单中，有一个下拉框，可以选择Core或者Collection，当使用Cloud模式时，可以选择Collection，选择后，会有一个二级菜单出现在下拉框下方。
![](http://i.imgur.com/JTFKXlA.png)

主要包含以下几个功能：

- Analysis 
通过Analysis页面可以根据字段检查数据的处理，字段类型和动态字段配置。可以分析内容将在索引期间或在查询处理期间如何进行处理，并分别或同时查看结果。理想情况下，你会希望内容会被持续处理，那么这个页面可以让你验证字段类型或者字段分析链中的设置。
![](http://i.imgur.com/jiAaW0A.png)





