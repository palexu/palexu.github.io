---
title: JStorm 源码分析 - 异步循环线程 AsyncLoopThread
date: 2019-01-17
---

> “虽然池化和重用线程相对于简单地为每个任务都创建和销毁线程是一种进步，但是它并不能消除由上下文切换所带来的开销，其将随着线程数量的增加很快变得明显，并且在高负载下愈演愈烈。此外，仅仅由于应用程序的整体复杂性或者并发需求，在项目的生命周期内也可能会出现其他和线程相关的问题。”
>
> 摘录来自: [美] Norman Maurer Marvin Allen Wolfthal. “Netty实战。” 

与 Netty 中的 EventLoop 类似, JStorm 中也建立了一套线程模型, 用于简化创建线程的过程, 并提高性能. 其具体的实现便是 AsyncLoopThread .

## AsyncLoopThread

com.alibaba.jstorm.callback.AsyncLoopThread

![](https://ws1.sinaimg.cn/large/006tNc79ly1fz9q4g0nmjj31dk0om75n.jpg)

核心方法： init

从上面的代码段中, 可以看到 AsyncLoopThread 的启动是去调用了持有的 Runnable#start ,实际上是 JStorm 设计的一个 Runnable 的子类 AsyncLoopRunable.

主要功能：

1. 封装了 Thread 的 start() 方法，对 Thread 做了一些常规的配置操作。
2. 自定义了一个 AsyncLoopDefaultKill 类, 用于自定义杀死进程的操作

每一个 AsyncLoopThread 对象, 持有各自的 Runnable (实际是 AsyncLoopRunable )

## AsyncLoopRunable

核心方法: run

在初始化时传入 RunnableCallback 对象, 业务代码写在 RunnableCallback#run 中.

```java
//构造函数
public AsyncLoopRunnable(RunnableCallback fn, RunnableCallback killFn) {
    this.fn = fn;
    this.killFn = killFn;
}
```

在线程启动后, AsyncLoopRunnable 会在 while 循环中, 不断执行 fn#run, 调用业务代码逻辑.

```java
while (!shutdown.get()) {
    //执行初始化时传入的 RunnableCallback 对象的 run 方法
    fn.run();
    if (shutdown.get()) {
        shutdown();
        return;
    }
    Exception e = fn.error();
    if (e != null) {
        throw e;
    }
    Object rtn = fn.getResult();
    if (this.needQuit(rtn)) {
        shutdown();
        return;
    }
}
```

## 总结


在启动后, AsyncLoopRunable 中存在一个循环, 会不断执行 RunnableCallback#run (这个是最终的业务代码逻辑).

AsyncLoopThread 的使用方法如上所示: 在 新建AsyncLoopThread时, 将业务逻辑封装为一个RunableCallback对象传入.

这样当AsyncLoopThread启动后, 会启动一个异步( Async )的线程, 该线程会循环( Loop )执行传入的业务逻辑, 直到终止条件被触发.
