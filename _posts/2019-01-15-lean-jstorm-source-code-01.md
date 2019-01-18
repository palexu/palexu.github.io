---
title: JStorm 源码分析 - tuple 在整个拓扑中的流转过程
date: 2019-01-15 23:42
---


我们先引入一些基本的概念, 对 JStorm 源码中使用的一些组件有一个基础的认识, 再在这个基础上加入源码的分析, 这样才能更好地学习源码. ^_^

下图展示了一个简单的拓扑结构.

![](https://ws2.sinaimg.cn/large/006tNc79ly1fz9mrm6en3j30oy0baaa1.jpg)



上图中共有 3 个 task, 其中 1 个 spout, 2 个 bolt, 它们分别被分配在 Worker 1、Worker 2 中. Spout 与 Bolt A 处在同一个 Worker 中,  属于同一个 JVM. 而 Bolt B 在另一个 Worker 中, 属于另一台机器的另一个 JVM 中.

当我们的 spout 发送一个 tuple 后, JStorm 会将这个 tuple 发送给下游的 task 消费( Bolt A 与 Bolt B ).  但下游的 bolt 处于不同的 Worker 中. 那么下游的 task 是如何拿到这个 tuple 的, 这两种情况下 JStorm 是怎么处理的呢?

## 基本的数据结构

tuple 在 JStorm 中的流动, 主要涉及到的有三个队列:

- deserializeQueues 待反序列化队列
- innerTaskTransfer 待task消费队列
- serializeQueue 待序列化队列

这三个队列是 JStorm 中最最核心的队列, 所有 spuot/bolt 发射的 tuple, 以及所有 bolt 消费的 tple 都会存放在这三个队列当中. 下图画出了 tuple 在这三个队列上的流动关系:

![](https://ws3.sinaimg.cn/large/006tNc79ly1fz9msa2z9vj311g0ikwev.jpg)

如果是发送到另一个 worker , 那么 spout 发射的 tuple 会被放入到 serializeQueue (待序列化队列).  后续这些 tuple 会被序列化后通过网络传输, 发送到另一个 worker 的 deserializeQueues (待反序列化队列) 中.  worker 在启动时, 会创建专门的反序列化线程. 这些序列化进程会不断地去消费 deserializeQueues, 将其中的消息解析为 bolt 可以识别的 tuple, 并丢到 innerTaskTransfer (待task消费队列).

如果是发送到同一个 worker, 就不需要上述这一系列操作, 直接丢到 innerTaskTransfer (待task消费队列) 就完事儿了! 

## 更细致些!

如果想要看的更详细一点, 请看下面这张图. 下图画出了 tuple 的反序列化/序列化/消费/网络传输所涉及的组件: TaskReceiver, TaskTransfer, Task, NettyClient/NettyServer

- TaskReceiver: 该组件内部有多个线程, 会持续消费 deserializeQueues, 对消息进行反序列化, 然后放入 innerTaskTransfer
- Task: 这个组件会持续取出 innerTaskTransfer 中的 tuple, 并交给内部持有的 Spout、Bolt A、 Bolt B 的实例化对象去消费
- TaskTransfer: 这个组件负责将 tuple 分发给不同的队列. 实际上, 是 Collector 内部持有这个对象, 当我们调用 Collector#emit, 内部会去调用 TaskTransfer#transfer 来分发 tuple.
- NettyClient/NettyServer: 负责序列化、反序列化、网络传输等操作.

这几个组件是 JStorm 运行过程中, 涉及到 tuple 发送、接受、序列化等操作的核心组件. 如果想要对 JStorm 的内部逻辑有一个清晰的概念, 就必须了解上面这几个组件.后续的源码分析过程中, 会对这些组件进行详细的介绍.

![](https://ws4.sinaimg.cn/large/006tNc79ly1fz9lgy4955j30za0mmdgd.jpg)

## 总体结构

下图是一张更加详细的高清大图了. 这张图使用黑色带箭头的实线, 画出了 tuple 的流动方向. 并且引入了 Worker、AsyncLoopThread 等新的组件

![jstorm](https://ws1.sinaimg.cn/large/006tNc79ly1fz8snni3xmj30u015itxg.jpg)

 (图片使用 OmniGraffle 绘制, 想要源文件或者高清原图的可以给我发邮件哦)

## 总结

本文介绍了以下这些重要的组件, 让我们回顾一下:

- 队列 deserializeQueues/innerTaskTransfer/serializeQueue
- 序列化 NettyClient
- 反序列化 NettyServer
- tuple路由 TaskTransfer
- 执行业务代码 Task
- 各个组件的容器 Worker

后续 JStorm 会更加详细地对这些组件进行介绍! 下次见^_^