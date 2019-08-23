---
title: JStorm 源码分析 - Nimbus
date: 2019-01-19
---
拓扑会通过 Nimbus 分发给 supervisor, 那么 Nimbus 内部是怎么操作的?
这里以本地模式为例, 对拓扑的提交过程做一个分析.

参考资料:
[理解storm拓扑并行度](https://meta.tn/a/4dd64fdd491b30afba2893e0a570ca8eb8fc1bfb21ccb8d8f8f44cbdc2bb6be4)

## 疑问:

1. 在zk上建立task信息,这些信息是用来做什么的?
2. notifyTopologyActionListener 做了什么?

## 启动

TODO ...

## 命令的入口

所有命令的入口, 都是由 ServiceHandler 实现的,  com.alibaba.jstorm.daemon.nimbus.ServiceHandler#submitTopologyWithOpts.

1. 配置校验
2. 判断拓扑是否已存在/重名/重复提交
3. 标准化conifg
4. 标准化topology (finalize component's task parallism)
5. 校验topology结构
    1. 校验 bolt/spout 的id 和 name
    2. 校验 bolt 的输入是否为空
6. 拷贝代码二进制文件到集群
7. 在zk上建立task信息 (supervisor会持续监控保存在zk的任务)
    ![F7A5DEAE-BED4-4A4E-8EB9-1E4DD9726223](https://ws2.sinaimg.cn/large/006tNc79ly1fzartnvu25j31em04i0to.jpg)
    1. 为bolt/spout等创建对应的 TaskInfo (多并行度的bolt/spout会创建出多个TaskInfo)
        com.alibaba.jstorm.cluster.Common#mkTaskMaker
    2. 注意, jstorm 的 setNumTasks 其实是无效的, 只有 paralleism 并行度会起作用.(见 jstorm作者之一cody的回答: https://stackoverflow.com/a/34316700/6275014 )

8. StartTopologyEvent.pushEvent, 
    1. 然后会异步地去执行 com.alibaba.jstorm.daemon.nimbus.TopologyAssign#mkAssignment
        1. com.alibaba.jstorm.schedule.default_assign.TaskScheduler#assign:
    将task分配给worker, 在这里做了一些定制化, 如有的task要求分配在不同的worker上等.
        2. 创建好 Assign 后, 会发布到 zk 上.
9. notifyTopologyActionListener

