# 2/9第03课：如何使用 Redis 客户端

之前在学习 Redis 基础的时候，我们都是使用 Redis 的命令行。但是除了命令行，还有主流语言支持的客户端。打开 Redis 的官网，在其中找到 Clients，就能找到你熟悉的语言对应的客户端，如下图所示。

![enter image description here](http://images.gitbook.cn/0a07cb50-f283-11e7-8314-6979f3340399)

本文我们将主要讲解 redis-py（Python 客户端）、Jedis（Java 客户端），其他语言的客户端也都差不多。当你选择某个语言点进去后，你会发有笑脸和黄色五角星。五角星表示 Redis 推荐的，笑脸表示社区比较活跃。建议选择有笑脸和五角星的客户端。

### 3.1 Python 客户端 redis-py

#### 1. 正确获取 redis-py

首先我们必须安装 Redis，注意不是安装 redis-py。打开你的命令行，如果你使用 CentOS 或者 Mac 系统，请直接使用 pip 工具安装。如果没有 pip，请先进行安装。下面我们介绍下如何安装它，已经安装的可以直接忽略。

第一步：下载 pip。进入 https://pypi.python.org/pypi/pip，下载第二项。

第二步：解压安装。

解压下载的文件（Windows 下用解压工具，如RAR，Linux 下终端输入 tar -xf pip-9.0.1.tar.gz，即tar -xf 文件名），进入解压后的文件夹中，调出命令行窗口或者终端。

在 Windows 下输入：

```
python setup.py install
```

在 Linux 下输入：

```
sudo python setup.py install
```

安装成功后测试下，输入以下命令，输出 pip 版本号即表示安装成功。

```
pip -v
```

然后选择执行以下 pip 命令，安装 Redis。

- pip install redis，如果你使用的是 CentOS 系统，打开命令行，使用该命令安装；
- easy_install redis，如果使用 Ubuntu 系统，使用easy_install 安装。

当然你也可以使用**源码安装**，如下。

```
 git clone https://github.com/andymccurdy/redis-py.git  
 cd redis-py  //进入 redis-py目录
 python setup.py install //开始安装
```

#### 2. 使用 redis-py 客户端

要使用 redis-py 客户端，我们需通过代码来连接。如何连接呢？打开你的 Python 编辑器。

redis-py 提供了两个类：StrictRedis 和 Redis，用于实现 Redis 的命令，这里推荐使用 StrictRedis。StrictRedis 实现了绝大部分官方的命令，并且使用原生的语法和命令。比如，SET 命令对应着 StrictRedis.set 方法。下面是个简单的例子。

```
//代码连接 Redis
import redis // 导入 Redis
client = redis.StrictRedis(host='127.0.0.1',port=6379) // 通过 redis.StrictRedis 连接 Redis
key = "welcome"
setResult = client.set(key, "redis-py") // SET 命令
print setResult //打印这个值
value = client.get(key) //在 Redis 使用 GET 命令获取 key 的值
print "key = "+ key +" value="+value //打印
```

#### 3. 简单的使用

上面我们讲了如何连接 redis-py 客户端，下面我们使用各种数据类型来设置和获取信息，后面都有注释。

- String：

```
client.set("redis","helloworld") // Redis 返回 True
client.get("redis") // redis返回 helloworld
client.incr("counter") //自增+1
```

- Hash：

```
client.hset("hashvalue","a1","b1") // Hash 设置 hashvalue
 的值 1
client.hset("hashvalue","a2","b2")// Hash 设置 hashvalue 的值 2
client.hgetall("hashvalue") // Redis 返回 {a1:b1,a2:b2}
```

- List：

```
client.rpush("listvalue","a") // List 设置是 rpush
client.rpush("listvalue","b")
client.rpush("listvalue","c")
client.lrange("listvalue",0,-1) // Redis 返回 ['a','b','c']
```

- Set：

```
client.sadd("setvalue","a") // Set 设置是 sadd
client.sadd("setvalue","b")
client.sadd("setvalue","c")
client.smembers("setvalue") // Redis 返回 ['a','b','c']
```

- ZSet：

```
client.zadd("zsetvalue","1","book1") // ZSet 设置是 zadd
client.zadd("zsetvalue","2","book2")
client.zadd("zsetvalue","3","book3")
client.zrange("zsetvalue",0,-1,withscores=True) // withscores 表示含索引值，Redis
 返回[('book3',3.0),('book2',2.0),('book1',1.0)]
```

怎么样，这些方法简单吗？我们再来讲一下如何使用 Java 客户端。

### 3.2 Java 客户端 Jedis

#### 1. 正确获取 Jedis

方式一，访问以下网址，获取 Jar 包：

> http://files.cnblogs.com/liuling/jedis-2.1.0.jar.zip

相信很多刚学 Java 的朋友就是这么找包的，这里不推荐，推荐大家使用 Maven。关于如何使用 Maven，大家可以找度娘，实在搞不定也可以咨询我。

当然如果你需要使用 Redis 连接池，那你还需要一个 Jar 包（还是不推荐，建议用 Maven），点击[这里](http://files.cnblogs.com/liuling/commons-pool-1.5.4.jar.zip)，获取下载地址。

方式二，使用 Maven 管理，只有在 Maven 里增加了 Jedis 客户端的配置，才能执行下面的代码，否则报错。

```
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
```

#### 2. 使用 Jedis 客户端

使用 Jedis 也是非常简单，只需要 new 一个 Jedis 对象，然后调用 SET 和 GET 就 OK 了，简单吧。

```
    Jedis jedis = new Jedis("127.0.0.1",6379); // IP，端口，连接超时，读写超时
    jedis.set("hello","world");
    jedis.get("hello");
    jedis.close(); // 记得要
 close 掉，这是一个良好的习惯。
```

#### 3. 简单的使用

连接完 Jedis，我们再介绍一下常用的三种数据类型，其他的参照 redis-py，都是大同小异。

- String：
- \-

```
jedis.set("hello","world"); //String 是最常用的，用 set 设置
jedis.get("hello"); // get 获取
jedis.incr("counter"); //自增1
```

- Hash：

```
jedis.hset("hello","v1","f1"); //Hash 设置用 hset
jedis.hset("hello","v2","f2");  
jedis.hgetAll("hello"); // Redis 返回 {v1=f1,v2=f2}
```

- List：

```
jedis.rpush("vss","1"); //List 设置用 rpush
jedis.rpush("vss","2");
jedis.rpush("vss","3");    
jedis.lrange("vss",0,-1); //Redis 返回 [1,2,3]
```

当然，我们不仅要使用 Redis，还会使用 Jedis 的连接池。为什么要使用连接池呢？因为 Jedis 对象不是线程安全的，在多线程下会发生并发问题。为了避免这些问题，Jedis 提供了 JedisPool 连接池，示例代码如下。

```
GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig(); //JedisPool 是单例的，需要 new
 一个 GenericObjectPoolConfig
 对象

JedisPool jedisPool = new JedisPool(poolConfig,"127.0.0.1",6379); // 建立连接池的连接

Jedis jedis = null; // 类似于 new 一个 Jedis 对象

try{
    jedis = jedisPool.getResource(); // 从连接池获取 Jedis 对象
    jedis.set("hello","world"); // 设置

}catch(Exception e){
    e.printStackTrace();
}finally{
    if(jedis!=null){
    jedis.close();// 注意不是关闭，而是归还到连接池
    }
}
```

其他语言都可以参照上面的 Python 和 Java 的客户端，内容方面都差不多，而且 Jedis 主要是掌握其原理和流程，操作其实是非常简单的。如果你有什么疑问，不妨给我留言。

如果你需要了解对应的 API，请点击[这里](https://redis.io/clients)，从中选择你要用的语言对应的客户端。

> 拓展阅读：[Redis 实战场景详解](https://gitbook.cn/gitchat/activity/5c81ff36aa24bf22d670e335?utm_source=chcsd001)