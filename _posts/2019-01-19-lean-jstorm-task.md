---
title: JStorm 源码分析 - Task 的启动与运行
date: 2019-01-19
---
本篇主要介绍task的创建与运行过程

## 文章开头, 先抛出一些疑问:

1. 为什么TaskTransfer、TaskReceiver 要在初始化Task的时候创建, 为什么不在Worker里直接创建好? 这么做有什么好处呢?
2. TaskTransfer/Receiver 与 Task 是1对1的关系么?  如果是,为什么? 如果不是,那设置了怎样的值, 为什么?
3. start up bolt 发给系统 bolt 是做什么用的?
4. 至于在创建拓扑时的并行度等属性, 在提交拓扑时(?确定是这个时候么?)就已经分配好了. 例如 BoltA 并行度为 3, 那么会创建3个Task对象, 每个对象都持有BoltA? 而不是一个Task 持有多个 Executor ,每个Executor使用执行bolt的业务代码 ?
5. 为什么创建executor时,要传递当前task进去,如 baseExecutor = new BoltExecutors(this) . 而不是传递必须的参数进去.这是否是为了强调task和executor之间的某种联系?如果是,那么又是什么联系呢?  在较早版本的源代码里面,是否有反映这个联系呢?

<img src="https://ws1.sinaimg.cn/large/006tNc79ly1fzark3184pj30im09kaa6.jpg" style="zoom:50%"/>

## 创建过程:
和worker的启动方式类似  (JStorm对初始化方法和启动方法使用了相同风格的命名, 减少了理解成本 ^_^)

1. 执行 Task#new, 将必需的参数从workerData中取出
    比较重要的一步: 获取taskObj. 
    是通过Common#get_task_object获取, 这个对象就是在建立拓扑时 new 出的Bolt/Spout对象 (原汁原味,没有执行过prepare)
2. 执行 Task#execute, 进行必要的初始化工作
    1. 发送“start up” bolt 给系统bolt
    2. 创建 TaskTransfer. 作用是当 Bolt 中的业务代码调用 collect#emit 时, collect 内部使用 TaskTransfer 来发送消息给下游 Task. 
    3. 创建 Executor 并在 AsyncLoopThread 中启动 详见 jstrom executor  
        Executor 是真正运行 Bolt/Spout 的地方. 
        1. 调用 Bolt/Spout#prepare 来初始化, Collector 也是在此时初始化并传递给 Bolt 的
        2. 循环调用 Bolt/Spout#execute 来对消息进行处理
        3. 若 Bolt/Spout 需发送消息, 需调用 Collector#emit, 消息会经过 TaskTransfer 投递给下游 Task
    4. 创建 TaskReceiver. 作用是反序列化其他 Worker 中的上游 Task 发来的消息 (当前 Worker 的消息在投递时没有序列化过,自然更不需要反序列化),并交由 Executor 消费.

运行过程:

![](https://ws2.sinaimg.cn/large/006tNc79ly1fzarn29cnbj31ei0cwmz8.jpg)

上文写到:task启动时,创建 Executor 并在 AsyncLoopThread 中启动.
AsyncLoopThread 启动后, 会在 while 循环中, 不断地执行 Executor 对象的 run 方法. 

Executor 对象有 3 个子类, 因为篇幅有限(为了偷懒), 这里仅介绍 BoltExecutors 所作的操作, 其他子类的原理类似, 可以自己研究.

<img src="https://ws3.sinaimg.cn/large/006tNc79ly1fzarne44coj30ly0g4wfj.jpg" style="zoom:50%"/>



首先来查看 BoltExecutors 的继承关系:
1. 由于实现了Runnable 接口, 因此 BoltExecutors 可以被 AsyncLoopThread 执行, 可以知道 BoltExecutors 代码执行的入口是 run 方法.
2. 继承了 EventHandler 接口 (这是LMAX 开发的 Disruptor 框架中提供的一个消息消费接口), 这告诉我们 BoltExecutors 具有从DisruptorQueue 消费消息的功能.

看run方法

![FC74A2FD-5C12-4E09-BCCE-D052C5D3B53B](https://ws3.sinaimg.cn/large/006tNc79ly1fzaroa3w9nj31f80fegp4.jpg)

BoltExecutors 会在 while 循环中不断批量消费 exeQueue(DisruptorQueue) 中的消息, 且调用时传入的 eventHandler 是自身.
exeQueue 中存放的是待消费的消息 ( 上游Task emit的消息最终会放在这个队列中 ).
因此这里的 consumeBatchWhenAvailable 会调用 BoltExecutors#onEvent 方法,如下所示:

代码过长,只截一部分.

![8524F169-452F-4D92-9192-E36E6BA2D95D](https://ws2.sinaimg.cn/large/006tNc79ly1fzaroih6toj31ei098gnp.jpg)

BoltExecutors的onEvent 方法做了以下操作:

1. 当 event 为 Tuple 时, 会调用 IBolt#execute 进行处理.
2. Tuple 的类型可能是single 或 batch, 如果是 batch, 那就 for 循环里调用 IBolt#execute
3. 实际上会做一些更加细致的判断,例如:
    1. tuple 是否支持事务
    2. 是否是 IRichBolt 所 emit 的 tuple
    3. 是否是 system bolt
    4. 等等
4. 实际上还会通过 Metric 做一些性能的记录

不过主要逻辑是1、2两点, 更加细致的操作, 后续有空可能会分享出来^_^.

 