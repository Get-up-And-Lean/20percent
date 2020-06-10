## 课程内容

| 序号 | 第一天           | 第二天         |
| ---- | ---------------- | -------------- |
| 1    | Tomcat基础       | Web应用配置    |
| 2    | Tomcat架构       | Tomcat管理配置 |
| 3    | Jasper           | JVM配置        |
| 4    | Tomcat服务器管理 | Tomcat集群     |
| 5    |                  | Tomcat安全     |
| 6    |                  | Tomcat性能调优 |
| 7    |                  | Tomcat附加功能 |

## 1. Tomcat基础

### 1.1 web概念

1) 软件架构

- CS
- BS

2) 资源分类

- 静态资源：所有用户访问，得到的结果都是一样的。静态资源可以被浏览器直接解析。
- 动态资源

3) 网络通信三要素

- IP
- 端口
- 传输协议：规定了数据传输的规则
  - 1.基础协议
    - TCP：安全协议，三次握手，速度稍慢
    - UDP：不安全协议，速度快

### 1.2 常见的Web服务器

### 1.3 Tomcat历史

1）Tomcat最初是由SUN公司软件架构师James Duncan Davidson开发，名称为"JavaWebServer"。

2）1999年，在Davidson的帮助下，该项目于1999年于Apache软件基金会旗下的JServ项目合并，并发布了第一个版本（3.X），既是现在的Tomcat，该版本实现了Servlet2.2和JSP1.1规范。

3）2001年，Tomcat发布了4.0版本，作为里程碑式的版本，Tomcat完全重新设计了其架构，并实现了Servlet2.3和JSP1.2规范。



目前Tomcat已经更新到9.0.X版本，但是目前企业中主要使用的7.X和8.X居多。本系列专题基于8.5版本展开。



### 1.4 Tomcat安装

### 1.5 Tomcat目录结构

| 目录    | 目录下的文件        | 说明                                                         |
| ------- | ------------------- | ------------------------------------------------------------ |
| bin     | /                   | 存放启动停止等脚本文件                                       |
| conf    | /                   | 存放配置文件                                                 |
|         | Catalina            | 存储针对每个虚拟机的Context配置                              |
|         | context.xml         | 定义所有web应用均需加载的Context配置，如果web应用指定了自己的context.xml，该文件将被覆盖 |
|         | catalina.properties | Tomcat环境变量配置                                           |
|         | catalina.policy     | 运行的安全策略                                               |
|         | logging.properties  | 日志配置文件，可以修改日志级别，日志路径                     |
|         | server.xml          | 核心配置文件                                                 |
|         | tomcat-users.xml    | 定义Tomcat默认的用户及用户映射信息配置                       |
|         | web.xml             | 所有应用默认的部署描述文件，主要定义了基础的Servlet和MIME映射 |
| lib     | /                   | 依赖的包                                                     |
| logs    | /                   | 默认的日志存放目录                                           |
| webapps | /                   | 默认的应用部署目录                                           |
| work    | /                   | Web应用JSP代码生成和编译的临时目录                           |

### 1.6 Tomcat启动停止

### 1.7 Tomcat源码

