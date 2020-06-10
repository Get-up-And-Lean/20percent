# 大数据实时计算引擎 Flink 实战与性能优化

抢先一步掌握它，你就是大数据行业的领头羊

专栏介绍

### 专栏亮点

- 全网首个使用最新版本 **Flink 1.9** 进行内容讲解（该版本更新很大，架构功能都有更新），领跑于目前市面上常见的 Flink 1.7 版本的教学课程。
- 包含大量的**实战案例和代码**去讲解原理，有助于读者一边学习一边敲代码，达到更快，更深刻的学习境界。目前市面上的书籍没有任何实战的内容，还只是讲解纯概念和翻译官网。
- 在专栏高级篇中，根据 Flink 常见的项目问题提供了**排查和解决的思维方法**，并通过这些问题探究了为什么会出现这类问题。
- 在实战和案例篇，围绕大厂公司的**经典需求**进行分析，包括架构设计、每个环节的操作、代码实现都有一一讲解。

### 为什么要学习 Flink？

随着大数据的不断发展，对数据的及时性要求越来越高，实时场景需求也变得越来越多，主要分下面几大类：

![img](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/zL93nD.jpg)

为了满足这些实时场景的需求，衍生出不少计算引擎框架。**现有市面上的大数据计算引擎的对比**如下图所示：

![img](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-07-07-063048.jpg)

可以发现无论从 Flink 的架构设计上，还是从其功能完整性和易用性来讲都是领先的，再加上 Flink 是**阿里巴巴主推**的计算引擎框架，所以从去年开始就越来越火了！

目前，阿里巴巴、腾讯、美团、华为、滴滴出行、携程、饿了么、爱奇艺、有赞、唯品会等大厂都已经将 Flink 实践于公司大型项目中，带起了一波 Flink 风潮，**势必也会让 Flink 人才市场产生供不应求的招聘现象。**

### 专栏内容

![avatar](https://images.gitbook.cn/Fk0F7MAGZ19D1vkzSzDrMiBkBENJ)

#### **预备篇**

介绍实时计算常见的使用场景，讲解 Flink 的特性，并且对比了 Spark Streaming、Structured Streaming 和 Storm 等大数据处理引擎，然后准备环境并通过两个 Flink 应用程序带大家上手 Flink。

#### **基础篇**

深入讲解 Flink 中 Time、Window、Watermark、Connector 原理，并有大量文章篇幅（含详细代码）讲解如何去使用这些 Connector（比如 Kafka、ElasticSearch、HBase、Redis、MySQL 等），并且会讲解使用过程中可能会遇到的坑，还教大家如何去自定义 Connector。

#### **进阶篇**

讲解 Flink 中 State、Checkpoint、Savepoint、内存管理机制、CEP、Table／SQL API、Machine Learning 、Gelly。在这篇中不仅只讲概念，还会讲解如何去使用 State、如何配置 Checkpoint、Checkpoint 的流程和如何利用 CEP 处理复杂事件。

#### **高级篇**

重点介绍 Flink 作业上线后的监控运维：如何保证高可用、如何定位和排查反压问题、如何合理的设置作业的并行度、如何保证 Exactly Once、如何处理数据倾斜问题、如何调优整个作业的执行效率、如何监控 Flink 及其作业？

#### **实战篇**

教大家如何分析实时计算场景的需求，并使用 Flink 里面的技术去实现这些需求，比如实时统计 PV／UV、实时统计商品销售额 TopK、应用 Error 日志实时告警、机器宕机告警。这些需求如何使用 Flink 实现的都会提供完整的代码供大家参考，通过这些需求你可以学到 ProcessFunction、Async I／O、广播变量等知识的使用方式。

#### **系统案例篇**

讲解大型流量下的真实案例：如何去实时处理海量日志（错误日志实时告警／日志实时 ETL／日志实时展示／日志实时搜索）、基于 Flink 的百亿数据实时去重实践（从去重的通用解决方案 --> 使用 BloomFilter 来实现去重 --> 使用 Flink 的 KeyedState 实现去重）。

![avatar](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-11-01-040805.png)

🔺 Flink 专栏思维导图



#### 多图讲解 Flink 知识点

![avatar](https://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/images/jvnREW.jpg)

🔺 Flink 支持多种时间语义



![Flink 提供灵活的窗口](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-05-19-074304.jpg)

🔺 Flink 提供灵活的窗口



![Flink On YARN](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-04-27-004-flink-on-yarn.png)

🔺 Flink On YARN



![Flink Checkpoint](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-10-07-151554.jpg)

🔺 Flink Checkpoint



![Flink 监控](http://zhisheng-blog.oss-cn-hangzhou.aliyuncs.com/img/2019-10-04-134810.png)

🔺 Flink 监控



### 你将获得什么

- 掌握 Flink 与其他计算框架的区别
- 掌握 Flink Time／Window／Watermark／Connectors 概念和实现原理
- 掌握 Flink State／Checkpoint／Savepoint 状态与容错
- 熟练使用 DataStream／DataSet／Table／SQL API 开发 Flink 作业
- 掌握 Flink 作业部署／运维／监控／性能调优
- 学会如何分析并完成实时计算需求
- 获得大型高并发流量系统案例实战项目经验

### 适宜人群

- Flink 爱好者
- 实时计算开发工程师
- 大数据开发工程师
- 计算机专业研究生
- 有实时计算场景场景的 Java 开发工程师

### 作者介绍

#### 作者一

![zhisheng](https://images.gitbook.cn/fc23af30-ff7e-11e9-8e1b-015a97d00819)

#### 作者二

范瑞 现负责数据仓库的研发、集群维护及 Flink 实时流处理开发。两年内经历了公司数据量的爆炸式增长，从中收益良多。

### 名人推荐

知道 zhisheng 很早就开始研究 Flink 并分享了大量 Flink 的优质文章，这次终于盼来了系统的专栏。专栏不仅有入门实战内容，还有原理剖析和大量线上问题的性能调优等，最后竟然还分享了两个大型案例。

—— 芋道源码 公众号号主

### 购买须知

- 本专栏为图文内容，共计 49 篇；
- 付费用户可享受文章永久阅读权限；
- 本专栏为虚拟产品，一经付费概不退款，敬请谅解；
- 本专栏可在 GitChat 服务号、App 及网页端 [gitbook.cn](https://gitbook.cn/) 上购买，一端购买，多端阅读。

### 订阅福利

- **本专栏限时特价 69 元，11 月 25 日恢复至原价 99 元。**
- 订购本专栏可获得专属海报（在 GitChat 服务号领取），分享专属海报每成功邀请一位好友购买，即可获得 25% 的返现奖励，多邀多得，上不封顶，立即提现；
- 提现流程：在 GitChat 服务号中点击「我-我的邀请-提现」；
- ①点击这里跳转至》[第 3 篇](https://gitbook.cn/gitchat/column/5dad4a20669f843a1a37cb4f/topic/5db69dd3f6a6211cb96164f4)《翻阅至文末获得入群口令。
- 

- ②购买本专栏后，服务号会自动弹出入群二维码和入群口令。如果你没有收到那就先关注微信服务号「GitChat」，或者加我们的小助手「GitChatty6」咨询。