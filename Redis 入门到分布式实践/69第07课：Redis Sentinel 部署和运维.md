# 6/9第07课：Redis Sentinel 部署和运维

上一篇，我们讲解了 Redis 复制的主要内容，但 Redis 复制有一个缺点，当主机 Master 宕机以后，我们需要人工解决。那么能不能自动解决主机宕机的问题呢？

Redis Sentinel 正是为了解决这样的问题而被开发的。Redis Sentinel 是一个分布式的架构，每一个 Sentinel 节点会对数据节点和其余 Sentinel 节点进行监控，当发现某个节点无法到达的时候，会自动标识该节点。如果这个节点是主节点，那么它会和其他 Sentinel 节点“协商”，大部分节点都认为主节点无法到达的时候，它们会选举一个 Sentinel 节点来完成自动故障转移，同时会告知 Redis 的应用方。

由于这个过程是自动化的，不需要人工参与，大大提高了 Redis 的高可用性。

接下来，我们将从实现流程、安装配置、客户端连接、实现原理、常见开发运维问题这五个方面来探讨。

### 7.1 实现流程

如下图所示，Sentinel 集群会监控每一个 Slave 和 Master。客户端不再直接从 Redis 获取信息，而是通过 Sentinel 集群来获取信息。

![img](http://images.gitbook.cn/d93125c0-0cb9-11e8-a370-e181c30bd776)

再看下面这张图，当 Master 宕机了，Sentinel 监控到 Master 有问题，就会和其他 Sentinel 进行选举，选举出一个 Sentinel 作为领导，然后选出一个 Slave 作为 Master，并通知其他 Slave。上图 Slave1 变成了 Master，如果原来的 Master 又连上了，也会变成 Slave 从机。

![enter image description here](http://images.gitbook.cn/0b10c960-0cba-11e8-9706-9106925a3925)

### 7.2 安装与配置

我们将从以下两个方面讲解如何安装和配置主从节点和 Sentinel 节点。

- 如何配置开启主从节点；
- 如何开启 Sentinel 监控主节点。

#### 1. 开启主从节点

Sentinel 对主节点和从节点的配置是不同的，需要分别配置，我们分开来讲解。

- 主节点配置

我们在命令行使用下面的命令进行主节点的启动。

```
redis-server redis-7000.conf
```

启动完成以后，我们参考下面的配置进行参数的设置。

```
port 7000
daemonize yes // 守护进程
pidfile /var/run/redis-7000.pid // 给出 pid
logfile “7000.log” // 日志查询
dir "/opt/redis/data" // 工作目录
```

- 从节点配置

我们在命令行使用下面的命令进行从节点的启动。

```
redis-server redis-7001.conf
redis-server redis-7002.conf
```

启动完成以后，和主节点配置一样，配置下面的参数。这个时候要注意，我们需要分别对 Slave 节点的每台机器进行配置。

Slave1 的配置如下。

```
port 7001
daemonize yes // 守护进程
pidfile /var/run/redis-7001.pid // 给出 pid
logfile “7001.log” // 日志查询
dir "/opt/redis/data" // 工作目录
slaveof 127.0.0.1 7000
```

Slave2 的配置如下。

```
port 7002
daemonize yes // 守护进程
pidfile /var/run/redis-7002.pid // 给出 pid
logfile “7002.log” // 日志查询
dir "/opt/redis/data" // 工作目录
slaveof 127.0.0.1 7000
```

#### 2. Sentinel 监控主要配置

开启了主从节点以后，我们需要对 Sentinel 进行监控上的配置，见下面的配置参数。

```
port 端口号
dir "/opt/redis/data/"
logfile "端口号.log"
sentinel monitor mymaster 127.0.0.1 7000 2
sentinel down-after-millseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

由于需要配置多台 Sentinel，从上面配置信息可以看到，除了修改端口号，其他配置都是相同的。重点看最后四个配置，这四个配置是 Sentinel 的核心配置。我们分别来解释一下这四个配置，斜杠后面的文字解释了该参数的意义。

```
sentinel monitor mymaster 127.0.0.1 7000 2 // 监控的主节点的名字、IP 和端口，最后一个 2 表示有 2 台 Sentinel 发现有问题时，就会发生故障转移；

sentinel down-after-millseconds mymaster 30000 // 这个是超时的时间。打个比方，当你去 ping 一个机器的时候，多长时间后仍 ping 不通，那么就认为它是有问题；

sentinel parallel-syncs mymaster 1 // 指出 Sentinel 属于并发还是串行。1 代表每次只能复制一个，可以减轻 Master 的压力；

sentinel failover-timeout mymaster 180000 // 表示故障转移的时间。
```

### 7.3 Sentinel 客户端原理

我们配置高可用的时候，如果只是配置服务端的高可用是不够的。如果客户端感知不到服务端的高可用，是不会起作用的。所以，我们不但要让服务端高可用，还要让客户端也是高可用的。

我们先来看下客户端基本原理。

第一步，客户端 Client 需要遍历 Sentinel 节点集合，找到一个可用的 Sentinel 节点，同时需要获取 Master 主机的 masterName。如下图所示。

![enter image description here](http://images.gitbook.cn/1ff79610-0cba-11e8-bdd7-1d78e572a792)

第二步，当客户端找到 Sentinel-2 节点的时候，Client 会通过 get-master-addr-by-name 命令获取 masterName，这个时候，Sentinel-2 会获取真正的名称和地址。如下图所示。

![enter image description here](http://images.gitbook.cn/73df8940-0cba-11e8-b141-a5ea9a5a91f4)

第三步，Client 获取到 Master 节点的时候，还会发出 role 或 role replication 命令，验证是不是 Master 节点，Sentinel-2 会返回这个节点信息加以验证。如下图所示。

![enter image description here](http://images.gitbook.cn/55760b50-0cba-11e8-a370-e181c30bd776)

第四步，如果 Sentinel 感知到 Master 宕机了，这时 Sentinel 集群应该是最先知道的。客户端和 Sentinel 集群之间其实是发布订阅，客户端 Client 去订阅某个 Sentinel 的频道，如果哪个 Sentinel 发现 Master 发生了变化，Client 是会被通知到这个信息的，如下图所示。**但是要注意这个不是代理模式**。

![enter image description here](http://images.gitbook.cn/86744fa0-0cba-11e8-b141-a5ea9a5a91f4)

总结一下，以上四步就是客户端和 Sentinel 集群的基本原理，任何客户端原理都是按照这个流程做的，只是内部的封装不同而已。

#### 1. Jedis

我们先通过使用率最高的 Java 的客户端 Jedis 讲起。

如何通过代码实现 Sentinel 的访问，让我们来看看代码如何连接 Sentinel 的资源池。

```
JedisSentinelPool sentinelPool = new JedisSentinelPool(masterName,sentinelSet,poolConfig,timeout); //内部的本质还是去连接 Master 主机，参数 masterName 表示 Master 名称，sentinelSet
 表示 Sentinel 集合，后面依次是 poolConfig 配置和超时时间
Jedis jedis = null;
try{
    //获得
 redisSentinelPool 资源
    jedis = redisSentinelPool.getResource();
    //Jedis 相关的命令
}catch (Exception e){
    logger.error(e.getMessage(),e);
}finally{
    if(jedis!=null){
        jedis.close(); // Jedis 归还
    }
}
```

#### 2. redis-py

接下来我们再来看下如何使用 Python 连 Redis 客户端的 Sentinel，和 Jedis 一样，我们将直接给出连接 Sentinel 的代码。

```
from redis.sentinel import Sentinel
sentinel = Sentinel([('localhost',26379),('localhost',26380),('localhost',26381)],socket_time=0.1) // 获取可用的 Sentinel，并设置超时时间。

sentinel.discover_master('mymaster') // 获取 Master 地址
>>> ('127.0.0.1',7000)

sentinel.discover_slaves('mymaster') //获取 Slave 地址
>>> [('127.0.0.1',7001),('127.0.0.1',7002)]
```

### 7.4 Sentinel 实现原理

讲完了 Sentinel 的代码实现，很多人还不懂 Sentinel 的原理。接下来我们就讲解下它的实现原理，主要分为以下三个步骤。

- 检测问题，主要讲的是三个定时任务，这三个内部的执行任务可以保证出现问题马上让 Sentinel 知道。
- 发现问题，主要讲的是主观下线和客观下线。当有一台 Sentinel 机器发现问题时，将对它主观下线，但是当多个 Sentinel 都发现有问题的时候，才会出现客观下线。
- 找到解决问题的人，主要讲的是领导者选举，如何在 Sentinel 内部多台节点中进行领导者选举，选出一个领导者。
- 解决问题，主要讲得是如何进行故障转移。

我们分开进行阐述。

#### 1. 三个定时任务

首先要讲的是内部 Sentinel 会执行以下三个定时任务。

- 每 10 秒每个 Sentinel 对 Master 和 Slave 执行一次 Info Replication。
- 每 2 秒每个 Sentinel 通过 Master 节点的 Channel 交换信息（Pub/Sub）。
- 每 1 秒每个 Sentinel 对其他 Sentinel 和 Redis 执行 Ping。

在这里一一解释下。

第一个定时任务，指的是 Redis Sentinel 可以对 Redis 节点做失败判断和故障转移，在 Redis 内部有三个定时任务作为基础，来 Info Replication 发现 Slave 节点，这个命令可以确定主从关系。

第二个定时任务，类似于发布订阅，Sentinel 会对主从关系进行判定，通过 _sentinel_:hello 频道交互。了解主从关系有助于更好地自动化操作 Redis。然后 Sentinel 会告知系统消息给其他 Sentinel 节点，最终达到共识，同时 Sentinel 节点能够互相感知到对方。

第三个定时任务，指的是对每个节点和其他 Sentinel 进行心跳检测，它是失败判定的依据。

#### 2. 主观下线和客观下线

我们先来回顾一下 Sentinel 的配置。

```
sentinel monitor mymaster 127.0.0.1 6379 3 //如不懂意思，请参见上面对 Sentinel 进行配置的说明；3 这个配置请记住，我将在后面讲解。

sentinel down-after-milliseconds mymaster 3000 //Sentinel 会 Ping 每个节点，如果超过 30 秒，依然没有恢复的话，做下线的判断。
```

那么什么是主观下线呢？

每个 Sentinel 节点对 Redis 节点失败存在“偏见”。之所以是偏见，只是因为某一台机器 30 秒内没有得到回复。

那么如何做到客观下线呢？

这个时候需要所有 Sentinel 节点都发现它 30 秒内无回复，才会达到共识。

#### 3. 领导者选举

Sentinel 集群会采用领导者选举的方式，完成 Sentinel 节点的故障转移。通过 sentinel is-master-down-by-addr 命令都希望成为领导者。

领导者选举的步骤请见下：

- 每个做主观下线的 Sentinel 节点向其他节点发送命令，要求将它设置为领导者；
- 收到命令的 Sentinel 节点，如果没有同意通过其他 Sentinel 节点发送的命令，那么将同意该要求，否则就会拒绝。
- 如果 Sentinel 节点发现自己的票数已经超过 Sentinel 半数同时也超过 Sentinel monitor mymaster 127.0.0.1 6379 3 中的 3 个的时候，那么它将成为领导者；
- 如果有多个 Sentinel 节点成为领导者，那么将等待一段时间后重新选举。

这里需要解释一下为什么要重新选举。因为如果有多个领导者，那么哪个节点能覆盖更多的节点，才会成为真正的领导者，盲目成为领导者，只会让 Sentinel 效率低下，只有不断确认保证最优的选举，才是高效的，当然这个过程是需要消耗时间的。

#### 4. 故障转移

故障转移主要包括以下四个步骤：

- 从 Slave 节点中选出一个“合适的”节点作为新节点；
- 对上面的节点执行 slaveof no one 命令让其成为 Master节点；
- 向剩余的 Salve 节点发送命令，让它们成为新的 Master 节点的 Slave节点，复制规则和同步参数；
- 将原来 Master 节点更新配置为 Slave 节点，并保持其“关注”。当其恢复后命令它去复制新的 Master 节点。

通过以上四步，就能获得“Master 断掉 -> 选出新的 Master -> 同步 -> 旧 Master 恢复后成为 Slave，同时同步新的 Master数据”这样一整套的流程。

#### 5. 如何选择“合适的”Slave 节点

Redis 内部其实是有一个优先级配置的，在配置文件中 slave-priority 这个参数是 Salve 节点的优先级配置，如果存在则返回，如果不存在则继续。

当上面这个优先级不满足的时候，Redis 还会选择复制偏移量最大的 Slave 节点，如果存在则返回，如果不存在则继续。之所以选择偏移量最大，这是因为偏移量越小，和 Master 的数据越接近，现在 Master挂掉了，说明这个偏移量小的机器数据也可能存在问题，这就是为什么要选偏移量最大的 Slave 的原因。

如果发现偏移量都一样，这个时候 Redis 会默认选择 runid 最小的节点。

### 7.5 常见的开发运维的问题

对于 Sentinel 来说，日常是不需要做太多运维工作的，主要说的是以下两点。

- 节点运维：偏运维，例如对 Master 和 Slave 节点的上下限进行操作。
- 高可用读写分离：偏开发，开发人员会思考是否能有更加好用的方式。例如用高可用的读写分离。

我们分别来讲解一下这两个问题。

#### 1. 节点运维

节点运维包括主节点、从节点和 Sentinel 节点的运维。

首先是机器下线问题，如过保等情况。

其次是机器性能不足问题，如 CPU、内存、硬盘、网络等硬件。

最后是节点自身故障，如服务器不稳定，可能因为系统、硬件等未知原因，这个时候我们只能对它进行下线，转移至其他机器上。

- 主节点

主节点的节点运维，主要是通过以下命令，对主节点做故障转移。

```
sentinel failover <masterName>
```

- 从节点

对于从节点，我们要区别是永久下线还是临时下线。例如是否做一些清理工作（如 RDB、AOF 文件的清理），但要考虑一下读写分离的情况。

我们再来看一下节点上线，我们需要把某台 Slave 晋升为主节点，就需要 Sentinel failover 进行替换；对于从节点的上线，我们只需要执行 slaveof 就可以了，Sentinel 节点会根据命令自动感知；对于 Sentinel 节点，我们只需要参考其他 Sentinel 节点启动就可以。

#### 2. 高可用读写分离

大家知道，从节点是高可用的基础，它的扩展功能是读的能力。我们先来看下面这张高可用读写分离使用之前的图。

![enter image description here](http://images.gitbook.cn/d59a7500-0cba-11e8-9706-9106925a3925)

如上图所示，Sentinel 其实是对 Master 做故障转移，对 Slave 只有下线的操作。Sentinel 集群是不会对 Slave 做故障转移的。那么我们应该怎么做呢？

其实我们需要使用一个客户端去监控 Slave，和 Master 类似。主要运用以下命令。

- switch-master：这个命令用来切换主节点，从节点晋升为主节点的操作；
- covert-to-slave：切换从节点，原来主节点需要降为从节点的时候使用该命令。
- sdown：这个命令在主观下线时使用。

我们再来看看使用高可用读写分离之后的图。

![enter image description here](http://images.gitbook.cn/b98fcea0-0cba-11e8-b141-a5ea9a5a91f4)

如图所示，和第一张不同的是，我们把 Slave 的机器全部做到同一个资源池中，让客户端每次都是访问这个 Slave 资源池。

但是这种高可用读写分离在实际应用场景中很少使用，主要因为这种高可用读写分离太过复杂，配置参数比较多。那怎么办呢？

当我们在实际运维中真正需要可扩展的时候，Redis 其实给我们提供了集群的模式，下一篇文章我们将讲解集群的知识。