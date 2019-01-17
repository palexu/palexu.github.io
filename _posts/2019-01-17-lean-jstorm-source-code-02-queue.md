---
title: JStorm 源码分析 - 高性能队列 LMAX DisruptorQueue
date: 2019-01-17 16:13:05
---

在上一次文章中( [JStorm 源码分析 - tuple 在整个拓扑中的流转过程](lean-jstorm-source-code-01)), 我们多次提到 JStorm 使用了 3 个队列来完成 tuple 的缓冲与消费. 

因此这些队列的性能会制约 JStorm 的总体的吞吐量. 

在日常的工程实践当中, 我们会使用一些 Java 的内置队列来完成这个工作, 比如 ArrayBlockingQueue. 实际上, 如果要追求更高的性能, DisruptorQueue 是一个更好的选择, 它虽然降低了 API 的易用性, 但是在性能上相比于 ArrayBlockingQueue 这类队列提升很大.

Storm/JStorm 就使用 Disruptor 作为队列, 来提高系统的性能.

> Disruptor 是英国外汇交易公司 LMAX 开发的一个高性能队列,研发的初衷是解决内存队列的延迟问题 ( 在性能测试中发现竟然与I/O操作处于同样的数量级 ).
>
> 需要特别指出的是，这里所说的队列是系统内部的内存队列，而不是Kafka这样的分布式队列.

> **封仲淹：**  (阿里巴巴中间件平台技术部高级技术专家， 花名纪君祥， 阿里巴巴JStorm 核心作者之一)
>
> Disruptor是谷歌后来开源的分布式高性能的队列，这个队列是非常赞的队列，它的性能非常高。我们做过性能测试，它比BlockingQueue、ConcurrentQueue要快一个数量级，它的核心原理是对cache有高速的亲和性，在内存有很好的防止GC。还有一个更重要的特性，它在锁这块控制的非常细。像BlockingQueue、ConcurrentQueue它们在头尾上有一个锁，这个锁的粒度远远高于Disruptor，Disruptor彻底利用了CPU里面的指令级的锁，所以它在锁这块非常的高效，这就是为什么Disruptor的性能比BlockingQueue、ConcurrentQueue高一个数量级的核心的原理。在社区上我知道很多人已经开始使用Disruptor queue，它已经发展到三了，是非常不错的。但是Disruptor我们在二的时候出现过一个问题，它会有一个CPU空转的问题。当时我们自己写了一段代码绕过去，但是这段代码其实可以contribute到社区让他们去改进。至于为什么当初选Disruptor，最重要的原因是storm用了Disruptor，我们也就跟进了。但是我们做了一些自己的分析跟性能测试，Disruptor确实是非常赞的高并发队列。
>
> 我个人的观点，就看你对性能的要求有多高。如果你要达到极致的性能，对延迟要求非常低，而且对高并发要求性能非常高的时候，你肯定要选择Disruptor。但是从易用性上来讲，Disruptor使用起来并没有传统的queue使用上更方便。你在百万级别并发的时候，我推荐大家使用Java的ConcurrentQueue跟BlockingQueue。但是如果你需要更低的延迟的话，我推荐用Disruptor。
>
> 出自 [封仲淹：如何优雅地使用Disruptor](http://www.qingpingshan.com/rjbc/qt/106547.html)

关于 Disruptor 的原理与使用, 网上已经有很多优秀的文章, 可以阅读以下文章来学习:

1. [[美团技术团队\] 2016-11-18 高性能队列--Disruptor](https://tech.meituan.com/disruptor.html)  这篇文章深入浅出,写的很好.
2. <https://lmax-exchange.github.io/disruptor/> 官方文档.