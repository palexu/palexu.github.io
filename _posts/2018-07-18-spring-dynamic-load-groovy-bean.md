---
title: 利用Spring管理热加载的Groovy对象
date: 2018-07-18 17:49
---

## 原因

最近做的项目属于数据分析类型，要求数据分析功能做到快速上线。一开始技术选型的时候，使用的是Java + Groovy。 使用Groovy的原因很简单，因为 Groovy 脚本支持热加载功能。简单的数据分析工作，如一些统计、排序、过滤等，都放在Groovy里完成。需要上线新的数据分析功能时，只需要编写一个新的脚本，并热加载到Jvm中即可。

现在希望将一些数据源访问、数据预处理的工作也放到 Groovy 脚本中完成。也就是说，Groovy 脚本需要具有访问数据源、调用rpc服务等等的功能。由于有Spring，这个功能就可以很方便地实现了。

## 解决方案
利用Spring对Groovy脚本进行管理。
Groovy 脚本保存在数据库中。定时任务不断轮训数据库检测Groovy脚本的更新时间，若有更新，则读取脚本内容，并解析为`Class`。
然后利用Spring提供的工具类`BeanDefinitionBuilder`，生成`BeanDefinition`。`BeanDefinition`中保存了Groovy脚本的meta信息，比如对其他类的依赖。接着，将`BeanDefinition`放入Spring上下文`ApplicationContext`中，并调用初始化方法，实例化Groovy脚本并对其进行依赖注入。最后，调用context.getBean("xxx")拿到该脚本。

![架构简图](https://ws3.sinaimg.cn/large/006tNc79gy1fte784mbcbj30hb0fvq2y.jpg)

## 简单实现

Hello.groovy
这是保存在数据库中的Groovy脚本。

```java
import org.springframework.beans.factory.annotation.Autowired

class Hello {
    @Autowired
    HelloService service;

    HelloService getService() {
        return service
    }

    def run() {
        print(service.hello())
    }
}
```

HelloService.java
这是项目中已经提供的服务，现实项目中可以是访问数据源等功能。

```java
import org.springframework.stereotype.Component;

@Component
public class HelloService {
    public String hello() {
        return "now hello";
    }
}
```

第一步，需要拿到Spring上下文 `ApplicationContext`。这个有很多种实现，比如继承`ApplicationContextAware`接口等。

第二步，获取到编译后的脚本，如下。

```java
//从数据库中获取到脚本内容
String scriptContent = "......";
//编译
Class clazz = new GroovyClassLoader().parseClass(scriptContent);
```

第三步，将bean放入上下文，并进行依赖注入

```java
BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(clazz);
BeanDefinition beanDefinition = beanDefinitionBuilder.getRawBeanDefinition();
context.getAutowireCapableBeanFactory().applyBeanPostProcessorsAfterInitialization(beanDefinition, "hello");
beanFactory.registerBeanDefinition("hello", beanDefinition);
```

第四步，从上下文中获取Groovy脚本

```java
Hello hello = context.getBean("hello");
hello.run();
//console中应当输出下面内容,此时说明HelloService已经成功注入到groovy脚本中了
//now hello
```

## 参考资料
1. [Groovy 使 Spring 更出色，第 2 部分 - 在运行时改变应用程序的行为 - 用 Groovy 为 Spring 应用程序添加可动态刷新的 bean](https://www.ibm.com/developerworks/cn/java/j-groovierspring2.html)
2. [动态注入 Bean 到 Spring 容器](https://blog.csdn.net/chao_1990/article/details/77370314)
3. [spring 动态注册 bean 王雁](https://zhuanlan.zhihu.com/p/30226423)
4. [Spring 动态注册 bean 李佳明](https://zhuanlan.zhihu.com/p/30070328)