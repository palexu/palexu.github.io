---
title: JStorm 源码分析 - tuple 在整个拓扑中的流转过程
date: 2019-01-16 23:42
---

# JStorm 源码分析 - tuple 在整个拓扑中的流转过程

介绍 JStorm 时, 如果上来就直接贴大段大段的源代码, 那我觉得还不如不看. 这种行为就像是让盲人去摸🐘, 难以窥见整个系统的全貌.

所以这里先丢一张图, 毕竟图总比文字更有表现力.

![](https://ws2.sinaimg.cn/large/006tNc79ly1fz8tfgh75uj30ta0iudg5.jpg)

这个图上画的是一个 tuple 在各个组件之间的流转过程, 主要涉及到的有三个队列:

- deserializeQueues 待反序列化队列
- innerTaskTransfer 待task消费队列
- serializeQueue 待序列化队列

线条代表了 tuple 被处理后流动的方向.

我们的 tuple 被 Spout/Bolt 所 emit 之后, 就一辈子都在这三个队列当中兜兜转转^_^.

当我们的 task (bolt/spout) 发送一个 tuple 时, 自然是希望这个 tuple 被下一个 task (bolt/spout) 所消费, 但是下一个 task 可能被分配在同一个 worker 当中, 也可能是在另一台机器的另一个 worker 当中. 

如果是另一个 worker , 那么 Collector 会将 worker 放入到 serializeQueue (待序列化队列), 有专门的线程会消费这个队列, 然后将消息发送到另一个 worker . 如果是同一个 worker , 那就没这么费事了, 直接丢到 innerTaskTransfer (待task消费队列) 就完事儿了! 那么 deserializeQueues (待反序列化队列) 又是做什么的呢? ^_^ 另一个接收到消息的 worker, 当然是需要反序列化收到的消息, 然后让 task 去消费啦, 那么这些刚接收到的消息, 都存在这个队列里面.



下面这张图画出了tuple 的反序列化/序列化/消费/网络传输所涉及的组件: TaskReceiver, TaskTransfer, Task, NettyClient, NettyServer

![image-20190116230849625](/Users/xj/code/palexu.github.io/_posts/assets/image-20190116230849625-7651329.png)



TODO 待更新





![jstorm](https://ws1.sinaimg.cn/large/006tNc79ly1fz8snni3xmj30u015itxg.jpg)

