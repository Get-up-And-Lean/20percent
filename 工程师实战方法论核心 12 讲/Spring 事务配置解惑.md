# Spring 事务配置解惑

### 一、前言

事务是数据库区别于文件系统的一个重要特征，数据库通过事务保证了数据库中数据的完整性，也就是一个事务内的 N 多操作要么全部都提交，要么全部都回滚。在 Spring 框架中使用事务，我们需要在 XML 里面配置好多 Bean，而这些 Bean 背后都做了哪些事情那，并不是每个人都清楚。通过本场 Chat 您将能弄清楚在 XML 文件里面配置事务时，每个配置项 Bean 背后究竟在做些什么？从而对 Spring 事务配置能够达到知其然，也知其所以然，避免入坑。

本 Chat 内容如下：

- 当您在 XML 里面配置了一个 SqlSessionFactoryBean 后，其究竟做了什么？
- 当您在 XML 里面配置了一个 MapperScannerConfigurer 后，其究竟做了什么?
- 上面两个 Bean 如何通过动态代理生成数据库操作类的？
- 当您执行 Mapper 接口的查询方法后，发生了什么？
- `<tx:advice/>`、`<aop:config>` 标签如何创建事务切面的？
- `<tx:annotation-driven/>` 标签添加后为何就可以使用注解式事务了？

### 二、Spring 事务配置

一般我们会在 datasource.xml 中进行事务的配置，配置内容如下：

```java
?xml version="1.0" encoding="UTF-8" standalone="no"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd   
                        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd   
                        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd   
                        http://www.springframework.org/schema/jee http://www.springframework.org/schema/jee/spring-jee-4.0.xsd   
                        http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd">

    <!-- (1) 数据源 -->
    <bean id="dataSource"
        class="com.alibaba.druid.pool.DruidDataSource" init-method="init"
        destroy-method="close">
        <property name="driverClassName"
            value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/test" />
        <property name="username" value="root" />
        <property name="password" value="123456" />
        <property name="maxWait" value="3000" />
        <property name="maxActive" value="28" />
        <property name="initialSize" value="2" />
        <property name="minIdle" value="0" />
        <property name="timeBetweenEvictionRunsMillis" value="300000" />
        <property name="testOnBorrow" value="false" />
        <property name="testWhileIdle" value="true" />
        <property name="validationQuery" value="select 1 from dual" />
        <property name="filters" value="stat" />
    </bean>

    <!-- (2) session工厂 -->
    <bean id="sqlSessionFactory"
        class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="mapperLocations"
            value="classpath*:mapper/*Mapper*.xml" />
        <property name="dataSource" ref="dataSource" />
    </bean>

    <!-- (3) 配置扫描器，扫描指定路径的mapper生成数据库操作代理类 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="annotationClass"
            value="javax.annotation.Resource"></property>
        <property name="basePackage" value="com.zlx.user.dal.sqlmap" />
        <property name="sqlSessionFactory" ref="sqlSessionFactory" />
    </bean>

    <!-- (4) 配置事务管理器 -->
    <bean id="transactionManager"
        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource" />
    </bean>

    <!-- (5)该配置创建了一个TransactionInterceptor的bean，作为事务切面的执行方法 -->
    <tx:advice id="defaultTxAdvice" transaction-manager="transactionManager">
        <tx:attributes>
            <tx:method name="*" rollback-for="Exception" />
        </tx:attributes>
    </tx:advice>

    <!-- (6) 该配置创建了一个类型为DefaultBeanFactoryPointcutAdvisor的bean，该bean是一个advisor,里面包含了pointcut和advice。pointcut说明切面加在哪里，advice说明具体执行逻辑。此处可以配多个advisor -->
    <aop:config>
        <aop:pointcut id="myCut"
            expression="(execution(* *..*BoImpl.*(..))) " />
        <aop:advisor pointcut-ref="myCut"
            advice-ref="defaultTxAdvice" />
    </aop:config>

    <!-- (7) 声明使用注解式事务 -->
    <tx:annotation-driven
        transaction-manager="transactionManager" />

    <!-- (8) 注册各种beanfactory处理器 -->
    <context:annotation-config />

</beans>
```

- 其中（1）是配置数据源，这里使用了 druid 连接池，用户可以根据自己的需要配置不同的数据源，也可以选择不适用数据库连接池，而直接使用具体的物理连接。
- 其中（2）创建 sqlSessionFactory，用来在（3）时候使用。
- 其中（3）配置扫描器，扫描指定路径的 mapper 生成数据库操作代理类
- 其中（4） 配置事务管理器
- 其中（5） 创建了一个 TransactionInterceptor 的 bean，作为事务切面的执行方法
- 其中（6）创建了一个类型为 DefaultBeanFactoryPointcutAdvisor 的 bean，该 bean 是一个 advisor，里面包含了pointcut和advice
- 其中（7） 声明使用注解式事务

### 三、Demo 结构

demo 结构如下图：

![image.png](https://upload-images.jianshu.io/upload_images/5879294-a00f1dc2e0d1d68e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

demo地址：https://github.com/zhailuxu/transaction-config

第一部分 boImpl 类：

- 其中 TestTransactionProgagationCourseImpl 用来操作 course 表，代码如下：

```Java
@Configuration
public class TestTransactionProgagationCourseImpl implements TestTransactionProgagationCourseBo{

    @Autowired
    private CourseDOMapper  courseDOMapper;

    @Override
    public boolean insertCourse(CourseDO course) {

        return courseDOMapper.insert(course)==1?true:false;

    }
}
```

- 其中 TestTransactionProgagationUserImpl 用来操作 user 表，代码如下：

```Java
@Configuration
public class TestTransactionProgagationUserImpl implements TestTransactionProgagationUserBo {

    @Autowired
    private UserDOMapper userDOMapper;
    @Autowired
    private TestTransactionProgagationCourseBo testTransactionProgagationCourseBo;

    @Override
    public boolean insertUser(UserDO user) {
        boolean result = userDOMapper.insert(user) == 1 ? true : false;


        return result;
    }
```

- 其中 UserManagerBoImpl 是个门面类，内部调用 TestTransactionProgagationUserImpl，代码如下：

```java
@Configuration
public class UserManagerBoImpl implements UserManagerBo{

    @Autowired
    TestTransactionProgagationUserBo  testTransactionProgagationUserBo;

    @Override
    public boolean addNewUser(UserDO userDO) {

        return testTransactionProgagationUserBo.insertUser(userDO);
    }

}
```

第二部分使用 mybaits generator 自动生成的 mapper 接口、xml、`*DOExample`、`*DO` 等类，可知这里每个表对应了一个 mapper.xml 和 mapper 接口类。

第三部分是 SpringBoot 应用的启动类：

```java
public class App {

    @Autowired
    private UserManagerBo userManagerBo;

    @Autowired
    private TestTransactionProgagationUserBo userBo;

    @Autowired
    private TestTransactionProgagationCourseBo courseBo;

    @RequestMapping("/home")
    String home() {
        return "Hello World!";
    }

    @RequestMapping("/addUser")
    String inserUser() {

        UserDO user = new UserDO();
        user.setAge(10);

        return userBo.insertUser(user) + "";
    }

    @RequestMapping("/addCourse")
    String addCourse() {

        CourseDO course = new CourseDO();
        course.setUserId(1);
        course.setCourseName("java");
        return courseBo.insertCourse(course) + "";
    }

    @RequestMapping("/testTransaction")
    String testTransaction() {

        UserDO user = new UserDO();
        user.setAge(10);;

        return userManagerBo.addNewUser(user) + "";
    }

    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

另外 applicationContext.xml 的内容如下：

```Java
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd   
                        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd   
                        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd   
                        http://www.springframework.org/schema/jee http://www.springframework.org/schema/jee/spring-jee-4.0.xsd   
                        http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd">

    <!-- (1) 数据源 -->
    <bean id="dataSource"
        class="com.alibaba.druid.pool.DruidDataSource" init-method="init"
        destroy-method="close">
        <property name="driverClassName"
            value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/test" />
        <property name="username" value="root" />
        <property name="password" value="123456" />
        <property name="maxWait" value="3000" />
        <property name="maxActive" value="28" />
        <property name="initialSize" value="2" />
        <property name="minIdle" value="0" />
        <property name="timeBetweenEvictionRunsMillis" value="300000" />
        <property name="testOnBorrow" value="false" />
        <property name="testWhileIdle" value="true" />
        <property name="validationQuery" value="select 1 from dual" />
        <property name="filters" value="stat" />
    </bean>

    <!-- (2) session工厂 -->
    <bean id="sqlSessionFactory"
        class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="mapperLocations"
            value="classpath*:mapper/*Mapper*.xml" />
        <property name="dataSource" ref="dataSource" />
    </bean>

    <!-- (3) 配置扫描器，扫描指定路径的mapper生成数据库操作代理类 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="annotationClass"
            value="javax.annotation.Resource"></property>
        <property name="basePackage" value="com.zlx.user.dal.sqlmap" />
        <property name="sqlSessionFactory" ref="sqlSessionFactory" />
    </bean>

    <!-- (4) 配置事务管理器 -->
    <bean id="transactionManager"
        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource" />
    </bean>

    <!-- (5)该配置创建了一个TransactionInterceptor的bean，作为事务切面的执行方法 -->
    <tx:advice id="defaultTxAdvice">
        <tx:attributes>
            <tx:method name="*" rollback-for="Exception" />
        </tx:attributes>
    </tx:advice>

    <!-- (6) 该配置创建了一个类型为DefaultBeanFactoryPointcutAdvisor的bean，该bean是一个advisor,里面包含了pointcut和advice。pointcut说明切面加在哪里，advice说明具体执行逻辑。此处可以配多个advisor -->
    <aop:config>
        <aop:pointcut id="myCut"
            expression="(execution(* *..*BoImpl.*(..))) " />
        <aop:advisor pointcut-ref="myCut"
            advice-ref="defaultTxAdvice" />
    </aop:config>

    <!-- (7) 声明使用注解式事务 -->
    <tx:annotation-driven
        transaction-manager="transactionManager" />

    <!-- (8) 注册各种beanfactory处理器 -->
    <context:annotation-config />

</beans>
```

然后需要建立两个表：

用户表：

```Java
create table user(
id int AUTO_INCREMENT PRIMARY KEY,
age int not null,
);
```

课程表：

```Java
create table course(
id int AUTO_INCREMENT PRIMARY KEY,
user_id int not null,
course_name varchar(50) not null
);
```

其中课程表中 user_id 是 user 表 id 的外键。

### 四、SqlSessionFactory 内幕

第二节配置中配置 SqlSessionFactory 的方式如下：

```Java
<!-- (2) session工厂 -->
    <bean id="sqlSessionFactory"
        class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="mapperLocations"
            value="classpath*:mapper/*Mapper*.xml" />
        <property name="dataSource" ref="dataSource" />
    </bean>
```

其中 mapperLocations 配置 `*mapper.xml` 文件所在的路径，dataSource 配置数据源，下面我们具体来看 SqlSessionFactoryBean 的代码，SqlSessionFactoryBean 实现了 FactoryBean 和 InitializingBean 扩展接口，所以具有 getObject 和 afterPropertiesSet 方法（[具体可以参考](https://gitbook.cn/gitchat/activity/5a84589a1f42d45a333f2a8e)），下面我们从时序图具体看这两个方法内部做了什么：

![enter image description here](https://images.gitbook.cn/deec6410-85e1-11e8-8d3d-91bd9348394f)

如上时序图其中步骤（2）代码如下：

```Java
 protected SqlSessionFactory buildSqlSessionFactory() throws IOException {

    Configuration configuration;

    XMLConfigBuilder xmlConfigBuilder = null;
    ...
    //(3.1)
    if (this.transactionFactory == null) {
      this.transactionFactory = new SpringManagedTransactionFactory();
    }
    //(3.2)
    configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));
    //(3.3)
    if (!isEmpty(this.mapperLocations)) {
      for (Resource mapperLocation : this.mapperLocations) {
        if (mapperLocation == null) {
          continue;
        }

        try {
          XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
              configuration, mapperLocation.toString(), configuration.getSqlFragments());
          xmlMapperBuilder.parse();
        } catch (Exception e) {
          throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
        } finally {
          ErrorContext.instance().reset();
        }

        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Parsed mapper file: '" + mapperLocation + "'");
        }
      }
    } else {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Property 'mapperLocations' was not specified or no matching resources found");
      }
    }
   //3.9
    return this.sqlSessionFactoryBuilder.build(configuration);
  }
```

- 如上代码（3.1）创建了一个 Spring 事务管理工厂。
- 代码（3.2）设置 configuration 对象的环境变量，其中 dataSource 为 demo 中配置文件中创建的数据源。
- 代码（3.3）中 mapperLocations 是一个数组，为 demo 中配置文件中配置的满足 `classpath*:mapper/*Mapper*.xml` 条件的所有 `*mapper.xml` 文件，本 demo 会发现存在

```
[file[/Users/zhuizhumengxiang/workspace/mytool/distributtransaction/transactionconfig/transaction-demo/deep-learn-java/Start/target/classes/mapper/CourseDOMapper.xml]
file[/Users/zhuizhumengxiang/workspace/mytool/distributtransaction/transactionconfig/transaction-demo/deep-learn-java/Start/target/classes/mapper/UserDOMapper.xml]]
```

两个文件

代码（3.3）循环遍历每个 mapper.xml，然后调用 XMLMapperBuilder 的 parse 方法进行解析。

XMLMapperBuilder 的 parse 代码中 configurationElement 方法做具体解析，代码如下：

```Java
 private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      ...
      //(3.4)
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      //(3.5)
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      //(3.6)
      sqlElement(context.evalNodes("/mapper/sql"));
      //(3.7)
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
```

- 代码（3.4）解析 mapper.xml 中 /mapper/parameterMap 标签下内容，本 demo 中的 XML 文件中没有配置这个。
- 代码（3.5）解析 mapper.xml 中 /mapper/resultMap 标签下内容，然后存放到 Configuration 对象的 resultMaps 缓存里面，这里需要提一下，所有的 `*mapper.xml` 文件共享一个 Configuration 对象，所有 `*mapper.xml` 里面的 resultMap 都存放到同一个 Configuration 对象的 resultMaps 里面，其中 key 为 mapper 文件的 namespace 和 resultMap 的 id 组成，比如 UserDoMapper.xml：

```Java
<mapper namespace="com.zlx.user.dal.sqlmap.UserDOMapper" >
  <resultMap id="BaseResultMap" type="com.zlx.user.dal.dao.UserDO" >
    <id column="id" property="id" jdbcType="INTEGER" />
    <result column="age" property="age" jdbcType="INTEGER" />
  </resultMap>
```

其中 key 为 com.zlx.user.dal.sqlmap.CourseDOMapper.BaseResultMap，value 则为一个 map，map 里面是 column 与 property 的映射。

- 代码（3.6）解析 mapper.xml 中 /mapper/sql 下的内容，然后保存到 Configuration 对象的 sqlFragments 缓存中，sqlFragments 也是一个 map，比如 UserDoMapper.xml 中的一个 sql 标签：

```Java
<sql id="Base_Column_List" >
    id, age
</sql>
```

其中 key 为 `com.zlx.user.dal.sqlmap.CourseDOMapper.Base_Column_List`，value 作为一个记录 sql 标签内容的 XNode 节点。

- 代码（3.7）解析 mapper.xml 中 select|insert|update|delete 增删改查的语句，并封装为 MappedStatement 对象保存到 Configuration 的 mappedStatements 缓存中，mappedStatements 也是一个 map 结构，比如：比如 UserDoMapper.xml 中的一个 select 标签：

```Java
<select id="selectByPrimaryKey" resultMap="BaseResultMap" parameterType="java.lang.Integer" >
    select 
    <include refid="Base_Column_List" />
    from user
    where id = #{id,jdbcType=INTEGER}
  </select>
```

其中 key 为 com.zlx.user.dal.sqlmap.CourseDOMapper.selectByPrimaryKey，value 为标签内封装为 MappedStatement 的对象。

至此 configurationElement 解析 XML 的步骤完毕了，下面我们看时序图中步骤（12）bindMapperForNamespace 代码如下：

```java
 private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
      Class<?> boundType = null;
      try {
        boundType = Resources.classForName(namespace);
      } catch (ClassNotFoundException e) {
        //ignore, bound type is not required
      }
      if (boundType != null) {
        if (!configuration.hasMapper(boundType)) {
          //(3.8)
          configuration.addLoadedResource("namespace:" + namespace);
          configuration.addMapper(boundType);
        }
      }
    }
  }
```

其中代码（3.8）注册 mapper 接口的 Class 对象到 configuration 中的 mapperRegistry 管理的缓存 knownMappers 中，knownMappers 是个 map, 其中 key 为具体 mapper 接口的 Class 对象，value 为 mapper 接口的代理对象 MapperProxyFactory。

> 注：SqlSessionFactoryBean 作用是扫描配置的 mapperLocations 路径下的所有 mapper.xml 文件，并对其进行解析，然后把解析的所有 mapper 文件的信息保存到一个全局的 configuration 对象的具体缓存中，然后注册每个 mapper.xml 对应的接口类到 configuration 中，并为每个接口类生成了一个代理 bean。

然后时序图步骤 15 创建了一 DefaultSqlSessionFactory 对象，并且传递了上面全局的 configuration 对象。

步骤 16 则返回创建的 DefaultSqlSessionFactory 对象。

### 五、MapperScannerConfigurer 内幕

第二节中 MapperScannerConfigurer 的配置方式如下：

```Java
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="annotationClass"
            value="javax.annotation.Resource"></property>
        <property name="basePackage" value="com.zlx.user.dal.sqlmap" />
        <property name="sqlSessionFactory" ref="sqlSessionFactory" />
    </bean>
```

其中 sqlSessionFactory 设置为第 4 节创建的 DefaultSqlSessionFactory，basePackage 为 mapper 接口类所在目录，annotationClass 这是为注解 @Resource，后面会知道标示只扫描 basePackage 路径下标注 @Resource 注解的 mapper 接口类进行代理。

MapperScannerConfigurer 实现了 BeanDefinitionRegistryPostProcessor, InitializingBean 接口，所以会重写下面方法：

```java
（5.1）
//在bean注册到ioc后创建实例前修改bean定义和新增bean注册，这个是在context的refresh方法被调用
void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

(5.2)
//set属性设置后被调用
void afterPropertiesSet() throws Exception;
```

更多关于 Spring 扩展接口的知识可以移步（[Spring 框架常用扩展接口揭秘](https://gitbook.cn/gitchat/activity/5a84589a1f42d45a333f2a8e)）

下面我们从时序图看这看 postProcessBeanDefinitionRegistry 和 afterPropertiesSet 扩展接口里面都做了些什么：

![enter image description here](https://images.gitbook.cn/dc64e250-8643-11e8-8395-25a125cc1274)

其中 afterPropertiesSet 代码如下：

```Java
  public void afterPropertiesSet() throws Exception {
    notNull(this.basePackage, "Property 'basePackage' is required");
  }
```

可知是校验 basePackage 是否为 null，为 null 会抛出异常。因为 MapperScannerConfigurer 作用就是扫描 basePackage 路径下的 mapper 接口类然后生成代理，所以不允许 basePackage 为 null。

postProcessBeanDefinitionRegistry 的代码如下：

```Java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    ...
    //5.3
    scanner.setAnnotationClass(this.annotationClass);

    //5.4
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    ...
    //5.5
    scanner.registerFilters();
    //5.6
  scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
 }
```

- 代码（5.3）设置注解类，这里设置的为 @Resource 注解，（5.4）设置 sqlSessionFactory 到 ClassPathMapperScanner。
- 代码（5.5）根据设置的 @Resource 设置过滤器，代码如下：

```Java
public void registerFilters() {
    boolean acceptAllInterfaces = true;

    if (this.annotationClass != null) {
      addIncludeFilter(new AnnotationTypeFilter(this.annotationClass));
      acceptAllInterfaces = false;
    }

    ...
  }

public void addIncludeFilter(TypeFilter includeFilter) {
    this.includeFilters.add(includeFilter);
  }
```

可知具体是把 @Resource 注解作为了一个过滤器

- 代码（5.6）具体执行扫描，其中 basePackage 为我们设置的 com.zlx.user.dal.sqlmap，basePackage 设置的时候允许设置多个包路径并且使用 `,` `;` `\t\n`进行分割。加上上面的过滤条件，就是说对 basePackage 路径下标注 @Resource 注解的 mapper 接口类进行代理。

具体执行扫描的是 doScan 方法，其代码如下：

```Java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
        for (String basePackage : basePackages) {
        //具体扫描符合条件的bean
            Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
            for (BeanDefinition candidate : candidates) {
                ...
                if (checkCandidate(beanName, candidate)) {
                    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    definitionHolder =
                            AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                    beanDefinitions.add(definitionHolder);
                    //注册到IOC容器
                    registerBeanDefinition(definitionHolder, this.registry);
                }
            }
        }
        return beanDefinitions;
}
```

如上代码可知是对每个包路径分别进行扫描，然后对符合条件的接口 bean 注册到 IOC 容器。

这里我们看下 findCandidateComponents 的逻辑：

```Java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
        Set<BeanDefinition> candidates = new LinkedHashSet<>();
        try {
            //5.8
            String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                    resolveBasePackage(basePackage) + '/' + this.resourcePattern;
            Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
            ...
            //5.9
            for (Resource resource : resources) {
                if (traceEnabled) {
                    logger.trace("Scanning " + resource);
                }
                if (resource.isReadable()) {
                    try {
                        //5.10
                        MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                        if (isCandidateComponent(metadataReader)) {
                            ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                            sbd.setResource(resource);
                            sbd.setSource(resource);
                            if (isCandidateComponent(sbd)) {
                                //5.11
                                candidates.add(sbd);
                            }
                            else {

                            }
                        }
                        ...
                    }
                    ...
                }
                ...
            }
        }
        ...
        return candidates;
    }
```

如上代码其中（5.8）是根据我们设置的 basePackage 得到一个扫描路径，这里根据我们 demo 设置的值，拼接后 packageSearchPath为 `classpath*:com/zlx/user/dal/sqlmap/**/*.class`，这里扫描出来的文件为：

```Java
file[/Users/zhuizhumengxiang/workspace/mytool/distributtransaction/transactionconfig/transaction-demo/deep-learn-java/Start/target/classes/com/zlx/user/dal/sqlmap/CourseDOMapper.class]
file[/Users/zhuizhumengxiang/workspace/mytool/distributtransaction/transactionconfig/transaction-demo/deep-learn-java/Start/target/classes/com/zlx/user/dal/sqlmap/CourseDOMapperNoAnnotition.class]
file[/Users/zhuizhumengxiang/workspace/mytool/distributtransaction/transactionconfig/transaction-demo/deep-learn-java/Start/target/classes/com/zlx/user/dal/sqlmap/UserDOMapper.class]
```

然后 isCandidateComponent 方法执行具体对上面扫描到的文件进行过滤，其代码：

```Java
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
        ...
        for (TypeFilter tf : this.includeFilters) {
            if (tf.match(metadataReader, getMetadataReaderFactory())) {
                return isConditionMatch(metadataReader);
            }
        }
        return false;
}
```

上面我们讲解过添加了一个 @Resource 注解的过滤器，这里执行时候器 match 方法如下：

```java
public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
            throws IOException {

        if (matchSelf(metadataReader)) {
            return true;
        }

        ...
        return false;

}
    //判断接口类是否有@Resource注解
    protected boolean matchSelf(MetadataReader metadataReader) {
        AnnotationMetadata metadata = metadataReader.getAnnotationMetadata();
        return metadata.hasAnnotation(this.annotationType.getName()) ||
                (this.considerMetaAnnotations && metadata.hasMetaAnnotation(this.annotationType.getName()));
    }
```

经过过滤后 CourseDOMapperNoAnnotition.class 接口类被过滤了，因为其没有标注 @Resource 注解。只有 CourseDOMapper 和 UserDOMapper 两个标注 @Resource 的类注册到了 IOC 容器。

如上时序图注册后，还需要执行 processBeanDefinitions 对满足过滤条件的 CourseDOMapper 和 UserDOMapper 的 bean 定义进行修改，以便生成代理类，processBeanDefinitions 代码如下：

```Java
 private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();

      // （5.12）
      definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue #59
      definition.setBeanClass(this.mapperFactoryBean.getClass());

     ...
     //5.13
     if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }
     ...
    }
  }
```

如上代码（5.12）修改 bean 定义的 BeanClass 为 MapperFactoryBean，然后设置 MapperFactoryBean 的泛型构造函数参数为真正的被代理接口。也就是如果当前 bean 定义是 com.zlx.user.dal.sqlmap.CourseDOMapper 接口的，则设置当前 bean 定义的 BeanClass 为 MapperFactoryBean，并设置 com.zlx.user.dal.sqlmap.CourseDOMapper 为 MapperFactoryBean 的构造函数参数。

代码（5.13）设置 session 工厂到 bean 定义。

> 注：MapperScannerConfigurer 的作用是扫描指定路径下的 Mapper 接口类，并且可以制定过滤策略，然后对符合条件的 bean 定义进行修改以便在 bean 创建时候生成代理类，最终符合条件的 mapper 接口都会被转换为 MapperFactoryBean，MapperFactoryBean 中并且维护了第 4 节生成的 DefaultSqlSessionFactory。

### 六、Mapper 接口的代理类生成

上节我们说了最终所有符合过滤条件的 mapper 接口类都被转换为了 MapperFactoryBean 并注册到了 IOC 容器，本节我们看 MapperFactoryBean 类是如何对 mapper 接口进行代理，生成数据库操作类的。

上节我们说过通过 MapperFactoryBean 的构造函数把具体的 mapper 接口放到了 MapperFactoryBean 内部：

```Java
  public MapperFactoryBean(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }
```

所以对本文 demo 来说 IOC 容器里面有两个 MapperFactoryBean 实例，其里面对应的 mapperInterface 接口分别为 CourseDOMapper 和 UserDOMapper。

另外上节我们提到会设置 DefaultSqlSessionFactory 到 MapperFactoryBean，其实是调用 MapperFactoryBean 的 set 方法进行设置的，其代码为：

```Java
   public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
    if (!this.externalSqlSession) {
      this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
    }
  }
```

如上可知其内部是使用 DefaultSqlSessionFactory 作为构造函数创建了 SqlSessionTemplate，并保证到了内部变量 sqlSession 里面。

MapperFactoryBean 类实现了 FactoryBean 接口，所以其是一个工厂 Bean，工厂 bean 最终会通过其 getObject 返回一个真正的 bean 并注入到 IOC 容器，其代码：

```Java
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
```

如上代码这里 getSqlSession() 返回的就是 SqlSessionTemplate，mapperInterface 是具体的 mapper 接口类，本 demo 中是指 CourseDOMapper 或者 UserDOMapper。

`getSqlSession().getMapper(this.mapperInterface)` 最终会调用 DefaultSqlSessionFactory 中的 Configuration 对象的 getMapper 接口获取具体 mapper 接口的代理类 Configuration 在第四节介绍过）。

Configuration 的 getMapper 接口：

```Java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    ...
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {...
    }
  }
```

可知会从 knownMappers 缓存获取第 4 节讲解的缓存到的接口对应的工厂代理类。

然后调用工厂代理类 mapperProxyFactory 的 newInstance 方法创建 mapper 接口的代理类：

```Java
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

    protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
```

可知是使用 JDK 动态代理来生成 mapper 接口的代理的，并且 InvocationHandler 为 MapperProxy。

> 注：可知 SqlSessionFactoryBean 的作用是扫描指定路径的 mapper.xml 文件并进行解析，解析完后放入到全局 Configuration 的缓存里面，然后每个 mapper.xml 缓存一个 mapperProxyFactory 代理对象，然后返回一个 DefaultSqlSessionFactory 对象。MapperScannerConfigurer 作用是扫描指定路径下 mapper 接口类，然后从 DefaultSqlSessionFactory 中的 Configuration 中获取接口类对应的 mapperProxyFactory 类，然后使用动态代理生成代理类。

需要注意的是 SqlSessionFactoryBean 的扫描路径范围要大于等于 MapperScannerConfigurer 的扫描范围，否者后者在 DefaultSqlSessionFactory 中的 Configuration 查找接口类对应的 mapperProxyFactory 类时候会找不到而报错。

另外需要注意 mapper.xml 里面的 namespace 一定要和 mapper 接口相同。

### 七、当执行 Mapper 接口的方法后，发生了什么

上节我们讲解了每个 mapper 接口类最终会生成一个代理类，当我们调用 mapper 接口的查询方法后，方法会被 MapperProxy 拦截，然后执行具体的 sql 操作，下面我们看 MapperProxy 的 invoke 代码：

```Java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
    ...
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

其中 MapperMethod 的 execute 代码：

```Java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
   ...
    return result;
  }
```

可知是具体是调用了 sqlSession 的方法进行进行数据库操作。

### 八、`<tx:advice/>`、`<aop:config>` 标签如何创建事务切面的

`<tx:advice/>` 标签作用是创建一个 TransactionInterceptor，作为事务切面的通知方法。在 Spring 中(可以参考：[Spring 框架之 AOP 原理剖析](https://gitbook.cn/gitchat/activity/5a8fdf6bf2e5dc2ca621a937)） Advisor 这个概念是从 Spring 1.2 的 AOP 支持中提出的，一个 Advisor 相当于一个小型的切面，不同的是它只有一个通知（Advice），Advisor 中还包含一个 pointcut(切点)，切点定义了对那些方法进行拦截，而通知是具体对拦截到的方法进行增强的逻辑。

具体对 `<tx:advice/>` 标签进行解析的是 TxAdviceBeanDefinitionParser，其时序图如下：

![enter image description here](http://upload-images.jianshu.io/upload_images/5879294-18712ffa5f7cd017.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先 TxAdviceBeanDefinitionParser 有 getBeanClass 方法代码：

```Java
    protected Class<?> getBeanClass(Element element) {
        return TransactionInterceptor.class;
    }
```

这说明该标签解析后生成的是 TransactionInterceptor 对象的 bean 定义。

- 其中时序图中步骤（2）是设置配置 demo 的 XML 配置文件里面创建的事务管理器到 TransactionInterceptor 对象

![image.png](https://upload-images.jianshu.io/upload_images/5879294-e7f6208b79c727a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 时序图（4）~（10）则解析 `<tx:advice/>` 标签中事务属性值设置到 TransactionInterceptor 对象里面属性里面。

> 注：也就是 `<tx:advice/>` 标签的作用是生成一个 TransactionInterceptor 拦击器对象，并设置该对象的一些事务属性，然后该对象将作为事务切面的通知方法。

`<aop:config>` 标签作用是创建一个DefaultBeanFactoryPointcutAdvisor（其实现了 Advisor 接口）对象作为作一个 Advisor，前面说了一个 Advisor 就是一个小型的切面，所以其中定义了切点和通知。该标签是 ConfigBeanDefinitionParser 类进行解析的，其时序图如下：

![enter image description here](http://upload-images.jianshu.io/upload_images/5879294-ac4939d5e25e09e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 时序图中步骤（2）创建了一个 DefaultBeanFactoryPointcutAdvisor 对象的 bean 定义，步骤（3）（4）则是设置上面创建的通知对象 (TransactionInterceptor) 到该 Advisor。
- 时序图中步骤（8）则是解析标签中的切点表达式，然后设置到 DefaultBeanFactoryPointcutAdvisor 对象的 bean 定义。
- 时序图步骤（4）注册了一个 AspectJAwareAdvisorAutoProxyCreator 到 Spring 容器，作用就是对满足 pointcut 表达式的类的方法进行代理，并且使用 advice 进行拦截处理，而 advice 就是事务拦截器。

由于 AspectJAwareAdvisorAutoProxyCreator 类实现了 BeanPostProcessor 接口，所以具有 postProcessAfterInitialization 方法，而对符合切点的方法进行代理就是在该方法内的 wrapIfNecessary 方法：

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        ...
        // 8.1
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
        if (specificInterceptors != DO_NOT_PROXY) {
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            //8.2
            Object proxy = createProxy(
                    bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
        }

        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
```

其中8.1查找所有可以对当前 bean 进行增强的切面，其中有一个条件就是看那些bean实现了 Advisor 接口，而 `<aop:config>` 标签作用是创建一个 DefaultBeanFactoryPointcutAdvisor，并且其实现了 Advisor 接口，所以这里会使用 DefaultBeanFactoryPointcutAdvisor 切面，然后会看当前 bean 的方法是否满足切面的切点表达式，具体是 AopUtils 的 canApply 方法进行判断：

![image.png](https://upload-images.jianshu.io/upload_images/5879294-323bb48eca2aa56b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果满足则执行 8.2 对方法进行代理,这里会对 TestTransactionProgagationUserImpl、TestTransactionProgagationCourseImpl、UserManagerBoImpl 类的所有方法进行事务代理。

> 注：Spring 框架中一个 Advisor 相当于一个小型的切面，`<tx:advice/>` 定义了这个切面的通知方法，而 `<aop:config>` 具体定义了一个 Advisor 切面，并且内部定义了一个切点，并且引入了 `<tx:advice/>` 定义的通知方法

### 九、`<tx:annotation-driven/>` 内幕

在 Spring 中我们可以使用注解式事务，也就是在需要开启事务的方法上加上 @Transactional 注解后，就可以对该方法进行事务增强，要使用该功能需要配置。

```Java
<!-- (7) 声明使用注解式事务 -->
    <tx:annotation-driven
        transaction-manager="transactionManager" />
```

标签，这时候可以注释掉掉demo配置文件中的（5）和（6）:

```java
    <!-- (5)-->
<!-- <tx:advice id="defaultTxAdvice" transaction-manager="transactionManager">
        <tx:attributes>
            <tx:method name="*" rollback-for="Exception" />
        </tx:attributes>
    </tx:advice> -->

    <!-- (6)  -->
    <!-- <aop:config>
        <aop:pointcut id="myCut"
            expression="(execution(* *..*BoImpl.*(..))) " />
        <aop:advisor pointcut-ref="myCut"
            advice-ref="defaultTxAdvice" />
    </aop:config> -->
```

上节我们讲解了（5）是创建了一个通知方法，（6）则是创建了一个小型切面，并且使用切点表达式说明要对哪些方法进行事务增强，然后对拦截的方法使用（5）创建的通知方法进行增强。而 `<tx:annotation-driven/>` 注解的功能也是类似的会创建一个小型切面，不同在于切点是添加了 @Transactional 注解的方法。

下面我们先看 `<tx:annotation-driven/>` 注解解析的时序图，该标签是 AnnotationDrivenBeanDefinitionParser 进行解析的：

![enter image description here](http://upload-images.jianshu.io/upload_images/5879294-e7594034a0ca636c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 如上时序图 `<tx:annotation-driven/>` 创建的小型切面是 BeanFactoryTransactionAttributeSourceAdvisor 而不是 DefaultBeanFactoryPointcutAdvisor，前者在内部定义了自己的 pointcut：

```java
    private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
        @Override
        @Nullable
        protected TransactionAttributeSource getTransactionAttributeSource() {
            return transactionAttributeSource;
        }
    };
```

- 另外 BeanFactoryTransactionAttributeSourceAdvisor 切面对应的通知方法还是 TransactionInterceptor，并且这里注册了 InfrastructureAdvisorAutoProxyCreator 类而不再是 AspectJAwareAdvisorAutoProxyCreator 到 IOC 容器了，这里 InfrastructureAdvisorAutoProxyCreator 负责收集标注了 @Transactional 注解的方法，然后使用切面 BeanFactoryTransactionAttributeSourceAdvisor 进行功能增强。

具体判断一个方法是否加了注解是：

![image.png](https://upload-images.jianshu.io/upload_images/5879294-02e11e2bfebd0178.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们具体看标注（2）的地方：

![image.png](https://upload-images.jianshu.io/upload_images/5879294-cba898fdf434b45c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后 parseTransactionAnnotation 方法具体是解析 @Transactional 上的属性值，比如事务隔离性了，事务传播性了等，然后保存到 TransactionAttribute 返回。

注：注解式事务也是创建了一个事务切面，其中通知方法还是事务拦击器 TransactionInterceptor，不同在于其内部创建了自己的切点用来匹配对加了 @Transactional 注解的方法进行拦截。

### 十、参考

- https://docs.spring.io/spring/docs/2.5.x/reference/aop.html
- https://gitbook.cn/books/5a92b44808ca63647471da8c/index.html