# Dubbo

## TOC

[TOC]

## 项目架构演变过程

### 单体架构

MVC结构，SSM框架，所有业务都在一个tomcat里面

![image-20200521102129024](https://gitee.com/jinxin.70/oss/raw/master/uPic/image-20200521102129024_2020_05_21_10_21_29.png)

优点：

- 小项目开发快，成本低
- 架构简单
- 易于测试
- 易于部署

缺点：

- 大项目模块耦合严重，不易开发，维护，沟通成本高
- 新增业务困难
- 核心业务与边缘业务混合在一块，出现问题互相影响
- 技术选型单一，不利于新技术的使用和探索，不利于个人发展，技术拓展不友好

### 垂直架构

基于业务划分模块，分布式部署

![image-20200521102152777](https://gitee.com/jinxin.70/oss/raw/master/uPic/image-20200521102152777_2020_05_21_10_21_53.png)



优点：

- 系统拆分实现了流量分担，解决了并发问题
- 可以针对不同系统进行优化
- 方便水平扩展，负载均衡，容错率提高
- 系统间相互独立，互不影响，新的业务迭代时更加高效

缺点：

- 服务系统之间接口调用硬编码
- 搭建集群之后，实现负载均衡比较复杂
- 服务系统接口调用监控不到位，调用方式不统一
- 服务监控不到位
- 数据库资源浪费，充斥慢查询，主从同步延迟大

### 分布式架构（SOA）

将每个项目拆分出多个具备松耦合的服务，一个服务通常以独立的形式存在于操作系统进程中。

为了解决接口协议不统一、服务无法监控、服务的负载均衡等问题，我们将通用业务逻辑下沉到服务层，通过接口暴露，供其他业务场景调用。同时引入Dubbo，它提供了三大核心功能：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。

![image-20200521183908230](https://gitee.com/jinxin.70/oss/raw/master/uPic/image-20200521183908230_2020_05_21_18_39_10.png)

分层：按照业务性质分层，每一层要求简单和容易维护

- 应用层
- 业务服务层
- 基础业务层
- 基础服务层
- 存储层

分级：同一层的业务，依据业务的重要性进行分级，按照二八定律，网站80%的流量都在核心功能上面，要优先保证核心业务的稳定。

隔离：业务、缓存、DB、中间件要做好隔离，比如核心业务的数据库要和活动相关的数据库隔离

调用：总体上调用要单向，可以跨层调用，但不能逆向调用



优点：

- 服务以接口为粒度，为开发者屏蔽远程调用底层细节，使用Dubbo面向接口远程方法调用屏蔽了底层调用细节
- 业务分层以后架构更清晰，并且每个业务模块职责单一，扩展性更强
- 数据隔离，权限回收，数据访问都通过接口，让系统更加稳定安全。
- 服务应用本身无状态化，应用本身不做内存及缓存，而是把数据出入DB
- 服务责任易确定，每个服务可以确定责任人，这样更容易保证服务质量和稳定

确定：

- 粒度控制复杂，模块越来越多，会引发超时，分布式事务等问题
- 服务接口数量不宜控制
- 版本升级兼容困难，尽量不要删除方法，字段，枚举类型的新增字段也可能不兼容
- 调用链长，服务质量不可监控，调用链路变长，下游抖动可能会影响到上游业务，最终形成连锁反应，服务质量不稳定，同时链路的变长使得服务质量的监控变得困难。

### 微服务架构

业务需要彻底的组件化和服务化。

将单个应用程序作为一套小型服务开发。

每个应用程序都在其自己的进程中独立运行，

并使用轻量级通信机制(通常是HTTP资源的API)进行通信。

围绕业务功能构建。

可以通过全自动部署机制进行独立部署。

这些服务的集中化管理非常少，可以用不同语言编写，并使用不同的数据存储技术。

## Dubbo架构与实战

### Dubbo架构描述

#### 什么是Dubbo

RPC框架

#### 特性

#### Dubbo的服务治理



### Dubbo的处理流程

![image-20200521185503176](https://gitee.com/jinxin.70/oss/raw/master/uPic/image-20200521185503176_2020_05_21_18_55_03.png)

### 服务注册中心Zookeeper

### Dubbo开发实战

#### 实战案例介绍



#### 开发过程

1.接口协定



2.创建接口提供者



3.创建消费者



#### 配置方式介绍



注解



XML



基于代码



#### XML方式

### Dubbo管理控制台



### Dubbo配置项说明

#### dubbo:application

#### dubbo:registry

#### dubbo:protocol

#### dubbo:service

#### dubbo:method

#### dubbo:service和dubbo:reference详解

一些必然会接触到的属性

mock:当作服务降级来统一对外返回结果

timeout:指定当前方法或者接口中所有方法的超时时间，对第三方服务依赖要放宽时长，防止第三方服务不稳定导致服务受损

check:用于在启动时，检查生产者是否有该服务。一般不开启检查，设为false，因为如果模块间存在循环引用，都check，会导致服务起不来。

retries:服务执行错误或超时时的重试机制

- 注意提供者是否幂等，否则可能出现数据不一致问题
- 注意提供者是否有类似缓存机制，如出现大面积错误时，可能因不停重试导致雪崩

executes:配置提供者最大并行度。

- 可能导致集群功能无法充分利用或者阻塞
- 但是也可以起到部分应用的保护功能
- 可以不做配置，结合后面的熔断限流使用

其他配置参考官网

## Dubbo高级实战

### 

### SPI

#### JDK中的SPI

SPI（Service Provider Interface），是JDK内置的一种服务提供发现机制。

![image-20200525204007491](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200525204007491.png)

1、约定一个接口，在``META-INF/services``目录下创建一个以“接口全限定名”为命名的文件，内容为实现类的全限定名。

2、接口实现类所在的jar包放在主程序的classpath中

3、主程序通过java.util.ServiceLoader动态装载实现模块，它通过扫描``META-INF/services``目录下的配置文件找到实现类的全限定名，把类加载到JVM

4、SPI的实现类，必须携带一个无参构造方法。

接口

```java
package com.lagou.service;

public interface HelloService {
    String  sayHello();
}
```

实现类

```java
package com.lagou.service.impl;

import com.lagou.service.HelloService;

public class DogHelloService  implements HelloService {
    @Override
    public String sayHello() {
        return "wang wang";
    }
}
```

```java
package com.lagou.service.impl;

import com.lagou.service.HelloService;

public class HumanHelloService   implements HelloService {
    @Override
    public String sayHello() {
        return "hello 你好";
    }
}
```

``META-INF/services/com.lagou.service.HelloService``

```
com.lagou.service.impl.DogHelloService
com.lagou.service.impl.HumanHelloService
```

使用：将实现类所在项目打包成jar，将接口所在项目达成jar。在使用的项目中引入两个jar

```java
package com.lagou.test;

import com.lagou.service.HelloService;

import java.util.ServiceLoader;

public class JavaSpiMain {
    public static void main(String[] args) {
        final ServiceLoader<HelloService> helloServices  = ServiceLoader.load(HelloService.class);
        for (HelloService helloService : helloServices){
            System.out.println(helloService.getClass().getName() + ":" + helloService.sayHello());
        }
    }
}
```

#### Dubbo中的SPI



### 负载均衡策略

### 异步调用

### 线程池

### 路由规则

### 服务动态降级



## Dubbo源码剖析