---
title: MySQL实战-读书笔记
date: 2021-01-12 14:35:27
tags: 极客
categories: 数据库
---
实习期间在导师的力荐下读到了这部极客网的 MySQL 课程，实在受益匪浅。初次速读，很多东西不能深刻的记在脑中，不能完全消化，先记录下来，日后再度翻阅。这份资料值得多次品味。

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

作为对比：
```sql
mysql> select l.operator from tradelog l, trade_detail d where d.tradeid=l.tradeid and d.id=4; # 两次查找都可以命中索引


mysql> select operator from tradelog where traideid =CONVERT($R4.tradeid.value USING utf8mb4);
# 与之前不同的是，这里的 CONVERT 函数是加在参数里的

```

# 参考文献
* [MySQL实战45讲](https://drive.google.com/drive/folders/168dQ754KYC9QFikDy6iTq0mKHWYJkz8p)