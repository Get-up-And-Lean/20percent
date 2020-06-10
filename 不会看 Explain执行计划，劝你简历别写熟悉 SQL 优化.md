# 不会看 Explain执行计划，劝你简历别写熟悉 SQL 优化

昨天中午在食堂，和部门的技术大牛们坐在一桌吃饭，作为一个卑微技术渣仔默默的吃着饭，听大佬们高谈阔论，研究各种高端技术，我 TM 也想说话可实在插不上嘴。

聊着聊着突然说到他上午面试了一个工作 6 年的程序员，表情挺复杂，他说：我看他简历写着熟悉`SQL`语句调优，就问了下 `Explain` 执行计划怎么看？结果这老哥一问三不知，工作 6 年这么基础的东西都不了解！

感受到了大佬的王之鄙视，回到工位我就开始默默写这个，哎~ 我 TM 也不太懂 `Explain` ，老哥你这是针对我啊！哭唧唧~

![](https://gitee.com/jinxin.70/oss/raw/master/uPic/Et8pwF_2020_06_09_11_15_36.jpg)

### Explain 有什么用

当`Explain` 与 `SQL`语句一起使用时，`MySQL` 会显示来自优化器关于 SQL 执行的信息。也就是说，`MySQL`解释了它将如何处理该语句，包括如何连接表以及什么顺序连接表等。

- 表的加载顺序
- `sql` 的查询类型
- 可能用到哪些索引，哪些索引又被实际使用
- 表与表之间的引用关系
- 一个表中有多少行被优化器查询 …..

### Explain 有哪些信息

`Explain` 执行计划包含字段信息如下：分别是 `id`、`select_type`、`table`、`partitions`、`type`、`possible_keys`、`key`、`key_len`、`ref`、`rows`、`filtered`、`Extra` 12 个字段。

![](https://gitee.com/jinxin.70/oss/raw/master/uPic/tdX3Vx_2020_06_09_11_15_57.jpg)

下边我们会结合具体的`SQL`示例，详细的解读每个字段以及每个字段中不同参数的含义，以下所有示例数据库版本为 `MySQL.5.7.17`。

```
mysql> select version() from dual;
+------------+
| version()  |
+------------+
| 5.7.17-log |
+------------+
```

我们创建三张表 `one`、`two`、`three`，表之间的关系 `one.two_id = two.two_id AND two.three_id = three.three_id`。

### Explain 执行计划详解

#### 一、id

`id：` ：表示查询中执行 select 子句或者操作表的顺序，**`id`的值越大，代表优先级越高，越先执行**。 `id`大致会出现 3 种情况：

##### 1、`id`相同

看到三条记录的`id`都相同，可以理解成这三个表为一组，具有同样的优先级，执行顺序由上而下，具体顺序由优化器决定。

```bash
mysql> EXPLAIN SELECT * FROM one o,two t, three r WHERE o.two_id = t.two_id AND t.three_id = r.three_id;
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref                  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | o     | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                 |    2 |      100 | NULL                                               |
|  1 | SIMPLE      | t     | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL                 |    2 |       50 | Using where; Using join buffer (Block Nested Loop) |
|  1 | SIMPLE      | r     | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | xin-slave.t.three_id |    1 |      100 | NULL                                               |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------------+------+----------+----------------------------------------------------+
```

##### 2、`id`不同

如果我们的 `SQL` 中存在子查询，那么 `id`的序号会递增，`id`值越大优先级越高，越先被执行 。当三个表依次嵌套，发现最里层的子查询 `id`最大，最先执行。

```
mysql> EXPLAIN select * from one o where o.two_id = (select t.two_id from two t where t.three_id = (select r.three_id  from three r where r.three_name='我是第三表 2'));
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | PRIMARY     | o     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |       50 | Using where |
|  2 | SUBQUERY    | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |       50 | Using where |
|  3 | SUBQUERY    | r     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |       50 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

\##### 3、以上两种同时存在

将上边的 `SQL` 稍微修改一下，增加一个子查询，发现 `id`的以上两种同时存在。相同`id`划分为一组，这样就有三个组，同组的从上往下顺序执行，不同组 `id`值越大，优先级越高，越先执行。

```
mysql>  EXPLAIN select * from one o where o.two_id = (select t.two_id from two t where t.three_id = (select r.three_id  from three r where r.three_name='我是第三表 2')) AND o.one_id in(select one_id from one where o.one_name="我是第一表 2");
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref                | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------------+------+----------+-------------+
|  1 | PRIMARY     | o     | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL               |    2 |       50 | Using where |
|  1 | PRIMARY     | one   | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | xin-slave.o.one_id |    1 |      100 | Using index |
|  2 | SUBQUERY    | t     | NULL       | ALL    | NULL          | NULL    | NULL    | NULL               |    2 |       50 | Using where |
|  3 | SUBQUERY    | r     | NULL       | ALL    | NULL          | NULL    | NULL    | NULL               |    2 |       50 | Using where |
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------------+------+----------+-------------+
```

#### 二、select_type

`select_type`：表示 `select` 查询的类型，主要是用于区分各种复杂的查询，例如：`普通查询`、`联合查询`、`子查询`等。

##### 1、SIMPLE

`SIMPLE`：表示最简单的 select 查询语句，也就是在查询中不包含子查询或者 `union`交并差集等操作。

##### 2、PRIMARY

`PRIMARY`：当查询语句中包含任何复杂的子部分，最外层查询则被标记为`PRIMARY`。

##### 3、SUBQUERY

`SUBQUERY`：当 `select` 或 `where` 列表中包含了子查询，该子查询被标记为：`SUBQUERY` 。

##### 4、DERIVED

`DERIVED`：表示包含在`from`子句中的子查询的 select，在我们的 `from` 列表中包含的子查询会被标记为`derived` 。

##### 5、UNION

`UNION`：如果`union`后边又出现的`select` 语句，则会被标记为`union`；若 `union` 包含在 `from` 子句的子查询中，外层 `select` 将被标记为 `derived`。

##### 6、UNION RESULT

`UNION RESULT`：代表从`union`的临时表中读取数据，而`table`列的`<union1,4>`表示用第一个和第四个`select`的结果进行`union`操作。

```
mysql> EXPLAIN select t.two_name, ( select one.one_id from one) o from (select two_id,two_name from two where two_name ='') t  union (select r.three_name,r.three_id from three r);

+------+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
| id   | select_type  | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra           |
+------+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
|    1 | PRIMARY      | two        | NULL       | ALL   | NULL          | NULL    | NULL    | NULL |    2 |       50 | Using where     |
|    2 | SUBQUERY     | one        | NULL       | index | NULL          | PRIMARY | 4       | NULL |    2 |      100 | Using index     |
|    4 | UNION        | r          | NULL       | ALL   | NULL          | NULL    | NULL    | NULL |    2 |      100 | NULL            |
| NULL | UNION RESULT | <union1,4> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL | NULL | NULL     | Using temporary |
+------+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
```

#### 三、table

查询的表名，并不一定是真实存在的表，有别名显示别名，也可能为临时表，例如上边的`DERIVED`、 `<union1,4>`等。

#### 四、partitions

查询时匹配到的分区信息，对于非分区表值为`NULL`，当查询的是分区表时，`partitions`显示分区表命中的分区情况。

```
+----+-------------+----------------+---------------------------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table          | partitions                      | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------------+---------------------------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | one            | p201801,p201802,p201803,p300012 | index | NULL          | PRIMARY | 9       | NULL |    3 |      100 | Using index |
+----+-------------+----------------+---------------------------------+-------+---------------+---------+---------+------+------+----------+-------------+
```

#### 五、type

```
type`：查询使用了何种类型，它在 `SQL`优化中是一个非常重要的指标，以下性能从好到坏依次是：`system` > `const` > `eq_ref` > `ref` > `ref_or_null` > `index_merge` > `unique_subquery` > `index_subquery` > `range` > `index` > `ALL
```

##### 1、system

`system`： 当表仅有一行记录时(系统表)，数据量很少，往往不需要进行磁盘 IO，速度非常快。

##### 2、const

`const`：表示查询时命中 `primary key` 主键或者 `unique` 唯一索引，或者被连接的部分是一个常量(`const`)值。这类扫描效率极高，返回数据量少，速度非常快。

```
mysql> EXPLAIN SELECT * from three where three_id=1;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | three | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |      100 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
```

##### 3、eq_ref

`eq_ref`：查询时命中主键`primary key` 或者 `unique key`索引， `type` 就是 `eq_ref`。

```
mysql> EXPLAIN select o.one_name from one o ,two t where o.one_id = t.two_id ; 
+----+-------------+-------+------------+--------+---------------+----------+---------+--------------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key      | key_len | ref                | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+----------+---------+--------------------+------+----------+-------------+
|  1 | SIMPLE      | o     | NULL       | index  | PRIMARY       | idx_name | 768     | NULL               |    2 |      100 | Using index |
|  1 | SIMPLE      | t     | NULL       | eq_ref | PRIMARY       | PRIMARY  | 4       | xin-slave.o.one_id |    1 |      100 | Using index |
+----+-------------+-------+------------+--------+---------------+----------+---------+--------------------+------+----------+-------------+
```

##### 4、ref

`ref`：区别于`eq_ref` ，`ref`表示使用非唯一性索引，会找到很多个符合条件的行。

```
mysql> select o.one_id from one o where o.one_name = "xin" ; 
+--------+
| one_id |
+--------+
|      1 |
|      3 |
+--------+
mysql> EXPLAIN select o.one_id from one o where o.one_name = "xin" ; 
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | o     | NULL       | ref  | idx_name      | idx_name | 768     | const |    1 |      100 | Using index |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
```

##### 5、ref*or*null

`ref_or_null`：这种连接类型类似于 ref，区别在于 `MySQL`会额外搜索包含`NULL`值的行。

```
mysql> EXPLAIN select o.one_id from one o where o.one_name = "xin" OR o.one_name IS NULL; 
+----+-------------+-------+------------+-------------+---------------+----------+---------+-------+------+----------+--------------------------+
| id | select_type | table | partitions | type        | possible_keys | key      | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+-------+------------+-------------+---------------+----------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | o     | NULL       | ref_or_null | idx_name      | idx_name | 768     | const |    3 |      100 | Using where; Using index |
+----+-------------+-------+------------+-------------+---------------+----------+---------+-------+------+----------+--------------------------+
```

##### 6、index_merge

`index_merge`：使用了索引合并优化方法，查询使用了两个以上的索引。

下边示例中同时使用到主键`one_id` 和 字段`one_name`的`idx_name` 索引 。

```
mysql> EXPLAIN select * from one o where o.one_id >1 and o.one_name ='xin'; 
+----+-------------+-------+------------+-------------+------------------+------------------+---------+------+------+----------+------------------------------------------------+
| id | select_type | table | partitions | type        | possible_keys    | key              | key_len | ref  | rows | filtered | Extra                                          |
+----+-------------+-------+------------+-------------+------------------+------------------+---------+------+------+----------+------------------------------------------------+
|  1 | SIMPLE      | o     | NULL       | index_merge | PRIMARY,idx_name | idx_name,PRIMARY | 772,4   | NULL |    1 |      100 | Using intersect(idx_name,PRIMARY); Using where |
+----+-------------+-------+------------+-------------+------------------+------------------+---------+------+------+----------+------------------------------------------------+
```

##### 7、unique_subquery

`unique_subquery`：替换下面的 `IN`子查询，子查询返回不重复的集合。

```
value IN (SELECT primary_key FROM single_table WHERE some_expr)
```

##### 8、index_subquery

`index_subquery`：区别于`unique_subquery`，用于非唯一索引，可以返回重复值。

```
value IN (SELECT key_column FROM single_table WHERE some_expr)
```

##### 9、range

`range`：使用索引选择行，仅检索给定范围内的行。简单点说就是针对一个有索引的字段，给定范围检索数据。在`where`语句中使用 `bettween...and`、`<`、`>`、`<=`、`in` 等条件查询 `type` 都是 `range`。

举个栗子：`three`表中`three_id`为唯一主键，`user_id`普通字段未建索引。

```
mysql> EXPLAIN SELECT * from three where three_id BETWEEN 2 AND 3;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | three | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    1 |      100 | Using where |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
```

从结果中看到只有对设置了索引的字段，做范围检索 `type` 才是 `range`。

```
mysql> EXPLAIN SELECT * from three where user_id BETWEEN 2 AND 3;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | three | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

##### 10、index

`index`：`Index` 与`ALL` 其实都是读全表，区别在于`index`是遍历索引树读取，而`ALL`是从硬盘中读取。

下边示例：`three_id` 为主键，不带 `where` 条件全表查询 ，`type`结果为`index` 。

```
mysql> EXPLAIN SELECT three_id from three ;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | three | NULL       | index | NULL          | PRIMARY | 4       | NULL |    1 |      100 | Using index |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
```

##### 11、ALL

`ALL`：将遍历全表以找到匹配的行，性能最差。

```
mysql> EXPLAIN SELECT * from two ;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | two   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |      100 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
```

#### 六、possible_keys

`possible_keys`：表示在`MySQL`中通过哪些索引，能让我们在表中找到想要的记录，一旦查询涉及到的某个字段上存在索引，则索引将被列出，**但这个索引并不定一会是最终查询数据时所被用到的索引**。具体请参考上边的例子。

#### 七、key

`key`：区别于`possible_keys`，key 是查询中实际使用到的索引，若没有使用索引，显示为`NULL`。具体请参考上边的例子。

> 当 `type` 为 `index_merge` 时，可能会显示多个索引。

#### 八、key_len

`key_len`：表示查询用到的索引长度（字节数），原则上长度越短越好 。

- 单列索引，那么需要将整个索引长度算进去；
- 多列索引，不是所有列都能用到，需要计算查询中实际用到的列。

> 注意：`key_len`只计算`where`条件中用到的索引长度，而排序和分组即便是用到了索引，也不会计算到`key_len`中。

#### 九、ref

`ref`：常见的有：`const`，`func`，`null`，字段名。

- 当使用常量等值查询，显示`const`，
- 当关联查询时，会显示相应关联表的`关联字段`
- 如果查询条件使用了`表达式`、`函数`，或者条件列发生内部隐式转换，可能显示为`func`
- 其他情况`null`

#### 十、rows

`rows`：以表的统计信息和索引使用情况，估算要找到我们所需的记录，需要读取的行数。

这是评估`SQL` 性能的一个比较重要的数据，`mysql`需要扫描的行数，很直观的显示 `SQL` 性能的好坏，一般情况下 `rows` 值越小越好。

```
mysql> EXPLAIN SELECT * from three;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | three | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |      100 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
```

#### 十一、filtered

`filtered` 这个是一个百分比的值，表里符合条件的记录数的百分比。简单点说，这个字段表示存储引擎返回的数据在经过过滤后，剩下满足条件的记录数量的比例。

在`MySQL.5.7`版本以前想要显示`filtered`需要使用`explain extended`命令。`MySQL.5.7`后，默认`explain`直接显示`partitions`和`filtered`的信息。

#### 十二、Extra

`Extra` ：不适合在其他列中显示的信息，`Explain` 中的很多额外的信息会在 `Extra` 字段显示。

##### 1、Using index

`Using index`：我们在相应的 `select` 操作中使用了覆盖索引，通俗一点讲就是查询的列被索引覆盖，使用到覆盖索引查询速度会非常快，`SQl`优化中理想的状态。

什么又是覆盖索引?

一条 `SQL`只需要通过索引就可以返回，我们所需要查询的数据（一个或几个字段），而不必通过二级索引，查到主键之后再通过主键查询整行数据（`select *` ）。

`one_id`表为主键

```
mysql> EXPLAIN SELECT one_id from one ;
+----+-------------+-------+------------+-------+---------------+------------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key        | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+------------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | one   | NULL       | index | NULL          | idx_two_id | 5       | NULL |    3 |      100 | Using index |
+----+-------------+-------+------------+-------+---------------+------------+---------+------+------+----------+-------------+
```

**注意**：想要使用到覆盖索引，我们在 `select` 时只取出需要的字段，不可`select *`，而且该字段建了索引。

```
mysql> EXPLAIN SELECT * from one ;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | one   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |      100 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
```

##### 2、Using where

`Using where`：查询时未找到可用的索引，进而通过`where`条件过滤获取所需数据，但要注意的是并不是所有带`where`语句的查询都会显示`Using where`。

下边示例`create_time` 并未用到索引，`type` 为 `ALL`，即`MySQL`通过全表扫描后再按`where`条件筛选数据。

```
mysql> EXPLAIN SELECT one_name from one where create_time ='2020-05-18';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | one   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

##### 3、Using temporary

`Using temporary`：表示查询后结果需要使用临时表来存储，一般在排序或者分组查询时用到。

```
mysql> EXPLAIN SELECT one_name from one where one_id in (1,2) group by one_name;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | one   | NULL       | range| NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where; Using temporary; Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

##### 4、Using filesort

`Using filesort`：表示无法利用索引完成的排序操作，也就是`ORDER BY`的字段没有索引，通常这样的 SQL 都是需要优化的。

```
mysql> EXPLAIN SELECT one_id from one  ORDER BY create_time;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | one   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |      100 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
```

如果`ORDER BY`字段有索引就会用到覆盖索引，相比执行速度快很多。

```
mysql> EXPLAIN SELECT one_id from one  ORDER BY one_id;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | one   | NULL       | index | NULL          | PRIMARY | 4       | NULL |    3 |      100 | Using index |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
```

##### 5、Using join buffer

`Using join buffer`：在我们联表查询的时候，如果表的连接条件没有用到索引，需要有一个连接缓冲区来存储中间结果。

先看一下有索引的情况：连接条件 `one_name` 、`two_name` 都用到索引。

```
mysql> EXPLAIN SELECT one_name from one o,two t where o.one_name = t.two_name;
+----+-------------+-------+------------+-------+---------------+----------+---------+----------------------+------+----------+--------------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref                  | rows | filtered | Extra                    |
+----+-------------+-------+------------+-------+---------------+----------+---------+----------------------+------+----------+--------------------------+
|  1 | SIMPLE      | o     | NULL       | index | idx_name      | idx_name | 768     | NULL                 |    3 |      100 | Using where; Using index |
|  1 | SIMPLE      | t     | NULL       | ref   | idx_name      | idx_name | 768     | xin-slave.o.one_name |    1 |      100 | Using index              |
+----+-------------+-------+------------+-------+---------------+----------+---------+----------------------+------+----------+--------------------------+
```

接下来删掉 连接条件 `one_name` 、`two_name` 的字段索引。发现`Extra` 列变成 `Using join buffer`，`type`均为全表扫描，这也是`SQL`优化中需要注意的地方。

```
mysql> EXPLAIN SELECT one_name from one o,two t where o.one_name = t.two_name;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |      100 | NULL                                               |
|  1 | SIMPLE      | o     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
```

##### 6、Impossible where

`Impossible where`：表示在我们用不太正确的`where`语句，导致没有符合条件的行。

```
mysql> EXPLAIN SELECT one_name from one WHERE 1=2;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra            |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL | NULL     | Impossible WHERE |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------+
```

##### 7、No tables used

`No tables used`：我们的查询语句中没有`FROM`子句，或者有 `FROM DUAL`子句。

```
mysql> EXPLAIN select now();
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL | NULL     | No tables used |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
```

`Extra`列的信息非常非常多，这里就不再一一列举了，详见 `MySQL`官方文档 ：https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype*index*merge ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200520124417784.png)

### 总结

上边只是简单介绍了下 `Explain` 执行计划各个列的含义，了解它不仅仅是要应付面试，在实际开发中也经常会用到。比如对慢`SQL`进行分析，如果连执行计划结果都不会看，那还谈什么`SQL`优化呢？