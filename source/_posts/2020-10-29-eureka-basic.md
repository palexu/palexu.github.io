---
title: Eureka 原理、组件生命周期、流程简介
date: 2020-10-29 19:02:57
tags: Eureka
---

> 这样一篇文章，主要是对自己近期所学重新梳理时，所做下的一篇笔记。
>
> 网络上已经存在大量对Eureka的介绍和原理剖析，不过这些文章存在很多重复雷同，对知识点总结不够清晰，对原理流程介绍又不够形象。本篇文章不算是完全原创，主要还是对这些文章的整理（见文末参考），对格式、内容有所删选，图片也来自不同的文章，在此感谢各个作者的分享。

## Eureka 简介

Eureka 是 Netflix 出品的用于实现服务注册和发现的工具。 Spring Cloud 集成了 Eureka，并提供了开箱即用的支持。其中， Eureka 又可细分为 Eureka Server 和 Eureka Client。

##### Eureka 有哪些组件？

![9A94ADCB-5226-46C1-97F0-878E5B76DC31](https://tva1.sinaimg.cn/large/0081Kckwly1gk6et3jj5ij30sg0dhtch.jpg)

从组件功能看：

* 黄色注册中心集群，分别部署在北京、天津、青岛机房；
* 红色服务提供者，分别部署北京和青岛机房；
* 淡绿色服务消费者，分别部署在北京和天津机房；

从机房分布看：
* 北京机房部署了注册中心、服务提供者和服务消费者；
* 天津机房部署了注册中心和服务消费者；
* 青岛机房部署了注册中心和服务提供者；

##### Eureka 各组件（client/server） 的生命周期是怎么样？

###### client-服务提供者

启动后（心跳+注册）
* 首先会创建一个心跳的定时任务（默认30秒），定时向服务端发送心跳信息，服务端会对客户端心跳做出响应。
* 如果响应状态码为404时，表示服务端没有该客户端的服务信息，那么客户端则会向服务端发送注册请求，注册信息包括服务名、ip、端口、唯一实例ID等信息。

停止时（注销）
* 向注册中心发起 cancel 请求，清空当前服务注册信息。

###### client-服务消费者

* 启动后，从注册中心拉取服务注册信息，并存放本地
* 运行中，定时更新本地服务注册信息。
* 发起远程调用：
    * 服务消费者会从服务注册信息中优先选择同机房的服务提供者，发起远程调用。只有同机房的服务提供者挂了才会选择其他机房的服务提供者。
    * 服务消费者因为同机房内没有服务提供者，则会按负载均衡算法选择其他机房服务提供者，发起远程调用。

###### server-注册中心

* 启动后，从其他节点拉取服务注册信息。
* 运行过程中，定时运行 evict 任务，剔除没有按时 renew 的服务（包括非正常停止和网络故障的服务）。
  > Eureka Server 检测到服务提供者因为宕机、网络原因不可用时，则在服务注册中心将服务置为DOWN状态，并把当前服务提供者状态向订阅者发布，订阅过的服务消费者更新本地缓存。
  > Eureka Server在一定的时间（默认90秒）未收到客户端的心跳，则认为服务宕机，注销该实例。
* 运行过程中，接收到的 register、renew、cancel 请求，都会同步至其他注册中心节点。

## Eureka的自我保护机制

在默认配置中，Eureka Server在默认90s没有得到客户端的心跳，则注销该实例，但是往往因为微服务跨进程调用，网络通信往往会面临着各种问题，比如微服务状态正常，但是因为网络分区故障时，Eureka Server注销服务实例则会让大部分微服务不可用，这很危险，因为服务明明没有问题。

为了解决这个问题，Eureka 有自我保护机制，通过在Eureka Server配置如下参数，可启动保护机制 : `eureka.server.enable-self-preservation=true`。

它的原理是，当Eureka Server节点在短时间内丢失过多的客户端时（可能发送了网络故障），那么这个节点将进入自我保护模式，不再注销任何微服务，当网络故障回复后，该节点会自动退出自我保护模式。
自我保护模式的架构哲学是宁可放过一个，决不可错杀一千。

##### 作为服务注册中心，Eureka与Zookeeper的优缺点？

著名的CAP理论指出，一个分布式系统不可能同时满足C(一致性)、A(可用性)和P(分区容错性)。由于分区容错性在是分布式系统中必须要保证的，因此我们只能在A和C之间进行权衡。在此Zookeeper保证的是CP, 而Eureka则是AP。

![777CF324-1FC4-45F8-A260-72B9AE87AE5A](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fassvduj30kw0j8mzi.jpg)

![A6EC5809-521A-4FB3-9BDB-554187402DA0](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fax1qfsj30ll0a5gm2.jpg)

###### Zookeeper保证CP

当向注册中心查询服务列表时，我们可以容忍注册中心返回的是几分钟以前的注册信息，但不能接受服务直接down掉不可用。也就是说，服务注册功能对可用性的要求要高于一致性。但是zk会出现这样一种情况，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于，选举leader的时间太长，30 ~ 120s, 且选举期间整个 zk 集群都是不可用的，这就导致在选举期间注册服务瘫痪。在云部署的环境下，因网络问题使得zk集群失去master节点是较大概率会发生的事，虽然服务能够最终恢复，但是漫长的选举时间导致的注册长期不可用是不能容忍的。

###### Eureka保证AP

Eureka 在设计时就优先保证可用性。Eureka各个节点都是平等的，几个节点挂掉不会影响正常节点的工作，剩余的节点依然可以提供注册和查询服务。而Eureka的客户端在向某个Eureka注册或时如果发现连接失败，则会自动切换至其它节点，只要有一台Eureka还在，就能保证注册服务可用(保证可用性)，只不过查到的信息可能不是最新的(不保证强一致性)。

除此之外，Eureka还有一种自我保护机制，如果在15分钟内超过85%的节点都没有正常的心跳，那么Eureka就认为客户端与注册中心出现了网络故障，此时会出现以下几种情况： 

1. Eureka不再从注册列表中移除因为长时间没收到心跳而应该过期的服务 
2. Eureka仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上(即保证当前节点依然可用) 
3. 当网络稳定时，当前实例新的注册信息会被同步到其它节点中

因此，Eureka可以很好的应对因网络故障导致部分节点失去联系的情况，而不会像zookeeper那样使整个注册服务瘫痪。

eureka有哪些不足： eureka consumer 本身有缓存，服务状态更新滞后，最常见的状况就是，服务下线了但是服务消费者还未及时感知，此时调用到已下线服务会导致请求失败，只能依靠consumer端的容错机制来保证。

## 数据存储结构
既然是服务注册中心，必然要存储服务的信息，我们知道 ZK 是将服务信息保存在树形节点上。

![fa1a581644096db7d30fba646b1a1445a681bc72](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fb0u6rcj30i00a0t9y.jpg)

而下面是 Eureka 的数据存储结构：

![55FC9DA4-0049-4760-9FFB-91D8B2567D6B](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fb2pqatj30ck0a13yx.jpg)

Eureka 的数据存储分了两层：存储、缓存

Eureka Client 在拉取服务信息时：
1. 先从缓存层获取
2. 如果获取不到，先把数据存储层的数据加载到缓存中，再从缓存中获取

Eureka 这样的数据结构设计是把内部的数据存储结构与对外的数据结构隔离开了，就像是我们平时在进行接口设计一样，对外输出的数据结构和数据库中的数据结构往往都是不一样的。

###### **数据存储层( 服务信息 )**

 rigistry 本质上是一个双层的 ConcurrentHashMap，存储在内存中的。

![5083627B-FCAE-45B6-BD46-4D03DBBB908B](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fb65g7yj30a0067jrk.jpg)

第一层Map：spring.application.name ---> Map<InstancId,Lease>
第二层Map：InstanceId ---> Lease(Lease 对象包含了服务详情和服务治理相关的属性。)

###### **缓存层 (二级缓存  经过处理加工过的、可以直接传输到 Eureka Client 的数据结构)**

Eureka 实现了二级缓存来保存即将要对外传输的服务信息，数据结构完全相同。

一级缓存：
ConcurrentHashMap<Key,Value> readOnlyCacheMap
无过期时间，保存服务信息的对外输出数据结构。

二级缓存：
Guava.Loading<Key,Value> readWriteCacheMap
包含失效机制，保存服务信息的对外输出数据结构。

##### 缓存的更新机制：

![3FEF2C9E-BD22-4556-B0DD-0EC958A90987](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fb8qrxlj30sg0cq0v5.jpg)

更新机制包含删除和加载两个部分，上图黑色箭头表示删除缓存的动作，绿色表示加载或触发加载的动作。

###### 删除二级缓存：

Eureka Client 发送 register、renew 和 cancel 请求并更新 registry 注册表之后，删除二级缓存；
Eureka Server 自身的 Evict Task 剔除服务后，删除二级缓存；
二级缓存本身设置了 guava 的失效机制，隔一段时间后自己自动失效；

###### 加载二级缓存：

Eureka Client 发送 getRegistry 请求后，如果二级缓存中没有，就触发 guava 的 load，即从 registry 中获取原始服务信息后进行处理加工，再加载到二级缓存中。
Eureka Server 更新一级缓存的时候，如果二级缓存没有数据，也会触发 guava 的 load。

###### 更新一级缓存：

Eureka Server 内置了一个 TimerTask，定时将二级缓存中的数据同步到一级缓存（这个动作包括了删除和加载）。
关于缓存的实现参考 ResponseCacheImpl

## 流程

### 服务续约机制（心跳）
服务注册后，要定时（默认 30S，可自己配置）向注册中心发送续约请求，告诉注册中心“我还活着”。

![6E19F5AA-BDDE-4D7F-8DEB-FD5DF3A88C3F](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fbbqyrlj30sg0bodi1.jpg)

注册中心收到续约请求后：
更新服务对象的最近续约时间，即 Lease 对象的 lastUpdateTimestamp;
同步服务信息，将此事件同步至其他的 Eureka Server 节点。

### 服务注册机制
服务提供者、服务消费者、以及服务注册中心自己，启动后都会向注册中心注册服务（如果配置了注册）。
下图是介绍如何完成服务注册的：

![27814EF0-C28C-4A69-B924-074031923AA9](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fbpobjgj30sg0bb76p.jpg)

注册中心服务接收到 register 请求后：

保存服务信息： 将服务信息保存到 registry 中，并清空读写缓存，即 readWriteCacheMap。
更新队列，将此事件添加到更新队列中，供 Eureka Client 增量同步服务信息使用。
更新阈值，供剔除服务使用。
同步服务信息，将此事件同步至其他的 Eureka Server 节点。

### 服务注销机制
服务正常停止之前会向注册中心发送注销请求，告诉注册中心“我要下线了”。

![FDC3826C-678B-4432-A005-96B8CF422D53](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fbuz1tzj30sg0bb76q.jpg)

注册中心服务接收到 cancel 请求后：

删除服务信息：将服务信息从 registry 中删除，清空读写缓存，即 readWriteCacheMap。
更新队列，将此事件添加到更新队列中，供 Eureka Client 增量同步服务信息使用。
更新阈值，供剔除服务使用。
同步服务信息，将此事件同步至其他的 Eureka Server 节点。

ps：服务正常停止才会发送 Cancel，如果是非正常停止，则不会发送（ Eureka Server 会定期扫描并移除非优雅停机的服务）

### 服务剔除机制（补偿机制，处理非正常下线的服务）

Eureka Server 提供了服务剔除的机制，用于剔除没有正常下线的服务。

![0C8790DB-9A7F-4062-9DA7-ACA403CE7C8E](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fbzddckj30i70n7wnd.jpg)

服务的剔除包括三个步骤:
1. 首先判断是否满足服务剔除的条件
2. 然后找出过期的服务，
3. 最后执行剔除。

判断是否满足服务剔除的条件
有两种情况可以满足服务剔除的条件：
* if 关闭了自我保护
统统认为是 Eureka Client 的问题，把没按时续约的服务都剔除掉（这里有剔除的最大值限制）。
* if 开启了自我保护
需要进一步判断是 Eureka Server（自己） 出了问题，还是 Eureka Client 出了问题，如果是 Eureka Client 出了问题则进行剔除。

Eureka 自我保护机制
Eureka 自我保护机制是为了防止误杀服务而提供的一个机制。
Eureka 的自我保护机制“谦虚”的认为如果大量服务都续约失败，则认为是自己出问题了（如自己断网了），也就不剔除了；反之，则是 Eureka Client 的问题，需要进行剔除。
而自我保护阈值是区分 Eureka Client 还是 Eureka Server 出问题的临界值：
* 如果超出阈值就表示大量服务可用，少量服务不可用，则判定是 Eureka Client 出了问题。
* 如果未超出阈值就表示大量服务不可用，则判定是 Eureka Server 出了问题。
这里比较难理解的是阈值的计算：
    自我保护阈值 = 服务总数 * 每分钟续约数 * 自我保护阈值因子。
    每分钟续约数 =（60S/ 客户端续约间隔）
最后自我保护阈值的计算公式为：
    自我保护阈值 = 服务总数 * （60S/ 客户端续约间隔） * 自我保护阈值因子。
举例：如果有 100 个服务，续约间隔是 30S，自我保护阈值 0.85。
    自我保护阈值 =100 * 60 / 30 * 0.85 = 170。
    如果上一分钟的续约数 =180>170，则说明大量服务可用，是服务问题，进入剔除流程；
    如果上一分钟的续约数 =150<170，则说明大量服务不可用，是注册中心自己的问题，进入自我保护模式，不进入剔除流程。

找出过期的服务
遍历所有的服务，判断上次续约时间距离当前时间大于阈值就标记为过期。并将这些过期的服务保存到集合中。
剔除服务
在剔除服务之前先计算剔除的数量，然后遍历过期服务，通过洗牌算法确保每次都公平的选择出要剔除的任务，最后进行剔除。
执行剔除服务后：
1. 删除服务信息，从 registry 中删除服务。
2. 更新队列，将当前剔除事件保存到更新队列中。
3. 清空二级缓存，保证数据的一致性。
实现过程参考 AbstractInstanceRegistry.evict() 方法。

### 服务获取机制
Eureka Client 获取服务有两种方式，全量同步和增量同步。获取流程是根据 Eureka Server 的多层数据结构进行的：

![2019012928](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fc2yj6qj30sg0qjdi0.jpg)

无论是全量同步还是增量同步，都是先从缓存中获取，如果缓存中没有，则先加载到缓存中，再从缓存中获取。（registry 只保存数据结构，缓存中保存 ready 的服务信息。）

先从一级缓存中获取
1. 先判断是否开启了一级缓存
2. 如果开启了则从一级缓存中获取，如果存在则返回，如果没有，则从二级缓存中获取
3. 如果未开启，则跳过一级缓存，从二级缓存中获取
再从二级缓存中获取
1. 如果二级缓存中存在，则直接返回；
2. 如果二级缓存中不存在，则先将数据加载到二级缓存中，再从二级缓存中获取。注意加载时需要判断是增量同步还是全量同步，增量同步从 recentlyChangedQueue 中 load，全量同步从 registry 中 load。

### 服务同步机制
服务同步机制是用来同步 Eureka Server 节点之间服务信息的。它包括 :
* Eureka Server 启动时的同步
* 和运行过程中的同步。

启动时同步

![C175BE44-1C7C-47F6-9662-BF25F5F92499](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fc77eh8j30sg0irjv8.jpg)

Eureka Server 启动后，遍历 eurekaClient.getApplications 获取服务信息，并将服务信息注册到自己的 registry 中。
注意这里是两层循环，第一层循环是为了保证已经拉取到服务信息，第二层循环是遍历拉取到的服务信息。

运行过程中同步

![FF270C49-B048-4FCD-AB5A-82B9A9222B97](https://tva1.sinaimg.cn/large/0081Kckwly1gk6fcegm0lj30sg07yq54.jpg)

当 Eureka Server 节点有 register、renew、cancel 请求进来时，会将这个请求封装成 TaskHolder 放到 acceptorQueue 队列中，然后经过一系列的处理，放到 batchWorkQueue 中。
TaskExecutor.BatchWorkerRunnable是个线程池，不断的从 batchWorkQueue 队列中 poll 出 TaskHolder，然后向其他 Eureka Server 节点发送同步请求。
这里省略了两个部分：
一个是在 acceptorQueue 向 batchWorkQueue 转化时，省略了中间的 processingOrder 和 pendingTasks 过程。
另一个是当同步失败时，会将失败的 TaskHolder 保存到 reprocessQueue 中，重试处理。


## 总结
Eureka作为单纯的服务注册中心来说要比zookeeper更加“专业”，因为注册服务更重要的是可用性，我们可以接受短期内达不到一致性的状况。不过Eureka目前1.X版本的实现是基于servlet的Java web应用，它的极限性能肯定会受到影响。期待正在开发之中的2.X版本能够从servlet中独立出来成为单独可部署执行的服务。
没有最好的选择，最好的选择是根据业务场景来进行架构设计；
如果要求一致性，则选择zookeeper，如金融行业。
如果要求可用性，则Eureka，如电商系统。


## 参考：
1、https://www.cnblogs.com/linjiqin/p/10080444.html
2、https://blog.csdn.net/u012105931/article/details/104659073
3、https://developer.aliyun.com/article/705399
4、https://zhuanlan.zhihu.com/p/88385121
5、http://www.iocoder.cn/Eureka/server-cluster/?vip
6、https://zhuanlan.zhihu.com/p/138542807

