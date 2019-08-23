---
title: 走近缓存
date: '2018-02-02 21:57'
tag: '缓存'
---

> 吾常闻，非人勤以求知，乃知者勤以求人也。然吾知其谬。其知者非求人，实乃出而逐人矣。其刻深无情者，如鹰犬逐兔。

目录
1. 为什么使用缓存
    1. 金字塔
    2. 2-8

* 从零写一个本地缓存
    * 哈希
    * 锁
    * 本地缓存的设计与实现
    * 缓存更新算法（http://blog.jobbole.com/30940/）
        1. Least Frequently Used（LFU）
        2. Least Recently User（LRU
        3. Least Recently Used 2（LRU2）
        4. Two Queues（2Q）
        5. Adaptive Replacement Cache（ARC）
        6. Most Recently Used（MRU）
        7. First in First out（FIFO）
        8. Second Chance
        9. CLock
        10. Simple time-based
        11. Extended time-based expiration
        12. Sliding time-based expiration
        13. Random Cache
    * 现代化的缓存更新算法
* 本地缓存
    * guava cache
* 分布式缓存
    * memcache
    * redis
    * 数据分片
* spring 注解缓存

案例
1. 缓存更新的四种套路
    1. Cache aside
    2. Read through
    3. Write through
    4. Write behind caching
2. 缓存可能遇到的问题
    1. 无底洞问题
    2. 穿透问题
    3. 热点key问题
    4. 雪崩问题

参考资料
1. 明辉，缓存那些事，https://tech.meituan.com/cache_about.html，2017-03-17 17:19
2. 陈浩，缓存更新的套路，https://coolshell.cn/articles/17416.html，2016-07-27
3.  jtraining，缓存、缓存算法和缓存框架简介，http://blog.jobbole.com/30940/，2013/03/30
4. https://weibo.com/ttarticle/p/show?hmsr=toutiao.io&id=2309404022116222639373&utm_medium=toutiao.io&utm_source=toutiao.io
5. 高可用架构 「ArchNotes」，微博数据库那些事儿：3个变迁阶段背后的设计思想，https://weibo.com/ttarticle/p/show?id=2309403948901471268848
6. 高可用架构 「ArchNotes」，从优化性能到应对峰值流量：微博缓存服务化的设计与实践，https://weibo.com/ttarticle/p/show?id=2309404013728432540615
7. 吳YH堅，后端的轮子（三）— 缓存，https://mp.weixin.qq.com/s__biz=MjM5ODczNTkwMA==&mid=2650107148&idx=1&sn=1f6d8610c21a55dc3490c16002ee8c1a&scene=0#wechat_redirect
8. 谢晞鸣，缓存使用总结,https://fdx321.github.io/2016/09/09/%E7%BC%93%E5%AD%98%E4%BD%BF%E7%94%A8%E6%80%BB%E7%BB%93/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io
9. carlosfu，缓存使用与设计系列文章—目录，http://carlosfu.iteye.com/blog/2269678?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io
10. 陈浩，分布式系统的事务处理，https://coolshell.cn/articles/10910.html
11. 沈剑，缓存架构设计细节二三事，https://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=404087915&idx=1&sn=075664193f334874a3fc87fd4f712ebc
12. 简直(译),现代化的缓存设计方案,http://ifeve.com/design-of-a-modern-cache/
13. 何锦彬,本地缓存的设计和实现,http://www.cnblogs.com/springsource/p/6475070.html
14. http://blog.csdn.net/dev_csdn/article/details/78949830
参考书籍
1. Redis开发与运维

下次主题：
1. 一致性hash
2. 大公司在真实业务场景如何使用缓存（以微博为例）
3. Guava Cache 实现原理：数据结构、缓存淘汰算法等
4. 现代的缓存淘汰算法

---

# 简介

## 为什么要使用缓存？

传统的后端业务场景中，访问量以及对响应时间的要求均不高，通常只使用DB即可满足要求。这种架构简单，便于快速部署，很多网站发展初期均考虑使用这种架构。但是随着访问量的上升，以及对响应时间的要求提升，单DB无法再满足要求。这时候通常会考虑DB拆分(sharding)、读写分离、甚至硬件升级(SSD)等以满足新的业务需求。

但是这种方式仍然会面临很多问题，主要体现在：性能提升有限，很难达到数量级上的提升，尤其在互联网业务场景下，随着网站的发展，访问量经常会面临十倍、百倍的上涨。成本高昂，为了承载N倍的访问量，通常需要N倍的机器，这个代价难以接受。

当然还有原因是没钱上全内存：）

### 每个开发者都应当知道的读写速度清单

【图】

### 存储器速度金字塔

【图】

### 二八定律

二八定律又名80/20定律、帕累托法则（定律）也叫巴莱特定律、最省力的法则、不平衡原则等，被广泛应用于社会学及企业管理学等。

是19世纪末20世纪初意大利经济学家巴莱多发现的。他认为，在任何一组东西中，最重要的只占其中一小部分，约20%，其余80%尽管是多数，却是次要的，因此又称二八定律。

## 日常开发中会接触到哪些缓存

常用的是本地缓存和使用一些分布式的缓存中间件。

对于单机的、数量不大的，我们常选用Map家族（如HashMap、ConcurrentHashMap），Guava Cache等。

对于分布式缓存中间件，常用的有Redis、Memcached等。

> 小知识:Buffer和Cache分别是什么，它们有什么区别？
>
>1、Buffer（缓冲区）是系统两端处理速度平衡（从长时间尺度上看）时使用的。
它的引入是为了减小短期内突发I/O的影响，起到流量整形的作用。比如生产者——消费者问题，他们产生和消耗资源的速度大体接近，加一个buffer可以抵消掉资源刚产生/消耗时的突然变化。
>
>2、Cache（缓存）则是系统两端处理速度不匹配时的一种折衷策略。因为CPU和memory之间的速度差异越来越大，所以人们充分利用数据的局部性（locality）特征，通过使用存储系统分级（memoryhierarchy）的策略来减小这种差异带来的影响。

## 常关注的缓存的属性

1. 命中率,它是衡量缓存有效性的重要指标。命中率越高，表明缓存的使用率越高。
2. 最大元素,根据不同的场景合理的设置最大元素值往往可以一定程度上提高缓存的命中率，从而更有效的时候缓存。同时它也和gc、回收策略有关。
3. 缓存清空算法,影响元素回收的效率

### 缓存清空算法

主要有这些：LRU（LRU2、2Q、ARC），MRU，LFU，FIFO（Second Chance），CLOCK

---

# 从零实现一个本地缓存
## 基本款
使用HashMap来作为对象内部的缓存

```java
class LocalCache{
  private Map<String,Data> cache = new HashMap<>();

  public Data get(String key){
    Data data = null;

    if(!cache.contains(key)){
      data = getDataFromSource();
      this.put(key,data);
    }else{
      data = cache.get(key);
    }

    return data;
  }

  public Data put(String key,Data data){
    return cache.put(key,data);
  }
}
```

优点：开发简单
缺点：仅限于类的自身作用域内，类间无法共享缓存，且存在并发问题。

对于`类的自身作用域内`这个问题，可以使用加上`static`来解决。并且更换`HashMap`为`ConcurrentHashMap`，来解决并发对
`Map`进行读写的问题。`ConcurrentHashMap`内部实现使用了16把读写锁（JDK7），读写效率和`HashMap`相同，
缺点是高并发下只能做到最终一致性（即一个线程put的数据后，另一个线程可能读到老的值）

## 升级款

```java
class LocalCache{
  private static Map<String,Data> cache = new ConcurrentHashMap<>();

  public static Data get(String key){
    Data data = null;

    if(!cache.contains(key)){
      data = getDataFromSource();
      this.put(key,data);
    }else{
      data = cache.get(key);
    }

    return data;
  }

  public static Data put(String key,Data data){
    return cache.put(key,data);
  }
}
```

优点：
静态变量实现类间可共享，进程内可共享。

缺点：
缓存的实时性稍差。
无法gc来回收占用的堆内存。

因此有了下面这种利用JVM的内存回收机制实现的本地内存

## 自动回收内存款

```java
class LocalCache{
  private static Map<String,SoftReference<Data>> cache = new ConcurrentHashMap<>();

  public static Data get(String key){
    SoftReference ref = (SoftReference) cache.get(key);

    if(null == ref){
      data = getDataFromSource();
      this.put(key,data);
    }else{
      return ref.get();
    }

  }

  public static Data put(String key,Data data){
    SoftReference ref = (SoftReference) cache.put(key,new SoftReference(data))

    if(null == ref){
      return null;
    }

    Data old = ref.get();
    ref.clear();

    return old;
  }
}
```

升级款使用了SoftReference（软引用），这是Java类库中提供的。
软引用有以下特征：
1. 使用 get() 方法取得对象的强引用从而访问目标对象。
2. 所指向的对象按照 JVM 的使用情况（Heap 内存是否临近阈值）来决定是否回收。
3. 可以避免 Heap 内存不足所导致的异常。它保持对使用对象的引用，同时 JVM 依然可以在内存不够用的时候对使用对象进行回收。

>Java.lang.ref 是 Java 类库中特殊的一个包.
提供垃圾回收器密切相关的引用类。
这些引用类对象可以指向其它对象，但它们不同于一般的引用，因为它们的存在并不防碍 Java 垃圾回收器对它们所指向的对象进行回收。

那么当内存占用过多时，JVM如何对这块内存进行回收？
1. 先清空它的 SoftReference，这时调用SoftReference 的 get() 方法将会返回 null
2. 调用value的 finalize() 方法，并在下一轮 GC 中对其真正进行回收。

也就是说，我们不用实现缓存淘汰算法，只需利用JVM的垃圾回收机制即可达到回收内存的目的。当然这也有坏处，
如果Map中会存放大量数据，频繁引发JVM的GC，会导致程序性能下降。至于为什么会性能下降，以及其他关于JVM内存回收相关的细节，
可以参考《深入理解JAVA虚拟机》一书。


## 总结

以上三种实现：基本款、升级款、内存自动回收款，具有如下优点
1. 能直接在heap区内读写
2. 快、方便

但是它们的缺点也很明显：
1. 受heap区域影响，缓存的数据量有限。且不同JVM中无法共享缓存。
2. 同时缓存时间受GC影响。

因此以上这些实现，主要满足单机场景下的小数据量缓存需求。
同时也适合对缓存数据的变更无需太敏感感知，如配置管理、基础静态数据等场景（这些场景通过定时任务去刷新缓存）。
如果对缓存新鲜度敏感，又要使用本地缓存的，可以考虑通过ZK统一监听资源然后通知所有关注该资源的应用去刷新缓存。
