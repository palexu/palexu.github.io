---
title: JStorm 源码分析 - Worker 的启动与运行
date: 2019-01-18
---

> 对一个 topology，JStorm 最终会调度成一个或多个 worker，每个 worker 即为一个真正的操作系统执行进程，分布到一个集群的一台或者多台机器上并行执行。
> 而每个 worker 中，又可以有多个 task，分别代表一个执行线程。每个 task 就是上面提到的组件(component)的实现，要么是 spout 要么是 bolt 。

<img src="https://ws4.sinaimg.cn/large/006tNc79ly1fzakkrdce6j30m00e6q2u.jpg" width="300px"/>

如上图所示, worker 是一个独立 JVM 的进程, 它其实是由 Supervisor 通过命令行执行 Worker#main 方法来启动. worker 进程内部, 运行着许多线程, 包括: Task 线程、序列化/反序列化线程等. 其对应的代码为: com.alibaba.jstorm.daemon.worker.Worker

注意: Storm 与 JStorm 的 worker 模型有所不同, JStorm 移除了 executor 的概念, 详见“[JStorm 中 task 与 executor 的关系]()”.

## 启动

worker 有两种被启动的方式, 这个在 supervisor 一节已经提到.
* (本地调试) 调用 mk_worker 方法来启动
* (线上部署) 通过命令行调用 main, 在新的 JVM 中启动 Worker

![](https://ws2.sinaimg.cn/large/006tNc79ly1fzajgfo71ej31di0em0to.jpg)

在 mk_worker 中, 执行了如下操作:

1. 打日志( sb 这个变量名我很喜欢)
2. new Worker ,并设置必要的属性 ( 这里设置的属性真的太多了, 得再开一个章节细讲 ). 总的来说是以下事情:
   - 算出当前worker会启动的task
   - todo
3. 执行 execute
   在 execute 方法内, worker 完成了初始化和启动 task 的工作, 主要做了以下事情:
   1. worker 之间互连并启动一个线程监控变化，如果worker任务变更会与启停 worker 重连
   2. 监控 topology 是否 active 并将这个状态赋给 storm-active-atom 变量，task 根据这个变量决定是否调用 spout 的 nextTuple
   3. worker 启动线程来执行具体的 tasks

### 让我们详细看一下 execute 方法内部

#### 在第一步中: 创建并启动分发控制信息的线程

```java
// create recv connection, reduce the count of netty client reconnect
AsyncLoopThread controlRvthread = startDispatchThread();
threads.add(controlRvthread);
```

1. 使用Disruptor的队列建立了一个属于worker的 recvControlQueue . 不过到目前为止,没有足够的信息告诉我们这个队列是在worker的生命周期中起到什么作用的.
2. 对recvControlQueue 的性能指标进行跟踪( 使用dropwizard/metrics,详见jstorm metric )
3. 使用netty创建了一个server, 对指定端口进行监听. NettyServer 会将收到的消息放入 recvControlQueue 和 deserializeQueues ( 忘了 deserializeQueues 是啥? [看下这篇“tuple 在 整个拓扑中的流转过程”](jstorm-source-code-01) )中
4. 新建并启动了一个 VirtulPortCtrlDispatch , 这又是个啥东西呢. 似乎只是用来在启动的过程中, 向各个 task 发送控制信息.
   上面所述的 recvControlQueue 中的元素会按照taskId 被分发到 VirtulPortCtrlDispatch 持有的 controlQueues 存放的task的 ctrlQueue 中.
   也就是 recvControlQueue 中装的东东会被放到 VirtulPortCtrlDispatch.controlQueues 的 ctrlQueue中.那么问题来了,  recvControlQueue 中的元素又是哪来的呢? (这个控制信息猜测是后续的)

<img src="https://ws4.sinaimg.cn/large/006tNc79ly1fzajcpu50sj30gu0c6jrd.jpg" width="300px"/>



#### 在第二步中: 创建并启动"维护task之间连接"的线程

创建了一个 RefreshConnections .

<img src="https://ws2.sinaimg.cn/large/006tNc79ly1fzajh0muefj30a207i0sl.jpg" width="200px"/>

为了配置这个类，需要从上下文中取出本次拓扑的拓扑结构,以及当前 worker 将会启动的 task 的taskid 列表  然后通过这两个信息算出当前 worker 会和那些 task 进行连接,不区分是本 worker 和其他 worker
主要工作:
该类会在 worker 的运行过程中，定时去做下面这些事：
维护 task 之间的通信（目前使用的是 netty ）：包括新建连接，移除旧的连接。（因为 worker 是可能挂掉掉并重启的，此时 task 之间的链接就需要重新维护）
其中会去计算出 task 的下游 task 在哪个 worker 中. 然后启动 NettyClient 去 connect 下游 worke r 的 port.

#### 在第三步中：创建并启动维护zk状态的线程

#### 在第四步中：创建并启动发送控制信息的线程（DrainerCtrlRunable）这名字取得挺有意思的, 排水控制？

![](https://ws3.sinaimg.cn/large/006tNc79ly1fzajhtzh5nj30qe05mmx9.jpg)

主要工作：

```java
//从transferCtrlQueue里消费
super(workerData.getTransferCtrlQueue (), idStr) 
```

这家伙会发送控制信息
```java
TaskMessage message = new TaskMessage(TaskMessage.CONTROL_MESSAGE, targetTask, tupleMessage);
conn.sendDirect(message);
```

#### 第五步中：创建并启动心跳维护线程
？？维护心跳，谁的心跳？怎么维护？为什么要维护？

#### 第六步中：为worker创建一个metric统计的类 
（如果后续datacore想要维护好性能的话，可以考虑加入这个东西来统计各个因子的数据。但是这个对代码改动还挺多的，因此还是看情况吧）

#### 第七步中: 创建并启动task线程.   (终于创建了task)

<img src="https://ws2.sinaimg.cn/large/006tNc79ly1fzaji64qh9j30lu0a674e.jpg" width="300px"/>

这里已经创建并启动了 task 线程. 

#### 第八步中：创建并启动n个序列化，反序列化线程。

![0A1663BC-76E3-4B6E-82A9-010A64F9E3B4](https://ws1.sinaimg.cn/large/006tNc79ly1fzal2b7hi0j30m806zjrq.jpg) 
这个n还挺讲究的，有专门的算法，代码先贴在下面，可以考虑下为啥要这么计算

#### 最后一步: 初始化步骤结束

经过哐哐哐一通创建，现在有了好多个不同的线程，这些线程被塞到workData里，然后一起返回给了 mk_worker 方法的调用者。
sd.join是说,当worker所创建的所有线程都运行结束后,worker线程才结束.

![](https://ws3.sinaimg.cn/large/006tNc79ly1fzajjvpru9j30of01r743.jpg)


那么到这里,worker已经完成了所有启动需要的操作.

## 运行时
todo