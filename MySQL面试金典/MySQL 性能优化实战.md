# MySQL 性能优化实战

鉴于公司项目及业务发展，技术人员从几人到如今几十人，后端团队技术人员日益剧增，可是随着项目人员的增长，大多研发人员及相关人员经常需要到测试环境使用 MySQL 数据库，比如移动端、测试、产品，然而他们需要普及 MySQL 知识点及性能优化的知识，其实性能优化的目标是针对后端研发人员及一些资深的前端人员，可能会从如下大的知识点讲解。

### 一、安装说明

首先学习数据库，当然是安装软件。千里之行，始于足下，如果连安装都不会，如何进行后续的入门学习。可是对于安装也有不同方式，比如 RPM 和源码编译安装呢。

#### 1.1 RPM 安装包和 Tar 安装包的区别？

RPM 直接安装，Tar 属于源码安装，可以设置更多的参数，可以和系统进行更紧密的优化。举个例子，RPM 是 180/96、185/100 之类的标准版型，Tar 源码安装类似于私人定制，量体裁衣的。其实个人觉得 RPM 安装适合小白入门，简单了解下 MySQL，不用做过多安装上的了解，然而源码编译安装适合比如经常跟 MySQL 打交道的工程师，比如 web 研发人员、c++ 工程师等，总不能只是知道如何简单的使用和 curd，其实对于技术人员的纵向知识体系打造是不好的，其实源码编译安装，还可能自己尝试些具体参数的配置。关于 MySQL 官方下载地址：

> https://dev.mysql.com/downloads/mysql/

#### 1.2 安装后需要配置哪些内容？

不管 RPM 还是源码编译安装后，有些东西必须要设置：

- root 初始密码问题：必须设置密码，如果是自己玩还好，如果是线上系统必须设置密码，要不然就是裸跑系统。
- 默认安装后会在指定文件中生成，如果忘记或找不到可以对 root 密码进行强制修改：

```
mysqld_safe –skip-grant-tables 2& 
```

- 用户远程访问问题：从安全性角度默认不允许远程访问，可以进行配置，允许远程访问，但是要注意安全性规范。

```
grant all privileges on . to ‘root’@’%’ identified by’Password’; 
flush privileges;
```

- UTF-8 编码问题： 关于字符集的问题，可能有些技术人员初次学习数据库时或者初次从事研发工作时，偶尔会碰到，为什么前端信息是正常录入，写入数据库时，变成了乱码？查看数据库的编码方式命令为：

```
show variables like 'character%';

参数说明：
character_set_client为客户端编码方式；
character_set_connection为建立连接使用的编码；
character_set_database数据库的编码；
character_set_results结果集的编码；
character_set_server数据库务器的编码；
```

#### 1.3 my.cnf 文件初始需要配置哪些内容？

- 数据文件位置： 确保数据不会把磁盘空间写满，如果有 ssd，可以充分利用 IO 优势。
- 日志文件位置： 快速定位错误日志的位置，根据日志排除错误的能力，是程序员的第一生产力。
- 其他基础参数
- Myisam系列参数（表级锁）：事务--锁--？

```
myisam_sort_buffer_size = 128M   
myisam_max_sort_file_size = 10G   
myisam_max_extra_sort_file_size = 10G
myisam_repair_threads = 1   
myisam_recover   
```

- InnoDB系统参数（行级锁？）：

```
innodb_additional_mem_pool_size = 16M   
innodb_buffer_pool_size = 2048M   
innodb_data_file_path = ibdata1:1024M:autoextend   
innodb_file_io_threads = 4   
innodb_thread_concurrency = 8   
innodb_flush_log_at_trx_commit = 2  
innodb_log_buffer_size = 16M  
innodb_log_file_size = 128M   
innodb_log_files_in_group = 3   
innodb_max_dirty_pages_pct = 90   
innodb_lock_wait_timeout = 120   
innodb_file_per_table = 0 
```

- 其他参数

```
[client]
port = 3306
socket = /data/3306/mysql.sock
[mysqld]
user = mysql
port = 3306
socket = /data/3306/mysql.sock
basedir = /usr/local/mysql
datadir = /data/3306/data
open_files_limit = 10240
[mysqldump]
max_allowed_packet = 32M
[mysqld_safe]
log-error=/data/mysql_err.log
pid-file=/data/mysqld.pid
```

- 常见的 my.cnf 文件类型

```
my-small.cnf 
my-medium.cnf 
my-large.cnf
my-huge.cnf
```

#### 1.4 MySQL 的版本选择

- 5.6 更成熟、更稳定，缺乏一些5.7开始支持的新特性。
- 5.7 支持更多新特性，支持 MGR、JSON 字段格式等。从 5.7 开始，MySQL 对 SQL 语法的检查变为严格，之前一些存在潜在问题和错误的 SQL 会无法执行。
- 8.0 拥有很多新的功能，包括 SQL 方面、JSON 方面以及 DevOps 方面，据说性能提升长达 15 倍。

#### 1.5 MySQL 之外的选择

- Oracle： 免费下载，但是商业用途收费（按 CPU 收费），功能和稳定性更佳，免费和收费的培训资料很多。由于 Oracle 的系统架构较老，代码难以进行整体颠覆性的修改，所以只能在每个版本中进行较小的改进。
- PostgreSQL：（国内有依托阿里德哥推广的强大知识分享社区） 和 MySQL 一样，社区版代码开源，SQL 风格和 Oracle 更加接近，功能和性能也比 MySQL 更加强大，支持 MPP（Greenplum）、LLVM、GIS、列式存储、图计算等特性。普及率相对较低，文档和资料比 Oracle、MySQL 要少。

### 二、MySQL 引擎选择和表设计上的优化

大多 Web 工程师，使用更多的引擎选择和表设计，并且随着业务量发展，会进行不同类型或程度上的优化。5.7 之后默认存储引擎为 InnoDB，主要 InnoDB 能应用绝大数场景。

#### 2.1 Myisam 和 InnoDB 的区别？

其实关于这两个最常用的存储引擎，无非就是看场景，其实没有绝对的好与坏，不要教条主义，适合自己业务的就是最好的。

- Myisam：表级锁，不支持事务，读性能更好，读写分离中做读（从）节点。老版本 MySQL 的默认存储引擎。表的存储分为三个文件：frm表格式，MYD/MYData 数据文件，myi 索引文件。
- InnoDB：有条件的行级锁，支持事务，更适合作为读写分离中的写（主）节点。新版本 MySQL（5.7开始）的默认存储引擎。

#### 2.2 其他的引擎介绍

- XtraDB：XtraDB 是一个 MySQL 的存储引擎，由 Percona 公司对于 InnoDB 存储引擎进行改进加强后的产品，其设计的主要目的是用以替代现在的 InnoDB。XtraDB 兼容 InnoDB 的所有特性，并且在 IO 性能，锁性能，内存管理等多个方面进行了增强。
- TokuDB：TokuDB 是一个高性能、支持事务处理的 MySQL 和 MariaDB 的存储引擎。TokuDB 的主要特点则是对高写压力的支持。

### 三、MySQL SQL 语句的优化

关于 SQL 优化，对于基本大多的 Web 研发人员，注意一个核心点：减少 IO 请求，网络传输。

1.应尽量避免在 where 子句中使用 != 或 <> 操作符，否则将引擎放弃使用索引而进行全表扫描

2.应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描。如：

```
select id from t where ais null
```

可以在 a 上设置默认值 0，确保表中 a 列没有 null 值，然后这样查询：

```
select id from t where a=0
```

3.尽量避免在 where 子句中使用 or 来连接条件，否则将导致引擎放弃使用索引而进行全表扫描，如：

```
select id from t where a=10 or a=20
```

可以这样查询：

```
select id from t where a=10
union all
select id from t where a=20
```

4.下面的查询也将导致全表扫描：

```
select id from t where name like‘%c%’
```

下面走索引

```
select id from t where name like‘c%’
```

若要提高效率，可以考虑全文检索。

5.in 和 not in 也要慎用，否则会导致全表扫描，如：

```
select id from t where a in(1,2,3)
```

对于连续的数值，能用 between 就不要用 in 了：

```
select id from t where a between 1 and 3 
```

如果在 where 子句中使用参数，也会导致全表扫描。因为 SQL 只有在运行时才会解析局部变量，但优化程序不能将访问计划的选择推迟到运行时；它必须在编译时进行选择。然而，如果在编译时建立访问计划，变量的值还是未知的，因而无法作为索引选择的输入项。如下面语句将进行全表扫描：

```
select id from t where a=@a
```

可以改为强制查询使用索引：

```
select id from t with(index(索引名)) where a=@a
```

6.应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。如：

```
select id from t where a/2=100
```

应改为：

```
select id from t where a=100*2
```

7.应尽量避免在 where 子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。如：

```
select id from t where substring(name,1,3)='abc'  name以abc开头的id
select id from t where datediff(day,createdate,'2005-11-30')= 0  '2005-11-30'生成的id
```

应改为:

```
select id from t where name like‘abc%’
select id from t where createdate>='2005-11-30′ and createdate<'2005-12-1′
```

8.不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用索引。

9.在使用索引字段作为条件时，如果该索引是复合索引，那么必须使用到该索引中的第一个字段作为条件时才能保证系统使用该索引，否则该索引将不会被使用，并且应尽可能的让字段顺序与索引顺序相一致。

10.很多时候用 exists 代替 in 是一个好的选择：

```
select num from a where num in (select num from b)
```

用下面的语句替换：

```
select num from a where exists (select 1 from b where num=a.num)
```

11.并不是所有索引对查询都有效，SQL 是根据表中数据来进行查询优化的，当索引列有大量数据重复时，SQL 查询可能不会去利用索引，如一表中有字段 sex，male、female 几乎各一半，那么即使在sex上建了索引也对查询效率起不了作用。

12.索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过6个，若太多则应考虑一些不常使用到的列上建的索引是否有 必要。

13.应尽可能的避免更新 clustered 索引数据列，因为 clustered 索引数据列的顺序就是表记录的物理存储顺序，一旦该列值改变将导致整个表记录的顺序的调整，会耗费相当大的资源。若应用系统需要频繁更新 clustered 索引数据列，那么需要考虑是否应将该索引建为 clustered 索引。

14.尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。这是因为引擎在处理查询和连接时会 逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。

15.尽可能的使用 varchar 代替 char，因为首先变长字段存储空间小，可以节省存储空间，其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。

16.任何地方都不要使用 `select * from t` ，用具体的字段列表代替 `*`，不要返回用不到的任何字段。这里就是典型的减少网络传输，尤其大多数业务中，用户优惠券列表，如果全是*，如果用户数据过多，程序在网络传输过程中会超时。

17.尽量使用表变量来代替临时表。如果表变量包含大量数据，请注意索引非常有限（只有主键索引）。

18.避免频繁创建和删除临时表，以减少系统表资源的消耗。

19.临时表并不是不可使用，适当地使用它们可以使某些例程更有效，例如，当需要重复引用大型表或常用表中的某个数据集时。但是，对于一次性事件，最好使 用导出表。

20.在新建临时表时，如果一次性插入数据量很大，那么可以使用 select into 代替 create table，避免造成大量 log，以提高速度；如果数据量不大，为了缓和系统表的资源，应先create table，然后insert。

21.如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先 truncate table ，然后 drop table ，这样可以避免系统表的较长时间锁定。

22.当只要一行数据时使用 Limit 1。当查询表已经知道结果只会有一条结果，在这种情况下，加上 Limit 1 可以增加性能。MySQ L数据库引擎会在找到一条数据后停止搜索，而不是继续往后查少下一条符合记录的数据。

23.如果应用程序有很多 Join 查询，应该确认两个表中 Join 的字段是被建过索引的。这样，MySQL 内部会启动优化 Join 的 SQL 语句的机制。这些被用来 Join 的字段，应该是相同的类型的。例如：如果要把 DECIMAL 字段和一个 INT 字段 Join 在一起，MySQL 就无法使用它们的索引。对于那些 STRING 类型，还需要有相同的字符集才行。

24.固定长度的表会更快，如果表中的所有字段都是 “固定长度” 的，整个表会被认为是 static 或 fixed-length。例如，表中没有如下类型的字段： VARCHAR，TEXT，BLOB。只要你包括了其中一个这些字段，那么这个表就不是 “固定长度静态表” 了，这样，MySQL 引擎会用另一种方法来处理。固定长度的表会提高性能，因为 MySQL 搜寻得会更快一些，因为这些固定的长度是很容易计算下一个数据的偏移量的，所以读取的自然也会很快。而如果字段不是定长的，那么，每一次要找下一条的话，需要程序找到主键。并且，固定长度的表也更容易被缓存和重建。不过，唯一的副作用是，固定长度的字段会浪费一些空间，因为定长的字段无论你用不用，他都是要分配那么多的空间。

25.尽量避免向客户端返回大数据量，若数据量过大，应该考虑相应需求是否合理。

26.尽量避免大事务操作，提高系统并发能力。

27.不同数据库的 SQL 执行顺序的差别。

28.MySQL Explain 执行计划 type 类型区别：

> 性能从好到差：`system`，`const`，`eq_ref`，`ref`，`fulltext`，`ref_or_null`，`unique_subquery`，`index_subquery`，`range`，`index_merge`，`index`，`all`。

除了 all 之外，其他的 type 都可以使用到索引，除了 index_merge 之外，其他的 type 只可以用到一个索引。

- system：表中只有一行数据或者是空表，且只能用于 myisam 和 memory 表。如果是 InnoDB 引擎表，type 列在这个情况通常都是 all 或者 index
- const：使用唯一索引或者主键，返回记录一定是 1 行记录的等值 where 条件时，通常 type 是 const。也叫做唯一索引扫描。
- `eq_ref`：出现在要连接多个表的查询计划中，驱动表只返回一行数据，且这行数据是第二个表的主键或者唯一索引，且必须为 not null，唯一索引和主键是多列时，只有所有的列都用作比较时才会出现 `eq_ref`。
- ref：不像 eq_ref 那样要求连接顺序，也没有主键和唯一索引的要求，只要使用相等条件检索时就可能出现。常见与辅助索引的等值查找，或者多列主键、唯一索引中，使用第一个列之外的列作为等值查找也会出现，总之，返回数据不唯一的等值查找就可能出现。
- fulltext：全文索引检索，要注意，全文索引的优先级很高，若全文索引和普通索引同时存在时，MySQL不管代价，优先选择使用全文索引。
- `ref_or_null`：与 ref 方法类似，只是增加了 null 值的比较。实际用的不多。
- unique_subquery：用于 where 中的 in 形式子查询，子查询返回不重复值唯一值
- index_subquery：用于 in 形式子查询使用到了辅助索引或者 in 常数列表，子查询可能返回重复值，可以使用索引将子查询去重。
- range：索引范围扫描，常见于使用 >,<,is null,between ,in ,like 等运算符的查询中。
- `index_merge`：表示查询使用了两个以上的索引，最后取交集或者并集，常见 and、or 的条件使用了不同的索引，官方排序这个在 `ref_or_null`之后，但是实际上由于要读取所个索引，性能可能大部分时间都不如 range
- index：索引全表扫描，把索引从头到尾扫一遍，常见于使用索引列就可以处理不需要读取数据文件的查询、可以使用索引排序或者分组的查询。
- all：这个就是全表扫描数据文件，然后再在 server 层进行过滤返回符合要求的记录。

### 四、MySQL 的缺陷与不足

1. 不支持 hash join，大表之间不适合做 joi n操作，没办法满足复杂的OLAP要求。
2. MySQL 不支持函数索引，也不支持并行更新
3. MySQL 连接的 8 小时问题，相对于使用 Oracle 数据库，使用MySQL需要注意更多的细节问题。
4. 对于 SQL 批处理和预编译，支持程度不如 Oracle 数据库。
5. MySQL 优化器还是比较欠缺，不及 Oracle 数据库。

### 五、MySQL 的优点

1. 互联网领域使用较多，文档资料丰富，使用案例非常多，对潜在的问题比较容提前做出应对方案。
2. 由于 MySQL 是开源的数据库，因此很多互联网公司都根据自己的业务需求，开发出了自己的 MySQL 版本，例如阿里云上的 RDS、腾讯云、美团云等。
3. MySQL 相关的开源解决方案众多，无需重复造轮子既可以获得包括读写分离、分库分表等高级特性，例如 Mycat、Sharding-JDBC 等。同时，MySQL 官方的解决方案也越来越丰富，例如 MySQL-Router 等。

### 六、MySQL 读写分离、分库分表

1. 读写分离的数据复制延迟问题 MySQL 通过 binlog 实现数据的复制，也就是主从节点间的数据同步。由于 binlog 复制默认是异步的，因此主从节点之间的数据存在延迟。
2. 分库分表带来的复杂性，难以执行全局的排序、聚合等操作？ ==> 由于同一个表的数据被写到了不同的表，不同的数据库（有可能在不同的服务器节点上），因此如果需要一个同一个表进行聚合操作或者全局的排序，会非常困难，且性能较差。如果是对多个表进行 join 操作，由于每个表都可能存储在多个服务器节点上，因此 join 操作的复杂度会变得很高，需要借助 MPP 的引擎才能完成 join 操作。
3. 目前市面最常用的 MySQL 中间件，无非 mycat（基于阿里 cobar 二次开发）、onesql（业界大牛楼方鑫基于 MySQL 官方，用 c/c++ 二次开发，不过是收费版，功能很强大，不过好像作者重回阿里了）、360 基于 MySQL 分支 Atlas 等。

### 七、MySQL 高可用

当大部分优化或者简单架构设计完成后，再就剩下数据的高可用，毕竟不能由于数据库的不可用导致业务的不可用，并且业务的不可用必然会导致企业损失大量用户，然而这也是技术人员最不愿意看到的，也是技术人员成长过程中的痛点。

在考虑 MySQL 数据库的高可用的架构时，主要要考虑如下几方面：

- 如果数据库发生了宕机或者意外中断等故障，能尽快恢复数据库的可用性，尽可能的减少停机时间，保证业务不会因为数据库的故障而中断
- 用作备份、只读副本等功能的非主节点的数据应该和主节点的数据实时或者最终保持一致。
- 当业务发生数据库切换时，切换前后的数据库内容应当一致，不会因为数据缺失或者数据不一致而影响业务。

关于MySQL高可用，常用架构方案如下：

#### 7.1 主从或主主半同步复制

主从架构基本是基于 binlog，最核心的就是 SQL 线程和 IO 线程。

#### 7.2 半同步复制优化

普通的 replication，即 MySQL 的异步复制，依靠 MySQL 二进制日志也即 binary log 进行数据复制。比如两台机器，一台主机 (master)，另外一台是从机 (slave)。

- 正常的复制为：事务一（t1）写入 binlog buffer；dumper 线程通知 slave 有新的事务 t1；binlog buffer 进行 checkpoint；slave 的 io 线程接收到 t1 并写入到自己的的 relay log；slave 的 sql 线程写入到本地数据库。 这时，master 和 slave 都能看到这条新的事务，即使 master 挂了，slave 可以提升为新的 master。
- 异常的复制为：事务一（t1）写入 binlog buffer；dumper 线程通知 slave 有新的事务 t1；binlog buffer 进行 checkpoint；slave 因为网络不稳定，一直没有收到 t1；master 挂掉，slave 提升为新的 master，t1 丢失。
- 很大的问题是：主机和从机事务更新的不同步，就算是没有网络或者其他系统的异常，当业务并发上来时，slave 因为要顺序执行 master 批量事务，导致很大的延迟。

为了弥补以上几种场景的不足，MySQL 从 5.5 开始推出了半同步。即在 master 的 dumper 线程通知 slave 后，增加了一个 ack，即是否成功收到 t1 的标志码。也就是 dumper 线程除了发送 t1 到 slave，还承担了接收 slave 的 ack 工作。如果出现异常，没有收到 ack，那么将自动降级为普通的复制，直到异常修复。

#### 7.3 高可用架构（MHA + 多节点集群）

MHA Manager 会定时探测集群中的 master 节点，当 master 出现故障时，它可以自动将最新数据的 slave 提升为新的 master，然后将所有其他的 slave 重新指向新的 master，整个故障转移过程对应用程序完全透明。

#### 7.4 zookeeper+proxy

Zookeeper 使用分布式算法保证集群数据的一致性，使用 zookeeper 可以有效的保证 proxy 的高可用性，可以较好的避免网络分区现象的产生。

#### 7.5 共享存储（SAN 共享储存、DRBD 磁盘复制）

共享存储实现了数据库服务器和存储设备的解耦，不同数据库之间的数据同步不再依赖于 MySQL 的原生复制功能，而是通过磁盘数据同步的手段，来保证数据的一致性。

基本上，上述三类架构是最常用的了，对于中小型公司，然而笔者公司也就是用到了上述三种了。

基本，讲述到这里，基本上从 MySQL 基本的安装到引擎选择乃至性能优化及高可用架构等，都捎带详细普及了下，想要 hold 住大部分的性能优化，要考虑的东西还是很多的。毕竟性能优化是个整体概念，宏观层面系统优化：应用服务器，数据库层面：MySQL、Oracle、PG 等。