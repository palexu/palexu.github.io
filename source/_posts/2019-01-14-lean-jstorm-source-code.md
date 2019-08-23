---
title: JStorm 源码分析 - 目录
date: 2019-01-14 18:42
---

## 简介

本次 JStorm 源码分析文章, 主要是为在公司的内部学习分享会准备的. 
生产环境使用的是 JStorm 2.2.1, 本次介绍的功能也以此为准 ( 本文不会涉及到一些高级特性, 主要还是围绕 Storm 的基本功能展开). 希望阅读本文的童鞋, 最好对 Storm/JStorm 的使用有一定的了解, 知道 Spout、Bolt 的一些基本的工作原理. 

如果不太了解, 本文最下方也提供了一些学习资料, 可以先花上半天时间学习一下.

本文主要会介绍以下这些内容:
1. [tuple 在整个拓扑中的流动过程](lean-jstorm-source-code-01)
    1. spout / bolt 如何接受并处理消息, 然后向后发送消息?
    2. 在发给内部的 task 和外部的 task 时, 发送方式有什么区别?
2. JStorm 如何对这个步骤进行抽象,形成不同的组件(TaskReceiver,TaskTransfer,Task,Executor)
3. 拓扑如何将这些组件启动?

这篇文章不会对着代码一行行介绍每一句代码干了啥, 而是希望从功能的角度出发, 去描述 JStrom 整个系统的架构.


## 涉及到以下这些方面:

启动、运行流程:
- 上传拓扑: jstorm Nimbus
- 启动 supervisor : SyncSupervisorEvent 实际启动worker的地方
- 启动 worker : storm Worker 的启动与运行
- 启动 task :storm task的启动与运行
- 在 JStrom 中 executor 与 task 之间的关系

关键组件:
- TaskTransfer/TaskReceiver
- [AsyncLoopThread 以及相关的类](lean-jstorm-AsyncLoopThread)
- SystemBolt
- TopologyContext
- BoltExecutors

依赖的库与框架
- Metric 用于性能统计
- [LMAX Disruptor 一个高性能队列](lean-jstorm-source-queue)
- Netty 网络通信
- thrift RPC通信

## 参考资料

### Storm 相关博客和书籍
1. [官方文档 http://storm.apache.org](http://storm.apache.org/releases/1.0.6/javadocs/index.html)
2. [InfoQ:Storm是如何成为Apache顶级项目的](http://www.infoq.com/cn/news/2014/10/storm-apache-top-level-project)
3. [InfoQ:Spotify如何对Apache Storm进行规模扩展](http://www.infoq.com/cn/articles/how-spotify-scales-apache-storm)
4. Storm分布式实时计算模式 
2. Getting Started with Storm 
3. [《Storm源码分析》](http://book.douban.com/subject/26115707/): 作为工具书在手边备一本，快速看一遍在文档之外多了解一些实现 

### JStorm 相关博客或网页 
1. [JStorm 官方文档 https://github.com/alibaba/jstorm/wiki](https://github.com/alibaba/jstorm/wiki)
2. [JStorm源码阅读汇总](https://blog.csdn.net/tjq980303/article/details/81806609) csdn上的博客,可以作为参考
2. [JStorm 源码解析](https://hk.saowen.com/source/site/www_zhenchao_org) 这个博客写得比较详细,不过它还是更多的关注于细节方面,没有对整体架构做一个总体的介绍.



