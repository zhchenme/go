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

### 二、SPI 案例分析

最早学习 Java 时连接数据库，都会先利用反射机制 `Class.forName("com.mysql.jdbc.Driver");` 加载数据库驱动。

这里其实可以引申出一个面试题，`Class.forName()` 与 `ClassLoader.loadClass` 区别，`Class.forName("com.mysql.jdbc.Driver");` 之所以能够加载数据库驱动，除了将类加载到虚拟机还会初始化该类，从而执行该类的静态方法，而加载数据库驱动，就是在那个静态方法里完成的，`ClassLoader.loadClass` 只会将类加载到虚拟机而不会执行类的初始化。

随着版本的更新（具体版本不清楚），现在连接数据库，不执行 `Class.forName("com.mysql.jdbc.Driver");` 也会加载数据库驱动，原因就是利用了 SPI 服务发现机制，下面我们一起来看一下。

`DriverManager` 类里也有一个静态代码块，如下

```java
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
```

下面接着来看 `loadInitialDrivers` 方法

```java
private static void loadInitialDrivers() {
        String drivers;
        try {
        	// 通过系统系统变量获取数据库驱动，系统变量中并没有这个值
            drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
                public String run() {
                	// 有兴趣的可以打印 System.getProperty 值看看
                    return System.getProperty("jdbc.drivers");
                }
            });
        } catch (Exception ex) {
            drivers = null;
        }

        AccessController.doPrivileged(new PrivilegedAction<Void>() {
        	// SPI 加载数据库驱动
            public Void run() {
            	// 加载服务资源
                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();
                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                // Do nothing
                }
                return null;
            }
        });

        println("DriverManager.initialize: jdbc.drivers = " + drivers);

        // 系统变量中有数据库驱动值，则直接使用 Class.forName 加载
        if (drivers == null || drivers.equals("")) {
            return;
        }
        // 数据库驱动可配置多个
        String[] driversList = drivers.split(":");
        println("number of Drivers:" + driversList.length);
        for (String aDriver : driversList) {
            try {
                println("DriverManager.Initialize: loading " + aDriver);
                Class.forName(aDriver, true,
                        ClassLoader.getSystemClassLoader());
            } catch (Exception ex) {
                println("DriverManager.Initialize: load failed: " + ex);
            }
        }
    }
```

我们要关注的是 `driversIterator.next()` 迭代器方法，如下

```java
        public S next() {
            if (acc == null) {
                return nextService();
            } else {
                PrivilegedAction<S> action = new PrivilegedAction<S>() {
                    public S run() { return nextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }
```

`next` 方法中调用了 `nextService()`

```java
        private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
            	// 加载数据库驱动
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service,
                     "Provider " + cn  + " not a subtype");
            }
            try {
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }
```

看了上面的代码就很清晰了，通过 SPI 机制，加载 `META-INF/services/` 路径下的配置，通过迭代器逐个注册数据库驱动。

### 三、线程上下文类加载器

在上面的例子中，SPI 会调用 `ServiceLoader.load(Driver.class)` 加载驱动类，详细代码如下

```java
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
```

加载 `ServiceLoader` 的 `BootstrapClassLoader` 是不能加载 SPI 的实现类的，因为 SPI 的实现类是由 `AppClassLoader` 加载的，而 `BootstrapClassLoader` 是不能委派 `AppClassLoader` 来加载类的，于是就使用线程上下文类加载器来解决这个问题。

线程上下文类加载器本质属于应用程序类加载器(Application ClassLoader)，下面可以用一个例子来验证

```java
    public static void main(String[] args) {
        System.out.println(Thread.currentThread().getContextClassLoader()); 
        System.out.println(Test.class.getClassLoader()); 
        System.out.println(ClassLoader.getSystemClassLoader()); 
    }
```

执行上面的代码你会发现输出的值是一样的，那为什么还要有线程上下文类加载器呢？个人觉得，当双亲委派模型遭到破坏时，就需要使用使用线程上下文类加载器去打破这种限制。

### 参考

[Java-SPI机制](https://www.jianshu.com/p/e4262536000d) by 贾博岩 <br>
[高级开发必须理解的Java中SPI机制](https://www.jianshu.com/p/46b42f7f593c) by caison <br>
[java中的SPI机制](https://blog.csdn.net/sigangjun/article/details/79071850) by sigangjun <br>
[理解TCCL：线程上下文类加载器](https://www.itcodemonkey.com/article/5859.html) by 蹲厕所的熊 <br>



