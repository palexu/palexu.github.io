---
title: Maven笔记
date: 2017-4-19 12:00:00
---
创建项目
`maven archetype:generate`
该指令为交互式，这里会卡住一会儿，因为要联网获取网上的项目骨架

也可以创建指定的骨架
#If you know for sure the spelling of the archetypeArtifactId you can use parameter -DinteractiveMode=false

`mvn archetype:generate
-DgroupId=my.groupid
-DartifactId=my-artifactId
-DarchetypeArtifactId=archetype-artifactId
-DinteractiveMode=false`

清理产生的项目
`mvn clean`

编译源代码
`mvn compile`

编译并打包
`mvn package`

编译测试代码
`mvn test-compile`

将你打好的jar包安装到你的本地库
`mvn install`

编译完成后，执行exec运行main方法。

不需要传递参数：
`mvn exec:java -Dexec.mainClass="com.vineetmanohar.module.Main"`

需要传递参数：
`mvn exec:java -Dexec.mainClass="com.vineetmanohar.module.Main" -Dexec.args="arg0 arg1 arg2"`

指定对classpath的运行时依赖：
`mvn exec:java -Dexec.mainClass="com.vineetmanohar.module.Main" -Dexec.classpathScope=runtime`

relativePath的默认值为../pom.xml
