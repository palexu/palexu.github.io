---
title: 教你如何阅读源代码，以及阅读 Spring 源代码的姿势？
date: '2017-08-07 21:57'
tag: 'Spring, SourceCode'
---

> 吾常闻，非人勤以求知，乃知者勤以求人也。然吾知其谬。其知者非求人，实乃出而逐人矣。其刻深无情者，如鹰犬逐兔。

# 前期准备

工欲善其事，必先利其器。对于阅读源代码而言，尤其是Spring这样发展了这么久，代码量这么大的开源项目来说，更是如此。 在开始之前，最好能充分准备一下：

- 趁手的 IDE ，这里我使用的是 IntelliJ IDEA
- 了解[开源框架代码读到什么程度算是读懂了]()
- spring 思维导图，来建立对spring架构的基本认识

## Java 框架开源代码读到什么程度算是读懂了

首先要会用，熟悉常见的API。 其次明确你的目标，你想要探究什么东西。 自己先了解基本的原理，尝试通过自己独立的思考，去实现相同的功能。这一步不需要编码，可以只华一下大致的流程、UML图，或者动笔写下关键步骤的文字或伪代码。

铭记一句话，阅读源代码一定是要有一个目的的，不然会陷在代码当中，不要什么都想看，要挑重点看。 从整体出发，如何实现，再看细节如何处理（比如异常处理、多线程的控制、资源回收、边界情况）。

## 如何导入 Spring 代码到 IDEA 中？

推荐一篇博文[Spring源码导入到Idea.2017的方法](http://blog.csdn.net/dalinsi/article/details/68943915)

环境

- JDK 1.8 以上
- Gradle 3.0 以上
- Git

`git clone https://github.com/spring-projects/spring-framework.git`

进入到spring-framework文件夹中 你可以按照`import-into-idea.md`中所说，尝试运行`./gradlew cleanIdea :spring-oxm:compileTestJava`。如果成功了，就可以直接按照它所提供的步骤继续走下去。

或者像我一样，直接执行`gradlew`，这个命令会帮助你下载spring的所有依赖包，并构建spring源代码。

当你看到`BUILD SUCCESSFUL`时，恭喜你，构建成功了。

接下来就是在IDEA中选择导入已经存在的Gradle工程，并尝试运行一下测试用例。

## 框架&结构介绍

![spring framework runtime](https://palexu.github.io/images/2017-08-07-how-to-read-source-code-and-the-read-ways-of-Spring/0.jpg)



待续

--------------------------------------------------------------------------------

有任何问题，可以在[github](https://github.com/palexu/palexu.github.io/issues/new)上向我提issue交流。
