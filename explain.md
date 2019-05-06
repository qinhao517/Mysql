# MySQL 性能优化神器 Explain 使用分析
## 简介
#### MySQL 提供了一个 EXPLAIN 命令, 它可以对 SELECT 语句进行分析, 并输出 SELECT 执行的详细信息, 以供开发人员针对性优化.
EXPLAIN 命令用法十分简单, 在 SELECT 语句前加上 Explain 就可以了, 例如:
```
EXPLAIN SELECT * from user_info WHERE id < 300;
```
## 准备
#### 为了接下来方便演示 EXPLAIN 的使用, 首先我们需要建立两个测试用的表, 并添加相应的数据:
```
CREATE TABLE `user_info` (
  `id`   BIGINT(20)  NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(50) NOT NULL DEFAULT '',
  `age`  INT(11)              DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `name_index` (`name`)
)
  ENGINE = InnoDB
  DEFAULT CHARSET = utf8

INSERT INTO user_info (name, age) VALUES ('xys', 20);
INSERT INTO user_info (name, age) VALUES ('a', 21);
INSERT INTO user_info (name, age) VALUES ('b', 23);
INSERT INTO user_info (name, age) VALUES ('c', 50);
INSERT INTO user_info (name, age) VALUES ('d', 15);
INSERT INTO user_info (name, age) VALUES ('e', 20);
INSERT INTO user_info (name, age) VALUES ('f', 21);
INSERT INTO user_info (name, age) VALUES ('g', 23);
INSERT INTO user_info (name, age) VALUES ('h', 50);
INSERT INTO user_info (name, age) VALUES ('i', 15);
CREATE TABLE `order_info` (
  `id`           BIGINT(20)  NOT NULL AUTO_INCREMENT,
  `user_id`      BIGINT(20)           DEFAULT NULL,
  `product_name` VARCHAR(50) NOT NULL DEFAULT '',
  `productor`    VARCHAR(30)          DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `user_product_detail_index` (`user_id`, `product_name`, `productor`)
)
  ENGINE = InnoDB
  DEFAULT CHARSET = utf8

INSERT INTO order_info (user_id, product_name, productor) VALUES (1, 'p1', 'WHH');
INSERT INTO order_info (user_id, product_name, productor) VALUES (1, 'p2', 'WL');
INSERT INTO order_info (user_id, product_name, productor) VALUES (1, 'p1', 'DX');
INSERT INTO order_info (user_id, product_name, productor) VALUES (2, 'p1', 'WHH');
INSERT INTO order_info (user_id, product_name, productor) VALUES (2, 'p5', 'WL');
INSERT INTO order_info (user_id, product_name, productor) VALUES (3, 'p3', 'MA');
INSERT INTO order_info (user_id, product_name, productor) VALUES (4, 'p1', 'WHH');
INSERT INTO order_info (user_id, product_name, productor) VALUES (6, 'p1', 'WHH');
INSERT INTO order_info (user_id, product_name, productor) VALUES (9, 'p8', 'TE');
```
## EXPLAIN 输出格式
#### EXPLAIN 命令的输出内容大致如下:
```
mysql> explain select * from user_info where id = 2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```
##### 各列的含义如下:

id: SELECT 查询的标识符. 每个 SELECT 都会自动分配一个唯一的标识符.

select_type: SELECT 查询的类型.

table: 查询的是哪个表

partitions: 匹配的分区

type: join 类型

possible_keys: 此次查询中可能选用的索引

key: 此次查询中确切使用到的索引.

ref: 哪个字段或常数与 key 一起被使用

rows: 显示此查询一共扫描了多少行. 这个是一个估计值.

filtered: 表示此查询条件所过滤的数据的百分比

extra: 额外的信息

接下来我们来重点看一下比较重要的几个字段.
##### select_type

select_type 表示了查询的类型, 它的常用取值有:

SIMPLE, 表示此查询不包含 UNION 查询或子查询

PRIMARY, 表示此查询是最外层的查询

UNION, 表示此查询是 UNION 的第二或随后的查询

DEPENDENT UNION, UNION 中的第二个或后面的查询语句, 取决于外面的查询

UNION RESULT, UNION 的结果

SUBQUERY, 子查询中的第一个 SELECT

DEPENDENT SUBQUERY: 子查询中的第一个 SELECT, 取决于外面的查询. 即子查询依赖于外层查询的结果.

最常见的查询类别应该是 SIMPLE 了, 比如当我们的查询没有子查询, 也没有 UNION 查询时, 那么通常就是 SIMPLE 类型, 例如:
```
mysql> explain select * from user_info where id = 2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```
##### 如果我们使用了 UNION 查询, 那么 EXPLAIN 输出 的结果类似如下:
```
mysql> EXPLAIN (SELECT * FROM user_info  WHERE id IN (1, 2, 3))
    -> UNION
    -> (SELECT * FROM user_info WHERE id IN (3, 4, 5));
+----+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
| id | select_type  | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra           |
+----+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
|  1 | PRIMARY      | user_info  | NULL       | range | PRIMARY       | PRIMARY | 8       | NULL |    3 |   100.00 | Using where     |
|  2 | UNION        | user_info  | NULL       | range | PRIMARY       | PRIMARY | 8       | NULL |    3 |   100.00 | Using where     |
| NULL | UNION RESULT | <union1,2> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL | NULL |     NULL | Using temporary |
+----+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------+
3 rows in set, 1 warning (0.00 sec)
```
#### table
##### 表示查询涉及的表或衍生表

#### type
##### type 字段比较重要, 它提供了判断查询是否高效的重要依据依据. 通过 type 字段, 我们判断此次查询是 全表扫描 还是 索引扫描 等.
##### type 常用类型
type 常用的取值有:

system: 表中只有一条数据. 这个类型是特殊的 const 类型.

const: 针对主键或唯一索引的等值查询扫描, 最多只返回一行数据. const 查询速度非常快, 因为它仅仅读取一次即可.
例如下面的这个查询, 它使用了主键索引, 因此 type 就是 const 类型的.
```
mysql> explain select * from user_info where id = 2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```
#### eq_ref: 此类型通常出现在多表的 join 查询, 表示对于前表的每一个结果, 都只能匹配到后表的一行结果. 并且查询的比较操作通常是 =, 查询效率较高. 例如:
```
mysql> EXPLAIN SELECT * FROM user_info, order_info WHERE user_info.id = order_info.user_id\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: order_info
   partitions: NULL
         type: index
possible_keys: user_product_detail_index
          key: user_product_detail_index
      key_len: 314
          ref: NULL
         rows: 9
     filtered: 100.00
        Extra: Using where; Using index
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: test.order_info.user_id
         rows: 1
     filtered: 100.00
        Extra: NULL
2 rows in set, 1 warning (0.00 sec)
```
#### ref: 此类型通常出现在多表的 join 查询, 针对于非唯一或非主键索引, 或者是使用了 最左前缀 规则索引的查询. 
例如下面这个例子中, 就使用到了 ref 类型的查询:
```
mysql> EXPLAIN SELECT * FROM user_info, order_info WHERE user_info.id = order_info.user_id AND order_info.user_id = 5\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: order_info
   partitions: NULL
         type: ref
possible_keys: user_product_detail_index
          key: user_product_detail_index
      key_len: 9
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using index
2 rows in set, 1 warning (0.01 sec)
```
###### range: 表示使用索引范围查询, 通过索引字段范围获取表中部分数据记录. 这个类型通常出现在 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, IN() 操作中.
当 type 是 range 时, 那么 EXPLAIN 输出的 ref 字段为 NULL, 并且 key_len 字段是此次查询中使用到的索引的最长的那个.

例如下面的例子就是一个范围查询:
```
mysql> EXPLAIN SELECT *
    ->         FROM user_info
    ->         WHERE id BETWEEN 2 AND 8 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
         type: range
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: NULL
         rows: 7
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```