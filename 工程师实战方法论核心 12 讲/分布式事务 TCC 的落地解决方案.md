# 分布式事务 TCC 的落地解决方案

### 分布式事务概述

随着互联网的发展、大数据时代的来临，使得很多企业以前的单体架构不再能够支撑大规模用户的应用，开始寻求突破性能瓶颈的技术方案，SOA、微服务正是在这种背景下产生的。无论是 SOA 和微服务想要支持大规模用户就必须使用分布式方式部署，分布式系统微服务应用由多个不同的子系统组成，各个子系统通过网络通讯传输数据共同配合完成各个功能。

如下图是一个典型的电子商务网站微服务架构，在分布式系统中每个子系统（微服务）都是相互独立的，拥有自己独立的数据库，在涉及到下单支付流程时就必须要订单服务、库存服务和账户服务一起配合完成，创建订单、扣减库存和扣减账户余额，这三步操作都会修改数据库数据，必须做到最终要么都成功，要么都失败，否则将会出现数据不一致情况。传统的单体架构数据库事务没办法解决这种问题，这种问题就被称为分布式事务问题。分布式事务的宗旨就是要在分布式系统中保持数据的一致性。

![在这里插入图片描述](https://images.gitbook.cn/5b15cd30-005f-11ea-85ee-29a5861d83e8)

介绍了解分布式事务之后，我们先来对分布式系统的一致性问题进行理论分析。首先介绍 CAP 理论，CAP 定理又被称作布鲁尔定理，是加州大学的计算机科学家布鲁尔在 2000 年提出的一个猜想。2002 年，麻省理工学院的赛斯 · 吉尔伯特和南希 · 林奇发表了布鲁尔猜想的证明，使之成为分布式计算领域公认的一个定理。

CAP 理论定义了在一个分布式系统中，当涉及读写操作时，只能保证一致性（Consistence）、可用性（Availability）、分区容错性（PartitionTolerance）三者中的两个，另外一个必须被牺牲。

![Alt](https://images.gitbook.cn/509ef090-0062-11ea-8a36-337a5cff47ad)

**C：Consistency 一致性**

> A read is guaranteed to return the most recent write for a given client.
>
> 对某个指定的客户端来说，读操作保证能够返回最新的写操作结果。

**A：Availability 可用性**

> A non-failing node will return a reasonable response within a reasonable amount of time (no error or timeout).
>
> 非故障的节点在合理的时间内返回合理的响应（不是错误和超时的响应）。

**P：Partition Tolerance 分区容忍性**

> The system will continue to function when network partitions occur.
>
> 当出现网络分区后，系统能够继续“履行职责”。

在大型互联网应用中，分布式部署、大规模集群的条件下，节点故障、网络故障是不可避免的，所以我们必须保证分区容错性。那么就需要在剩下的一致性和可用性中选择其一，如果选择一致性，那么就要将整个系统在处理用户请求时锁定各个子系统，这时候其他的请求将被阻塞或者提示错误（放弃可用性），用户体验会非常差，所以基本不会选择。选择分区容错性和可用性，放弃一致性（这里说的一致性是强一致性）是现在很多分布式系统的选择，当一个请求涉及到多个子系统时，修改数据在各个子系统之间传递会存在网络延迟、机器故障等问题，使得数据不能时时一致，但是仍然可以获取到一个合理的结果，而不是阻塞或者错误。

有人在 CAP 理论的基础上又提出了一个 BASE 理论。BASE 是指基本可用（Basically Available）、软状态（ Soft State）、最终一致性（ Eventual Consistency），核心思想是即使无法做到强一致性（CAP 的一致性就是强一致性），但应用可以采用适合的方式达到最终一致性。

![Alt](https://images.gitbook.cn/c3b29ac0-01d7-11ea-b417-c70197b7f6af)

**BA：Basically Available 基本可用**

> 分布式系统在出现故障时，允许损失部分可用性，即保证核心可用。

**S：Soft State 软状态**

> 允许系统存在中间状态，而该中间状态不会影响系统整体可用性。这里的中间状态就是 CAP 理论中的数据不一致。

**E：Eventual Consistency 最终一致性**

> 系统中的所有数据副本经过一定时间后，最终能够达到一致的状态。

BASE 理论明确指出了在保证分布式系统可用性时，允许在数据一致性（强一致性）上面放宽限制，允许子系统之间存在中间状态，例如上面列举的用户下单例子，可能会存在账户钱扣去了，但是库存还没有扣减这样类似的中间状态，但是只要保证最终库存是一定会扣减的，这对用户来说是可以接受的。

放弃了数据强一致性，选择最终一致性，换来的是可以将系统部署在分布式环境中，同时可以提高系统的可用性、扩展性、维护性，能够支撑更高的并发访问数量，好处一目了然，以至于这样的系统架构已经称为现在大型互联网应用的必然选择。

### TCC 落地方案

理论在前，实践在后，在了解了分布式事务理论知识之后，我们接下来就实践一下分布式事务。我们首先需要知道分布式事务都有哪些实现方案，其实分布式事务的方案整体来说大概有[四到五种方案](https://blog.csdn.net/u010739551/article/details/100096867)，但是真正在企业大规模使用就两种：基于消息队列的最终一致性和基于 TCC 近实时一致性。这篇文章将通过实践 TCC 方案来帮助各位快速理解和掌握分布式事务。

下面这个图描绘了整个 TCC 方案的实现思路。[TCC 方案](https://www.cnblogs.com/jajian/p/10014145.html)由三个单词首字母 Try、Confirm、Cancel 组合而成，在具有分布式事务的业务操作中，例如仍然是前面提到的用户下单业务，整个系统的订单服务流程如下：

- 在接收到下单请求之后，需要依次执行各节点服务 try 方法，try 方法预留资源为后面执行 confirm 做准备，对于订单服务它就负责创建未支付成功的订单；
- 接着 RPC 调用库存服务和账户服务的 try 方法，调用完成之后再执行订单服务的 confirm 方法；
- 同时同样 RPC 调用库存服务和账户服务的 confirm 方法，最后将结果返回给用户；
- try 阶段和 confirm 阶段出现错误异常时都将执行各个服务的 cancel 方法。

try 阶段预留的资源其实就是中间状态，通过执行 confirm 或者 cancel 方法最终达成整个系统数据一致性，只是 TCC 方案相比较于基于消息队列的最终一致性来说实时性更好，但是需要的开发成本更高，因为各个子系统服务的三个阶段方法需要开发者根据业务自己实现，同时需要 confirm 方法和 cancel 方法保持幂等性。

![在这里插入图片描述](https://images.gitbook.cn/80ee6f70-01f0-11ea-9bd3-bf59687abb68)

### 实现自己的分布式事务框架

通过上面的的介绍，大家应该已经大概了解了整个 TCC 方案的实现思路，但是有一些地方还是没搞清楚的，例如事务日志应该如何创建，结构是如何的呢？try 阶段执行完成之后是如何调用 confirm 或者 cancel 的呢？自恢复定时任务又是做什么的呢？这些问题唯有实践才能知道。

下面我就来源码解析作者自己的开源的[分布式事务框架 Milo](https://gitee.com/luke2017/milo)。

![在这里插入图片描述](https://images.gitbook.cn/6a48e710-04f6-11ea-a521-696a4b4a1d50)

项目中以 cloud 开头的是测试 Demo，分别代表了注册中心、订单服务、账户服务和库存服务。以 milo 开头的则是分布式事务框架实现代码。

#### 框架介绍

**milo-common 框架的公共信息**

包含了事务日志、配置信息、事务上下文、枚举信息、异常类和序列化器。

![在这里插入图片描述](https://images.gitbook.cn/5bc7b7b0-04f7-11ea-8d4f-79586e558671)

**milo-core 框架的核心内容**

包含了事务注解、AOP 切面处理器、框架启动器、事务日志操作类、事务上下文传递工具类、自恢复定时任务、业务逻辑处理类、事务上下文线程本地类和容器存储工具类。

![在这里插入图片描述](https://images.gitbook.cn/19152c80-04f8-11ea-bd6d-cf3c3ec89ddf)

**milo-spring-cloud-starter 框架启动器**

框架启动器是外部项目接入 Milo 的依赖项目，包含了从外部项目引入配置和启动整个 Milo 框架。

![在这里插入图片描述](https://images.gitbook.cn/f28b7e10-04f8-11ea-8d4f-79586e558671)

**milo-spring-cloud 框架整合 spring-cloud 内容**

该部分内容主要是实现和 Spring Cloud 的整合，包括切面处理器的入口实现、拦截 feign 请求远程传递事务上线文和事务上下文在使用 Hystrix 时跨线程传递。

![在这里插入图片描述](https://images.gitbook.cn/6ea89050-04f9-11ea-bd6d-cf3c3ec89ddf)

#### 框架实现思路

实现分布式事务 TCC 解决方案需要把握的地方就是如何实现服务之间的数据一致，如何能够将代码更好的封装，降低依赖项目的耦合度，最后一点就是需要保证事务能够在异常状态下仍然保证数据的最终一致性。

将一个现有的项目改造成支持分布式事务，降低对现有项目的改造就是使用注解方式，而 TCC 方案是围绕 try、confirm 和 cancel 三个方法展开的，所以我们需要使用者告知我们每个具有分布式事务业务 TCC 对应的方法和参数。

为此我定义了 TCC 注解，让用户使用该注解标明这是一个分布式事务业务，然后我们通过拦截该注解实现对分布式数据一致性的同步控制。

##### **定义 TCC 注解**

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface MiloTCC {

    /**
     * 指明TCC的confirm方法，具备幂等性
     * @return string
     */
    String confirmMethod() default "";

    /**
     * 指明TCC的cancel方法，具备幂等性
     * @return
     */
    String cancelMethod() default "";

}
```

在 cloud-order 订单服务，是用户下单的入口，在 OrderController 控制类的 payment 方法是一个具有分布式事务的业务方法（创建订单、远程调用扣减库存和扣减账户），该方法用户只需引入 @MiloTCC 注解，标明是 TCC 的 try 方法，并指定了 paymentConfirm 作为 confirm 方法、paymentCancel 作为 cancel 方法。

这种方式是低入侵方式，使用者是起来非常方便，唯一麻烦的地方就是使用者需要开发对应具备幂等性的 confirm 和 cancel 方法（入参一致，失败主动抛出异常），这也是 TCC 方式的特点，那就是开发成本比较高，好处就是实时性也比较高，使用者需要根据自己的业务需求选择不同的分布式事务方案，而像用户下单这种对实时性要求较高的业务，使用 TCC 方法是再合适不过了。

```
    @MiloTCC(confirmMethod = "paymentConfirm",cancelMethod = "paymentCancel")
    @Override
    public void payment(PaymentOrderReq req) {
        log.info("==================payment开始=======================");
        Order order = orderMapper.selectById(req.getOrderId());
        order.setStatus(2);
        order.setUpdateTime(new Date());
        orderMapper.updateById(order);

        String remoteResult = "failure";
        //扣减库存
        DecreaseStockReq stockReq = new DecreaseStockReq();
        stockReq.setOrderId(order.getId());
        stockReq.setProductName(req.getProductName());
        stockReq.setStock(req.getCount());
        remoteResult = stockInter.decreaseStock(stockReq);
        if ("failure".equals(remoteResult)){
            throw new RuntimeException("decreaseStock error");
        }

        //扣减账户余额
        DecreaseAccountReq accountReq = new DecreaseAccountReq();
        accountReq.setOrderId(order.getId());
        accountReq.setUsername(req.getUsername());
        accountReq.setAmount(order.getAmount());
        remoteResult = accountInter.decreaseAccount(accountReq);
        if ("failure".equals(remoteResult)){
            throw new RuntimeException("decreaseAccount error");
        }
        log.info("==================payment结束=======================");
    }

    /**
     * paymentConfirm 失败时需要抛出异常
     * @param req
     */
    @Transactional(rollbackFor = Exception.class)
    public void paymentConfirm(PaymentOrderReq req){
        log.info("==================paymentConfirm开始=======================");
        Order dbOrder = orderMapper.selectById(req.getOrderId());
        if(dbOrder != null){
            dbOrder.setStatus(4);
            dbOrder.setUpdateTime(new Date());
            int influence = orderMapper.updateById(dbOrder);
            if(influence < 1){
                throw new MiloException("paymentConfirm exception");
            }
        }else{
            log.warn("confirm找不到订单，req:{}",req);
        }
        log.info("==================paymentConfirm结束=======================");
    }

    /**
     * paymentCancel 失败时需要抛出异常
     * @param req
     */
    @Transactional(rollbackFor = Exception.class)
    public void paymentCancel(PaymentOrderReq req){
        log.info("==================paymentCancel开始=======================");
        Order dbOrder = orderMapper.selectById(req.getOrderId());
        if(dbOrder != null){
            dbOrder.setStatus(3);
            dbOrder.setUpdateTime(new Date());
            int influence = orderMapper.updateById(dbOrder);
            if(influence < 1){
                throw new MiloException("paymentCancel exception");
            }
        }else{
            log.warn("cancel找不到订单，req:{}",req);
        }
        log.info("==================paymentCancel结束=======================");
    }
```

##### **拦截定义 TCC 注解**

接着我们可以通过 Spring AOP 去拦截注解，这部分内容 spring 的基础知识，定义 @Pointcut 拦截标注有 @MiloTCC 注解的方法，拦截之后通过 @Around 将拦截之后交给 miloTransactionAspectHandler 处理。

```
    /**
     * 拦截MiloTCC注解 {@linkplain MiloTCC}
     */
    @Pointcut("@annotation(com.luke.milo.core.annotation.MiloTCC)")
    public void interceptPointcut(){
    }

    /**
     * 拦截器处理方法
     * @param pjp
     * @return
     * @throws Throwable
     */
    @Around("interceptPointcut()")
    public Object interceptPointcutHandleMethod(final ProceedingJoinPoint pjp) throws Throwable {
        return miloTransactionAspectHandler.handleAspectPointcutMethod(pjp);
    }
```

##### **定义控制流程**

为了能够更加方便地控制整个分布式事务过程，我将整个 TCC 方案流程分为四个阶段，分别是 READY、TRYING、CONFIRMING 和 CANCELING。

- READY 阶段负责创建事务日志，事务上下文和当前线程绑定（利用线程本地 ThreadLocal 实现）。
- TRYING 阶段负责执行各个参与者（分布式事务每一个 try 都是一个参与者，例如下单参与者由三个方法组成，cloud-order 的下单方法、cloud-stock 的扣减库存方法和 cloud-account 的扣减账户方法）的 try 方法，同时 feign 远程调用时需要传递事务上下文（通过在 HTTP 的 Header 添加一个参数值）。
- CONFIRMING 阶段负责执行各个参与者对应的 confirm 方法，执行完后删除事务日志。
- CANCELING 阶段负责处理出现异常错误时执行参与者对应的 cancel 方法，执行完后删除事务日志。

事务上下文包含了整个事务的 id，阶段枚举和角色枚举，传递事务上下文能够让事务日志的主键相同，每一次的分布式事务业务操作都有一个唯一的事务 id，称为 transId。

```
public enum  MiloPhaseEnum {

    /**
     * TCC READY阶段
     */
    READY(0,"READY准备阶段"),

    /**
     * TCC try阶段
     */
    TRYING(1,"try阶段"),

    /**
     * TCC confirm阶段
     */
    CONFIRMING(2,"confirm阶段"),

    /**
     * TCC cancel阶段
     */

    CANCELING(3,"cancel阶段");
/**
 * @Descrtption TCC事务上下文
 * @Author luke
 * @Date 2019/9/18
 **/
@Data
public class MiloTransactionContext implements Serializable {

    private static final long serialVersionUID = 5712013353357711388L;

    /**
     * 事务id
     */
    private String transId;

    /**
     * 事务执行阶段 {@linkplain MiloPhaseEnum}
     */
    private int phase;

    /**
     * 事务参与的角色 {@linkplain MiloRoleEnum}
     */
    private int role;

}
```

同时我还定义了 TCC 事务的角色，每一个事务参与者（订单服务的支付方法、库存服务的扣减方法和账户服务的扣减方法）又细分为发起者、消费者和提供者。订单服务的支付方法就是发起者 INITIATOR，是整个分布式事务的入口方法，支付方法里面通过 feign 远程调用的代理方法被称为消费者 CONSUMER，对应的实现方库存服务和账户服务的扣减方法就被称为提供者 PROVIDER。

```
public enum MiloRoleEnum {

    /**
     * TCC事务发起者（分布式事务事务的发起的调用）
     */
    INITIATOR(1, "发起者"),

    /**
     * TCC事务消费者（分布式事务跨进程rpc的调用）
     */
    CONSUMER(2, "消费者"),

    /**
     * TCC事务提供者（分布式事务rpc调用的实现方）
     */
    PROVIDER(3, "提供者"),

    /**
     * TCC事务提供者(本地嵌套事务) TODO 嵌套事务
     */
    NESTER(4,"嵌套者");
```

每一个微服务都有一个对应的相同表结构的事务日志表，用于保存事务日志。invocation 是参与者反射调用的集合序列化值，对于订单服务事务日志来说 invocation 包含了三个参与者，而对于库存服务和账户服务来说则只有自己本身。retried_times 是用于记录自恢复定时任务处理遗留的事务日志（遗留的事务日志主要是没办法执行整个 TCC 流程时留下的，例如网络出现问题、服务宕机问题等）的重试次数，当超过最大重试次数时，需要通过人工介入解决。

![在这里插入图片描述](https://images.gitbook.cn/9ff45560-051a-11ea-a521-696a4b4a1d50)

##### **编写发起者控制流程**

我定义了三个事务处理器分别对应前面定义的角色枚举类的三个值 InitiatorMiloTransactionHandler（发起者事务处理器）、ConsumerMiloTransactionHandler（消费者事务处理器）和 ProviderMiloTransactionHandler（提供者事务处理器）。Spring AOP 拦截到下单服务之后，拦截处理器将会根据事务上线文去选择对应的事务处理器去处理。对于下单服务操作时事务上下文是空值，同时 HTTP 的 Header 也没有事务上下文，最终会路由到 InitiatorMiloTransactionHandler。

![在这里插入图片描述](https://images.gitbook.cn/16e37fa0-051d-11ea-bd6d-cf3c3ec89ddf)

![在这里插入图片描述](https://images.gitbook.cn/f3e696e0-051c-11ea-a7e5-67381fcbb097)

![在这里插入图片描述](https://images.gitbook.cn/f54ee400-051d-11ea-8d4f-79586e558671)

事务发起者控制整个 TCC 的执行过程，tryPhase 执行 try 方法，执行成功之后执行 confirmPhase（处理 confirm 阶段），出现异常则执行 cancelPhase（处理 cancel 阶段）。 try 阶段负责创建保存事务日志，创建事务上下文并绑定当前线程，执行 try 方法，更新事务日志状态为 try，表示 try 阶段成功完成。

![在这里插入图片描述](https://images.gitbook.cn/9719f540-051e-11ea-bd6d-cf3c3ec89ddf)

![在这里插入图片描述](https://images.gitbook.cn/bf09d880-051f-11ea-8d4f-79586e558671)

confirmPhase 方法负责通过从数据库将事务日志取出获得参与者，对于本地服务则反射调用对应的 @MiloTCC 定义的 confirm 目标方法，而对于远程则是反射调用 feign 代理方法同时传递事务上下文。cancelPhase 方法和 confirmPhase 方法类似，只是执行目标换成了 cancel 方法。

![在这里插入图片描述](https://images.gitbook.cn/2058e310-0520-11ea-a521-696a4b4a1d50)

##### **编写消费者控制流程**

当发起远程调用时（例如订单服务下单时需要远程调用库存服务和账户服务），事务上下文不为空且事务角色为事务发起者，根据前面的事务拦截器代码会修改角色为 CONSUMER，最终会路由到 ConsumerMiloTransactionHandler。每次 feign 调用都会添加一个参与者，更新到事务日志，为后面处理 confirm 和 cancel 做准备。在 confirm 和 cancel 阶段时则不需要做特殊的处理（发起者通过 feign 远程调用提供者）。

![在这里插入图片描述](https://images.gitbook.cn/4830dc10-0522-11ea-a7e5-67381fcbb097)

##### **编写提供者控制流程**

当服务的提供者接收到 feign 的远程调用（库存服务和账户服务）时，事务拦截器可以从 Header 获取到事务日志，将角色修改为 PROVIDER，最后会路由到 ProviderMiloTransactionHandler。在 try 阶段事务提供者需要创建自己服务的事务日志，同时事务日志的 id 值从事务上下文中获取，确保整个事务过程拥有相同事务 id，接着执行提供者的 try 方法，该方法同样需要标明 @MiloTCC 注解。在 confirm 阶段时将通过事务日志执行到扣减方法对应的 confirm 方法，cancel 阶段类似，执行成功后删除事务日志，否则仍然保留事务日志，所以也就意味着 confirm 方法和 cancel 方法会有被重复调用的可能性，这也就是前面提到的需要使用者实现 confirm 方法和 cancel 方法幂等性的原因。

![在这里插入图片描述](https://images.gitbook.cn/2a543f60-0523-11ea-bd6d-cf3c3ec89ddf)

![在这里插入图片描述](https://images.gitbook.cn/3c613820-0523-11ea-8d4f-79586e558671)

![在这里插入图片描述](https://images.gitbook.cn/ab3d58a0-0523-11ea-8d4f-79586e558671)

##### **自恢复定时任务**

自恢复定时任务会在框架启动器启动的时候开始定时执行，其目的就是处理超时遗留的事务日志。通过查询出没有超过最大处理次数同时超时的事务日志，处理逻辑其实很简单，那就是事务日志阶段值如果是 CONFIRMING 则执行 confirm 流程，否则执行 cancel 流程。执行过程和前面说的相同，也是通过从事务日志中获取事务参与者，接着通过反射执行目标类目标方法。

![在这里插入图片描述](https://images.gitbook.cn/f5847950-0525-11ea-a521-696a4b4a1d50)

![在这里插入图片描述](https://images.gitbook.cn/099139b0-0526-11ea-8d4f-79586e558671)

### 总结

这篇文章从介绍分布事务产生、实现分布式事务的理论知识，到 TCC 解决落地方案、举例用户下单业务流图，到最后分析了作者自己是如何实现分布式事务框架的。读者朋友们如果对 Milo 感兴趣的话可以到码云上找到[源码](https://gitee.com/luke2017/milo)，最后希望读者朋友们可以从这篇文章中有所收获，早日攻克分布式事务难题。