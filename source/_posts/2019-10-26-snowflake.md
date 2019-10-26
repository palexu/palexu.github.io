# 分布式id生成器-DID(Distribute Id Generator)

## 1. 需求



目前每个应用如果想要使用分布式id功能, 都需要在各自的数据库中新增did表,在代码中加入mapper等,使用较麻烦. 现在希望将分布式id生成器功能封装成工具类提供.

- 期望做到引入maven依赖即可使用, 且做到本地生成分布式id时高性能、不重复.
- **并且需要适用于之后容器化部署的场景**.
- 设计简单可靠, 尽量减少第三方业务的依赖

## 2. 现状



![img](https://cdn.nlark.com/yuque/0/2019/png/188373/1565335431957-77a89972-0ae2-4c58-ad24-ce3300892f79.png)

鲸鱼的公司原先使用的是snowflake算法的变种, 逻辑分片id(下称workerId)从数据库中获取,上图表示了该算法的原理.

## 3. 改造的方向:

1. 仍是在外部(数据库/缓存)管理workerId, 应用启动时去获取

1. 1. 数据库方案:
       提供client jar包, sofa应用启动时通过rpc向server申请workerId, 业务方需支持sofa-rpc、扫描xml
   2. redis方案:
       应用启动时从redis获取workerId, 业务方需配置redis相关参数

1. 本地通过一定的算法生成workerId, 不包含任何外部数据库/缓存等依赖

1. 1. 通过ip生成workerId
       ipv4: 如 255.255.255.255, 各位相加最大为 1020, 小于1024, 可以对机器较好地划分,但遇到如 1.1.1.2 和 1.1.2.1 等时, 所生成的workerId是相同的.
       ipv6: IP最大 ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff ,将每个 Bit 位的后6位相加。这样在一定程度上也可以满足workerId不重复的问题
   2. 通过hostName生成workerId
       见下文方案

### 3.1 适用于物理机器直接部署:本地根据hostName直接生成workerId



原理: 使用机房+机器编号生成worker.

目前线上机器具有良好的命名规范: {env}-{appName}-{机房}{机器编号}, 如 prod-miniapp-b01, 因此可以考虑这种形式.

例如: 打卡小程序与用户中心业务分离良好,  打卡小程序不会去本地生成用户id并创建新用户, 因此即使打卡小程序的某一台机器与用户中心的某一台机器使用了相同的workerid, 也不会存在业务上使用的id重复的情况.



![img](https://cdn.nlark.com/yuque/0/2019/png/188373/1565335449458-3892d3ff-bcd6-4f58-a680-70cd40afa8cc.png)



- 机房支持: 2^2=4
- 机器支持: 2^8=256
- 自增序列号:2^12, 每毫秒最多2^12=4096个, 单机每秒最大生成400万个id



机房x机器: 单业务下, 4机房最多支持1024台机器,双机房支持最大512台机器.

优点: 无任何外部依赖, 相同业务的机器不会出现重复id的情况.

缺点: 不同业务、相同后缀名的机器,  同一毫秒内生成的id可能会重复

#### 改进:扩展逻辑分片位,分配4bit给业务

通过减少每秒生成的最大id数, 增加逻辑分片的位数.(逻辑分片位数 10bit->12bit, 自增序列号 12bit->10bit)



![img](https://cdn.nlark.com/yuque/0/2019/png/188373/1565335598952-ad9aad01-f347-4311-a7e9-e4532e9153e0.png)



- 业务支持:2^4=16
- 机房支持:2^2=4
- 机器支持:2^6=64
- 自增序列号:2^10, 每毫秒最多2^10=1024个, 单机每秒最大生成100万个id

业务x机房x机器: 单业务下,4机房最多支持256台机器,双机房最多支持128台机器

优点:一定程度上避免了各个业务线同时为同一目的(如生成用户id并入库)生成ID时产生碰撞的问题

### 3.2 同时适用于物理机及K8S部署:本地根据ip生成workerId



由于后续公司考虑将所有环境迁移到k8s上进行部署, 因此分布式id生成的方案需要进行改动.

k8s上部署的java应用有以下几个特点:

1. 无法获取到物理机器的hostName
    类似通过 `InetAddress.getLocalHost().getHostName()`获取到的是容器的ID, 例如:`801ccacd9a3a`

2. 在一个k8s集群上, 一个pod分配一个唯一的ip

> 在Kubernetes集群中Pod有如下两种使用方式：
>
> - **一个Pod中运行一个容器**。“每个Pod中一个容器”的模式是最常见的用法；在这种使用方式中，你可以把Pod想象成是单个容器的封装，kuberentes管理的是Pod而不是直接管理容器。
> - **在一个Pod中同时运行多个容器**。一个Pod中也可以同时封装几个需要紧密耦合互相协作的容器，它们之间共享资源。这些在同一个Pod中的容器可以互相协作成为一个service单位——一个容器共享文件，另一个“sidecar”容器来更新这些文件。Pod将这些容器的存储资源作为一个实体来管理。

和运维同学确认过, 目前及之后, 我们集群的运行方式都属于“一个pod中运行一个容器”, 即使某些情况下修改为“在一个pod中同时运行多个容器”, 也会保证多个容器属于不同的业务应用.

3. pod所分配的ip, 一般使用的是b类, 例如 172.17.0.1/16

> 既然Kubernetes中将容器的联网通过插件的方式来实现，那么该如何解决容器的联网问题呢？
>
> 如果您在本地单台机器上运行docker容器的话会注意到所有容器都会处在`docker0`网桥自动分配的一个网络IP段内（172.17.0.1/16）。该值可以通过docker启动参数`--bip`来设置。这样所有本地的所有的容器都拥有了一个IP地址，而且还是在一个网段内彼此就可以互相通信了。

因此只需要 ip 的后16个比特去进行workerId生成

4. 不确定ip的分配方式, 连续or随机
    因此需要保证在ip连续分配时, 具有一定的随机性

综上, 考虑使用 ip 的后16个比特进行 workerId 的生成.

## 4. 具体实现

#### 流程图



![image-20190813163618114.png](https://cdn.nlark.com/yuque/0/2019/png/188373/1565685491583-263b69ac-10bd-4ce7-b10a-cb80c7db3cec.png)



#### workerId的生成

经过第三节的讨论, 建议使用3.2小节的方案: 本地根据ip生成workerId

![image-20190813161459168.png](https://cdn.nlark.com/yuque/0/2019/png/188373/1565685709143-5932ff93-37ba-4ca3-b11e-8b478c5c2966.png)

ip的后16位参与workerId的生成:

- 多次输入相同的ip,将生成相同的workerId
- 输入一串连续的ip, 生成的workerId是连续且不同的(在分片长度允许范围内)
- k8s 为 pod 分配 ip 是随机从 ip 池中获取, 因此可以保证生成workerId的随机性

## 5. 注意

关于时间戳字段与 EPOCH 字段:

**不建议改动这两个字段, 因为目前线上业务线希望平滑迁移到DID上**

如果改动时间戳字段(例如增加或减少 bit 位), 会导致生成的id出现跳变(比如从 30000000000000... 跳变为 2000000000..., 因为时间戳会进行截取和位移, 导致首位数字可能变小), 当业务线依赖于 id 的递增属性, 那么就会出现问题, 可能导致后续生成的id重复

如果改动 EPOCH 字段(目前为1473955200000L,从2016-09-16 00:00:00开始的毫秒数), 也可能导致后续生成的id重复.

## 参考资料

1. [Github snowflake](https://github.com/twitter-archive/snowflake/tree/snowflake-2010)
2. [58 细聊分布式ID生成方法](https://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=403837240&idx=1&sn=ae9f2bf0cc5b0f68f9a2213485313127)
3. [当当对snowflake算法的实践与坑](https://mp.weixin.qq.com/s?src=11&timestamp=1565331459&ver=1779&signature=TO48UgJmgwjopML-auTc7lzoTy3PNJ7i5hQfIj7Bka0w0nCBA*cq7lY8xYp7fY9Mz-cck67Kj7VDKZYFxWkIDn82ZrVScFI0Wb7gnU7MyZG-5g3*MJdChzKrHZSwJ2cx&new=1)
4. [百度 uid-generator](https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md)
5. [生成全局唯一ID的3个思路](https://mp.weixin.qq.com/s?__biz=MzI4MTY5NTk4Ng==&mid=2247489561&idx=1&sn=7396f373af4efa62ba4dbecc6d7f83b3&source=41#wechat_redirect)
6. [Leaf——美团点评分布式ID生成系统](https://tech.meituan.com/2017/04/21/mt-leaf.html)
7. [Kubernetes中的网络](https://jimmysong.io/kubernetes-handbook/concepts/networking.html) 