title: MySQL 最佳实践
date: 2015-06-21 23:55:00
tags:
- MySQL
categories:
- Develop
---

实习期间接触到最多的数据库除了 Redis，就是 MySQL 了。在做 MySQL 相关操作的时候，发现之前自己玩泥巴时期对 MySQL 的理解是完全不够用的，所以重新看了一些 MySQL 方面的资料，这篇大概算是的 MySQL 方面的实践的总结。

<!-- more -->

测试环境的 Schema：

```sql
CREATE TABLE `sql_test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `text` varchar(100) DEFAULT NULL,
  `created` date DEFAULT NULL,
  `a` int(11) NOT NULL,
  `b` int(11) NOT NULL,
  `c` int(11) NOT NULL,
  `d` int(11) NOT NULL,
  `e` int(11) NOT NULL,
  `f` int(11) NOT NULL,
  `g` int(11) NOT NULL,
  `h` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=100001 DEFAULT CHARSET=utf8;
```

在测试环境下一共生成了 15 万条随机数据。

### 1. 限定查询语句的数量

如果查询语句并不需要全部的结构，使用 `limit` 来限定查询的数量可以大量的节省查询的实践。

例如我们只想知道表中有没有 `e == 6666` 的序列，分别对于一下两种语句：

```python
# SQL Query 1
# SELECT id FROM sql_test WHERE sql_test.e = 6666
sql = select([sql_test.c.id]).where(
    sql_test.c.e == 6666
)
conn.execute(sql).first()

# SQL Query 2
# SELECT id FROM sql_test WHERE sql_test.e = 6666 LIMIT 1
sql = select([sql_test.c.id]).where(
    sql_test.c.e == 6666
).limit(1)
conn.execute(sql).first()
```

分别执行 100 次，两者花费的时间为：

```bash
# SQL Query 1
Cost: 4.616106987

# SQL Query 2
Cost: 0.09752202034
```

### 2. 不要使用 ORDER BY RAND()

如果需要一个满足条件的随机行，不要使用 `ORDER BY RAND()`，因为 `ORDER BY RAND()` 在给你随机的一行之前，会对整张表进行随机排序。

建议自己生成随机数 `rand` 后，限定查询数量为 `rand` 个，然后取最后一个。

例如我们需要查询 `e == 6666` 的随机一行，对于以下两种语句：

```python
# SQL Query 1
# SELECT id FROM sql_test WHERE sql_test.e = 6666 ORDER BY RAND() LIMIT 1
sql = select([sql_test.c.id]).where(
    sql_test.c.e == 6666
).order_by(func.random()).limit(1)
conn.execute(sql).first()

# SQL Query 2
# SELECT id FROM sql_test WHERE sql_test.e = 6666 LIMIT :rand
rand = randint(1, item_cnt)
sql = select([sql_test.c.id]).where(
    sql_test.c.e == 6666
).limit(rand)
conn.execute(sql).fetchall()[:-1]
```

分别执行 100 次，两者花费的时间为：

```bash
# SQL Query 1
Cost: 4.82608103752

# SQL Query 2
Cost: 1.95495295525
```

### 3. 避免 SELECT *

`SELECT *` 是一个很浪费时间的查询过程，应该按需索取，需要什么就查询什么。

例如我们需要查询 `e == 6666` 的前五行信息，对于以下两种语句：

```python
# SQL Query 1
# SELECT * FROM sql_test WHERE sql_test.e = 6666
sql = select("*").where(
    sql_test.c.e == 6666
).limit(5)
conn.execute(sql).first()

# SQL Query 2
# SELECT id FROM sql_test WHERE sql_test.e = 6666
sql = select([sql_test.c.id]).where(
    sql_test.c.e == 6666
).limit(5)
conn.execute(sql).first()
```

分别执行 100 次，两者时间为：

```bash
# SQL Query 1
Cost: 4.07271790504

# SQL Query 2
Cost: 1.93964004517
```

### 4. 为常查询的字段增加索引

索引不止使用于 `Unique` 的列或者主键，如果在程序运行过程总是使用某些字段来查询，可以考虑增加索引来提高查询速度：

例如对于查询 `e=6666` 的前五列：

```python
# SELECT id FROM sql_test WHERE sql_test.e = 6666 
sql = select([sql_test.c.id]).where(
    sql_test.c.e == 6666
).limit(5)
conn.execute(sql).first()
```

执行 100 次，对于增加索引前后：

```bash
# Before index
Cost: 2.00303196907

# After index
Cost: 0.0680429935455
```

索引同样对于 `LIKE` 有效，对于以下查询：

```python
# SELECT id WHERE sql_test.text LIKE 'a%e'
sql = select([sql_test.c.id]).where(
    sql_test.c.text.like("a%e")
)
conn.execute(sql).first()
```

建立索引前后：

```bash
# Before index
Cost: 5.52424716949

# After index
Cost: 0.367518186569
```

### 5. 尽量用 ENUM 代替 VARCHAR

如果可以用 `ENUM` 代替 `VARCHAR`，那么用 `ENUM` 来代替 `VARCHAR`。

相对来说 `ENUM` 要比 `VARCHAR` 快。

对于以下两个表：

```sql
CREATE TABLE `sql_enum_test_a` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `text` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=100001 DEFAULT CHARSET=utf8;

CREATE TABLE `sql_enum_test_b` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `text` enum('success','info','warning','error') DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=100001 DEFAULT CHARSET=utf8;
```

随机生成 10 万条数据后，去查找 `text = 'info'` 的前 1000 行，如下：

```python
# SQL Query 1
# SELECT id FROM sql_enum_test_a WHERE sql_enum_test_a.text = 'info'
sql = select([sql_enum_test_a.c.id]).where(  
    sql_enum_test_a.c.text == 'info'         
).limit(1000)
conn.execute(sql)                 

# SQL Query 2
# SELECT id FROM sql_enum_test_b WHERE sql_enum_test_b.text = 'info'
sql = select([sql_enum_test_b.c.id]).where(  
    sql_enum_test_b.c.text == 'info'         
).limit(1000)
conn.execute(sql)               
```

分别耗时：

```bash
# SQL Query 1
Cost: 0.336887121201

# SQL Query 2
Cost: 0.286870002747
```

### 6. 通过 EXPLAIN 语句来查看查询语句

可以通过 `EXPLAIN` 语句来了解语句的结构或者性能，从而更清楚优化方向。

### 7. 选择正确的 MySQL 引擎

MyISAM 和 InnoDB 是 MySQL 的两种主要引擎。

MyISAM 适用于读较多的场合，在完成 `SELECT COUNT(*)` 类型的语句时比 InnoDB 要快，只支持表级别的锁，而对于大量写的场合性能表现较差。

InnoDB 使用于写较多的场合，支持行级别的锁，初次之外还有很多优秀的特性，可以参考 [「Features of the InnoDB Storage Engine」](http://dev.mysql.com/doc/innodb/1.1/en/innodb-introduction-features.html)。

也可以在下表简单看到对比，在不同场景下该使用哪一种引擎：

| | MyISAM | InnoDB
|----|----|---
| 需要全文搜索| ✓ | >= 5.6.4
| 需要事务 | | ✓
| 频繁从表中获取数据 | ✓ | |
| 频繁插入、更新和删除 | | ✓
| 需要行锁（对于同一个表中的并发操作） |  | ✓ 
| 关系基础的数据库设计 |  | ✓ 

在测试环境下，用进程池大小为 4 ，执行 20 次更新所有 `e == 6666` 的 `text` 字段：

```python
def update_one(cnt):
    print cnt
    # UPDATE sql_test SET text='aaaaa' WHERE sql_test.e = 6666
    sql = update(sql_test).where(sql_test.c.e == 6666).values(text="aaaaa")
    conn.execute(sql)

pool = multiprocessing.Pool(4)
ret = pool.map(update_one, [i for i in range(20)])
```

在 MyISAM 和 InnoDB 下分别耗时为：

```bash
# MyISAM
Cost: 10.5139839649
# InnoDB
Cost: 2.71518301964
```

对于读非常多的情况，还有其他的像 Redis 这样性能更好的解决方案，所以，**尽量使用 InnoDB**。

### 8. 选择合理的排序规则

对于 UTF-8 编码的表，一般来说又两种排序规则：`utf8_general_ci` 和 `utf8_unicode_ci`。

`utf8_unicode_ci` 广泛地适用于各种语言类型，排序非常精确，但排序速度相对慢一些。

`utf8_general_ci` 排序相对要不精确一些，但排序速度较快，因为它针对性能做出了一些优化。

`utf8_general_ci` 和 `utf8_unicode_ci` 排序精确度的差异主要因为在 unicode 排序中，有些字符是不参与排序的，需要跳到下一个字符继续进行字典序排列，`utf8_unicode_ci` 相对于 `utf8_general_ci` 在方面处理得更加合理。

### 9. 尽量保持有 id 一列

尽量保持有一个类型为 INT 的自增的主键。

首先，相对于其他类型的主键，在你查询或者建立某一行要快得多。

其次，由于 MySQL 的内部机制，对于 MySQL 的某些内置操作也有意义，这些内置操作主要通过主键来关联，这样来说主键的性能要重要的多。

这个规则适用于大部分情况，有一个可能的例外是关联表里的外键，会适用其他几张表里的主键共同作为其主键。

### 10. 尽可能适用 NOT NULL

除非有特别的原因，尽量将每一列设为 NOT NULL。

首先，对于很多语言，我们需要特地去分辨 `NULL`、`0` 和 `""` 这样的值。

其次，从 [Limits on Table Column Count and Row Size - MySQL Doc](https://dev.mysql.com/doc/refman/5.0/en/column-count-limit.html) 中能看到，NULL 的一列在比较过程中需要额外的空间。

--------------

参考资料：
1. [Top 20+ MySQL Best Practices](http://code.tutsplus.com/tutorials/top-20-mysql-best-practices--net-7855)  
2. [MyISAM versus InnoDB - Stack Overflow](http://stackoverflow.com/questions/20148/myisam-versus-innodb)  
3. [What's the difference between utf8_general_ci and utf8_unicode_ci - Stack Overflow](http://stackoverflow.com/questions/766809/whats-the-difference-between-utf8-general-ci-and-utf8-unicode-ci)
