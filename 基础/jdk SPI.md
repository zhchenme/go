### 一、Java SPI 介绍

**1.1 简介**

SPI 全称为 Service Provider Interface，是 JDK 内置的一种服务提供发现机制。简单来说，它就是一种动态替换发现机制。例如：有个接口想在运行时才发现具体的实现类，那么你只需要在程序运行前添加一个实现即可，并把新加的实现描述给 JDK 即可。此外，在程序的运行过程中，也可以随时对该描述进行修改，完成具体实现的替换。

Java SPI 实际上是“基于接口的编程＋策略模式＋配置文件”组合实现的动态加载机制。

**1.2 SPI 类加载机制**

SPI 的接口是 Java 核心库的一部分，是由启动类加载器 (Bootstrap Classloader) 来加载的。SPI 的实现类是由系统/应用程序类加载器 (System/Application ClassLoader) 来加载的。

关于 SPI 类加载，涉及到一个破坏双亲委派模型的例子，不是很理解，先贴出来，有机会深入了解类加载器后再拿出来参考：

[真正理解线程上下文类加载器（多案例分析）](https://blog.csdn.net/yangcheng33/article/details/52631940) by 小杨Vita


**1.3 应用场景**

- 数据库驱动加载接口实现类加载（JDBC）
- 日志门面接口实现类加载（SLF4J）
- Spring Dubbo 允许使用 SPI 对接口进行扩展

**1.4 SPI 约定**

 java SPI 的具体约定为:当服务的提供者，提供了服务接口的一种实现之后，在 jar 包的 META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该 jar 包 META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 

 基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。jdk 提供服务实现查找的一个工具类：java.util.ServiceLoader

**1.5 SPI demo**

关于 jdk SPI 的 demo，可以参考：[Java-SPI机制](https://www.jianshu.com/p/e4262536000d) by 贾博岩 

### 参考

[Java-SPI机制](https://www.jianshu.com/p/e4262536000d) by 贾博岩 <br>
[高级开发必须理解的Java中SPI机制](https://www.jianshu.com/p/46b42f7f593c) by caison <br>
[java中的SPI机制](https://blog.csdn.net/sigangjun/article/details/79071850) by sigangjun <br>