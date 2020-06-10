# 一个高可靠性商用 Redis 集群方案介绍

本场 Chat 是基于读者已经具备 Redis 基础知识的假设，故，如果读者不熟悉 Redis，可阅读文章《[基于 Redis 的分布式缓存实现方案及可靠性加固策略](http://gitbook.cn/books/5aa4cb50c2ff6f2e120891d0/index.html)》和《[一组 Redis 实际应用中的异常场景及其根因分析和解决方案](http://gitbook.cn/books/5ad0921f7e92727030f21e7f/index.html)》。 本场 Chat 涉及如下内容：

- Redis 集群模式原理简介；
- Redis 高级 Java 客户端 Lettuce 介绍；
- Redis SSL 双向认证通信介绍；
- 基于 Lettuce 的 Redis 集群运维软件设计及实现；
- Redis 集群节点故障替换实现方案；

### 1. Redis 集群模式原理简要回顾

> 为了更好的阐述 Redis 集群方案，我们首先简要回顾一下 Redis 集群的实现原理。

#### 1.1 Redis 集群实现基础

Redis 集群实现的基础是分片，即将数据集有机的分割为多个片，并将这些分片指派给多个 Redis 实例，每个实例只保存总数据集的一个子集。利用多台计算机内存和来支持更大的数据库，而避免受限于单机的内存容量；通过多核计算机集群，可有效扩展计算能力；通过多台计算机和网络适配器，允许我们扩展网络带宽。

![enter image description here](http://images.gitbook.cn/5bf6bb00-628c-11e8-8a60-1bdde4cc4659)

#### 1.2 Redis-Cluster 原理

**（1）Hash slot**

基于“分片”的思想，Redis 提出了 Hash Slot。Redis-Cluster 把所有的物理节点映射到预分好的16384个 Slot，当需要在 Redis 集群中放置一个 Key-Value 时，根据 CRC16(key) Mod 16384的值，决定将一个 Key 放到哪个 Slot 中。

**（2）集群内的每个 Redis 实例监听两个 TCP 端口**

6379（默认）用于服务客户端查询，16379（默认服务端口 + 10000）用于集群内部通信。

**（3）节点间状态同步**

Gossip 协议，最终一致性。所有的 Redis 节点彼此互联（PING-PONG机制），节点间通信使用轻量的二进制协议，减少带宽占用。

#### 1.3 Redis-Cluster 请求路由方式

客户端直连 Redis 服务，进行读写操作时，Key 对应的 Slot 可能并不在当前直连的节点上，经过“重定向”才能转发到正确的节点。如下图所示：以集群模式登陆 127.0.0.1:6379 客户端（注意：-c 表示集群模式)，进行 set 操作，当 Key 对应的 Slot 不在当前节点时（如 key-test)，则可以清楚的看到“重定向”的信息，并且客户端也发生了切换：“6379”->“6381”。

![enter image description here](http://images.gitbook.cn/fac3dd40-6286-11e8-b977-f33e31f528f0)

以三节点为例，上述操作的路由查询流程示意图如下所示：

![enter image description here](http://images.gitbook.cn/21304a40-6287-11e8-b977-f33e31f528f0)

和普通的查询路由相比，Redis-Cluster 借助客户端实现的请求路由是一种混合形式的查询路，它并非从一个 Redis 节点到另外一个 Redis，而是借助客户端转发到正确的节点。

Redis-Cluster 模式使用的混合查询路由效率并不高，在实际应用中，可以在客户端缓存 Slot 与 Redis 节点的映射关系，当接收到 MOVED 响应时修改缓存中的映射关系。如此，基于保存的映射关系，请求时会直接发送到正确的节点上，从而减少一次交互，听上去是不是科学很多？

目前，包括 Lettuce（下文将介绍），Jedis 在内的许多 Redis Client，都已经实现了对 Redis-Cluster 的支持。

### 2. Redis 高级 Java 客户端 Lettuce

如果你在网上搜索 Redis 的 Java 客户端，你会发现，大多数文献介绍的都是 Jedis，不可否认，Jedis 是一个优秀的基于 Java 语言的 Redis 客户端，但是，其不足也很明显：

Jedis 在实现上是直接连接 redis-server，在多个线程间共享一个 Jedis 实例时是线程不安全的，如果想要在多线程环境下使用 Jedis，需要使用连接池；每个线程都使用自己的 Jedis 实例，当连接数量增多时，物理资源成本就较高。

鉴于此，本场 Chat 介绍的方案中采用的并不是 Jedis，而是更加优秀的 Lettuce（[官网](https://lettuce.io/)）。

#### 2.1 Lettuce 简介

![enter image description here](http://images.gitbook.cn/0296fb00-628d-11e8-b977-f33e31f528f0)

和 Jedis 相比，[Lettuce](https://lettuce.io/) 则完全克服了其线程不安全的缺点，先来一段官方介绍：Lettuce 是一个可伸缩的线程安全的 Redis 客户端，支持同步、异步和响应式模式。多个线程可以共享一个连接实例，而不必担心多线程并发问题。它基于优秀 netty NIO 框架构建，支持 Redis 的高级功能，如 Sentinel，集群，流水线，自动重新连接和 Redis 数据模型。

**Lettuce 有很多优点：**

- 基于 Netty，支持事件模型
- 支持 同步、异步、响应式的模式
- 可以方便的连接 Redis Sentinel
- 完全支持 Redis Cluster
- SSL 连接
- Streaming API
- CDI 和 Spring 的集成
- 兼容 Java 8 和 9

#### 2.2 Lettuce 使用实例

以下是两个简单的 Lettuce 使用实例：

**单机模式**

```
// 利用redis-server所绑定的IP和Port创建URI，
RedisURI redisURI = RedisURI.create("100.x.x.152", 6379);

// 创建集Redis单机模式客户端
RedisClient redisClient = RedisClient.create(redisURI);
StatefulRedisConnection<String, String> connect = redisClient.connect();
RedisCommands<String, String> cmd = connect.sync();

// 执行基本的set、get操作
cmd.set("key", "value");
cmd.get("key");
```

**集群模式**

```
// 利用redis-server所绑定的IP和Port创建URI，
List<RedisURI> redisURIList = new ArrayList<>();
String[] ipSet = {"100.x.x.152","100.x.x.153","100.x.x.154"};
int port = 6379;
for (int i=0; i<3; i++)
{
    RedisURI temp = RedisURI.create(ipSet[i], port);
    redisURIList.add(temp);
}

// 创建集Redis集群模式客户端
RedisClusterClient redisClusterClient = RedisClusterClient.create(redisURIList);
StatefulRedisClusterConnection<String, String> clusterConnection = redisClusterClient.connect();   
RedisClusterCommands<String, String> commandsForCluster = clusterConnection.sync();
// 执行基本的set、get操作
commandsForCluster.set("key", "value");
commandsForCluster.get("key");
```

### 3. Redis SSL 双向认证通信实现

#### 3.1 Redis 自带的鉴权访问模式

默认情况下，Redis 服务端是不允许远程访问的，打开其配置文件 redis.conf，可以看到如下配置：

![enter image description here](http://images.gitbook.cn/86c021b0-6259-11e8-a59f-c7ac04233ce1)

根据说明，如果我们要远程访问，可以手动改变 “protected-mode” 的配置，将 yes 状态置为 no 即可，也可在本地客服端 redis-cli，键入命令：`config set protected-mode no`。但是，这明显不是一个好的方法，去除保护机制，意味着严重安全风险。

鉴于此，我们可以采用鉴权机制，通过秘钥来鉴权访问，修改 redis.conf，添加 requirepass mypassword ，或者键入命令：`config set requirepass password` 设置鉴权密码。

![enter image description here](http://images.gitbook.cn/14a74bc0-6264-11e8-8a60-1bdde4cc4659)

设置密码后，Lettuce 客户端访问 redis-server 就需要鉴权，增加一行代码即可，以单机模式为例：

![enter image description here](http://images.gitbook.cn/1f3fe880-6322-11e8-b864-0bd1f4b74dfb)

**补充**

除了通过密码鉴权访问，出于安全的考量，Redis 还提供了一些其它的策略：

- 禁止高危命令

修改 redis.conf 文件，添加如下配置，将高危原生命令重命名为自定义字符串：

```
rename-command FLUSHALL "user-defined"
rename-command CONFIG   "user-defined"
rename-command EVAL     "user-defined"
```

虽然禁止高危命令有助于安全，但本质上只是一种妥协，也许是对加密鉴权信心不足吧。

- 禁止外网访问

Redis 配置文件 redis.conf 默认绑定本机地址，即 Redis 服务只在当前主机可用，配置如下：

```
bind 127.0.0.1
```

这种方式，基本斩断了被远程攻击的可能性，但局限性更明显，Redis 基本退化成本地缓存了。

#### 3.2 SSL 双向认证通信

通过上面的介绍，相信读者已经对 Redis 自带的加固策略有了一定了解。客观地讲，Redis 自带的安全策略很难满足对安全性要求普遍较高的商用场景，鉴于此，有必要优化。就 Client-Server 模式而言，成熟的安全策略有很多，本文仅介绍其一：SSL 双向认证通信。关于 SSL 双向认证通信的原理和具体实现方式，网上有大量的博文可供参考，并非本文重点，因此不做详细介绍。

**总体流程**

![enter image description here](http://images.gitbook.cn/132d6a10-6271-11e8-b977-f33e31f528f0)

1. 客户端安装服务器根证书 ca.crt 到客户端信任证书库中，服务器端安装服务器根证书 ca.crt 到服务器信任证书库中。
2. SSL 握手时，服务器先将服务器证书 server.p12 发给客户端，客户端到客户端信任证书库中进行验证，因为 server.p12 是根证书 CA 颁发的，所以验证通过；
3. 然后客户端将客户端证书 client.p12 发给服务器，同理因为 client.p12 是根证书 CA 颁发的，所以验证通过。
4. 需要注意：从证书库中取证书是需要提供密码的，这个密码需要保存到服务端和客户端的配置文件中，如果以明文形式保存，存在安全风险，因此，通常会对明文密码进行加密，配置文件中保存加密后的密文。然后，客户端和服务端对应的鉴权程序中首先对密文解密获得证书库明文密码，再从证书库中取得证书。

**实现方案**

- 服务端

Redis 本身不支持 SSL 双向认证通信，因此，需要修改源码，且涉及修改较多，本文仅列出要点，具体实现层面代码不列。

**config.c**

SSL 双向认证通信涉及的 keyStore 和 trustStore 秘密码密文、路径等信息可由 Redis 的配置文件 redis.conf 提供，如此，我们需要修改加载配置文件的源码（config.c->loadServerConfigFromString(char *config)），部分修改如下：

![enter image description here](http://images.gitbook.cn/0baace00-6955-11e8-b501-5b633bc9aed2)

**redis.h**

Redis 的客户端（redisClient）和服务端（redisServer）都需要适配，部分代码如下：

![enter image description here](http://images.gitbook.cn/0fdb8d20-695a-11e8-8614-25103124b6ae)

![enter image description here](http://images.gitbook.cn/19cabbd0-695a-11e8-8feb-7f10d09adc13)

**hiredis.h**

修改创建连接的原函数：

![enter image description here](http://images.gitbook.cn/266d0720-695b-11e8-b501-5b633bc9aed2)

**anet.h**

定义 SSL 通信涉及的一些函数(实现在 anet.c 中)：

![enter image description here](http://images.gitbook.cn/1fb78a30-695c-11e8-8614-25103124b6ae)

- 客户端

Lettuce 支持 SSL 双向认证通信，需要增加一些代码，以单机模式为例：

```
// 获取trustStore的密码的密文
String trustPd = System.getProperty("redis_trustStore_password");
// 对密码的密文进行解密获得明文密码，解密算法很多，这里是自研代码
trustPd = CipherMgr.decrypt(trustPd);

// 获取keyStore的密码的密文
String keyPd = System.getProperty("redis_keyStore_password");
// 对密码的密文进行解密获得明文密码
keyPd = CipherMgr.decrypt(keyPd);

// 加密算法套件，此处client_cipher=TLS_RSA_WITH_AES_128_GCM_SHA256
List<String> cipherList = new ArrayList<String>();
cipherList.add(System.getProperty("client_cipher"));

// 构建SslOptions
SslOptions sslOptions = SslOptions.builder()
.truststore(new File(System.getProperty("redis_trustStore_location")),
        trustPd)
.keystore(new File(System.getProperty("redis_keyStore_location")), keyPd)
.cipher(cipherList).build();

ClusterClientOptions option = (ClusterClientOptions) ClusterClientOptions.builder()
.sslOptions(sslOptions).build();

// 利用redis-server所绑定的IP和Port创建URI，
RedisURI redisURI = RedisURI.create("100.x.x.152", 6379);

// 创建集Redis集群模式客户端
RedisClient redisClient = RedisClient.create(redisURI);
// 为客户端设置SSL选项
redisClient.setOptions(option);
StatefulRedisConnection<String, String> connect = redisClient.connect();
RedisCommands<String, String> cmd = connect.sync();

// 执行基本的set、get操作
cmd.set("key", "value");
cmd.get("key");
```

### 4. Redis 集群方案

为了便于理解（同时也为了规避安全违规风险），本文将原方案进行了适度简化，以 3 主 3 备 Redis 集群为例阐述方案（redis-cluster 模式最少需要三个主节点），如下所示：其中，A-M 表示主节点 A，A-S 表示主节点 A 对应的从节点，以此类推。

![enter image description here](http://images.gitbook.cn/90bad990-628f-11e8-b864-0bd1f4b74dfb)

**特别说明：**

事实上，Redis 集群节点间是两两互通的，如下图所示（5 节点），上面作为示意图，进行了适当简化。

![enter image description here](http://images.gitbook.cn/68c726b0-6347-11e8-8a60-1bdde4cc4659)

#### 4.1 可靠性问题1

Redis 集群并不是将 redis-server 进程启动便可自行建立的，在各个节点启动 redis-server 进程后，形成的只是 6 个 “孤立” 的 Redis 节点而已，它们相互不知道对方的存在，拓扑结构如下：

![enter image description here](http://images.gitbook.cn/5e716e00-6347-11e8-b977-f33e31f528f0)

查看每个 Redis 节点的集群配置文件 cluster-config-file，你将看到类似以下内容：

```
2eca4324c9ee6ac49734e2c1b1f0ce9e74159796 192.168.1.3:6379 myself,master - 0 0 0 connected
vars currentEpoch 0 lastVoteEpoch 0
```

很明显，每个 Redis 节点都视自己为 master 角色，其拓扑结构中也只有自己。为了建立集群，Redis 官方提供一个基于 Ruby 语言的工具 redis-trib.rb，使用命令便可以创建集群。以 3 主 3 备集群为例，假设节点 IP 和 Port 分别为：

```
192.168.1.3:6379,192.168.1.3:6380,192.168.1.4:6379,192.168.1.4:6380,192.168.1.5:6379,192.168.1.5:6380 
```

则建立集群的命令如下：

```
redis-trib.rb create --replicas 1 192.168.1.3:6379 192.168.1.3:6380 192.168.1.4:6379 192.168.1.4:6380 192.168.1.5:6379 192.168.1.5:6380
```

使用 redis-trib.rb 建立集群虽然便捷，不过，由于 Ruby 语言本身的一系列安全缺陷，有些时候并不是明智的选择。考虑到 Lettuce 提供了极为丰富的 Redis 高级功能，我们完全可以使用 Lettuce 来创建集群，在上一场 Chat《[基于 Redis 的分布式缓存实现方案及可靠性加固策略》](http://gitbook.cn/books/5aa4cb50c2ff6f2e120891d0/index.html)中，就是基于 Lettuce 实现的创建集群功能。

#### 4.2 节点故障

三个物理节点，分别部署两个 redis-server，且交叉互为主备，这样做可以提高可靠性：如节点 1 宕机，主节点 A-M 对应的从节点 A-S 将发起投票，作为唯一的备节点，其必然升主成功，与 B-M、C-M 构成新的集群，继续提供服务，如下图所示：

![enter image description here](http://images.gitbook.cn/087976e0-6294-11e8-b977-f33e31f528f0)

#### 4.3 故障节点恢复

接续上一节，如果宕机的节点 1 经过修复重新上线，根据 Redis 集群原理，节点 1 上的 A-M 将意识到自己已经被替代，将降级为备，形成的集群拓扑结构如下：

![enter image description here](http://images.gitbook.cn/12a93100-6294-11e8-a59f-c7ac04233ce1)

#### 4.4 可靠性问题2

基于上述拓扑结构，如果节点 3 宕机，Redis 集群将只有一个主节点 C-M 存活，存活的主节点总数少于集群主节点总数的一半 (1<3/2+1)，集群无法自愈，不能继续提供服务。

为了解决这个问题，我们可以设计一个常驻守护进程对 Redis 集群的状态进行监控，当出现主-备状态不合理的情况（如节点 1 重新上线后的拓扑结构），守护进程主动发起主备倒换（clusterFailover），将节点 1 上的 A-S 升为主，节点 3 上的 A-M 降为备，如此，集群拓扑结构恢复正常，并且能够支持单节点故障。

> 注：Lettuce 提供了主备倒换的方法，示例代码如下：

```
// slaveConn为Lettuce与从节点建立的连接
slaveConn.sync().clusterFailover(true)
```

![enter image description here](http://images.gitbook.cn/297a8fa0-6294-11e8-8a60-1bdde4cc4659)

#### 4.5 可靠性问题3

接续 4.1 节，如果节点 1 故障后无法修复，为了保障可靠性，通常会用一个新的节点来替换掉故障的节点——所谓故障替换。拓扑结构如下：

![enter image description here](http://images.gitbook.cn/2ea5ff40-6295-11e8-a59f-c7ac04233ce1)

新的节点上面部署两个 redis-server 进程，由于是新建节点，redis-server 进程对应的集群配置文件 cluster-config-file 中只包含自身的信息，并没有整个集群的信息，简言之，新建的节点上的两个 redis-server 进程是“孤立”的。

为了重新组成集群，我们需要两个步骤：

1. 将新节点上的两个 redis-server 纳入现有集群，通过 clusterMeet() 方法可以完成；
2. 为新加入集群的两个 redis-server 设置主节点：节点 3 上的两个主 A-M 和 B-M 都没有对应的从节点，因此，可将新加入的两个 redis-server 分别设置为它们的从节点。

完成上述两个步骤后，Redis 集群的拓扑结构将演变成如下形态：

![enter image description here](http://images.gitbook.cn/c83e7400-6297-11e8-8a60-1bdde4cc4659)

很明显，变成了问题 1 的形态，继续通过问题 1 的解决方案便可修复。

#### 4.6 其它

上面仅介绍了几个较为常见的问题，在实际使用 Redis 的过程中可能遇到的问题远不止这些。在[《一组 Redis 实际应用中的异常场景及其根因分析和解决方案》](http://gitbook.cn/books/5ad0921f7e92727030f21e7f/index.html)中，我介绍了一些更为复杂的异常场景，感兴趣的读者可以看一下。

### 5. 基于 Lettuce 的 Redis 集群运维软件设计及实现

不同的应用场景，关注的问题、可能出现的异常不尽相同，上文中介绍问题仅仅是一种商业应用场景中遇到的。为了解决上述问题，可基于Lettuce设计一个常驻守护进程，实现集群创建、添加节点、平衡主备节点分布、集群运行状态监测、故障自检及故障自愈等功能。

#### 5.1 总体流程图

注：为避免违规，以下流程图为删减后的版本

![enter image description here](http://images.gitbook.cn/e9bc2e40-6280-11e8-b864-0bd1f4b74dfb)

**说明**

流程图中，ETCD 选主部分需要特别说明一下，ETCD 和 ZooKeeper 类似，可提供 leader 选举功能。Redis 集群模式下，在各个 Redis 进程所在主机上均启动一个常驻守护进程，以提高可靠性，但是，为了避免冲突，只有被 ETCD 选举为 leader 的节点上的常驻守护进程可以执行 “守护” 流程，其它主机上的守护进程呈 “休眠” 状态。关于 Leader 选举的实现，方式很多，本文仅以 ETCD 为例。

#### 5.2 实现

**集群状态检测**

读者应该知道，Redis 集群中每个节点都保存有集群所有节点的状态信息，虽然这些信息可能并不准确。通过状态信息，我们可以判断集群是否存在以及集群的运行状态，基于 Lettuce 提供的方法，简要代码如下 (代码中只从一个节点的视角进行了检查，完整的代码将遍历所有节点，从所有节点的视角分别检查)：

![enter image description here](http://images.gitbook.cn/cdc1eb40-6333-11e8-a59f-c7ac04233ce1)

**Redis 集群创建**

**(1) 相互感知，初步形成集群**

成功拉起了 6 个 redis-server 进程后，每个进程视为一个节点，这些节点仍处于孤立状态，它们相互之间无法感知对方的存在，既然要创建集群，首先需要让这些孤立的节点相互感知，形成一个集群。该过程可用 Lettuce 的 clusterMeet() 方法实现：以其中一个节点为基准，其它节点 Meet 该节点，从而形成初步集群：简要代码如下：

![enter image description here](http://images.gitbook.cn/575ad230-6330-11e8-b864-0bd1f4b74dfb)

**(2) 分配 Slot 给期望的主节点**

形成集群之后，仍然无法提供服务，Redis 集群模式下，数据存储于 16384 个 Slot 中，我们需要将这些 Slot 指派给期望的主节点。何为期望呢？我们有 6 个节点，3 主 3 备，我们只能将 Slot 指派给 3 个主节点，至于哪些节点为主节点，我们可以自定义。

![enter image description here](http://images.gitbook.cn/62ec8d50-6330-11e8-b977-f33e31f528f0)

**(3) 设置从节点**

Slot 分配完成后，被分配 Slot 的节点将成为真正可用的主节点，剩下的没有分到 Slot 的节点，即便状态标志为 Master，实际上也不能提供服务。接下来，处于可靠性的考量，我们需要将这些没有被指派 Slot 的节点指定为可用主节点的从节点（Slave）。

![enter image description here](http://images.gitbook.cn/3c1952c0-6331-11e8-a59f-c7ac04233ce1)

经过上述三个步骤，一个精简的 3 主 3 备 Redis 集群就搭建完成了。

**替换故障节点**

**(1) 加入新节点**

替换上来的新节点本质上是“孤立”的，需要先加入现有集群：通过集群命令 RedisAdvancedClusterCommands 对象调用 clusterMeet() 方法，便可实现：

![enter image description here](http://images.gitbook.cn/640cbba0-6336-11e8-b864-0bd1f4b74dfb)

**(2) 为新节点设置主备关系**

首先需要明确，当前集群中哪些 master 没有 slave，然后，新节点通过 clusterReplicate() 方法成为对应 master 的 slave：

```
slaveConn.sync().clusterReplicate(masterNode);
```

**平衡主备节点的分布**

**(1) 状态检测**

常驻守护进程通过遍历各个节点获取到的集群状态信息，可以确定某些 host 上 master 和 slave 节点数量不平衡，比如，经过多次故障后，某个 host 上的 Redis 节点角色全部变成了 master，不仅影响性能，还会危及可靠性。这个环节关键点是如何区分 master 和 slave，通常我们以是否被指派 slot 为依据：

![enter image description here](http://images.gitbook.cn/eb68ea40-6338-11e8-8a60-1bdde4cc4659)

**(2) 平衡**

如何平衡呢，在创建 Redis 集群的时候，开发者需要制定一个合理的集群拓扑结构（或者算法）来指导集群的创建，如本文介绍的 3 主 3 备模式。那么，在平衡的时候，同样可以依据制定的拓扑结构进行恢复。具体操作很简单：调用 Lettuce 提供的 clusterFailover() 方法即可。

**致谢**

- 双向认证部分参考资料：[enter link description here](https://www.cnblogs.com/qiuxiangmuyu/p/6405511.html) https://www.cnblogs.com/qiuxiangmuyu/p/6405511.html