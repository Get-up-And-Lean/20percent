# Spring Cloud微服务学习笔记

SOA->Dubbo

微服务架构->Spring Cloud提供了一个一站式的微服务解决方案

## 第一部分 微服务架构

### 1 互联网应用架构发展

那些迫使系统演进的因素：

业务量上去了后，负载能力满足不了，对高负载能力的需求，以及高性能高可用、自动化运维管理、等的需求。

1、单体应用架构

优点：

缺点：

2、垂直应用架构

优点：

缺点：

3、SOA应用架构

优点：

缺点

4、微服务架构

### 2 微服务架构体现的思想及优缺点

优点：

- 微服务很小，便于特定业务功能的聚焦
- 微服务很小，每个微服务都可以被一个团队单独实施（开发、测试、部署上线、运维），团队合作一定程度解耦，便于实施敏捷开发
- 微服务很小，便于重用和模块之间的组装
- 微服务很独立，那么不同的微服务可以使用不同的语言开发，松耦合
- 更容易引入新技术
- 可以更好的实现DevOps开发运维一体化

缺点：

- 服务数量越多越难管理
- 链路难跟踪



### 3 微服务架构中的一些概念

#### 服务注册与发现

服务注册：服务提供者将所提供的服务信息注册/登记到注册中心

服务发现：服务消费者能够从注册中心获取到较为实时的服务列表，然后根据一定的策略选择一个服务访问

#### 负载均衡

将请求压力分配到多个服务器（应用服务器、数据库服务器等），以此来提高服务等性能、可靠性

#### 熔断

即断路保护。如果下游服务因访问压力过大而响应变慢或失败，上游服务为了保护系统整体可用性，可以暂时切断对下游服务等调用。这种牺牲局部，保全整体的措施就叫做熔断。

熔断的本质是**为了保护服务的整体可用性**。

#### 链路追踪

对一次请求涉及的很多个服务链路进行日志记录、性能监控

#### API网关

如果没有网关，客户端直接与各个微服务通信的问题：

1、客户端调用不同的url地址，增加了维护调用难度

2、在一定的场景下，也存在跨域请求的问题（可以使用Nginx做反向代理服务器）

3、每个微服务都需要进行单独的身份认证

网关除了转发请求，更专注安全、路由、流量问题的处理，它的功能有：

1、统一接入（路由）

2、安全防护（统一鉴权，负责网关访问身份验证，与“访问认证中心”通信，实际认证业务逻辑交移“访问认证中心”处理）

3、黑白名单（实现通过IP地址控制禁止访问网关功能，控制访问）

4、协议适配（实现通信协议校验、适配转换的功能）

5、流量管控（限流）

6、长短链接支持

7、容错能力（负载均衡）

## 第二部分 Spring Cloud概述

### Spring Cloud是什么

Spring cloud是一套规范，间commons下的接口

实现有SCN和SCA

### Spring Cloud解决什么问题

解决微服务架构实施过程中存在的问题，比如服务注册发现、网络问题（熔断场景）、统一认证安全授权、负载均衡问题、链路追踪问题等。

### Spring Cloud架构

#### Spring Cloud组件

|                | 第一代SCN                  | 第二代SCA                           |
| -------------- | -------------------------- | ----------------------------------- |
| 注册中心       | Eureka                     | Nacos                               |
| 客户端负载均衡 | Ribbon                     | Dubbo LB、Spring Cloud Loadbalancer |
| 熔断器         | Hystrix                    | Sentinel                            |
| 网关           | Zuul                       | Spring Cloud Gateway                |
| 配置中心       | Spring Cloud Config        | Nacos、Apollo                       |
| 服务调用       | Feign                      | Dubbo RPC                           |
| 消息驱动       | Spring Cloud Stream        |                                     |
| 链路追踪       | Spring Cloud Sleuth/Zipkin |                                     |
|                |                            | Seata分布式事务解决方案             |

#### Spring Cloud 体系结构（组件协同工作机制）

![image-20200620212327068](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200620212327068.png)

- 注册中心负责服务的注册和发现
- 网关负责转发所有外来的请求
- 断路器负责监控服务之间的调用情况，连续多次失败进行熔断保护
- 配置中心提供了统一的配置信息管理服务，可以实时的通知各个服务获取最新的配置信息

#### Spring Cloud与Dubbo对比

Dubbo定位于高性能RPC框架，不是一站式的微服务解决方案

#### Spring Cloud与Spring Boot的关系

Spring Boot是实现Spring Cloud的基础，提供依赖版本管理、自动配置、快速启动

## 第三部分 案例准备

### 案例说明

### 数据库环境准备

### 工程环境准备

### 使用Maven聚合工程

### 案例核心微服务开发及通信调用

### 案例代码问题分析

存在的问题：

- 在服务消费者中，我们把url地址硬编码到代码中，不方便后期维护
- 服务提供者只有一个服务，即便服务提供者形成集群，服务消费者还需要自己实现负载均衡
- 在服务消费者中，不清楚服务提供者的状态
- 服务消费者调用服务提供者的时候，如果出现故障能否及时发现不向用户抛出异常页面
- RestTemplate这种请求调用方式是否还要优化空间？能不能类似于Dubbo那样玩
- 这么多微服务统一认证如何实现
- 配置文件每次都修改好多个很麻烦

微服务架构面临的共有的问题：

- 服务管理：自动注册与发现、状态监管
- 服务负载均衡
- 熔断
- 远程过程调用
- 网关拦截、路由转发
- 统一认证
- 集中式配置管理，配置信息实时自动更新



## 第四部分 第一代Spring Cloud核心组件

![image-20200620214721693](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200620214721693.png)

从形式上说，Feign一个顶三，Feign=RestTemplate+Ribbon+Hystrix

### Eureka注册中心

#### 关于服务注册中心

**解耦服务提供者和服务调用者**

服务注册中心一般原理

![image-20200620214915243](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200620214915243.png)

#### 注册中心做哪些事



#### 主流服务注册中心对比



### 服务注册中心Eureka

#### Eureka基础架构

![image-20200620215145893](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200620215145893.png)

#### Eureka交互流程及原理

![image-20200620215217312](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200620215217312.png)

### Eureka应用及高可用集群



### Eureka细节详解

#### Eureka元数据



#### Eureka客户端详解



#### Eureka服务端详解



### Eureka核心源码剖析



## 第五部分 常见问题及解决方案





## 第六部分 Spring Cloud高级进阶

### Turbine聚合监控

参照Hustrix部分

### 微服务监控之分布式链路追踪技术Sleuth+Zipkin

#### 分布式链路追踪技术适用场景（问题场景）

场景描述

1. 如何动态展示服务的调用链路
2. 如何分析服务调用链路中的瓶颈节点并对其进行调优？
3. 如何快速进行服务链路的故障发现

分布式链路追踪技术

如果我们在一个请求的调用处理过程中，在各个链路节点都能够记录下日志，并最终将日志进行集中可视化展示，那么我们想监控调用链路中一些指标就有希望了。

比如，请求到达哪个服务实例？请求倍处理的状态怎样？这些都能够分析出来了

分布式环境下基于这种想法实现的监控技术就是分布式链路追踪（全链路追踪）。

市场上的分布式链路追踪方案

分布式链路追踪技术方案：

- Spring Cloud Sleuth+Twitter Zipkin
- 阿里巴巴的"鹰眼"
- 大众点评的"CAT"
- 美团的"Mtrace"
- 京东的"Hydra"
- 新浪的"Watchman"
- Apache Skywalking

#### 分布式链路追踪技术的核心思想

本质：记录日志，作为一个完整的技术，分布式链路追踪技术有自己的理论和概念

微服务架构中，针对请求处理的调用链路可以展示为一棵树：

![image-20200705213247275](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200705213247275.png)

上图描述了一个常见的调用场景，一个请求通过网关服务路由到下游的微服务1，然后微服务1调用微服务2，拿到结果再调用微服务3，最后组合微服务2和微服务3的结果，通过网关返回给用户



追踪调用链路就要记录日志，Google的论文，《Dapper, a Large-Scale Distributed Systems Tracing Infrastructure》

![image-20200705213543752](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200705213543752.png)

上图标识一个请求链路，一条链路通过TraceId唯一标识，span标识发起的请求信息，各span通过parrentId关联起来。

**Trace**：服务追踪的追踪单元是从客户发起请求抵达被追踪系统的边界开始，到被追踪系统向客户返回响应为止的过程

**Trace ID**：为了实现请求追踪，当请求发送到分布式系统的入口端点时，只需要服务跟踪框架为该请求创建一个唯一的跟踪标识Trace ID，同时在分布式系统内部流转的时候，框架始终保持该唯一标识，直到返回给请求方

一个Trace由一个或多个Span组成，每一个Span都有一个SpanId，Span中会记录TraceId，同时还有一个叫做parentId，指向了另一个Span的SpanId，表明父子关系，其实本质表达了依赖关系。

**Span ID**：为了统计各处理单元的时间延迟，当请求达到各个组件时，也是通过一个唯一标识Span ID来标记他的开始，具体过程以及结束。对每个Span来说，它必须有开始和结束两个节点，通过记录开始Span和结束Span时间戳，就能统计该Span的时间延迟，每个Span还包含了时间名称、请求信息等元数据。

每个Span都会有一个唯一跟踪标识SpanId，若干个有序的span就组成一个trace。

Span可以认为是一个日志数据结构，在一些特殊时机点记录了一些日志信息，比如有时间戳、spanid、traceId、parentId等。Span中也抽象出另外一个概念，叫做事件，核心事件如下：

- CS：client send/start 客户端/消费者发出一个请求，描述的是一个span开始
- SR：server received/start 服务端/生产者接收请求SR-CS属于请求发送的网络延迟
- SS：server send/finish 服务端/生产者发送应答SS-SR属于服务端消耗时间
- CR：client received/finished 客户端/消费者接受应答CR-SS表示回复需要的时间（响应的网络延迟）

Spring Cloud Sleuth（追踪服务框架）可以追踪服务之间的调用，Sleuth可以记录一个服务请求经过哪些服务、服务处理时长，根据这些信息，我们能够理清各微服务间的调用关系及进行问题追踪分析

- 耗时分析：通过Sleuth了解采样请求的耗时，分析服务性能问题（哪些服务调用比较耗时）
- 链路优化：发现频繁调用的服务，针对性优化等

Sleuth就是通过记录日志的方式来记录追踪链路的

注意：我们往往把Spring Cloud Sleuth和Zipkin一起使用，把Sleuth的数据信息发送给Zipkin进行聚合，利用Zipkin存储并展示数据。

![image-20200705215456677](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200705215456677.png)

![image-20200705215633659](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200705215633659.png)

#### Sleuth + Zipkin

1、每一个需要被追踪踪迹的微服务工程都需要引入依赖坐标

```xml
<!--链路追踪-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency>
```

2、每一个微服务都修改application.yml配置文件，添加日志级别

```yaml
#分布式链路追踪
logging:
  level:
    org.springframework.web.servlet.DispatcherServlet: debug
    org.springframework.cloud.sleuth: debug
```

请求到来时，我们可以看到控制台Sleuth输出的日志（全局TraceId、SpanId等）

![image-20200705220510153](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200705220510153.png)

这样的日志不易观察，另外日志分散在各个微服务服务器上，接下来我们使用Zipkin统一聚合轨迹日志并进行存储展示

3、结合Zipkin展示追踪数据

##### Zipkin server构建

pom.xml

```xml
<!--zipkin-server的依赖坐标-->
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-server</artifactId>
            <version>2.12.3</version>
            <exclusions>
                <!--排除掉log4j2的传递依赖，避免和springboot依赖的日志组件冲突-->
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-log4j2</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!--zipkin-server ui界面依赖坐标-->
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-autoconfigure-ui</artifactId>
            <version>2.12.3</version>
        </dependency>
```

入口启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;
import zipkin2.server.internal.EnableZipkinServer;

import javax.sql.DataSource;

@SpringBootApplication
@EnableZipkinServer // 开启Zipkin 服务器功能
public class ZipkinServerApplication9411 {

    public static void main(String[] args) {
        SpringApplication.run(ZipkinServerApplication9411.class,args);
    }

}
```

application.yml

```yaml
server:
  port: 9411
management:
  metrics:
    web:
      server:
        auto-time-requests: false # 关闭自动检测
```

##### Zipkin Client构建（在具体微服务中修改）

pom.xml文件中添加zipkin依赖

```xml
				<!--链路追踪-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency>
				<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
```

application.yml中添加对zipkin server的引用

```yaml
spring:
  application:
    name: lagou-service-autodeliver
  zipkin:
    base-url: http://127.0.0.1:9411 # zipkin server的请求地址
    sender:
      # web 客户端将踪迹日志数据通过网络请求的方式传送到服务端，另外还有配置
      # kafka/rabbit 客户端将踪迹日志数据传递到mq进行中转
      type: web
    sleuth:
      sampler:
        # 采样率 1 代表100%全部采集 ，默认0.1 代表10% 的请求踪迹数据会被采集
        # 生产环境下，请求量非常大，没有必要所有请求的踪迹数据都采集分析，对于网络包括server端压力都是比较大的，可以配置采样率采集一定比例的请求的踪迹数据进行分析即可
        probability: 1
```

对于log日志，依然保持开启debug状态，在各个微服务中配置开启debug日志

```yaml
logging:
  level:
    # Feign日志只会对日志级别为debug的做出响应
    com.lagou.edu.controller.service.ResumeServiceFeignClient: debug
    org.springframework.web.servlet.DispatcherServlet: debug
    org.springframework.cloud.sleuth: debug
```

##### Zipkin server页面解读

会方便我们查看服务调用依赖关系及一些性能指标和异常信息

##### 追踪数据Zipkin持久化到mysql

mysql中创建名为zipkin的数据库，并执行官方提供的sql语句

zipkin server的pom文件引入依赖

```xml
<!--zipkin针对mysql持久化的依赖-->
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-autoconfigure-storage-mysql</artifactId>
            <version>2.12.3</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.10</version>
        </dependency>
        <!--操作数据库需要事务控制-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-tx</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
        </dependency>
```

修改配置文件，添加如下内容

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/zipkin?useUnicode=true&characterEncoding=utf-8&useSSL=false&allowMultiQueries=true
    username: root
    password: 123456
    druid:
      initialSize: 10
      minIdle: 10
      maxActive: 30
      maxWait: 50000
# 指定zipkin持久化介质为mysql
zipkin:
  storage:
    type: mysql
```

启动类中注入事务管理器

```java
// 注入事务控制器
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
```

### 微服务统一认证方案Spring Cloud OAuth2+JWT

认证：验证用户的合法身份，比如输入用户名和密码，系统会在后台验证用户名和密码是否合法，合法的前提下，才能够进行后续的操作，访问受保护的资源。

#### 微服务架构下统一认证场景

每个服务实现一套认证逻辑不现实，需要由独立的认证服务处理系统认证请求

![image-20200705222447169](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200705222447169.png)

#### 微服务架构下统一认证思路

1、基于session的认证方式

用session存储用户信息。解决分布式场景下session不一致，要使用Session共享方案、Session黏贴等方案

session方案的缺点：基于cookie，移动端不能有效使用

2、基于token的认证方式

服务端不用存储认证数据，易维护扩展性强，客户端可以把token存在任意地方，并且可以实现web和app统一认证机制。其缺点也很明显，token由于自身包含信息，因此一般数据量较大，而且每次请求都需要传递，因此比较占带宽。另外，token的签名验签操作也会给cpu带来额外的负担

#### OAuth2开放授权协议/标准

OAuth（开放授权）是一个开放协议/标准，允许用户授权第三方应用访问他们存储在另外的服务提供者上的信息，而不需要将用户名和密码提供给第三方应用或分享他们数据的所有内容。

允许用户授权第三方应用访问他们存储在另外的服务提供者上的信息，而不需要将用户名和密码提供给第三方应用或分析他们数据的所有内容。



结合“使用QQ登录拉钩”这个场景拆分理解上述那句话

用户：我们自己

第三方应用：拉勾网

另外的服务提供者：QQ

OAuth2是OAuth协议的延续版本，但不向后兼容OAuth1，即完全废止了OAuth1。

#### OAuth2协议角色和流程

拉勾网要开发使用QQ登录这个功能的话，那么拉勾网要提前到QQ平台进行登记（否则QQ凭什么陪着拉勾网玩授权登录这件事）

1）拉勾网——登记——QQ平台

2）QQ平台会颁发一些参数给拉勾网，后续上线进行授权登录的时候（打开授权页面）需要携带这些参数

client_id:客户端id（QQ最终相当于一个认证授权服务服务器，拉勾网就相当于一个客户端了，所以会给一个客户端ID），相当于账号

secret:相当于密码

![image-20200709200309449](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200709200309449.png)

- 资源拥有者：可以理解为用户自己
- 客户端：我们想登陆的网站或应用，比如拉勾网
- 认证服务器：可以理解为微信或QQ
- 资源服务器：可以理解为微信或QQ

#### 什么情况下需要使用OAuth2

第三方授权登录的场景：比如，微信授权登录、QQ授权登录、微博授权登录、抖音授权登录、钉钉授权登录。

单点登录场景：如果项目中有很多微服务或公司的内部有很多服务，可以专门做一个认证中心（充当认证平台的角色），所有的服务都要到这个认证中心做认证，只做一次登录，就可以在多个授权范围内的服务中自由串行。

#### OAuth2的颁发Token授权方式

1）授权码

2）密码 提供用户名+密码换取token令牌

3）隐藏式

4）客户端凭证

授权码模式使用到了回调地址，是最复杂的授权方式，微博、微信、QQ等第三方登录就是这种模式。我们重点讲解接口对接中常用的密码模式，也就是第二种。

### Spring Cloud OAuth2+JWT实现

#### Spring Cloud OAuth2介绍

Spring Cloud OAuth2是Spring Cloud体系对OAuth2协议的实现，可以用来做多个微服务的统一认证（验证身份合法性）授权（验证权限）。通过向OAuth2服务发送某个类型的grant_type进行集中认证和授权，从而获得access_token（访问令牌），而这个token是受其他微服务信任的。



**注意：使用OAuth2解决问题的本质是，引入一个认证授权层，认证授权连接了资源的拥有者，在授权层里面，资源的拥有者可以给第三方应用授权去访问某些受保护的资源。**

#### Spring Cloud OAuth2构建微服务统一认证服务思路

![image-20200709224323241](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200709224323241.png)

注意：在我们统一认证的场景中，Resource Server其实就是我们的各种受保护的微服务，微服务中的各种API访问接口就是资源，发起http请求的浏览器就是Client客户端（对应为第三方应用）

#### 搭建认证服务器

新建项目lagou-cloud-oauth-server-9999

**pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>lagou-parent</artifactId>
        <groupId>com.lagou.edu</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>lagou-cloud-oauth-server-9999</artifactId>



    <dependencies>
        <!--导入Eureka Client依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>


        <!--导入spring cloud oauth2依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-oauth2</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.security.oauth.boot</groupId>
                    <artifactId>spring-security-oauth2-autoconfigure</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.security.oauth.boot</groupId>
            <artifactId>spring-security-oauth2-autoconfigure</artifactId>
            <version>2.1.11.RELEASE</version>
        </dependency>
        <!--引入security对oauth2的支持-->
        <dependency>
            <groupId>org.springframework.security.oauth</groupId>
            <artifactId>spring-security-oauth2</artifactId>
            <version>2.3.4.RELEASE</version>
        </dependency>




        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.10</version>
        </dependency>
        <!--操作数据库需要事务控制-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-tx</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
        </dependency>


        <dependency>
            <groupId>com.lagou.edu</groupId>
            <artifactId>lagou-service-common</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

    </dependencies>

</project>
```

**application.yml**（配置文件误特别之处）

```yml
server:
  port: 9999
Spring:
  application:
    name: lagou-cloud-oauth-server
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/oauth2?useUnicode=true&characterEncoding=utf-8&useSSL=false&allowMultiQueries=true
    username: root
    password: 123456
    druid:
      initialSize: 10
      minIdle: 10
      maxActive: 30
      maxWait: 50000
eureka:
  client:
    serviceUrl: # eureka server的路径
      defaultZone: http://lagoucloudeurekaservera:8761/eureka/,http://lagoucloudeurekaserverb:8762/eureka/ #把 eureka 集群中的所有 url 都填写了进来，也可以只写一台，因为各个 eureka server 可以同步注册表
  instance:
    #使用ip注册，否则会使用主机名注册了（此处考虑到对老版本的兼容，新版本经过实验都是ip）
    prefer-ip-address: true
    #自定义实例显示格式，加上版本号，便于多版本管理，注意是ip-address，早期版本是ipAddress
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}:@project.version@

```

**入口类无特别之处**

**认证服务器配置类**

```java
package com.lagou.edu.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.jwt.crypto.sign.MacSigner;
import org.springframework.security.jwt.crypto.sign.SignatureVerifier;
import org.springframework.security.jwt.crypto.sign.Signer;
import org.springframework.security.oauth2.config.annotation.configurers.ClientDetailsServiceConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configuration.AuthorizationServerConfigurerAdapter;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;
import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerEndpointsConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerSecurityConfigurer;
import org.springframework.security.oauth2.provider.ClientDetailsService;
import org.springframework.security.oauth2.provider.client.JdbcClientDetailsService;
import org.springframework.security.oauth2.provider.token.*;
import org.springframework.security.oauth2.provider.token.store.InMemoryTokenStore;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;
import org.springframework.security.oauth2.provider.token.store.JwtTokenStore;

import javax.sql.DataSource;
import java.util.ArrayList;
import java.util.List;


/**
 * 当前类为Oauth2 server的配置类（需要继承特定的父类 AuthorizationServerConfigurerAdapter）
 */
@Configuration
@EnableAuthorizationServer  // 开启认证服务器功能
public class OauthServerConfiger extends AuthorizationServerConfigurerAdapter {


    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private LagouAccessTokenConvertor lagouAccessTokenConvertor;


    private String sign_key = "lagou123"; // jwt签名密钥


    /**
     * 认证服务器最终是以api接口的方式对外提供服务（校验合法性并生成令牌、校验令牌等）
     * 那么，以api接口方式对外的话，就涉及到接口的访问权限，我们需要在这里进行必要的配置
     * @param security
     * @throws Exception
     */
    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        super.configure(security);
        // 相当于打开endpoints 访问接口的开关，这样的话后期我们能够访问该接口
        security
                // 允许客户端表单认证
                .allowFormAuthenticationForClients()
                // 开启端口/oauth/token_key的访问权限（允许）
                .tokenKeyAccess("permitAll()")
                // 开启端口/oauth/check_token的访问权限（允许）
                .checkTokenAccess("permitAll()");
    }

    /**
     * 客户端详情配置，
     *  比如client_id，secret
     *  当前这个服务就如同QQ平台，拉勾网作为客户端需要qq平台进行登录授权认证等，提前需要到QQ平台注册，QQ平台会给拉勾网
     *  颁发client_id等必要参数，表明客户端是谁
     * @param clients
     * @throws Exception
     */
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        super.configure(clients);


        // 从内存中加载客户端详情

        /*clients.inMemory()// 客户端信息存储在什么地方，可以在内存中，可以在数据库里
                .withClient("client_lagou")  // 添加一个client配置,指定其client_id
                .secret("abcxyz")                   // 指定客户端的密码/安全码
                .resourceIds("autodeliver")         // 指定客户端所能访问资源id清单，此处的资源id是需要在具体的资源服务器上也配置一样
                // 认证类型/令牌颁发模式，可以配置多个在这里，但是不一定都用，具体使用哪种方式颁发token，需要客户端调用的时候传递参数指定
                .authorizedGrantTypes("password","refresh_token")
                // 客户端的权限范围，此处配置为all全部即可
                .scopes("all");*/

        // 从数据库中加载客户端详情
        clients.withClientDetails(createJdbcClientDetailsService());

    }

    @Autowired
    private DataSource dataSource;

    @Bean
    public JdbcClientDetailsService createJdbcClientDetailsService() {
        JdbcClientDetailsService jdbcClientDetailsService = new JdbcClientDetailsService(dataSource);
        return jdbcClientDetailsService;
    }


    /**
     * 认证服务器是玩转token的，那么这里配置token令牌管理相关（token此时就是一个字符串，当下的token需要在服务器端存储，
     * 那么存储在哪里呢？都是在这里配置）
     * @param endpoints
     * @throws Exception
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        super.configure(endpoints);
        endpoints
                .tokenStore(tokenStore())  // 指定token的存储方法
                .tokenServices(authorizationServerTokenServices())   // token服务的一个描述，可以认为是token生成细节的描述，比如有效时间多少等
                .authenticationManager(authenticationManager) // 指定认证管理器，随后注入一个到当前类使用即可
                .allowedTokenEndpointRequestMethods(HttpMethod.GET,HttpMethod.POST);
    }


    /*
        该方法用于创建tokenStore对象（令牌存储对象）
        token以什么形式存储
     */
    public TokenStore tokenStore(){
        //return new InMemoryTokenStore();
        // 使用jwt令牌
        return new JwtTokenStore(jwtAccessTokenConverter());
    }

    /**
     * 返回jwt令牌转换器（帮助我们生成jwt令牌的）
     * 在这里，我们可以把签名密钥传递进去给转换器对象
     * @return
     */
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
        jwtAccessTokenConverter.setSigningKey(sign_key);  // 签名密钥
        jwtAccessTokenConverter.setVerifier(new MacSigner(sign_key));  // 验证时使用的密钥，和签名密钥保持一致
        jwtAccessTokenConverter.setAccessTokenConverter(lagouAccessTokenConvertor);

        return jwtAccessTokenConverter;
    }




    /**
     * 该方法用户获取一个token服务对象（该对象描述了token有效期等信息）
     */
    public AuthorizationServerTokenServices authorizationServerTokenServices() {
        // 使用默认实现
        DefaultTokenServices defaultTokenServices = new DefaultTokenServices();
        defaultTokenServices.setSupportRefreshToken(true); // 是否开启令牌刷新
        defaultTokenServices.setTokenStore(tokenStore());

        // 针对jwt令牌的添加
        defaultTokenServices.setTokenEnhancer(jwtAccessTokenConverter());

        // 设置令牌有效时间（一般设置为2个小时）
        defaultTokenServices.setAccessTokenValiditySeconds(20); // access_token就是我们请求资源需要携带的令牌
        // 设置刷新令牌的有效时间
        defaultTokenServices.setRefreshTokenValiditySeconds(259200); // 3天

        return defaultTokenServices;
    }
}
```

**关于三个configure方法**

- configure(ClientDetailServiceConfigurer clients)

用来配置客户端详情服务（ClientDetailService），客户端详情信息在这里进行初始化，你能够把客户端详情信息写死在这里或者通过数据库来存储调取详情信息

- configure(AuthorizationServerEndpointsConfigurer endpoints)

用来配置令牌（token）的访问端点和令牌服务（token services）

- configure(AuthorizationServerSecurityConfigurer oauthServer)

用来配置令牌端点的安全约束

**关于TokenStore**

- InMemoryTokenStore

默认采用，可以在开发阶段使用

- JdbcTokenStore

这是一个基于JDBC的实现版本，令牌会被保存进关系型数据库。可以在不同的服务器之间共享令牌信息，使用这个版本的时候请注意把spring-jdbc依赖加入到classpath中

- JwtTokenStore

JSON Web Token(JWT)，它可以把令牌相关的数据进行编码（因此对于后端服务来说，它不需要进行存储，这将是一个重大的优势），缺点就是这个令牌占用的空间会比较大，如果你加入了比较多的用户凭证信息，JwtTokenStore不会保存任何数据。

**认证服务器安全配置类**

```java
package com.lagou.edu.config;

import com.lagou.edu.service.JdbcUserDetailsService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.password.NoOpPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;


/**
 * 该配置类，主要处理用户名和密码的校验等事宜
 */
@Configuration
public class SecurityConfiger extends WebSecurityConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private JdbcUserDetailsService jdbcUserDetailsService;

    /**
     * 注册一个认证管理器对象到容器
     */
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }


    /**
     * 密码编码对象（密码不进行加密处理）
     * @return
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }

    /**
     * 处理用户名和密码验证事宜
     * 1）客户端传递username和password参数到认证服务器
     * 2）一般来说，username和password会存储在数据库中的用户表中
     * 3）根据用户表中数据，验证当前传递过来的用户信息的合法性
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 在这个方法中就可以去关联数据库了，当前我们先把用户信息配置在内存中
        // 实例化一个用户对象(相当于数据表中的一条用户记录)
        /*UserDetails user = new User("admin","123456",new ArrayList<>());
        auth.inMemoryAuthentication()
                .withUser(user).passwordEncoder(passwordEncoder);*/

        auth.userDetailsService(jdbcUserDetailsService).passwordEncoder(passwordEncoder);
    }
}
```

测试

获取token:http://localhost:9999/oauth?client_secret=abcxyz&grant_type=password&username=admin&password=123456&client_id=client_lagou

- endpoint:/oauth/token
- 获取token携带的参数
  - client_id:
  - client_secret:
  - grant_type:指定使用哪种颁发类型，password
  - username:
  - password:

校验token:http://localhost:9999/oauth/check_token=

刷新token:http://localhost:9999/oauth/token?grant_type=refresh_token&client_id=client_lagou&client_secret=abcxyz&refresh_token=

**资源服务器（希望访问被认证的微服务）Resource Server配置**

- 资源服务配置类

```java
package com.lagou.edu.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.jwt.crypto.sign.MacSigner;
import org.springframework.security.jwt.crypto.sign.RsaVerifier;
import org.springframework.security.jwt.crypto.sign.SignatureVerifier;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableResourceServer;
import org.springframework.security.oauth2.config.annotation.web.configuration.ResourceServerConfigurerAdapter;
import org.springframework.security.oauth2.config.annotation.web.configurers.ResourceServerSecurityConfigurer;
import org.springframework.security.oauth2.provider.token.RemoteTokenServices;
import org.springframework.security.oauth2.provider.token.TokenStore;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;
import org.springframework.security.oauth2.provider.token.store.JwtTokenStore;

@Configuration
@EnableResourceServer  // 开启资源服务器功能
@EnableWebSecurity  // 开启web访问安全
public class ResourceServerConfiger extends ResourceServerConfigurerAdapter {

    private String sign_key = "lagou123"; // jwt签名密钥

    @Autowired
    private LagouAccessTokenConvertor lagouAccessTokenConvertor;

    /**
     * 该方法用于定义资源服务器向远程认证服务器发起请求，进行token校验等事宜
     * @param resources
     * @throws Exception
     */
    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {

        /*// 设置当前资源服务的资源id
        resources.resourceId("autodeliver");
        // 定义token服务对象（token校验就应该靠token服务对象）
        RemoteTokenServices remoteTokenServices = new RemoteTokenServices();
        // 校验端点/接口设置
        remoteTokenServices.setCheckTokenEndpointUrl("http://localhost:9999/oauth/check_token");
        // 携带客户端id和客户端安全码
        remoteTokenServices.setClientId("client_lagou");
        remoteTokenServices.setClientSecret("abcxyz");

        // 别忘了这一步
        resources.tokenServices(remoteTokenServices);*/


        // jwt令牌改造
        resources.resourceId("autodeliver").tokenStore(tokenStore()).stateless(true);// 无状态设置
    }


    /**
     * 场景：一个服务中可能有很多资源（API接口）
     *    某一些API接口，需要先认证，才能访问
     *    某一些API接口，压根就不需要认证，本来就是对外开放的接口
     *    我们就需要对不同特点的接口区分对待（在当前configure方法中完成），设置是否需要经过认证
     *
     * @param http
     * @throws Exception
     */
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http    // 设置session的创建策略（根据需要创建即可）
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .and()
                .authorizeRequests()
                .antMatchers("/autodeliver/**").authenticated() // autodeliver为前缀的请求需要认证
                .antMatchers("/demo/**").authenticated()  // demo为前缀的请求需要认证
                .anyRequest().permitAll();  //  其他请求不认证
    }




    /*
       该方法用于创建tokenStore对象（令牌存储对象）
       token以什么形式存储
    */
    public TokenStore tokenStore(){
        //return new InMemoryTokenStore();

        // 使用jwt令牌
        return new JwtTokenStore(jwtAccessTokenConverter());
    }

    /**
     * 返回jwt令牌转换器（帮助我们生成jwt令牌的）
     * 在这里，我们可以把签名密钥传递进去给转换器对象
     * @return
     */
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
        jwtAccessTokenConverter.setSigningKey(sign_key);  // 签名密钥
        jwtAccessTokenConverter.setVerifier(new MacSigner(sign_key));  // 验证时使用的密钥，和签名密钥保持一致
        jwtAccessTokenConverter.setAccessTokenConverter(lagouAccessTokenConvertor);
        return jwtAccessTokenConverter;
    }

}
```

思考：当我们第一次登录之后，认证服务器颁发token并将其存储在认证服务器中，后期我们访问资源服务器时会携带token，资源服务器会请求认证服务器验证token有效性，如果资源服务器很多，那么认证服务器压力会很大。。。。

另外，资源服务器向认证服务器check_token，获取的也是用户信息UserInfo，能否把用户信息存储到令牌中，让客户端一值持有这个令牌，令牌的验证也在资源服务器进行，这样避免和认证服务器频繁的交互。。。

我们可以考虑使用JWT进行改造，使用JWT机制之后，资源服务器不需要访问认证服务器。。。

#### JWT改造统一认证授权中心的令牌存储机制

**JWT令牌介绍**

通过上边的测试我们发现，当资源服务和授权服务不在一起时资源服务使用RemoteTokenServices远程请求授权服务验证token，如果访问量较大将会影响系统的性能。

解决上边的问题：令牌采用JWT格式即可解决上边的问题，用户认证通过会得到一个JWT令牌，JWT令牌中已经包括了用户相关的信息，客户端只需要携带JWT访问资源服务，资源服务根据事先约定的算法自行完成令牌校验，无需每次请求认证服务完成授权。

> 资源服务接收到请求，需要请求认证服务器验证token，当资源服务器很多时，这个可能存在瓶颈，使用JWT令牌，令牌本身包含用户信息，可以通过事先约定的算法在资源服务器内部完成令牌校验。

1）什么是JWT？

JSON Web Token(JWT)是一个开放的行业标准（RFC 7519），它定义了一种简洁的、自包含的协议格式，用于在通信双方传递json对象，传递的信息经过数字签名可以被验证和信任。JWT可以使用HMAC算法或使用RSA的公钥/私钥来对签名验证，防止被篡改。

2）JWT令牌结构

JWT令牌由三部分组成，每部分中间使用点(.)分割，比如：xxx.yyy.zzz

- Header

头部包括令牌的类型（即JWT）及使用的哈希算法（如HMAC SHA256或RSA），例如

```json
{
  "alg":"HS256",
  "typ":"JWT"
}
```

将上边的内容使用Base64Url编码，得到一个字符串就是JWT令牌的第一部分。

- Payload

第二部分是负载，内容也是一个json对象，它是存放有效信息的地方，它可以存放jwt提供的现成字段，比如：iss(签发者)，ex(过期时间戳)，sub(面向的用户)等，也可以自定义字段。此部分不建议存放敏感信息，因为此部分可以解码还原原始内容。最后将第二部分负载使用Base64Url编码，得到一个字符串就是JWT令牌的第二部分，一个例子：

```json
{
  "sub":"1234567890",
  "name":"Johh Doe",
  "iat":1516239022
}
```

- Signature

第三部分是签名，此部分用于防止jwt内容被篡改。这个部分使用base64url将前两部分进行编码，编码后使用点(.)连接组成字符串，最后使用header中声明签名算法进行签名。

```json
HMACSHA256(
	base64UrlEncode(header) + "."+
  base64UrlEncode(payload),
  secret
)
```

base64UrlEncode(header):jwt令牌的第一部分

base64UrlEncode(payload):jwt令牌的第二部分

secret:签名所使用的密钥

**认证服务器端JWT改造**（改造主配置类）

```java
/*
        该方法用于创建tokenStore对象（令牌存储对象）
        token以什么形式存储
     */
    public TokenStore tokenStore(){
        //return new InMemoryTokenStore();
        // 使用jwt令牌
        return new JwtTokenStore(jwtAccessTokenConverter());
    }

    /**
     * 返回jwt令牌转换器（帮助我们生成jwt令牌的）
     * 在这里，我们可以把签名密钥传递进去给转换器对象
     * @return
     */
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
        jwtAccessTokenConverter.setSigningKey(sign_key);  // 签名密钥
        jwtAccessTokenConverter.setVerifier(new MacSigner(sign_key));  // 验证时使用的密钥，和签名密钥保持一致
        jwtAccessTokenConverter.setAccessTokenConverter(lagouAccessTokenConvertor);

        return jwtAccessTokenConverter;
    }
```

修改jwt令牌服务方法

![image-20200712131804267](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200712131804267.png)

**资源服务器校验JWT令牌**

不需要和远程认证服务器交互，添加本地tokenStore

```java
package com.lagou.edu.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.jwt.crypto.sign.MacSigner;
import org.springframework.security.jwt.crypto.sign.RsaVerifier;
import org.springframework.security.jwt.crypto.sign.SignatureVerifier;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableResourceServer;
import org.springframework.security.oauth2.config.annotation.web.configuration.ResourceServerConfigurerAdapter;
import org.springframework.security.oauth2.config.annotation.web.configurers.ResourceServerSecurityConfigurer;
import org.springframework.security.oauth2.provider.token.RemoteTokenServices;
import org.springframework.security.oauth2.provider.token.TokenStore;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;
import org.springframework.security.oauth2.provider.token.store.JwtTokenStore;

@Configuration
@EnableResourceServer  // 开启资源服务器功能
@EnableWebSecurity  // 开启web访问安全
public class ResourceServerConfiger extends ResourceServerConfigurerAdapter {

    private String sign_key = "lagou123"; // jwt签名密钥

    @Autowired
    private LagouAccessTokenConvertor lagouAccessTokenConvertor;

    /**
     * 该方法用于定义资源服务器向远程认证服务器发起请求，进行token校验等事宜
     * @param resources
     * @throws Exception
     */
    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {

        /*// 设置当前资源服务的资源id
        resources.resourceId("autodeliver");
        // 定义token服务对象（token校验就应该靠token服务对象）
        RemoteTokenServices remoteTokenServices = new RemoteTokenServices();
        // 校验端点/接口设置
        remoteTokenServices.setCheckTokenEndpointUrl("http://localhost:9999/oauth/check_token");
        // 携带客户端id和客户端安全码
        remoteTokenServices.setClientId("client_lagou");
        remoteTokenServices.setClientSecret("abcxyz");

        // 别忘了这一步
        resources.tokenServices(remoteTokenServices);*/


        // jwt令牌改造
        resources.resourceId("autodeliver").tokenStore(tokenStore()).stateless(true);// 无状态设置
    }


    /**
     * 场景：一个服务中可能有很多资源（API接口）
     *    某一些API接口，需要先认证，才能访问
     *    某一些API接口，压根就不需要认证，本来就是对外开放的接口
     *    我们就需要对不同特点的接口区分对待（在当前configure方法中完成），设置是否需要经过认证
     *
     * @param http
     * @throws Exception
     */
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http    // 设置session的创建策略（根据需要创建即可）
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .and()
                .authorizeRequests()
                .antMatchers("/autodeliver/**").authenticated() // autodeliver为前缀的请求需要认证
                .antMatchers("/demo/**").authenticated()  // demo为前缀的请求需要认证
                .anyRequest().permitAll();  //  其他请求不认证
    }




    /*
       该方法用于创建tokenStore对象（令牌存储对象）
       token以什么形式存储
    */
    public TokenStore tokenStore(){
        //return new InMemoryTokenStore();

        // 使用jwt令牌
        return new JwtTokenStore(jwtAccessTokenConverter());
    }

    /**
     * 返回jwt令牌转换器（帮助我们生成jwt令牌的）
     * 在这里，我们可以把签名密钥传递进去给转换器对象
     * @return
     */
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
        jwtAccessTokenConverter.setSigningKey(sign_key);  // 签名密钥
        jwtAccessTokenConverter.setVerifier(new MacSigner(sign_key));  // 验证时使用的密钥，和签名密钥保持一致
        jwtAccessTokenConverter.setAccessTokenConverter(lagouAccessTokenConvertor);
        return jwtAccessTokenConverter;
    }

}

```

#### 从数据库加载OAuth2客户端信息

- 创建表并初始化数据（表名和字段保持固定）

```sql
SET NAMES utf8mb4; 
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------

-- Table structure for oauth_client_details

-- ---------------------------
DROP TABLE IF EXISTS `oauth_client_details`; 
CREATE TABLE `oauth_client_details` ( `client_id` varchar(48) NOT NULL, `resource_ids` varchar(256) DEFAULT NULL, `client_secret` varchar(256) DEFAULT NULL, `scope` varchar(256) DEFAULT NULL, `authorized_grant_types` varchar(256) DEFAULT NULL, `web_server_redirect_uri` varchar(256) DEFAULT NULL, `authorities` varchar(256) DEFAULT NULL, `access_token_validity` int(11) DEFAULT NULL, `refresh_token_validity` int(11) DEFAULT NULL, `additional_information` varchar(4096) DEFAULT NULL, `autoapprove` varchar(256) DEFAULT NULL, PRIMARY KEY (`client_id`) 
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------

-- Records of oauth_client_details

-- ---------------------------
BEGIN; 
INSERT INTO `oauth_client_details` VALUES ('client_lagou123', 'autodeliver,resume', 'abcxyz', 'all', 'password,refresh_token', NULL, NULL, 7200, 259200, NULL, NULL); 
COMMIT;

SET FOREIGN_KEY_CHECKS = 1;
```

- 配置数据源

```yml
server:
  port: 9999
Spring:
  application:
    name: lagou-cloud-oauth-server
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/oauth2?useUnicode=true&characterEncoding=utf-8&useSSL=false&allowMultiQueries=true
    username: root
    password: 123456
    druid:
      initialSize: 10
      minIdle: 10
      maxActive: 30
      maxWait: 50000
eureka:
  client:
    serviceUrl: # eureka server的路径
      defaultZone: http://lagoucloudeurekaservera:8761/eureka/,http://lagoucloudeurekaserverb:8762/eureka/ #把 eureka 集群中的所有 url 都填写了进来，也可以只写一台，因为各个 eureka server 可以同步注册表
  instance:
    #使用ip注册，否则会使用主机名注册了（此处考虑到对老版本的兼容，新版本经过实验都是ip）
    prefer-ip-address: true
    #自定义实例显示格式，加上版本号，便于多版本管理，注意是ip-address，早期版本是ipAddress
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}:@project.version@

```

- 认证服务器主配置类改造

```java
 /**
     * 客户端详情配置，
     *  比如client_id，secret
     *  当前这个服务就如同QQ平台，拉勾网作为客户端需要qq平台进行登录授权认证等，提前需要到QQ平台注册，QQ平台会给拉勾网
     *  颁发client_id等必要参数，表明客户端是谁
     * @param clients
     * @throws Exception
     */
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        super.configure(clients);


        // 从内存中加载客户端详情

        /*clients.inMemory()// 客户端信息存储在什么地方，可以在内存中，可以在数据库里
                .withClient("client_lagou")  // 添加一个client配置,指定其client_id
                .secret("abcxyz")                   // 指定客户端的密码/安全码
                .resourceIds("autodeliver")         // 指定客户端所能访问资源id清单，此处的资源id是需要在具体的资源服务器上也配置一样
                // 认证类型/令牌颁发模式，可以配置多个在这里，但是不一定都用，具体使用哪种方式颁发token，需要客户端调用的时候传递参数指定
                .authorizedGrantTypes("password","refresh_token")
                // 客户端的权限范围，此处配置为all全部即可
                .scopes("all");*/

        // 从数据库中加载客户端详情
        clients.withClientDetails(createJdbcClientDetailsService());

    }

    @Autowired
    private DataSource dataSource;

    @Bean
    public JdbcClientDetailsService createJdbcClientDetailsService() {
        JdbcClientDetailsService jdbcClientDetailsService = new JdbcClientDetailsService(dataSource);
        return jdbcClientDetailsService;
    }
```

#### 从数据库验证用户合法性

- 创建数据表users（表名不需固定），初始化数据

```sql
SET NAMES utf8mb4; 
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------

-- Table structure for users

-- ---------------------------
DROP TABLE IF EXISTS `users`; 
CREATE TABLE `users` ( `id` int(11) NOT NULL AUTO_INCREMENT, `username` char(10) DEFAULT NULL, `password` char(100) DEFAULT NULL, PRIMARY KEY (`id`) 
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8;

-- ----------------------------

-- Records of users

-- ---------------------------
BEGIN; 
INSERT INTO `users` VALUES (4, 'lagou-user', 'iuxyzds'); 
COMMIT;

SET FOREIGN_KEY_CHECKS = 1;
```

- 操作数据表的JPA配置及DAO接口

```java
public interface UsersRepository extends JpaRepository<Users,Long> {
    Users findByUsername(String username);
}
```

- 开发UserDetailsService接口的实现类，根据用户名从数据库加载用户信息

```java
@Service
public class JdbcUserDetailsService implements UserDetailsService {

    @Autowired
    private UsersRepository usersRepository;

    /**
     * 根据username查询出该用户的所有信息，封装成UserDetails类型的对象返回，至于密码，框架会自动匹配
     * @param username
     * @return
     * @throws UsernameNotFoundException
     */
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Users users = usersRepository.findByUsername(username);
        return new User(users.getUsername(),users.getPassword(),new ArrayList<>());
    }
}
```

- 使用自定义的用户详情服务对象

```java
@Autowired
    private JdbcUserDetailsService jdbcUserDetailsService;

/**
     * 处理用户名和密码验证事宜
     * 1）客户端传递username和password参数到认证服务器
     * 2）一般来说，username和password会存储在数据库中的用户表中
     * 3）根据用户表中数据，验证当前传递过来的用户信息的合法性
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 在这个方法中就可以去关联数据库了，当前我们先把用户信息配置在内存中
        // 实例化一个用户对象(相当于数据表中的一条用户记录)
        /*UserDetails user = new User("admin","123456",new ArrayList<>());
        auth.inMemoryAuthentication()
                .withUser(user).passwordEncoder(passwordEncoder);*/

        auth.userDetailsService(jdbcUserDetailsService).passwordEncoder(passwordEncoder);
    }
```

#### 基于OAuth2的JWT令牌信息扩展

OAuth2帮我们生产的JWT令牌载荷部分信息有限，关于用户信息只有一个user_name，有些场景下我们希望放入一些扩展信息项，比如，之前我们经常向session中存入userid，或者现在我们希望在JWT的载荷部分存入当时请求令牌的客户端IP，客户端携带令牌访问资源服务器时，可以对比当前请求的客户端真是IP和令牌存放的客户端IP是否匹配，不匹配拒绝请求，以此进一步提高安全性。那么如何在OAuth2环境下向JWT令牌中存入扩展信息呢？

- 认证服务器生存JWT令牌时存入扩展信息（比如clientIp）

继承DefaultAccessTokenConverter类，重写convertAccessToken方法，存入扩展信息

```java
@Component
public class LagouAccessTokenConvertor extends DefaultAccessTokenConverter {


    @Override
    public Map<String, ?> convertAccessToken(OAuth2AccessToken token, OAuth2Authentication authentication) {
        // 获取到request对象
        HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.getRequestAttributes())).getRequest();
        // 获取客户端ip（注意：如果是经过代理之后到达当前服务的话，那么这种方式获取的并不是真实的浏览器客户端ip）
        String remoteAddr = request.getRemoteAddr();
        Map<String, String> stringMap = (Map<String, String>) super.convertAccessToken(token, authentication);
        stringMap.put("clientIp",remoteAddr);
        return stringMap;
    }
}
```

将自定义的转换器对象注入

```java
    /**
     * 返回jwt令牌转换器（帮助我们生成jwt令牌的）
     * 在这里，我们可以把签名密钥传递进去给转换器对象
     * @return
     */
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
        jwtAccessTokenConverter.setSigningKey(sign_key);  // 签名密钥
        jwtAccessTokenConverter.setVerifier(new MacSigner(sign_key));  // 验证时使用的密钥，和签名密钥保持一致
        jwtAccessTokenConverter.setAccessTokenConverter(lagouAccessTokenConvertor);

        return jwtAccessTokenConverter;
    }
```

#### 资源服务器取出JWT令牌扩展信息

资源服务器也需要自定义一个转换器类，继承DefaultAccessTokenConverter类，重写extractAuthentication提取方法，把载荷信息设置到认证对象的details属性中

```java
@Component
public class LagouAccessTokenConvertor extends DefaultAccessTokenConverter {


    @Override
    public OAuth2Authentication extractAuthentication(Map<String, ?> map) {

        OAuth2Authentication oAuth2Authentication = super.extractAuthentication(map);
        oAuth2Authentication.setDetails(map);  // 将map放入认证对象中，认证对象在controller中可以拿到
        return oAuth2Authentication;
    }
}
```

业务类比如Controller类中，可以通过如下方法获取认证对象，进一步获取到扩展信息

```java
 @GetMapping("/test")
    public String findResumeOpenState() {
        Object details = SecurityContextHolder.getContext().getAuthentication().getDetails();
        return "demo/test!";
    }
```

#### JWT注意事项

关于JWT令牌我们需要注意

- JWT令牌就是一种可以被验证的数据组织格式，玩法灵活，我们这里基于Spring Cloud OAuth2创建、校验JWT令牌
- 我们也可以自己写工具类生成、校验JWT令牌
- JWT令牌中不要存放过于敏感的信息，因为我们拿到令牌后，可以解码看到载荷部分的信息
- JWT令牌每次请求都会携带，内容过多的话，会增加网络宽带占用

## 第七部分 第二代Spring Cloud核心组件（Spring Cloud Alibaba）

Nacos(服务注册和配置中心)

Sentinel哨兵（服务熔断、限流）

Dubbo RPC/LB

Seata分布式事务解决方案

### Nacos服务注册和配置中心

#### Nacos介绍

Nacos就是注册中心+配置中心的组合（Nacos=Eureka+Config+Bus）

#### Nacos功能特性

- 服务发现于健康检查
- 动态配置管理
- 动态DNS服务
- 服务和元数据管理，动态的服务权重调整、动态服务优雅下线

#### Nacos单实例部署

运行

unix：``sh startup.sh -m standalone``

访问nacos管理控制台

http://127.0.0.1/8848/nacos

用户名和密码默认都是nacos



#### Nacos注册中心

保护阈值

可以设置0-1之间的浮点数，是一个比值（当前服务健康实例数/当前服务总实例数）

当保护阈值触发，nacos会把该服务的所有实例信息（健康的+不健康的）全部提供给消费者，消费者可能访问到不健康的实例，请求失败，但这样也比造成雪崩好。

#### 服务消费者从Nacos获取服务

#### 负载均衡

Nacos客户端引入的时候，会关联引入Ribbon的依赖包，我们使用OpenFiegn的时候也会引入Ribbon的依赖，Ribbon包括Hystrix都按原来方式进行配置即可。

> Nacos会默认使用Ribbon客户端负载均衡，如果想换成Dubbo LB呢

#### Nacos数据模型（领域模型）

Namespace命名空间、Group分组、集群这些都是为了进行归类管理，把服务和配置文件进行归类，归类之后就可以实现一定的效果，比如隔离

比如，对于服务来说，不同命名空间中的服务不能互相访问调用

![image-20200712210310147](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200712210310147.png)

Namespace：命名空间，对不同环境进行隔离，比如开发环境、测试环境、生产环境

Group：分组，将若干个服务或若干个配置集归为一组，通常习惯一个系统归位一个组

Service：某一个服务，比如简历微服务

DataId：配置集或者可以认为是一个配置文件

**Namespace+Group+Service如同Maven中的GAV坐标，GAV坐标是为了锁定jar，而这里是为了锁定服务**

**Namespace+Group+DataId如同Maven中的GAV坐标，GAV坐标是为了锁定jar，而这里是为了锁定配置文件**

**最佳实践**

Nacos抽象了Namespace、Group、Service、DataId等概念，具体代表什么取决于怎么用（非常灵活），推荐用法如下：

| 概念      | 描述                                           |
| --------- | ---------------------------------------------- |
| Namespace | 代表不同的环境，如开发环境、测试环境、生产环境 |
| Group     | 代表某项目，比如拉勾云项目                     |
| Service   | 某个项目中具体的服务                           |
| DataId    | 某个项目中具体的配置文件                       |

- Nacos服务的分级模型

![image-20200712211009939](https://gitee.com/jinxin.70/oss/raw/master/uPic2/image-20200712211009939.png)

#### Nacos Server数据持久化

Nacos默认使用嵌入式数据库进行数据存储，它支持改为外部MySQL存储

- 新建数据库nacos_config，数据库初始化脚本文件${nacoshome}/conf/nacos-mysql.sql
- 修改${nacoshome}/conf/appcalition.properties

```properties
spring.datasource.platform=mysql

db.num=1

db.url.0=jdbc:mysql://127.0.0.1:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=123456
```

#### Nacos Server集群

- 安装3个或3个以上的Nacos

复制解压后的nacos文件夹，分别命名为nacos-01、nacos-02、nacos-03

- 修改配置文件

  - 同一台机器模拟，将上述三个文件夹中application.properties中的server.port分别改为8848、8849、8850，同时给当前实例节点绑定ip，因为服务器可能绑定多个ip。

  ```properties
  nacos.inetutils.ip-address=127.0.0.1
  ```

  - 复制一份conf/cluster.conf.example文件，命名为cluster.conf，在配置文件中设置集群中每一个节点的信息

  ```properties
  127.0.0.1:8848
  127.0.0.1:8849
  127.0.0.1:8850
  ```

- 分别启动每一个实例（可以批处理脚本完成）

```shell
sh startup.sh -m cluster
```

#### Nacos配置中心

在Nacos控制台添加配置，或者通过API添加配置

一个DataId表示一个配置文件