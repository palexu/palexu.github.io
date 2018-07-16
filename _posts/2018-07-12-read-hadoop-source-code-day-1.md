---
title: Hadoop源码阅读-day1
date: 2018-07-12 12:00
---
首先从github checkout最新的代码。现在Hadoop总体代码体积接近400MB了，还是比较大的。

## 1.尝试在Mac下编译最新版本

代码clone成功后，可以看到现在默认分支在trunk。

首先尝试进行maven编译
```bash
mvn package -Pdist,native -DskipTests -Dtar
```

此时发现我的Mac没有安装2.5.0版本的protobuf，而最新版本的Hadoop的序列化默认使用了protobuf，因此maven编译出错。

```bash
mvn install protobuf@2.5
brew link --force --overwrite protobuf@2.5
```

遇到Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.1:testCompile (default-testCompile) on project hadoop-common这个问题，网上说删除~/.m2/repository即可，代码如下：
```
rm -rf ~/.m2/repository
```

但是这一步我始终过不去，不清楚是不是java版本的原因，最终还是决定学习1.0.0版本的代码。

## 2.在Mac下编译1.0.0版本

注意，此时需要在jdk1.7以下进行编译（毕竟是7年前的老代码了），并且这时候的代码还没有引入Maven，因此需要使用Ant进行编译，操作如下：

```bash
git checkout branch-1.0
ant -Dversion=1.0.0 jar
```

编译通过，day1到此顺利结束。

---
## 参考
1.董西成的 “Hadoop技术内幕：深入解析MapReduce架构设计与实现原理 (大数据技术丛书)

---
