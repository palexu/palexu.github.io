---
title: 教你如何阅读源代码，以及阅读 Spring 源代码的姿势？
date: 2017-08-07 21:57
tag: Spring,
     SourceCode
---
# 前期准备
工欲善其事，必先利其器。对于阅读源代码而言，尤其是Spring这样发展了这么久，代码量这么大的开源项目来说，更是如此。
在开始之前，最好能充分准备一下：
- 趁手的 IDE ，这里我使用的是 IntelliJ IDEA
-

## 如何导入 Spring 代码到 IDEA 中？
推荐一篇博文[Spring源码导入到Idea.2017的方法](https://github.com/spring-projects/spring-framework.git)

环境
- JDK 1.8 以上
- Gradle 3.0 以上
- Git

```git
git clone https://github.com/spring-projects/spring-framework.git
```

进入到spring-framework文件夹中
你可以按照`import-into-idea.md`中所说，尝试运行`./gradlew cleanIdea :spring-oxm:compileTestJava`。如果成功了，就可以直接按照它所提供的步骤继续走下去。

或者像我一样，直接执行`gradlew`，这个命令会帮助你下载spring的所有依赖包，并构建spring源代码。

当你看到`BUILD SUCCESSFUL`时，恭喜你，构建成功了。

接下来就是在IDEA中选择导入已经存在的Gradle工程，并尝试运行一下测试用例。

##
