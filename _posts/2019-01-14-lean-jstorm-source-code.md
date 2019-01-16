---
title: JStorm 源码分析(目录)
date: 2019-01-14 18:42
---
# JStorm 源码分析(目录)

## 简介

本次 JStorm 源码分析文章, 主要是为在公司的内部学习分享会准备的. 
生产环境使用的是 JStorm 2.2.1, 本次介绍的功能也以此为准(当然, 本文不会涉及到一些高级特性, 主要还是围绕 Storm 的基本功能展开). 希望阅读本文的童鞋, 最好对 Storm/JStorm 的使用有一定的了解, 知道 Spout、Bolt 的一些基本的工作原理. 如果不太了解, 本文最下方也提供了一些学习资料, 可以先花上半天时间学习一下.

本文主要会介绍以下这些内容:
1. tuple 在整个拓扑中的流动过程
    1. spout / bolt 如何接受并处理消息, 然后向后发送消息?
    2. 在发给内部的 task 和外部的 task 时, 发送方式有什么区别?
2. JStorm 如何对这个步骤进行抽象,形成不同的组件(TaskReceiver,TaskTransfer,Task,Executor)
3. 拓扑如何将这些组件启动?

这篇文章不会对着代码一行行介绍每一句代码干了啥, 而是希望从功能的角度出发, 去描述 JStrom 整个系统的架构.


## 涉及到以下这些方面:

启动、运行流程:
- 上传拓扑: jstorm Nimbus
- 启动supervisor SyncSupervisorEvent 实际启动worker的地方
- 启动worker storm Worker 的启动与运行
- 启动task storm task的启动与运行
- jstrom executor

关键组件:
- TaskTransfer/TaskReceiver
- AsyncLoopThread 以及相关的类
- SystemBolt
- TopologyContext
- BoltExecutors

依赖的库与框架
- Metric 用于性能统计
- LMAX Disruptor 一个高性能队列
- Netty 网络通信
- thrift RPC通信

## 参考博客或网页 

### 书籍： 
1. Storm分布式实时计算模式 
2. Getting Started with Storm 
3. [《Storm源码分析》](http://book.douban.com/subject/26115707/): 作为工具书在手边备一本，快速看一遍在文档之外多了解一些实现 

### Storm 相关博客或网页 
1. [InfoQ:Storm是如何成为Apache顶级项目的](http://www.infoq.com/cn/news/2014/10/storm-apache-top-level-project)
2. [InfoQ:Spotify如何对Apache Storm进行规模扩展](http://www.infoq.com/cn/articles/how-spotify-scales-apache-storm)
3. 入门资料极客学院  <http://wiki.jikexueyuan.com/project/storm/> 
4. [Storm中Spout使用注意事项小结 - 大圆那些事 - 博客园](evernote:///view/9880513/s38/f829c119-81db-4dec-a67c-b075ef27ef16/f829c119-81db-4dec-a67c-b075ef27ef16/)
5. [storm spout的速度抑制问题 - sanmutongzi - 博客园](evernote:///view/9880513/s38/86d2f45e-8c31-4618-94dc-5ec8b281c17f/86d2f45e-8c31-4618-94dc-5ec8b281c17f/)
6. [Storm中Spout和Bolt的生命周期 - CSDN博客](evernote:///view/9880513/s38/3c37a490-32aa-41ab-a756-a273d24d5446/3c37a490-32aa-41ab-a756-a273d24d5446/)
7. [storm topology生命周期](evernote:///view/9880513/s38/f2e03b49-0ec4-4042-8a48-77532aaffea5/f2e03b49-0ec4-4042-8a48-77532aaffea5/)
8. 官方文档 <http://storm.apache.org/releases/1.0.6/javadocs/index.html> 
9. docker ：[Docker集群轻松部署Apache Storm-云栖社区](evernote:///view/9880513/s38/ff8c2edd-9f5d-4ba6-8c00-ab7b5e21d292/ff8c2edd-9f5d-4ba6-8c00-ab7b5e21d292/) 

### JStorm 相关博客或网页 
1. [jstorm源码阅读汇总](https://blog.csdn.net/tjq980303/article/details/81806609) csdn上的博客,可以作为参考
2. [JStorm 源碼解析](https://hk.saowen.com/source/site/www_zhenchao_org) 这个博客写得比较详细,不过它还是更多的关注于细节方面,没有对整体架构做一个总体的介绍.



