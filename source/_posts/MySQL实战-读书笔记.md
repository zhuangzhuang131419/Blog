---
title: MySQL实战-读书笔记
date: 2021-01-12 14:35:27
categories: 数据库
tags: [极客,读书笔记]
---
实习期间在导师的力荐下读到了这部极客网的 MySQL 课程，实在受益匪浅。初次速读，很多东西不能深刻的记在脑中，不能完全消化，先记录下来，日后再度翻阅。这份资料值得多次品味。



# 01.基础架构：一条SQL查询语句是如何执行的？



{% asset_img 01-MySQL的逻辑架构图.jpg 01-MySQL的逻辑架构图 %}

* MySQL可以分为Server层和存储引擎层两部分
  * Server 层包括连接器、查询缓存、分析器、优化器、执行器等，涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。
  * 存储引擎负责数据的存储和获取，是插件式的，InnoDB在5.5.5之后变成了默认的存储引擎。



## 连接器

> 连接器负责跟客户端建立连接、获取权限、维持和管理连接。



```bash
mysql -h$ip -P$port -u root -p
```

一个用户成功建立连接后，即使你用管理员账号对这个用户的权限做了修改，也不会影响已经存在连接的权限。修改完成后，只有再新建的连接才会使用新的权限设置。



可以通过`show processlist`命令来查看当前进程表。

```mysql
mysql> show processlist;
+----+-----------------+-----------+------+---------+------+------------------------+------------------+
| Id | User            | Host      | db   | Command | Time | State                  | Info             |
+----+-----------------+-----------+------+---------+------+------------------------+------------------+
|  5 | event_scheduler | localhost | NULL | Daemon  |  586 | Waiting on empty queue | NULL             |
|  8 | root            | localhost | NULL | Query   |    0 | init                   | show processlist |
+----+-----------------+-----------+------+---------+------+------------------------+------------------+
2 rows in set (0.00 sec)
```



长连接是指连接成功后，如果客户端持续有请求，则一直使用同一个链接。短连接则是指每次执行完很少的几次查询就断开连接，下次查询再重新建立一个。

大部分的并发业务建议不要使用短连接，因为数据库连接是高cost的。但是全部使用长连接又会导致MySQL内存占用的特别快。这是因为 MySQL 在执行过程中临时使用的内存是管理在连接对象里面的。这些资源会在连接断开的时候才释放。所以如果长连接累积下来，可能导致内存占用太大，被系统强行杀掉（OOM），从现象看就是 MySQL 异常重启了。

可以采取以下两种解决方案：

1. 定期断开长连接。
2. 通过执行`mysql_reset_connection`来重新初始化连接资源，将连接恢复到刚刚创建完时的状态。



## 查询缓存

**大多数情况建议不要使用查询缓存**

* 查询缓存的失效非常频繁，只要对一个表的更新，这个表上所有的查询缓存都会被清空。
* 除非你的业务就是一张静态表，很长时间更新一次。比如，一个系统配置表，那这张表上的查询才适合查询缓存。



## 分析器

1. 分析其首先要做“词法分析”。例如：识别出关键字“select”，这就是一个查询语句。
2. 做完识别之后，就要做“语法分析”。判断输入的SQL语句是否满足MySQL语法。



如果表T中没有字段k，而执行了语句`select * from T where k=1`，会报错`Unknown column 'k' in 'where clause'`。这个错是在分析器这个阶段报出来的。



## 优化器

在开始执行前，还需要经过优化器的处理。例如：决定使用哪个索引；在有多表关联的时候，决定各个表的连接顺序。



```mysql
mysql> select * from t1 join t2 using(ID)  where t1.c=10 and t2.d=20;
```

* 可以先从t1中取出c=10的记录的ID值，再根据ID值关联到t2，再判断t2里面d的值是否等于20
* 可以先从t2中取出d=20的记录的ID值，再根据ID值关联到t1，再判断t1里面c的值是否等于10



## 执行器

1. 开始执行前，首先判断一下是否有查询权限。（如果有触发器，得在执行器阶段才能确定）
2. 如果有权限，就打开表继续执行。打开表的时候，执行器根据表的引擎定义，去使用这个引擎提供的接口。



# 02.日志系统：一条SQL更新语句是如何执行的

一条查询语句的执行过程一般是经过连接器、分析器、优化器、执行器等功能模块，最后到达存储引擎。



## redo log（重做日志）

### WAL (Write-Ahead Logging)

* **先写日志，再写磁盘**

* 具体来说，当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。
* 只要事务提交成功，那么对数据库做的修改就被永久保存下来了，不可能因为任何原因再回到原来的状态。



{% asset_img 02-redo log.jpg 02-redo log %}

write pos 是当前记录的位置，一边写一边后移。checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。write pos 和 checkpoint 之间的部分，可以用来记录新的操作。如果 write pos 追上 checkpoint，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。



## binlog（归档日志）

> redo log是InnoDB引擎特有的日志，而Server层也有自己的日志，称为binlog

* binlog和redo log的区别：

  * redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。

  * redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。依靠binlog是没有`crash-safe`能力的。

  * redo log 是循环写的，不持久保存，空间固定会用完，不具备binlog的归档功能；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

  * `crash-safe`是崩溃恢复；binlog恢复是制造一个副本，只能恢复到上个版本。

    

binlog有两种模式：

* statement格式记的是sql语句
* row格式会记录行的内容，更新前和更新后都有





## 更新流程

{% asset_img 02-update语句执行流程.jpg 02-update语句执行流程 %}



把redo log的写入拆成了两个步骤：prepare和commit。这就是两阶段提交。

### 两阶段提交

* 如果先写redo log后写binlog：
  * 由于我们前面说过的，redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 c 的值是 1。但是由于 binlog 没写完就 crash 了，这时候 binlog 里面就没有记录这个语句。因此，之后备份日志的时候，存起来的 binlog 里面就没有这条语句。然后你会发现，如果需要用这个 binlog 来恢复临时库的话，由于这个语句的 binlog 丢失，这个临时库就会少了这一次更新，恢复出来的这一行 c 的值就是 0，与原库的值不同。

* 如果先写binlog再写redo log：
  * 如果在 binlog 写完之后 crash，由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 c 的值是 0。但是 binlog 里面已经记录了“把 c 从 0 改成 1”这个日志。所以，在之后用 binlog 来恢复的时候就多了一个事务出来，恢复出来的这一行 c 的值就是 1，与原库的值不同。



## 在两阶段提交的不同时刻，MySQL异常重启会出现什么现象

* 在写入redo log处于prepare阶段之后、写binlog之前发生了崩溃

  由于此时binlog还没写，rede log也还没提交，所以崩溃恢复的时候，这个事务会回滚。binlog还没写，所以也不会传到备库

* 在binlog写完，redo log还没commit前发生crash

  我们先来看一下崩溃恢复时的判断规则。

  1. 如果 redo log 里面的事务是完整的，也就是已经有了 commit 标识，则直接提交；

  2. 如果 redo log 里面的事务只有完整的 prepare，则判断对应的事务 binlog 是否存在并完整：

     a. 如果是，则提交事务；

     b. 否则，回滚事务。

  这里我们对应的就是2(a)的情况



Q: MySQL怎么知道binlog是完整的？

A: 一个事务的binlog是有完整格式的：

* statement格式的binlog，最后会有COMMIT
* row格式的binlog，最后会有一个XID event
* 在MySQL5.6.2版本后，还引入了binlog-checksum参数，用来验证binlog内容的正确性



Q: redo log 和 binlog 是怎么关联起来的？

A:它们有一个共同的数据字段，叫 XID。



Q: 处于 prepare 阶段的 redo log 加上完整 binlog，重启就能恢复，MySQL 为什么要这么设计?

A: 在binlog 写完以后 MySQL 发生崩溃，这时候 binlog 已经写入了，之后就会被从库（或者用这个 binlog 恢复出来的库）使用。所以，在主库上也要提交这个事务。采用这个策略，主库和备库的数据就保证了一致性。防止恢复时，主备数据库不一致。



集群正常运行的前提：要有完整的binlog来保证从库的数据一致性。要有commit状态的redo log来保证主库的数据的一致性



Q: 能不能只用binlog来支持崩溃恢复，又能支持归档

{% asset_img 02-只用binlog支持崩溃恢复.jpg 02-只用binlog支持崩溃恢复 %}

A: binlog是**逻辑日志**，没有记录数据页的更新细节，binlog没有能力恢复“数据页”。

重启后，引擎内部事务 2 会回滚，然后应用 binlog2 可以补回来；但是对于事务 1 来说，系统已经认为提交完成了，不会再应用一次 binlog1。InnoDB 引擎使用的是 WAL 技术，执行事务的时候，写完内存和日志，事务就算完成了。如果之后崩溃，要依赖于日志来恢复数据页。在图中这个位置发生崩溃的话，事务1也是可能丢失了的，而且是数据页级的丢失。此时，binlog里面没有记录数据页的更新细节，无法补回来。





Q: 能不能只用redo log，不要binlog

A: 如果只考虑崩溃恢复的功能是可以的，但是binlog也有redo log无法代替的功能。

* 归档。redo log是循环写，这样历史日志就无法保留
* MySQL系统依赖于binlog



Q: 数据写入后的最终落盘，是从redo log更新过来的还是从buffer pool更新过来的

A: redo log并没有记录数据页的完整数据，所以它没有能力自己去更新磁盘数据页。

* 如果是正常运行的实例的话，数据页被修改以后，跟磁盘的数据页不一致，称为脏页。最终数据落盘，就是把内存中的数据页写盘。这个过程，甚至与 redo log 毫无关系。
* 在崩溃恢复场景中，InnoDB 如果判断到一个数据页可能在崩溃恢复的时候丢失了更新，就会将它读到内存，然后让 redo log 更新内存内容。更新完成后，内存页变成脏页，就回到了第一种情况的状态。









# 03.事务隔离：为什么你改了我还看不见？







# 18.为什么有的 SQL 语句逻辑相同，性能差异巨大

## 案例一：

```sql
CREATE TABLE `tradelog` (
    `id`            int(11)     NOT     NULL,
    `tradeid`       varchar(32) DEFAULT NULL,
    `operator`      int(11)     DEFAULT NULL,
    `t_modified`    datetime    DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `tradeid` (`tradeid`),
    KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

```sql
mysql> select count(*) from tradelog where month(t_modified) = 7;  # 无法命中索引
mysql> select count(*) from tradelog where 
    (t_modified >= '2016-7-1' and t_modified < '2016-8-1') or
    (t_modified >= '2017-7-1' and t_modified < '2017-8-1') or 
    (t_modified >= '2018-7-1' and t_modified < '2018-8-1') ...     # 可以命中索引
```

> 如果对字段做了函数计算，就用不上索引了。
> 
> 原因是：对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能。

## 案例二：
```sql
# tradeid 的类型是 varchar(32)
mysql> select * from tradelog where tradeid=110717;  # 无法命中索引
mysql> select * from tradelog where CAST(tradidAS signed int) = 110717;   # 等价于执行这条语句
```

**MySQL 类型转换的规则是将字符串转换成数字。**
```sql
mysql> select "10" > 9 # 返回 1
```
```sql
# id 的类型是 int
mysql> select * from tradelog where id="83126";  # 可以命中索引
```

## 案例三：
```sql
CREATE TABLE `trade_detail` (
    `id`            int(11)     NOT NULL,
    `tradeid`       varchar(32) DEFAULT NULL,
    `trade_step`    int(11)     DEFAULT NULL, /*操作步骤*/
    `step_info`     varchar(32) DEFAULT NULL, /*步骤信息*/
    PRIMARY KEY (`id`),
    KEY `tradeid` (`tradeid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

```sql
mysql> select d.* from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2;
# 优化器先在 tradelog 上查到 id=2 的行，使用了主键索引
# 没有使用 trade_detail 上的 trade_id 进行了全表扫描
```

> 原因：因为这两个表的字符集不同，一个是 utf8，一个是 utf8mb4

> 字符集 utf8mb4 是 utf8 的超集，所以在作比较的时候，先把 utf8 转成 utf8mb4。

```sql
mysql> select * from trade_detail where CONVERT(traideid USING utf8mb4)=$L2.tradeid.value; # 等价于执行这条语句
```
**字符集不同只是条件之一，连接过程中要求在被驱动表的索引字段上加函数操作，是导致对被驱动表做全表扫描的原因。**

1. 当使用left join时，左表是驱动表，右表是被驱动表 
2. 当使用right join时，右表时驱动表，左表是驱动表
3. 当使用join时，mysql会选择数据量比较小的表作为驱动表，大表作为被驱动表

作为对比：
```sql
mysql> select l.operator from tradelog l, trade_detail d where d.tradeid=l.tradeid and d.id=4; # 两次查找都可以命中索引


mysql> select operator from tradelog where traideid =CONVERT($R4.tradeid.value USING utf8mb4);
# 与之前不同的是，这里的 CONVERT 函数是加在参数里的

```


# 19 为什么我只查一行的语句，也执行这么慢
```sql
mysql> CREATE TABLE `t` (  
    `id`    int(11) NOT NULL,
    `c`     int(11) DEFAULT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```
我们再往里插入十万条数据

## 查询时间长时间不返回
```sql
mysql> select * from t where id=1;
```

{% asset_img 等MDL锁.png 等MDL锁 %}

出现这个状态表示的是，现在有一个线程正在表上请求或者持有 MDL 写锁，把 ```select``` 语句 block 住了。

有一个 ```flush table``` 命令被别的语句 block 住了，然后又 block 住了我们的 ```select``` 语句。
## 查询慢
> 坏查询不一定是慢查询

```sql
mysql> select * from t where id=1;                    # 慢
mysql> select * from t where id=1 lock in share mode; # 快
```

带 lock in share mode 的 SQL 是当前读，因此会直接读到最后结果；而没有 share mode 的是一致读，要从最后的结果，依次执行 undo log，才能返回正确的结果。

# 20 幻读是什么，幻读有什么问题
## 幻读是什么
> **幻读**指的是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行。

在可重读度隔离级别下，普通的查询时快照读，是不会看到别的事务插入数据的。因此幻读在当前读下才会出现。

## 幻读有什么问题

```sql
CREATE TABLE `t` (
    `id`    int(11) NOT NULL,
    `c`     int(11) DEFAULT NULL,
    `d`     int(11) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

|   |  sessionA  | sessionB | sessionC |
| - | - | - | - |
| T1 | ```begin;``` <br> ```select * from t where d=5 for update;``` | | |
| T2 | | ```update t set d=5 where id=0;``` <br> ```update t set c=5 where id=0;``` | |
| T3 | ```select * from t where d=5 for update;``` | | |
| T4 | | | ```insert into t values(1, 1, 5);``` <br> ```update t set c=5 where id=1;``` |
| T5 | ```select * from t where d=5 for update;``` | | |
| T6 | ```commit;``` | | |


1. 语义上的问题
sessionA 在 T1 时刻声明 “我要将d=5的行锁住，不允许别的事务进行读写操作”。
2. 数据不一致
会导致数据和日志在逻辑上的不一致
```sql
update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/

insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

update t set d=100 where d=5;/*所有d=5的行,d改成100*/
```
拿这个语句序列，无法克隆一个一模一样的库，会存在数据的不一致。

## 如何解决幻读
即使把所有的记录都加上锁，还是阻止不了新插入的记录。
### InnoDB
InnoDB 引入新的锁，也就是间隙锁 (Gap Lock)。在一行行扫描的过程中，不仅将给行加上了行锁，还给两边的空隙，也加上了间隙锁，就可以确保无法再插入新的记录。

间隙锁和行锁合称 **next-key lock**。但是间隙锁的引入，可能会导致同样的语句锁住更大的范围


|   |  sessionA  | sessionB |
| - | - | - | - |
| T1 | ```begin;``` <br> ```select * from t where id=9 for update;``` | |
| T2 | | ```begin;``` <br> ```select * from t where id=9 for update;``` |
| T3 | | ```insert into t values(9, 9, 9)``` <br> (blocked)|
| T4 | ```insert into t values(9, 9, 9);``` <br> (ERROR: DEADLOCK)| |

上述情况出现的原因是 sessionA 被 sessionB 的间隙锁 block 了，同时 sessionB 也被 sessionA block 了。（因为间隙锁之间不会冲突）

## 结论
如果把隔离级别设置为 Read Commit 就没有间隙锁了，但这时有需要解决数据与日志不一致的问题，需要把 binlog 格式设置为 row。

如果业务场景 Read Commit级别够用，即业务不需要可重复读的保证，考虑到读提交下操作数据的锁范围更小，这个选择是合理的。

# 21 为什么我只改一行语句，锁这么多

本篇还是在之前 20 的场景下

## 两个原则，两个优化，一个bug
1. 原则1：加锁的基本单位是 next-key lock。next-key lock 是前开后闭区间。
2. 原则2：查找过程中访问到的对象才会加锁
3. 优化1：索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。
4. 优化2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。
5. 一个bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止

## 案例一：等值查询间隙锁
|   |  sessionA  | sessionB | sessionC |
| - | - | - | - |
| T1 | ```begin;``` <br> ```update t set d=d+1 where id=7;``` | | |
| T2 | | ```insert into t values(8, 8, 8);``` <br> (blocked) | |
| T3 | | | ```update t set d=d+1 where id=10;``` <br> (Query OK) |

根据**优化2**，next-key lock会退化为间隙锁，最终加锁的范围是(5, 10)，所以 sessionC 的修改是可以成功的。

## 案例二：非唯一索引等值锁
|   |  sessionA  | sessionB | sessionC |
| - | - | - | - |
| T1 | ```begin;``` <br> ```select id from t where c=5 lock in share mode;``` | | |
| T2 | | ```update t set d=d+1 where id=5;``` <br> (Query OK) | |
| T3 | | | ```insert into t values(7, 7, 7);``` <br> (blocked) |

1. 根据**原则1**，加锁单位是 next-key lock，因此会给(0, 5]加上 next-key lock。
2. 因为c是普通索引，仅访问c=5这一条记录是不能马上停下来的，需要向右遍历，查到c=10才放弃。根据**原则2**，访问到的都要加锁，因此要给(5, 10]加next-key lock。
3. 根据**优化2**，等值判断，向右遍历，最后一个值不满足c=5这个等值条件，退化成间隙锁(5, 10)
4. 根据**原则2**，只有访问到的对象才会加锁，这个查询使用覆盖索引，并不需要访问主键索引，所以主键索引上没有加任何锁，所以 sessionB 的 update 语句可以执行完成。但是 sessionC 要插入一个(7, 7, 7)的记录，就会被sessionA 的间隙锁锁住。

### 小结
* 锁是加在索引上的
* 如果要用 lock in share mode 来给行加读锁避免数据被更新的话，就必须得绕过覆盖索引的优化，在查询字段中加入所有中不存在的字段。

## 案例三：主键索引范围锁
|   |  sessionA  | sessionB | sessionC |
| - | - | - | - |
| T1 | ```begin;``` <br> ```select * from t where id>=10 and id<11 for update;``` | | |
| T2 | | ```insert into t values(8, 8, 8);``` <br> (Query OK) <br> ```insert into t values(13, 13, 13);``` <br> (blocked) | |
| T3 | | | ```update t set d=d+1 where id=15;``` <br> (blocked) |


1. 开始执行的时候，要找到一个id=10的行，next-key lock(5, 10]。根据**优化一**，主键id上的等值条件，退化成行锁，只加了id=10这一行的行锁。
2. 范围查找就往后继续找，找到id=15这一行，需要next-key lock(10, 15]
3. sessionA 这时候锁的范围就是主键索引上，行锁id=10和next-key lock(10, 15]


## 案例四：非唯一索引范围锁
|   |  sessionA  | sessionB | sessionC |
| - | - | - | - |
| T1 | ```begin;``` <br> ```select * from t where c>=10 and c<11 for update;``` | | |
| T2 | | ```insert into t values(8, 8, 8);``` <br> (blocked)| |
| T3 | | | ```update t set d=d+1 where c=15;``` <br> (blocked) |


1. 由于c是普通索引，不能退化成行锁，因此最终 sessionA 加的锁是，索引c上的(5, 10]和(10, 15]这两个next-key lock
2. InnoDB 要扫到c=15 才知道不需要继续往后找了。
## 案例五：唯一索引范围锁bug
|   |  sessionA  | sessionB | sessionC |
| - | - | - | - |
| T1 | ```begin;``` <br> ```select * from t where id>10 and id<=15 for update;``` | | |
| T2 | | ```update t set d=d+1 where id=20;``` <br> (blocked)| |
| T3 | | | ```insert t values(16, 16, 16);``` <br> (blocked) |

1. 根据**原则1**，应该是索引id上只加(10, 15]这个next-key lock，并且因为id是唯一键，所以循环判断到id=15这一行就应该停止了
2. 但是根据**一个bug**，InnoDB 会往前扫描到第一个不满足条件的行 为止，因此索引id上的(15, 20]这个next-key lock也会被锁上。

## 案例六：非唯一索引上存在“等值”的例子

先插入一条新纪录
```sql
mysql> insert into t values(30, 10, 30);
```

|   |  sessionA  | sessionB | sessionC |
| - | - | - | - |
| T1 | ```begin;``` <br> ```delete from t where c=10;``` | | |
| T2 | | ```insert into t values(12, 12, 12);``` <br> (blocked)| |
| T3 | | | ```update t set d=d+1 where c=15;``` <br> (Query OK) |

{% asset_img 21-案例六.png 21-案例六 %}

两个c=10的记录之间，也是有间隙的。

{% asset_img 21-案例六加锁示意图.png 21-案例六加锁示意图 %}

根据**优化2**，这是一个等值查询，向右查找到了不满足条件的行，所以会退化成 (c=10, id=10) 到 (c=15, id=15)的间隙锁。

## 案例七：limit 语句加锁
|   |  sessionA  | sessionB |
| - | - | - | - |
| T1 | ```begin;``` <br> ```delete from t where c=10 limit 2;``` | |
| T2 | | ```insert into t values(12, 12, 12);``` <br> (Query OK) |

```delete``` 语句明确加了 limit 2 的限制，因此在遍历到(c=10, id=30)这一行后，满足条件的语句就已经有两条，循环就结束了。

{% asset_img 21-案例七加锁示意图.png 21-案例七加锁示意图 %}

> 在删除数据的时候尽量加limit

## 案例八：一个死锁的例子

|   |  sessionA  | sessionB |
| - | - | - | - |
| T1 | ```begin;``` <br> ```select id from t where c=10 lock in share mode;``` | |
| T2 | | ```update t set d=d+1 where c=10;``` <br> (blocked) |
| T3 | ```insert into t values(8, 8, 8)```| |
| T4 | | (ERROR: DEADLOCK) |

1. sessionA 启动事务后执行查询语句加 lock in share mode，在索引c上加了next-key locks(5, 10]和间隙锁(10, 15)
2. sessionB 的update语句也要在索引c上加next-key lock(5, 10]，进入锁等待
3. 然后sessionA 要再插入(8, 8, 8)这一行，被sessionB 的间隙锁锁住。由于出现了死锁，需要回滚。



## 案例九：desc导致的顺序差异

|   |  sessionA  | sessionB |
| - | - | - | - |
| T1 | ```begin;``` <br> ```select * from t where c>=15 and c<=20 order by c desc in share mode;``` | |
| T2 | | ```insert into t values(6, 6, 6);``` <br> (blocked) |

1. 因为这里使用的是 ```desc``` 所以是从右往左扫描的。
2. 根据**优化2**，先判断条件c<=20，普通索引等值c=20，所以间隙锁(20, 25)
3. 20 到 15，所以next-key locks(20, 15]
4. 判断c>=15，普通索引c=15，向左匹配到c=10这个记录，因为next-key locks是前开后闭的，所以只能是(5, 10]
5. 最后的范围是 (5, 15)+[15, 20)+[20, 25)
6. c=20、c=15、c=10 都存在值， select *主键 id 加三个行锁。

如果没有desc 锁的应该是(10, 15] + (15, 20] + (20, 25) 加上 15 20 主键id 两个行锁


# 22 MySQL 有哪些“饮鸩止渴”提高性能的方法

## 短连接风暴
1. 先处理掉那些占着连接但是不工作的线程
* 如果是连接数过多，你可以优先断开事务外空闲太久的连接。
* 如果这样还不够，可以考虑断开事务内空闲太久的连接。 

> 从数据库端断开连接可能是有损的，尤其是有的应用端收到这个错误后，不重新连接，而是直接用这个已经不能用的 handler 重试查询。这会导致从应用端看上去——“MySQL一直没恢复”。

2. 减少连接过程的消耗

如果现在数据库确认是被连接行为打挂了，那么可以让数据库跳过权限验证阶段。重启数据库，并使用 ```-skip-grant-tables``` 参数启动。但是这个方法风险极高。

## 慢查询性能问题
### 索引没有设计好
通过紧急创建索引来解决
1. 在备库B上执行 ```set sql_log_bin=off```, 也就是不写 binlog, 然后执行 ```alter table``` 语句加上索引
2. 执行主备切换
3. 这时候主库是B, 备库是A。在A上执行 ```set sql_log_bin=off```, 然后执行 ```alter table``` 语句加上索引
### SQL 语句没有写好
详见 [18 为什么这些SQL语句逻辑相同，性能却差异巨大]()
### MySQL 选错了索引
详见 [10 MySQL 为什么有时候会选错索引]()
可以通过添加语句 ```force index``` 

### 小结
那么我们如何尽量避免这类性能问题的发生呢？
1. 上线前，在测试环境，把慢查询日志（slow log）打开，并且把long_query_time设置成0，确保每个语句都会被记录入慢查询日志
2. 在测试表里插入模拟线上的数据，做一遍回归测试
3. 观察慢查询日志里每类语句的输出，特别留意Rows_examined字段是否与预期一致。

## QPS 突增问题
有时候由于业务突然出现高峰，或者应用程序bug，导致某个语句的QPS突然暴涨，也可能导致MySQL压力过大，影响服务。



# 23 MySQL 是怎么保证数据不丢的



只要redo log和binlog保证持久化到磁盘，就能确保MySQL异常重启后，数据可以恢复



## binlog 的写入机制
> 事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。



事务提交的时候，执行器把binlog cache里的完整事务写入到binlog中，并清空binlog cache

{% asset_img 23-binlog写盘状态.jpg 23-binlog写盘状态 %}

* 每个线程有自己binlog cache，但是共用一份binlog文件
* 图中的 write，指的就是指把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快。
* 图中的 fsync，才是将数据持久化到磁盘的操作。一般情况下，我们认为 fsync 才占磁盘的 IOPS。



write 和 fsync 的时机，是由参数 sync_binlog 控制的：

1. sync_binlog=0 的时候，表示每次提交事务都只 write，不 fsync；
2. sync_binlog=1 的时候，表示每次提交事务都会执行 fsync；
3. sync_binlog=N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。



## redo log的写入机制

事务还没提交，redo log buffer中的部分日志也有可能被持久化到磁盘。



{% asset_img 23-redo log存储状态.jpg 23-redo log存储状态 %}

1. 存在 redo log buffer 中，物理上是在 MySQL 进程内存中，就是图中的红色部分；
2. 写到磁盘 (write)，但是没有持久化（fsync)，物理上是在文件系统的 page cache 里面，也就是图中的黄色部分；
3. 持久化到磁盘，对应的是 hard disk，也就是图中的绿色部分。



InnoDB有一个后台线程，每隔1秒，就会把redo log buffer中的日志，调用write写到文件系统的page cache，然后调用fsync持久化到磁盘。



实际上，除了后台线程每秒一次的轮询操作外，还有两种场景会让一个没有提交的事务的 redo log 写入到磁盘中。

* **redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘。**注意，由于这个事务并没有提交，所以这个写盘动作只是 write，而没有调用 fsync，也就是只留在了文件系统的 page cache。
* **并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘。**假设一个事务 A 执行到一半，已经写了一些 redo log 到 buffer 中，这时候有另外一个线程的事务 B 提交，如果 innodb_flush_log_at_trx_commit 设置的是 1，那么按照这个参数的逻辑，事务 B 要把 redo log buffer 里的日志全部持久化到磁盘。这时候，就会带上事务 A 在 redo log buffer 里的日志一起持久化到磁盘。



为了控制 redo log 的写入策略，InnoDB 提供了 innodb_flush_log_at_trx_commit 参数，它有三种可能取值：

1. 设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中 ;
2. 设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘；
3. 设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache。



通常我们说MySQL的“双1”配置，指的就是sync_binlog和innodb_flush_log_at_trx_commit都设置成1。也就是说一个事务完整提交前，需要等待两次刷盘，一次是redo log（prepare阶段），一次是binlog



## 组提交

日志逻辑序列号（log sequence number, LSN）是单调递增的，用来对应redo log的一个个写入点。每次写入长度为length




# 24 MySQL 是怎么保证主备一致的？

## MySQL 主备的基本原理


{% asset_img 24-MySQL主备切换流程.png 24-MySQL主备切换流 %}

这就是基本的主备切换流程。

{% asset_img 24-主备流程图.png 24-主备流程图 %}
1. 在备库B上通过 change master 命令，设置主库A 的 IP、端口、用户名、密码，以及要从哪个位置开始请求 binlog，这个位置包含文件名和日志偏移量。
2. 在备库B上执行 start slave 命令，这时候备库会启动两个线程，就是图中的 io_thread 和 sql_thread。其中 io_thread 负责与主库建立连接。
3. 主库A校验完用户名、密码后，开始按照备库B传过来的位置，从本地读取 binlog，发给B。
4. 备库B拿到 binlog 后，写到本地文件，称为中转日志(relay log)
5. sql_thread 读取中转日志，解析出日志里的命令，并执行。


# 参考文献
* [MySQL实战45讲](https://drive.google.com/drive/folders/168dQ754KYC9QFikDy6iTq0mKHWYJkz8p)