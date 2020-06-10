# Dubbo 框架内核原理剖析

### 前言

Dubbo 是阿里巴巴开发的一个开源的高性能的远程服务调用框架，致力于提供高性能和透明化的 RPC 远程调用服务解决方案。作为阿里巴巴 SOA 服务化治理方案的核心框架，目前它已进入 Apache 卵化器项目，其前景可谓无限光明。

本 Chat 我们来探讨支撑 Dubbo 框架的内核原理，包含：

- Dubbo 框架整体架构分析；
- Dubbo 的适配器原理；
- Dubbo 的动态编译原理；
- JDK 标准 SPI 原理，Dubbo 增强 SPI 原理,扩展点的自动包装原理；
- Dubbo 如何使用 JavaAssist 减少反射调用开销。

本文使用 Dubbo2.7.1 版本进行讲解

### Dubbo 内核原理剖析

#### Dubbo 分层架构概述

本节我们从整体上来看看 Dubbo 的分层架构设计，架构分层是一个比较经典的模式，比如网络中的 7 层协议，每层执行固定的功能，上层依赖下层提供的功能，下层对上层提供功能，下层的改变对上层不可见，并且每层都是一个可被替换的组件。

如下图 2.1.1 是 Dubbo 官方提供的 Dubbo 的整体架构图：

![在这里插入图片描述](https://images.gitbook.cn/33ae5d20-a925-11e9-8b3f-69de6a04c825) 图 2.1.1

Dubbo 官方提供的该架构图很复杂，一开始我们没必要深入细节，下面我们简单讲解下其中的主要模块：

- 其中 Service 和 Config 层为 API 接口层，是为了方便的让 Dubbo 使用方发布服务和引用服务；对于服务提供方来说需要实现服务接口，然后使用 ServiceConfig API 来发布该服务；对于服务消费方来说需要使用 ReferenceConfig 对服务接口进行代理。Dubbo 服务发布与引用方可以直接初始化配置类，也可以通过 Spring 配置自动生成配置类。
- 其它各层均为 SPI 层，SPI 意味着下面各层都是组件化可以被替换的，这也是 Dubbo 设计的比较好的一点。Dubbo 增强了 JDK 中提供的标准 SPI 功能，在 Dubbo 中除了 Service 和 Config 层外，其它各层都是通过实现扩展点接口来提供服务的；Dubbo 增强的 SPI 增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点；并且不会一次性实例化扩展点的所有实现类，这避免了当扩展点实现类初始化很耗时，但当前还没用上它的功能时仍进行加载实例化，浪费资源的情况；增强的 SPI 是在具体用某一个实现类的时候才对具体实现类进行实例化。后续会具体讲解 Dubbo 增强的 SPI 的实现原理。
- Proxy 服务代理层：该层主要是对服务消费端使用的接口进行代理，把本地调用透明的转换为远程调用；另外对服务提供方的服务实现类进行代理，把服务实现类转换为 Wrapper 类，这是为了减少反射的调用，后面会具体讲解到。Proxy 层的 SPI 扩展接口为 ProxyFactory，Dubbo 提供的实现主要有 JavassistProxyFactory（默认使用）和 JdkProxyFactory，用户可以实现 ProxyFactory SPI 接口，自定义代理服务层的实现。
- Registry 服务注册中心层：服务提供者启动时候会把服务注册到服务注册中心，消费者启动时候会去服务注册中心获取服务提供者的地址列表，Registry 层主要功能是封装服务地址的注册与发现逻辑，扩展接口 Registry 对应的扩展实现为 ZookeeperRegistry、RedisRegistry、MulticastRegistry、DubboRegistry 等。扩展接口 RegistryFactory 对应的扩展接口实现为 DubboRegistryFactory、DubboRegistryFactory、RedisRegistryFactory、ZookeeperRegistryFactory。另外该层扩展接口 Directory 实现类有 RegistryDirectory、StaticDirectory 用来透明的把 invoker 列表转换为一个 invoker;用户可以实现该层的一系列扩展接口，自定义该层的服务实现。
- Cluster 路由层：封装多个服务提供者的路由规则、负载均衡、集群容错的实现，并桥接服务注册中心；扩展接口 Cluster 对应的实现类有 FailoverCluster(失败重试)、FailbackCluster（失败自动恢复）、FailfastCluster（快速失败）、FailsafeCluster（失败安全）、ForkingCluster（并行调用）等；负载均衡扩展接口 LoadBalance 对应的实现类为 RandomLoadBalance（随机）、RoundRobinLoadBalance（轮询）、LeastActiveLoadBalance（最小活跃数）、ConsistentHashLoadBalance（一致性 hash)等。用户可以实现该层的一系列扩展接口，自定义集群容错和负载均衡策略。
- Monitor 监控层：用来统计 RPC 调用次数和调用耗时时间，扩展接口为 MonitorFactory，对应的实现类为 DubboMonitorFactroy。用户可以实现该层的 MonitorFactory 扩展接口，实现自定义监控统计策略。
- Protocol 远程调用层：封装 RPC 调用逻辑，扩展接口为 Protocol， 对应实现有 RegistryProtocol、DubboProtocol、InjvmProtocol 等。
- Exchange 信息交换层：封装请求响应模式，同步转异步，扩展接口 Exchanger，对应扩展实现有 HeaderExchanger 等。
- Transport 网络传输层：抽象 mina 和 netty 为统一接口。扩展接口为 Channel，对应实现有 NettyChannel（默认）、MinaChannel 等;扩展接口 Transporter 对应的实现类有 GrizzlyTransporter、MinaTransporter、NettyTransporter（默认实现）；扩展接口 Codec2 对应实现类有 DubboCodec、ThriftCodec 等
- Serialize 数据序列化层：提供可以复用的一些工具，扩展接口为 Serialization，对应扩展实现有 DubboSerialization、FastJsonSerialization、Hessian2Serialization、JavaSerialization 等，扩展接口 ThreadPool 对应扩展实现有 FixedThreadPool、CachedThreadPool、LimitedThreadPool 等。

综上可知 Dubbo 的分层架构使得 Dubbo 的每层的功能都是可被替换的，这使得 Dubbo 的扩展性极强，上面说了那么多关于扩展点的东西，那么具体什么是扩展点呢，下面看下 Dubbo 扩展点一个简单例子。以扩展点 Protocol 为例:

```java
@SPI("dubbo")
public interface Protocol {
...
}
```

扩展点接口的类上面都含有@SPI 注解，这里注解里面的"dubbo"说明 Protocol 扩展接口 SPI 的默认实现是 DubboProtocol。

如果我们想自己写一个 Protocol 扩展接口的实现类，那么我们需要在实现类所在的 Jar 包内的 `META-INF/dubbo/` 目录下创建一个名字为 org.apache.dubbo.rpc.Protocol 的文本文件，然后配置它的内容为：

```
myprotocol=com.alibaba.user.MyProtocol
```

假设该实现类 MyProtocol 的内容如下：

```
package com.alibaba.user;
public class MyProtocol implemenets Protocol {
// ...
}
```

那么如何使用我们自定义的扩展实现呢？Dubbo 配置模块中，扩展点均有对应配置属性或标签，如下代码通过配置标签方式指定使用哪个扩展实现：

```
<dubbo:protocol name="myprotocol" />
```

注意这里的 name 必须与 jar 包内 `META-INF/dubbo/` 目录下 org.apache.dubbo.rpc.Protocol 文件中的等号左侧的 key 的名字一致。

#### Dubbo 远程调用细节

本节我们先从整体来看下 Dubbo 服务发布与消费的概要流程，以便对其中涉及到的概念有个了解，更详细的过程后面的章节会具体讲解。

##### **服务提供者暴露一个服务的概要过程**

如下图 2.2.1.1 是服务提供者暴露一个服务的概要过程：

![在这里插入图片描述](https://images.gitbook.cn/41f85a70-a925-11e9-8b3f-69de6a04c825)图 2.2.1.1

- 首先 ServiceConfig 类引用对外提供服务的实现类 ref（如：GreetingServiceImpl），然后通过 ProxyFactory 接口的扩展实现类的 getInvoker 方法使用 ref 生成一个 AbstractProxyInvoker 实例，到这一步就完成了具体服务到 Invoker 的转化。接下来就是 Invoker 转换到 Exporter 的过程。 Dubbo 协议的 Invoker 转为 Exporter 发生在 DubboProtocol 类的 export 方法中，Dubbo 处理服务暴露的关键就在 Invoker 转换到 Exporter 的过程，在这个过程中会先启动 Netty Server 监听服务连接，然后注册服务到服务注册中心。

##### **服务消费者消费一个服务的概要过程**

如下图 2.2.2.1 服务消费者消费一个服务的概要过程 ![在这里插入图片描述](https://images.gitbook.cn/49261f30-a925-11e9-8fca-f70fa6e8acb0) 图-2.2.2.1

- 首先 ReferenceConfig 类的 init 方法调用 Protocol 扩展接口实现类的 refer 方法生 成 Invoker 实例（如上图中的红色部分），这是服务消费的关键。接下来把 Invoker 转换为客户端需要的接口（如：GreetingService）。
- Dubbo 协议的 invoker 转换为客户端需要的接口，发生在 ProxyFactory 接口的扩展实现类的的 getProxy 方法中，它主要是使用代理对服务接口的调用转换为对 invoker 的调用。

#### Dubbo 的适配器原理

前面谈论 Dubbo 的分层架构时候我们谈到 Dubbo 为每个功能点提供了一个 SPI 扩展接口，dubbo 框架使用扩展点功能的时候是对接口进行依赖的，而一个扩展接口对应了一系列的扩展实现类，那么如何选择使用哪一个扩展接口的实现类那？其实是使用适配器模式来做的。

首先我们来看看什么是适配器模式，比如 dubbo 提供的扩展接口 Protocol，Protocol 的定义如下：

```java
@SPI("dubbo")
public interface Protocol {
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
    ....
}
```

Dubbo 则会使用我们下节将要讲解的动态编译技术为接口 Protocol 生成一个适配器类`Protocol$Adaptive`的对象实例，Dubbo 框架中需要使用 Protocol 的实例的时候实际就是使用的`Protocol$Adaptive`的对象实例来获取具体 SPI 实现类，其代码如下：

```java
package org.apache.dubbo.rpc;
...
public class Protocol$Adaptive implements Protocol {   
 ...
public Exporter export(Invoker invoker) throws RpcException {
    String string;
    ...
    //(1)
    URL uRL = invoker.getUrl();
    String string2 = string = uRL.getProtocol() == null ? "dubbo" : uRL.getProtocol();
    if (string == null) {
        throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (").append(uRL.toString()).append(") use keys([protocol])").toString());
    }
    //(2)
    Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(string);
    //(3)
    return protocol.export(invoker);
}
```

在 dubbo 框架中 protocol 的一个定义为： `private static final Protocol protocol =ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();`当调用`protocol.export(wrapperInvoker)`时候，实际是调用的`Protocol$Adaptive`的对象实例的 export 方法，然后后者根据 wrapperInvoker 中的 url 里面的协议类型参数执行代码（2）使用 Dubbo 增强 SPI 方法 getExtension 获取对应的 SPI 实现类，然后调用代码（3）执行具体 SPI 实现类的 export 方法。

需要注意的是在 Dubbo 中 URL 是一个核心概念，Dubbo 框架把所需的参数都拼接到了 URL 对象里面了，这里假设执行代码（2）传递的协议类型为 dubbo，则说明使用增强 SPI 返回扩展接口 Protocol 的 dubbo 实现，也就是返回 DubboProtocol 的实例，那么代码（3）则返回调用 DubboProtocol 的 export 的返回结果。也就是框架调用`protocol.export(wrapperInvoker)`时候，实际上是调用了 DubboProtocol 的 export 方法。

总结一点，也就是说适配器类`Protocol$Adaptive`会根据传递的协议参数的不同，加载不同的 Protocol 的 SPI 实现。

其实在 Dubbo 框架中框架会给每个 SPI 扩展接口动态生成一个对应的适配器类，用来根据参数使用增强 SPI 选择不同的 SPI 实现。比如扩展接口 ProxyFactory 的适配器类为`ProxyFactory $Adaptive`，用来根据参数 proxy 选择是使用 JdkProxyFactory 还是使用 JavassistProxyFactory 做代理工厂；扩展接口 Registry 的适配器类`Registry$Adaptive`则根据参数 register 来决定是使用 ZookeeperRegistry、RedisRegistry、MulticastRegistry、DubboRegistry 中的哪一个作为服务治理中心等等。

#### Dubbo 的动态编译原理

众所周知 Java 程序要想运行首先需要使用 javac 把源代码编译为 class 字节码文件，然后使用 JVM 加载 class 字节码文件到内存后创建 Class 对象后，使用 Class 对象创建对象实例。正常情况下我们是把所有的源文件静态编译为字节码文件后，由 JVM 统一加载，而动态编译则是在 JVM 进程运行时把源文件编译为字节码文件，然后使用字节码文件创建对象实例。

上节我们提到 Dubbo 框架中框架会给每个 SPI 扩展接口动态生成一个对应的适配器类，那么如何生成的那？其就使用了动态编译技术，在 dubbo 中提供了一个 Compiler 的 spi:

```java
@SPI("javassist")
public interface Compiler {
    Class<?> compile(String code, ClassLoader classLoader);
}
```

dubbo 提供了 Compiler 的实现有 JavassistCompiler（默认实现）和 JdkCompiler。

这里我们从 Dubbo 框架如何使用动态编译生成扩展接口对应的适配器类入手，首先我们打开 ExtensionLoader 的 createAdaptiveExtensionClass 方法，就是该方法动态编译源文件为 Class 对象的，有了 Class 对象然后我们就可以使用 newInstance（）方法创建对象实例了：

```Java
private Class<?> createAdaptiveExtensionClass() {
    //3.1.4-1
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    ClassLoader classLoader = findClassLoader();
    //3.1.4-2
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    //3.1.4-3
    return compiler.compile(code, classLoader);
}
    }
```

- 如上代码 3.1.4-1 则是根据 SPI 扩展接口生成其对应的适配器类的源码，其返回的是一个字符串，比如对于 Protocol 扩展接口，则这里返回的字符串内容为：

```Java
package org.apache.dubbo.rpc;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.ExtensionLoader;
import org.apache.dubbo.rpc.Exporter;
import org.apache.dubbo.rpc.Invoker;
import org.apache.dubbo.rpc.Protocol;
import org.apache.dubbo.rpc.RpcException;

public class Protocol$Adaptive
implements Protocol {
    @Override
    public void destroy() {
        throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    @Override
    public int getDefaultPort() {
        throw new UnsupportedOperationException("The method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public Exporter export(Invoker invoker) throws RpcException {
        String string;
        if (invoker == null) {
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        }
        if (invoker.getUrl() == null) {
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        }
        URL uRL = invoker.getUrl();
        String string2 = string = uRL.getProtocol() == null ? "dubbo" : uRL.getProtocol();
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (").append(uRL.toString()).append(") use keys([protocol])").toString());
        }
        Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(string);
        return protocol.export(invoker);
    }

    public Invoker refer(Class class_, URL uRL) throws RpcException {
        String string;
        if (uRL == null) {
            throw new IllegalArgumentException("url == null");
        }
        URL uRL2 = uRL;
        String string2 = string = uRL2.getProtocol() == null ? "dubbo" : uRL2.getProtocol();
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (").append(uRL2.toString()).append(") use keys([protocol])").toString());
        }
        Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(string);
        return protocol.refer(class_, uRL);
    }
}
```

- 代码 3.1.4-2 则使用增强 SPI 选择扩展接口 Compiler 的实现，这里默认为 JavassistCompiler，然后代码 3.1.4-3 则调用 JavassistCompiler 的 compile 方法根据源代码生成 Protocol$Adaptive 的 Class 对象。

总结下，Dubbo 框架会为每个扩展接口生成其对应的适配器类的源码，然后选择具体的动态编译类的扩展实现对源码进行编译生成适配器类的 Class 对象，然后就可以调用 Class 对象的 newInstance()方法生成扩展接口对应的适配器类的实例。

#### Dubbo 增强 SPI 原理

前面我们讲解了 Dubbo 框架如何使用动态编译技术给每个扩展接口生成适配器类，并讲解了适配器类内根据参数来选择对应的 SPI 实现，下面我们来讲解适配器类中是如何根据参数来装载具体的 SPI 的实现的。

##### **JDK 中标准 SPI 原理**

Dubbo 增强的 SPI 功能是从 JDK 标准 SPI 演化而来的，所以有必要先讲讲标准 SPI 的原理。

JDK 中的 SPI（Service Provider Interface）是面向接口编程的，服务规则提供者会在 JRE 的核心 API 里面提供服务访问接口，而具体实现则由其他开发商提供。

例如规范制定者在 rt.jar 包里面定义了数据库的驱动接口 java.sql.Driver。那么 MySQL 实现的开发商则会在 MySQL 的驱动包的 META-INF/services 文件夹下建立名称为 java.sql.Driver 的文件，文件内容就是 MySQL 对 java.sql.Driver 接口的实现类，如下图 2.5.1.1：

![在这里插入图片描述](https://images.gitbook.cn/51df3ee0-a925-11e9-9fa0-158085a98524)图 2.5.1.1

如下代码可知 com.mysql.jdbc.Driver 就是实现了 java.sql.Driver 接口：

```
public class com.mysql.jdbc.Driver extends com.mysql.jdbc.NonRegisteringDriver implements java.sql.Driver
```

上面讲解了如何使用 SPI 扩展自定义自己的实现，下面来说说 SPI 实现原理，我们知道 Java 核心 API，比如 rt.jar 包，是使用 Bootstrap ClassLoader 类加载器加载的，而用户提供的 Jar 包是由 App classloader 加载。并且我们知道如果一个类由类加载器 A 加载，那么这个类依赖的类也是由相同的类加载器加载。

而用来搜索开发商提供的 SPI 扩展实现类的 API 类（ServiceLoader）是使用 Bootstrap ClassLoader 加载的，那么 ServiceLoader 里面依赖的类应该也是由 Bootstrap ClassLoader 来加载。而上面说了用户提供的包含 SPI 实现类的 Jar 包是由 Appclassloader 加载，所以需要一种违反双亲委派模型的方法，线程上下文类加载器 ContextClassLoader 就是为了解决这个问题。

下面我们写个测试代码，看看具体是如何工作的。

```
    public static void main(String[] args) {
               //(1)
        ServiceLoader<Driver> loader = ServiceLoader.load(Driver.class);
               //(2)
        Iterator<Driver> iterator = loader.iterator();
        while (iterator.hasNext()) {
            Driver driver = (Driver) iterator.next();
            System.out.println("driver:" + driver.getClass() + ",loader:" + driver.getClass().getClassLoader());
        }
        //(3)
        System.out.println("current thread contextloader:" + Thread.currentThread().getContextClassLoader());
               //(4)
        System.out.println("ServiceLoader loader:" + ServiceLoader.class.getClassLoader());
    }

}
```

然后引入 MySQL 驱动的 Jar 包，执行结果如下。

```
driver:class com.mysql.jdbc.Driver,loader:sun.misc.Launcher$AppClassLoader@4554617c
current thread contextloader:sun.misc.Launcher$AppClassLoader@4554617c
ServiceLoader loader:null
```

从结果可知找到了 MySQL 的驱动，如果你在引入 Oracle 数据库 驱动的 Jar 包后再运行，则会输出找到了 MySQL 和 Oracle 的驱动，这也说明了，JDK 标准的 SPI 会同时把 SPI 接口的所有实现类都提前加载好实例:

```java
driver:class com.mysql.cj.jdbc.Driver,loader:sun.misc.Launcher$AppClassLoader@4e25154f
driver:class oracle.jdbc.OracleDriver,loader:sun.misc.Launcher$AppClassLoader@4e25154f
current thread contextloader:sun.misc.Launcher$AppClassLoader@4e25154f
ServiceLoader loader:null
```

另外从执行结果可以知道 ServiceLoader 的加载器为 Bootstarp，因为这里输出了 null，并且从该类在 rt.jar 里面，也可以证明。

下面我们来看下 ServiceLoader 的 load 方法源码。

```
public final class ServiceLoader<S> implements Iterable<S> {
    public static <S> ServiceLoader<S> load(Class<S> service) {
        // （5）获取当前线程上下文加载器
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }

    public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader) {
        return new ServiceLoader<>(service, loader);
    }
       //(6)
    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        service = svc;
        loader = cl;
        reload();
    }
```

代码（5）获取了当前线程上下文加载器，这里是 AppClassLoader。

代码（6）传递该类加载器到新构造的 ServiceLoader 的成员变量 loader。那么这个 loader 什么时候使用的呢？下面我们看下 LazyIterator 的 next() 方法。

```
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
private S nextService() {
          ...
            String cn = nextName;//hasNext 中设置的 nextName
            try {
              //（7）使用 loader 类加载器加载
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
              ...
            }
           ...
            try {//(8)根据 Class 对象 c 创建对象实例
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
               ....
            }
              ...
        }
```

代码（7）使用 loader 也就是 AppClassLoader 加载具体的驱动实现类的 Class 对象，代码（8）则使用 Class 对象调用 newInstance()方法创建对象实例。至于 cn 是怎么来的，读者可以参见 LazyIterator 的 hasNext() 方法：

```Java
public boolean hasNext() {
   ...
   return hasNextService();
   ...
}

private boolean hasNextService() {
    //扩展实现存在
    if (nextName != null) {
        return true;
    }
    //从 jar 的 META-INF/services/下查找接口名称的文件并保存到 configs
    if (configs == null) {
        try {
            //META-INF/services/service 接口名称
            String fullName = PREFIX + service.getName();
            ...
            configs = loader.getResources(fullName);
        } catch (IOException x) {
        }
    }
    //遍历每个文件
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        pending = parse(service, configs.nextElement());
    }
    //文件内容，比如 com.mysql.cj.jdbc.Driver
    nextName = pending.next();
    return true;
}
```

##### **Dubbo 增强 SPI 原理**

Dubbo 的扩展点加载机制是基于 JDK 标准的 SPI 扩展点机制增强而来的，Dubbo 解决了 JDK 标准的 SPI 的以下问题：

- JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但没用上，则加载会很浪费资源。
- 如果扩展点加载失败，就失败了，不会友好的给用户通知具体异常。比如：JDK 标准的 ScriptEngine，如果 Ruby ScriptEngine 因为所依赖的 jruby.jar 不存在，导致 Ruby ScriptEngine 类加载失败，这个失败原因被吃掉了，当用户执行 ruby 脚本时，会报空指针异常，而不是报 Ruby ScriptEngine 不存在。
- 增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点，也可以对扩展点使用 wrapper 类进行功能增强。

本节我们结合服务提供者配置类 ServiceConfig 来讲解如何使用增强 SPI 加载扩展接口 Protocol 的实现类，在 ServiceConfig 类中，有如下代码：

```
public class ServiceConfig<T> extends AbstractServiceConfig {
...
    private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
...
}
```

这里 ExtensionLoader 类似 JDK 标准 SPI 里面的 ServiceLoader 类，代码 ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension() 的作用是获取 Protocol 接口的适配器类，在 Dubbo 中每个扩展接口都有一个对应的适配器类，如前面所述这个适配器类是动态生成的一个类，这里我们给出 Protocol 扩展接口对应的适配器类的代码，如下：

```
package org.apache.dubbo.rpc;
...
public class Protocol$Adaptive implements Protocol {   
 ...
public Exporter export(Invoker invoker) throws RpcException {
    String string;
    ...
    //(1)
    URL uRL = invoker.getUrl();
    String string2 = string = uRL.getProtocol() == null ? "dubbo" : uRL.getProtocol();
    if (string == null) {
        throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (").append(uRL.toString()).append(") use keys([protocol])").toString());
    }
    //(2)
    Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(string);
    //(3)
    return protocol.export(invoker);
}
```

所以当我们调用 protocol.export(invoker) 方法的时候实际调用的是动态生成的 Protocol$Adaptive 实例的 export(invoker) 方法，其内部代码（1）首先获取参数里面的 URL 对象，然后从 URL 对象里面获取用户设置的 Protocol 的实现类的名称，然后调用代码（2）根据名称获取具体的 Protocol 协议的实现类（后面我们会知道获取的是实现类被使用 Wrapper 类增强后的类），最后代码（3）具体调用 Protocol 协议的实现类的 export(invoker) 方法。

下面我们结合下面的时序图 2.5.2.1 来讲解 ExtensionLoader 的 getAdaptiveExtension() 方法是如何动态生成扩展接口对应的适配器类，以及 getExtension 方法如何根据扩展实现类的名称找到对应的扩展实现类的：

![在这里插入图片描述](https://images.gitbook.cn/5a427660-a925-11e9-9fa0-158085a98524)图 2.5.2.1

- 时序图 2.5.2.1 的步骤（1）获取当前扩展接口对应的 ExtensionLoader 对象，在 Dubbo 中每个扩展接口对应着自己的 ExtensionLoader 对象，如下代码，内部通过并发 Map 来缓存扩展接口与对应的 ExtensionLoader 的映射，其中 key 为扩展接口的 Class 对象，value 为对应的 ExtensionLoader 实例：

```
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null)
            throw new IllegalArgumentException("Extension type == null");
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type +
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }

        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
```

可知第一次访问某个扩展接口时候需要 new 一个对应的 ExtensionLoader 放入缓存，后面就直接从缓存获取。

- 步骤（2）获取当前扩展接口对应的适配器对象，getAdaptiveExtension 的代码如下：

```
 @SuppressWarnings("unchecked")
    public T getAdaptiveExtension() {
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            instance = createAdaptiveExtension();
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }
```

如上代码使用双重检查创建 cachedAdaptiveInstance 对象，接口对应的适配器对象就保存到了这个对象里面。

- 我们重点看步骤（3）createAdaptiveExtension 方法，因为具体创建适配器对象的是这个方法。createAdaptiveExtension 代码如下：

```
    private T createAdaptiveExtension() {
        try {
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
```

可知首先调用了步骤（4）getAdaptiveExtensionClass().newInstance() 获取适配器对象的一个实例，然后调用步骤（7）injectExtension 方法进行扩展点相互依赖注入（扩展点之间依赖自动注入）。下面首先看下步骤（4）getAdaptiveExtensionClass() 是如何动态生成适配器类的 Class 对象的。

```
    private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
```

如上代码首先调用了步骤（5）getExtensionClasses 获取了该扩展接口所有实现类的 Class 对象，然后调用了步骤（6）createAdaptiveExtensionClass 创建具体的适配器对象的 Class 对象，前面我们讲解过 createAdaptiveExtensionClass，该方法根据字符串代码生成适配器的 Class 对象并返回，然后通过 getAdaptiveExtensionClass().newInstance() 创建适配器类的一个对象实例。至此扩展接口的适配器对象已经创建完毕。

- 下面我们在看步骤（7）前面看看步骤（5）getExtensionClasses 如何加载扩展接口的所有实现类的 Class 对象。其内部最终调用了 loadExtensionClasses 方法进行加载，loadExtensionClasses 代码如下：

```
   private Map<String, Class<?>> loadExtensionClasses() {
    //获取默认扩展名
    cacheDefaultExtensionName();

    //在指定目录的 jar 里面查找扩展点
    Map<String, Class<?>> extensionClasses = new HashMap<>();
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());//META-INF/dubbo/internal/
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());//META-INF/dubbo/
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());//META-INF/services/
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    return extensionClasses;
}

private void cacheDefaultExtensionName() {
    //获取扩展接口上 SPI 注解
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    //是否存在注解
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                throw new IllegalStateException("More than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            //默认实现类的名称放到 cachedDefaultName
            if (names.length == 1) {
                cachedDefaultName = names[0];
            }
        }
    }
}
```

如上代码，拿 Protocol 协议来说，这里 SPI 注解为 `@SPI("dubbo")`，那么这里 cachedDefaultName 就是 dubbo。然后 loadDirectory 方法去 `META-INF/dubbo/internal/`、`META-INF/dubbo/`、`META-INF/services/` 目录下去加载具体的扩展实现类，比如 Protocol 协议默认实现类如下图 2.5.2.2： ![在这里插入图片描述](https://images.gitbook.cn/6249ab30-a925-11e9-80b9-071d990090b2) 图 2.5.2.2

- 步骤（7）injectExtension 方法进行扩展点实现类相互依赖自动注入（IOC 功能）：

```java
private T injectExtension(T instance) {
    try {

        if (objectFactory != null) {
            //遍历扩展点实现类所有的方法
            for (Method method : instance.getClass().getMethods()) {
                if (isSetter(method)) {
                    /**
                     * 如果方法含有 DisableInject 注解，说明该属性就不需要自动注入
                     */
                    if (method.getAnnotation(DisableInject.class) != null) {
                        continue;
                    }
                    //第一个参数是原始类型，则不需要自动注入  
                    Class<?> pt = method.getParameterTypes()[0];
                    if (ReflectUtils.isPrimitives(pt)) {
                        continue;
                    }
                    //看 set 方法设置的变量是不是有扩展接口实现
                    try {
                        //获取 setter 器对应的属性名，比如 setVersion, 则返回 "version"
                        String property = getSetterProperty(method);
                        //看该属性类似是否存在扩展实现
                        Object object = objectFactory.getExtension(pt, property);
                        //如果存在则反射调用 setter 方法进行属性注入
                        if (object != null) {
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("Failed to inject via method " + method.getName()
                                + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}

//获取 setter 器对应的属性名，比如 setVersion, 则返回 "version"
private String getSetterProperty(Method method) {
    return method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
}
```

至此 ExtensionLoader 的 getAdaptiveExtension() 方法是如何动态生成扩展接口对应的适配器类的以及如何加载扩展接口的实现类的 Class 对象的已经讲解完毕，下面我们看 getExtension 方法如何根据扩展实现类的名称找到对应的实现类的：

```Java
public T getExtension(String name) {
    //扩展实现名称合法性校验
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    //如果为 true 则加载默认扩展
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    //根据 name 获取实例
    Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                //不存则创建
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

其中 createExtension 代码如下：

```Java
 private T createExtension(String name) {
    //根据 name 查找对应的扩展实现的 Class 对象
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    //实例缓存里面不存在则，使用 Class 创建实例
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        //ioc 依赖注入  
        injectExtension(instance);
        //wraper 对扩展实现进行功能增强
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        ...
    }
}
```

至此如何使用 getExtension(String name) 加载具体的扩展实现类也讲完了。

##### **Dubbo 增强 SPI 原理-扩展点的自动包装**

在 Spring AOP 中我们可以使用多个切面对指定类的方法进行增强，在 Dubbo 中也提供了类似的功能，在 Dubbo 中你可以指定多个 Wrapper 类对指定的扩展点的实现类的方法进行增强。

如下代码当执行`protocol.export(wrapperInvoker)`时候，实际调用的是适配器 Protocol$Adaptive 的 export 方法，如果 URL 对象里面的 protocol 为 dubbo，那么在没有扩展点自动包装时，protocol.export 返回的就是 DubboProtocol 的对象。

```Java
    private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

    Exporter<?> exporter = protocol.export(wrapperInvoker);
```

而真正情况下 Dubbo 里面使用了 ProtocolFilterWrapper、ProtocolListenerWrapper 等 Wrapper 类对 DubboProtocol 对象进行了包装增强。

ProtocolFilterWrapper、ProtocolListenerWrapper、DubboProtocol 三个类都有一个拷贝构造函数，这个拷贝构造函数的参数就是扩展接口 Protocol，所谓包装是指下面意思：

```Java
public class XxxProtocolWrapper implemenets Protocol {
  private Protocol impl;
  public XxxProtocol(Protocol protocol) { 
       impl = protocol; 
  }

 public void export() {
 //... 在调用 DubboProtocol 的 export 前做些事情
   impl.export();
 //... 在调用 DubboProtocol 的 export 后做些事情
 }
 ...
}
```

比如这里会进行两次包装，第一次可能首先使用 ProtocolListenerWrapper 类对 DubboProtocol 进行包装，这时候 ProtocolListenerWrapper 类里面的 impl 就是 DubboProtocol，然后第二次使用 ProtocolFilterWrapper 对 ProtocolListenerWrapper 进行包装，也就是 ProtocolFilterWrapper 里面的 impl 是 ProtocolListenerWrapper，那么这时候调用适配器 Protocol$Adaptive 的 export 方法，如果 URL 对象里面的 protocol 为 dubbo，那么在有扩展点自动包装时候，这时候 protocol.export 返回的 就是 ProtocolFilterWrapper 的实例了。

下面我们看下 Dubbo 增强的 SPI 中如何去收集的这些包装类，以及如何实现的使用包装类对 SPI 实现类的自动包装(AOP 功能)。

其实上节 getExtensionClasses 里面的 loadDirectory 方法除了加载扩展接口的所有实现类的 Class 对象外还对包装类（wrapper 类）进行了收集，我们看 loadDirectory->loadResource->loadClass 代码如下：

```Java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    //类型不匹配则抛出异常
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + " is not subtype of interface.");
    }
    //如果是适配器类，则调用 cacheAdaptiveClass 来做缓存
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        cacheAdaptiveClass(clazz);
    //如果是 Wrapper 类，则调用 cacheWrapperClass 来做缓存
    } else if (isWrapperClass(clazz)) {
        cacheWrapperClass(clazz);
    } else {
        //剩下的则是扩展点的扩展实现类
        clazz.getConstructor();
        if (StringUtils.isEmpty(name)) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }

        String[] names = NAME_SEPARATOR.split(name);
        if (ArrayUtils.isNotEmpty(names)) {
            cacheActivateClass(clazz, names[0]);
            for (String n : names) {
                cacheName(clazz, n);
                saveInExtensionClass(extensionClasses, clazz, name);
            }
        }
    }
}

//如果 clazz 的构造函数参数是 type 类型，做说明是 wrapper 类
private boolean isWrapperClass(Class<?> clazz) {
    try {
        clazz.getConstructor(type);
        return true;
    } catch (NoSuchMethodException e) {
        return false;
    }
}
```

如上代码使用 isWrapperClass 方法判断 clazz 是否为 wrapper 类，其中调用 clazz.getConstructor(type)，这里是判断 SPI 实现类 clazz 是否有扩展接口 type 为参数的拷贝构造函数，如果没有直接抛异常 NoSuchMethodException，该异常被 catch 掉了然后返回 false，如果有则说明 clazz 类为 wrapper 类，则返回 true,并调用 cacheWrapperClass 方法收集起来放入到 cachedWrapperClasses 集合，到这里 wrapper 类的收集已经完毕。

而具体对扩展实现类使用收集的 wrapper 类进行自动包装是在 createExtension 方法里做的：

```Java
private T createExtension(String name) {
    ...
    try {

         //cachedWrapperClasses 里面有元素，即为 wrapper 类
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            //使用循环一层层对包装类进行包装，可以参考上面讲解的使用 
           // ProtocolFilterWrapper、ProtocolListenerWrapper 对 DubboProtocol 进行包装的流程
           for (Class<?> wrapperClass : wrapperClasses) {
              instance = injectExtension((T) 
              wrapperClass.getConstructor(type).newInstance(instance));
           }
         }
        return instance;
    } catch (Throwable t) {
        ...
    }
}
```

如上代码遍历所有 wrapper 类，使用 injectExtension 一层层对扩展实现类进行功能增强。

#### Dubbo 使用 JavaAssist 减少反射调用开销

Dubbo 会给每个服务提供者的实现类生产一个 Wrapper 类，这个 wrapper 类里面最终调用服务提供者的接口实现类，wrapper 类的存在是为了减少反射的调用。当服务提供方接受到消费方发来的请求后需要根据消费者传递过来的方法名和参数反射调用服务提供者的实现类，而反射本身是有性能开销的，所以 dubbo 把每个服务提供者的实现类通过 JavaAssist 包装为一个 Wrapper 类，那么 Wrapper 类为何能减少反射调用那？

假设我们的服务提供方实现类为 GreetingServiceImpl， 其代码如下：

```java
public class GreetingServiceImpl implements GreetingService {

    @Override
    public String sayHello(String name) {
        return "Hello " + name + " " + RpcContext.getContext().getAttachment("company");
    }

    @Override
    public Result<String> testGeneric(PoJo poJo) {
      ...
        return result;
    }
} 
```

那么其对应的 wrapper 类的源码如下：

```Java
public class Wrapper1 extends Wrapper implements ClassGenerator.DC {
...
    public Object invokeMethod(Object object, String string, Class[] arrclass, Object[] arrobject) throws InvocationTargetException {
        GreetingServiceImpl greetingServiceImpl;
        try {
            greetingServiceImpl = (GreetingServiceImpl)object;
        }
        catch (Throwable throwable) {
            throw new IllegalArgumentException(throwable);
        }
        try {
            if ("sayHello".equals(string) && arrclass.length == 1) {
                return greetingServiceImpl.sayHello((String)arrobject[0]);
            }
            if ("testGeneric".equals(string) && arrclass.length == 1) {
                return greetingServiceImpl.testGeneric((PoJo)arrobject[0]);
            }
        }
        catch (Throwable throwable) {
            throw new InvocationTargetException(throwable);
        }
        throw new NoSuchMethodException(new StringBuffer().append("Not found method \"").append(string).append("\" in class com.books.dubbo.demo.provider.GreetingServiceImpl.").toString());
    }
...
}
```

如上代码可知 Wrapper1 的 invokeMethod 最终是直接调用的 GreetingServiceImpl 的具体方法，这避免了反射开销，而 wrapper1 类的生成是在 dubbo 服务启动时候，所以不会对运行时带来开销。

下面我们看看 Dubbo 哪里生成的 Wrapper 类，在 Dubbo 分层架构概述中我们讲了 Proxy 层的 SPI 扩展接口为 ProxyFactory，Dubbo 提供的实现主要有 JavassistProxyFactory（默认使用）和 JdkProxyFactory，其实就是 JavassistProxyFactory 为每个服务提供者实现类生成了 wrapper 类：

```Java
public class JavassistProxyFactory extends AbstractProxyFactory {
...
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // 生成 Wrapper 类
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName, 
                                      Class<?>[] parameterTypes, 
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }

}
```

### 参考

- [dubbo 用户手册](http://dubbo.apache.org/en-us/docs/user/preface/background.html)
- [dubbo 开发手册](http://dubbo.apache.org/en-us/docs/user/preface/background.html)
- https://github.com/apache/incubator-dubbo