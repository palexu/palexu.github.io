处于学习netty的目的, 需要进行一些项目时间. 目前的工作暂时接触不到相关的内容,  所以考虑自己造一个项目.

目前网上已经有对应的项目了: [Netty 入门与实战：仿写微信 IM 即时通讯系统](https://juejin.im/book/5b4bc28bf265da0f60130116), 花了几分钟看了一下, 似乎没有看到比较困难的点, 基本上就是实现了基础功能吧. 这篇文章主要介绍的还是netty的一些常用操作和属性, 适合我用来入门查看.

开新项目的目标:

- 学习 netty, 上手netty的使用方法
- 不要太关注im系统的详细设计, 而是围绕基本的功能点进行底层原理的学习, 做到掌握 netty.
- 从基本的原型开始 , 不断迭代, 并记录每一次优化后的性能差距, 形成文档.
- 对一些技术难点, 一定要做好记录.
- 最好能够应用到一些场景, 如实时客服等.
- 其次, 希望能运用上ddd的思想
- 在跑满本机性能的情况下(90%cpu, 6G jvm), 尽可能多得优化代码性能, 并作出总结.                                                       baidu
- 掌握压测工具: jmeter 高并发压测



任务安排:

第一次迭代:

1. 了解im系统相关设计

2. 给出v1版本系分, 满足基本的需求

   1. 参考文档:

      1. [**移动端IM开发需要面对的技术问题**](http://www.52im.net/thread-133-1-1.html)
         「写的很好」

      2. [为自己搭建一个分布式 IM(即时通讯) 系统](https://juejin.im/post/5c2bffdc51882509181395d7)
         「较简单」

      3. [**一套海量在线用户的移动端IM架构设计实践分享(含详细图文)**]([http://www.52im.net/thread-812-1-1.html](http://www.52im.net/thread-812-1-1.html))
         「这个文章真的太丰富了, 参考资料都列了一大坨」
      
         > 如果在没有太多经验可借鉴的情况下，要设计一套完整可用的移动端IM架构，难度是相当大的。原因在于，IM系统（尤其是移动端IM系统）是多种技术和领域知识的横向应用综合体：网络编程、通信安全、高并发编程、移动端开发等，如果要包含实时音视频聊天的话，则还要加上难度更大的音视频编解码技术（内行都知道，把音视频编解码及相关技术玩透的，博士学位都可以混出来了），凡此种种，加上移动网络的特殊性、复杂性，设计和开发难度不言而喻。
      
      4. [IM即时通讯：如何跳出传统思维来设计聊天室架构？](http://yunxin.163.com/blog/im13-0622/)
         「这篇介绍了可能遇到的难点与问题」
      
      5. [一个海量在线用户即时通讯系统（IM）的完整设计]([http://www.yunliaoim.com/im/1075.html](http://www.yunliaoim.com/im/1075.html))
         「真的是完整设计, 数据库表、流程图一应俱全, 建议阅读」
      
      6. [**P2P技术如何将实时视频直播带宽降低75%？**](http://www.52im.net/thread-1289-1-1.html)
         这篇虽然和本文无关,但是写的也很好….


3. 开发并压测

第二次迭代:

...





## 第一次迭代

### 需求

1. 实时消息推送(点对点)
2. 实时群消息推送(点对多)
3. 登录认证(非核心功能, 不做)



### 疑问

1. 客户端之间发送消息: p2p/server
   p2p不现实(因为nat限制, 且不支持离线、群组), 所以采取server方式.

2. 网络通讯协议: 使用tcp长连接, or http短链接?  以及这两个的原理和区别是啥?
   https://www.cnblogs.com/beifei/archive/2011/06/26/2090611.html
   https://www.zhihu.com/question/22677800
   https://juejin.im/post/5b3649d751882552f052703b
   https://www.jianshu.com/p/56881801d02c
   https://blog.51cto.com/ironfighter/1881283

3. tcp要解决什么问题? 为什么要握手挥手?什么场景下用tcp?

4. 端口上限是65535, 为什么会有 c10k,c100k的问题呢?
   https://www.zhihu.com/question/66553828
   「核心原因是: quadruple, tcp4元组决定一个链接(相当于数据库的组合键)」
   linux4.2之前, 客户端最多~65k

   >在Linux里，如果是作为客户端或者负载均衡器的节点连接多个服务器，在connect()服务器之前，调用bind()先绑定IP地址(通常是在多网卡的场景)，即使使用bind(IP, port=0)，Kernel也会帮你选定一个端口。这样就会出现只能使用～65k的连接
   >
   >直到Kernel 4.2版本，一个新的socket option IP_BIND_ADDRESS_NO_PORT的引入，这个问题才算解决。
   >
   >> **IP_BIND_ADDRESS_NO_PORT** (since Linux 4.2)
   >>               Inform the kernel to not reserve an ephemeral port when using
   >> [bind(2)](https://link.zhihu.com/?target=http%3A//man7.org/linux/man-pages/man2/bind.2.html) with a port number of 0.  The port will later be auto‐
   >>               matically chosen at [connect(2)](https://link.zhihu.com/?target=http%3A//man7.org/linux/man-pages/man2/connect.2.html) time, in a way that allows
   >>               sharing a source port as long as the 4-tuple is unique.
   >
   >作者：wipan 链接：https://www.zhihu.com/question/66553828/answer/244748823 来源：知乎

5. http 复用的方式 [HTTP复用](http://blog.zhanglun.me/2017/10/10/HTTP复用/)

6. 在以TCP为连接方式的服务器中，为什么在服务端设计当中需要考虑心跳？ https://www.zhihu.com/question/35013918

