# 4/9第05课：正确认识 Redis 持久化和开发运维问题

### 5.1 持久化的概念及其作用

首先我们来看下什么是 Redis 的持久化。Redis 的所有数据都保持在内存中，对数据的更新将异步保存到磁盘上。如果 Redis 需要恢复时，就是从硬盘再到内存的过程。

![enter image description here](http://images.gitbook.cn/0b4ffe60-f9b8-11e7-b881-8fdce9c8497b)

由上图可知，持久化就是把内存中的数据保存到硬盘中的过程。，因为 Redis 本身就在内存中运行的；当突然断电或者死机后，我们可以从硬盘中拷贝数据到内存，这是整个过程。

这就是 Redis 持久化的用处。**持久化是为了让硬盘成为备份，以保证数据库数据的完整。**

持久化有哪些方式呢？你会看到网上和书上有很多分类，但是其实大致只分成两种。

- 快照 RDB
- 日志 AOF

### 5.2 快照 RDB

顾名思义，快照就是拍个照片，做个备份，而这种备份是 Redis 自动完成的。

这个功能是 Redis 内置的一个持久化方式，即使你不设置，它也会通过配置文件自动做快照持久化的操作。你可以通过配置信息，设置每过多长时间超过多少条数据做一次快照。具体的操作如下：

```
save 500 100 // 每过 500 秒超过 100 个 key 被修改就执行快照
```

我们再来看看它们的运行状况。

- Redis 调用 fork() 进程，同时拥有父进程和子进程；
- 子进程将数据都写到一个临时 RDB 文件之中；
- 当子进程完成对新 RDB 文件的写入时，Redis 用新 RDB 文件替换旧的 RDB 文件。

这个过程使 Redis 可以从**写时复制**中得到备份。

接下来我们讲解三种 RDB 的触发机制：save（ 同步 ）、bgsave（ 异步 ）和自动触发 RDB 文件。

#### 1. save（同步）

即其他的命令都要排队。

![enter image description here](http://images.gitbook.cn/0548abb0-f9cd-11e7-a366-e3aeb8ba8d5e)

如果有 1000 万条数据，执行 save 命令，Redis 就会对 1000 万条数据打包，而这个时候是同步的，Redis 就会进入阻塞状态。这也是 save 的缺点，接下来我们讲异步命令 bgsave。

#### 2. bgsave（异步命令）

即返回 OK，后台会新开一个线程去执行。

![enter image description here](http://images.gitbook.cn/60ed2100-f9cb-11e7-b881-8fdce9c8497b)

上图可知，客户端会在 Redis 发出 bgsave 命令，另外开一个进程调用 fork() 方法，这个进程同时会创建 RDB 文件，也就是我们上面提到的自动触发 RDB 文件机制。这样，你在父进程里操作别的命令，就不会受影响了。

#### 3. 自动触发 RDB 文件

快照方式虽然是 Redis 自动的，但是如果 Redis 服务器挂掉后，那么最近写入的，是不会被拷入快照中的。所以 RDB 存在两方面的缺点。

- 耗时耗性能：Redis 写 RDB 文件是 dump 操作，所以需要的时间是 O（n），需要所有命令都执行一遍，非常耗时耗性能。
- 容易丢失数据和不可控制：如果我们在某个时间 T1 内写多个命令，这个时候 T2 时间执行 RDB 文件操作，T3 时间又执行多个命令，T4 时间就会出现宕机了。那么 T3 到 T4 的数据就会丢失。

![enter image description here](http://images.gitbook.cn/381c4270-fa60-11e7-9f83-d55f581a6c37)

虽然 RDB 有缺陷，但是依然在生产环境中会使用，RDB 适合冷备，就是当用户数据不高的时候，比如在午夜时分就可使用 RDB 备份。其他时候我们通常使用 AOF 日志备份。

### 5.3 日志 AOF

AOF（Append-only File）是用日志方式，通俗点讲就是当写一条命令的时候，如 Set 某个值，Redis 就会去日志里写一条 Set 某个值的语句。如下图：

![enter image description here](http://images.gitbook.cn/ea14c190-fa61-11e7-acb1-c520391eb94c)

当服务器宕机后，Redis 就会调用 AOF 日志文件，并且这个过程一般是实时的，不需要时间消耗。

![enter image description here](http://images.gitbook.cn/c4eba2a0-fa64-11e7-acb1-c520391eb94c)

上图表示客户端向 AOF 文件写入的时候，是会通过缓冲的，缓冲是系统机制，是为了提高文件的写入效率。

AOF 三种策略分别是 always 、everysec 和 no。

- always

客户端是不会直接把命令写入 AOF 文件的，Liunx 系统会有一个缓冲机制，把一部分命令打包再同步到 AOF 文件，从而提高效率。

但是如果你使用的是 always 命令，就表示每条命令都写入 AOF 文件中，这样是为了保证每条命令都不丢失。

- everysec

即每秒策略，简而言之，就是说每一秒的缓冲区的数据都会刷新到硬盘当中。但是它不像 always 那样，每条数据都会写入硬盘中，如果硬盘发生故障有可能丢失 1 秒的数据。

- no

这个 no 的配置相当于把控制权给了操作系统，操作系统来决定什么时候刷新数据到硬盘，以及不需要我们考虑哪种情况。

| 命令 | always                          | everysec                 | no       |
| :--- | :------------------------------ | :----------------------- | :------- |
| 优点 | 不丢失数据                      | 每秒一次同步，丢一秒数据 | 不需要管 |
| 缺点 | IO 开销大，一般 SATA 盘只有 TPS | 丢一秒数据               | 不可控制 |

关于 AOF，我们补充一点。对于 AOF 操作，Redis 在写入的时候，会压缩命令。它既可以减少硬盘的占用量，同时可以提高恢复硬盘的速度。例如如下表格。

| 原生 AOF      | AOF 复写          |
| :------------ | :---------------- |
| set hello a1  | set hello a3      |
| set hello a2  |                   |
| set hello a3  |                   |
| incr counter  | set counter 2     |
| incr counter  |                   |
| rpush hello a | rpush hello a b c |
| rpush hello b |                   |
| rpush hello c |                   |

从上表可以看到，set hello 有三个值，但是 a1 和 a2 是无效的，最终 AOF 会自动 set 最后一个 a3 的值；incr counter 两次，AOF 自动识别 2 次；rpush 三个值，rpush 会自动简写为一条数据。

针对上面的 Redis 的 AOF 复写。Redis 提供了两种命令。这两种命令是 bgrewriteaof 和 AOF 重写配置。bgrewriteaof 类似 RDB 中的 bgsave 命令，它还是 fork() 子进程，然后完成 AOF 的过程。

AOF 重写配置包含两个配置命令，见如下表格。

| 配置名                      | 含义                   |
| :-------------------------- | :--------------------- |
| auto-aof-rewrite-min-size   | AOF 文件重写最小的尺寸 |
| auto-aof-rewrite-percentage | AOF 文件增长率         |

auto-aof-rewrite-min-size 表示配置最小尺寸，超过这个尺寸就进行重写。

auto-aof-rewrite-percentage ，这里说的是 AOF 文件增长比例，指当前 AOF 文件比上次重写的增长比例大小。AOF 重写即 AOF 文件在一定大小之后，重新将整个内存写到 AOF 文件当中，以反映最新的状态（相当于 bgsave）。这样就避免了 AOF 文件过大而实际内存数据小的问题（频繁修改数据问题）。

接下来看下统计配置，如下表所示，有了它，就可以对上面的配置命令进行控制。

| 配置名           | 含义                               |
| :--------------- | :--------------------------------- |
| aof-current-size | AOF 当前尺寸（字节）               |
| aof-base-size    | AOF 上一次启动和重写的尺寸（字节） |

![enter image description here](http://images.gitbook.cn/fbf5bcb0-fb2f-11e7-8c30-4fe386329305)

由上图可知，bgrewriteaof 命令发出后，Redis 会在父进程中 fork 一个子进程，同时父进程会分别对旧的 AOF 文件和新的 AOF 文件发出 aof_buf 和 aof_rewrite_buf 命令，同时子进程写入新的 AOF 文件，并通知父进程，最后 Redis 使用 aof_rewrite_buf 命令写入新的 AOF 文件。

实际配置过程如下。

```
appendonly yes // appendonly 
 默认是 no
appendfilename "append only - ${port}.aof" //设置 AOF 名字
appendfsync everysec // 每秒同步
dir /diskpath // 新建一个目录
no-appendfsync-on-rewrite yes // 为了减少磁盘压力，AOF 性能上需要权衡。默认是 no，不会丢失数据，但是延迟会比较高。为了减低延迟，一般我们设置成 yes，这样可能丢数据
```

### 5.4 Redis 持久化开发运维时遇到的问题

问题可总结为四种，即 fork 操作、进程外的开销和优化、AOF 追加阻塞和单机多实例部署。

#### 1. fork 操作

fork 操作包括以下三种：

- 同步操作，即 bgsave 时是否进行同步；
- 与内存量息息相关：内存越大，耗时越长；
- info：lastest_fork_usec，持久化操作。

改善 fork 的方式有以下四种：

- 使用物理机或支持高效 fork 的虚拟技术；
- 控制 Redis 实例最大可用内存 maxmemory；
- 合理配置 Liunx 系统内存分配策略：vm.overcommit_memory=1；
- 降低频率，如延长 AOF 重写 RDB，不必要的全量复制。

#### 2. 子进程的开销以及优化

这里主要指 CPU、内存、硬盘三者的开销与优化。

- CPU

  开销：AOF 和 RDB 生成，属于 CPU 密集型，对 CPU 是巨大开销；

  优化：不做 CPU 绑定，不与 CPU 密集型部署。

- 内存

  开销：需要通过 fork 来消耗内存的，如 copy-on-write。

  优化：echo never > /sys/kernel/mm/transparent_hugepage/enabled，有时启动的时候会出现警告的情况，这个时候需要配置这个命令。

![enter image description here](http://images.gitbook.cn/78b91270-fc1f-11e7-b435-c9c42b4c17e4)

- 硬盘

  开销：由于大量的 AOF 和 RDB 文件写入，导致硬盘开销大，建议使用 iostat、iotop 分析硬盘状态。

  优化：

- 不要和高硬盘负载部署到一起，比如存储服务、消息队列等等；

- 配置文件中的 no-appendfsync-on-rewrite 设置成 yes；

- 当写入量很大的时候，建议更换 SSD 硬盘；

- 单机多实例持久化文件考虑硬盘分配分盘。

#### 3. AOF 追加阻塞

我们如果使用 AOF 策略，通常就会使用每秒刷盘的策略（everysec），主线程会阻塞，直到同步完成。首先我们知道主线程是非常宝贵的资源，其次我们每秒刷盘实际上未必是 1 秒，可能是 2 秒的数据。

![enter image description here](http://images.gitbook.cn/47dd7580-fc18-11e7-b435-c9c42b4c17e4)

我们如何定位 AOF 阻塞？

- 通过 Redis 日志

![enter image description here](http://images.gitbook.cn/776f3c10-fc19-11e7-b435-c9c42b4c17e4)

上图可以看到， Redis 日志会出现上述的语句，告诉你异步 IO 同步时间太长，你的硬盘是否有问题，同时会拖慢 Redis。

- 当然除了上述的问题，你还可以用 Redis 的 info 方式来确定问题。

```
info rersistence // 直接在命令中打这个命令即可。
...
aof_delayed_fsync : 100 // 会记录你发生阻塞的次数，每一次加 1
...
...
```

但是这个命令无法看到当前的问题，因为它是历史的累计值。

当然你还可以使用 Liunx 命令 top。

![enter image description here](http://images.gitbook.cn/bf8a2360-fc1a-11e7-afbd-0bc562d2596d)

上图能看到 wa 值，wa 值是表示 IO 瓶颈，如果超过 50%，就表示 IO 出现阻塞了。

本文，我们首先讲了持久化的概念，持久化的方式有 RDB 和 AOF 两种。RDB 主要是 save（同步）、bgsave（异步）、自动触发 RDB 文件三种触发方式；AOF 则是 always、everysec、no 三种方式，同时我们补充了 AOF 重写的命令和如何配置。

最后我们讨论了 Redis 持久化开发运维时遇到的问题，主要有 fork 操作、进程外的开销和优化、AOF 追加阻塞、单机多实例部署四种问题。

这要求我们在做 RDB 和 AOF 备份时，要注意到这些问题。特别是大数据，需要监控 Redis 是否阻塞、开销是否过大等。