+++
date = '2025-06-23T09:45:24+08:00'
draft = false
title = 'MySQL 性能调优利器：深入解读 EXPLAIN 执行计划'
tags = ["数据库", "MySQL", "后端基础"]
+++
---
## 1. 为什么要引入explain
**痛点**：你写了个 SQL 查询，逻辑正确，但速度慢得像蜗牛？数据库服务器负载飙升？加索引好像没用？你不知道 MySQL 内部到底怎么执行它的。
**解决办法**：mysql提供explain机制，为你剖析你的sql执行的底层逻辑，指明sql查询的类型，扫描的行数以及使用的索引等。
**核心价值**
- 理解查询如何被执行
- 验证索引是否被正确使用
- 指导sql优化和索引设计
- 识别性能瓶颈（全表扫描？临时表？文件排序？）
总的来说，explain可以帮助我们分析sql的执行计划，进而辅助我们优化sql，提高应用执行效率以及减少mysql性能开销。

---
## 2. 如何使用explain
```sql
EXPLAIN [FORMAT = {TRADITIONAL | JSON | TREE}] your_sql_statement;
-- 最常见、最易读的是 TRADITIONAL (表格形式)，也是默认格式
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';
```
如上，`explain`关键字后面直接跟sql即可，可以以表格的形式输出执行计划。
### 输出列解读
- `id`: 查询中 SELECT 子句的执行顺序号。相同 id 表示同一执行层级（如 UNION），id 越大越先执行，id 相同从上往下执行。
- `select_type`: 查询类型 (SIMPLE, PRIMARY, SUBQUERY, DERIVED, UNION, UNION RESULT 等)。揭示查询的复杂结构。
- `table`: 当前行正在访问哪个表（或派生表、别名）。
- partitions: 使用的分区（如果是个分区表）
- type: **重要！！** 表示 `MySQL` 如何查找表中的行。从最优到最差常见的有：
system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL
`const` (主键/唯一索引等值), `eq_ref` (关联查询主键/唯一索引), `ref` (非唯一索引等值), `range` (索引范围扫描), `index` (全索引扫描), `ALL` (全表扫描)。强调 `ALL` 和 `index` (如果索引不覆盖) 通常是性能杀手。
- `possible_keys`: 查询可能会使用哪些索引。理论上的候选。
- `key`: 查询实际决定使用的索引。如果为 NULL，则表示未使用索引。
- `key_len`: 使用的索引的长度（字节数）。可推断实际使用了索引的哪些部分（前缀索引）。
- `ref`: 显示索引的哪一列或常量被用来与 key 列指定的索引进行比较。
- `rows` **(关键指标！)**: MySQL 估计为了找到所需的行而必须检查的行数。一个非常重要的性能指标，理想情况下应尽可能小。注意这是估计值！
- `filtered`: 表示存储引擎返回的数据在 server 层过滤后，剩余行数的百分比估计。rows * filtered / 100 可以估算出将与下一表连接的行数。
- `Extra` **(包含重要信息！)**: 包含 MySQL 解决查询的额外信息。常见且需要关注的值：
    - `Using index`: 表示使用了覆盖索引 (查询的列都在索引中)，性能极佳。
    - `Using where`: 表示 Server 层对存储引擎返回的行进行了过滤 (WHERE 条件中的列可能不在索引中或索引没完全覆盖)。
    - `Using temporary`: 表示 MySQL 需要创建临时表来处理查询（常见于 GROUP BY, ORDER BY 非索引列）。警惕！
    - `Using filesort`: 表示 MySQL 需要进行额外的排序操作（常见于 ORDER BY 非索引列）。警惕！意味着无法利用索引排序。
    - `Using join buffer (Block Nested Loop)`: 表示使用了连接缓冲区。关联字段无索引时常见。
    - `Select tables optimized away`: 优化得很好（例如通过 MIN()/MAX() 使用索引）。

---
## 3. 实例分析
首先新建两张表，建表语句如下：
```sql
CREATE TABLE `user_test` (
  `id` int NOT NULL,
  `name` varchar(100) DEFAULT NULL,
  `age` int DEFAULT NULL,
  `email` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name_age` (`name`,`age`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

CREATE TABLE `user_extra` (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_id` int DEFAULT NULL,
  `name_md5` char(32) DEFAULT NULL,
  `email_md5` char(32) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=500001 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```
使用脚本为每个表插入五十万行数据。
##### 1. 基础查询 - 全表扫描
```sql
mysql> EXPLAIN SELECT * FROM user_test WHERE age = 30;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | user_test | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 498481 |    10.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
如上，`user_test`表对name以及age字段加了联合索引，单用age查询会违背最左匹配原则导致走全表扫描，
**分析**：
- `select_type = SIMPLE`，代表是简单查询
- `type = ALL`，走了全表扫描
- `rows = 498481`，预估扫描行数498481行
**优化方案**：
可以对age字段添加索引
```sql
CREATE INDEX idx_age ON user_test(age);
```
再次执行：
```sql
mysql> EXPLAIN SELECT * FROM user_test WHERE age = 30;
+----+-------------+-----------+------------+------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | user_test | NULL       | ref  | idx_age       | idx_age | 5       | const | 4982 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
可以看到，type已经改为了效率比较高的ref，并且可能的行数也只有4982行，查询效率大大提升。
##### 2. 主键查询 - 最优情况
因为mysql的innoDB引擎的主键索引为聚簇索引，所以主键等值查询的效率非常高。
```sql
mysql> EXPLAIN SELECT * FROM user_test WHERE id = 123456;
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | user_test | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
如上，扫描的预期行数只有一行，type为const，使用的索引为主键索引，在常数级时间内即可完成查询。效率最高。
##### 3. join查询条件未带索引
```sql
mysql> EXPLAIN
    -> SELECT u.name, ue.name_md5
    -> FROM user_test u
    -> JOIN user_extra ue ON u.id = ue.user_id
    -> WHERE u.email = 'user123@example.com';
+----+-------------+-------+------------+------+---------------+-------------+---------+------------------+--------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key         | key_len | ref              | rows   | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+-------------+---------+------------------+--------+----------+-------------+
|  1 | SIMPLE      | u     | NULL       | ALL  | PRIMARY       | NULL        | NULL    | NULL             | 498481 |    10.00 | Using where |
|  1 | SIMPLE      | ue    | NULL       | ref  | idx_user_id   | idx_user_id | 5       | falcon_loan.u.id |      1 |   100.00 | NULL        |
+----+-------------+-------+------------+------+---------------+-------------+---------+------------------+--------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```
如上是一个连表查询，由于`user_test`表没有对`email`字段的索引，所以会走全表扫描，效率极慢。`user_extra`由于有`user_id`的索引，所以联表时可以走索引，所以主要优化目标在`user_test`表上，需要增加`email`索引。
##### 4. join查询条件带索引
```sql
mysql> EXPLAIN SELECT u.name, ue.name_md5 FROM user_test u JOIN user_extra ue ON u.id = ue.user_id WHERE u.age = 30;
+----+-------------+-------+------------+------+-----------------+-------------+---------+------------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys   | key         | key_len | ref              | rows | filtered | Extra |
+----+-------------+-------+------------+------+-----------------+-------------+---------+------------------+------+----------+-------+
|  1 | SIMPLE      | u     | NULL       | ref  | PRIMARY,idx_age | idx_age     | 5       | const            | 4982 |   100.00 | NULL  |
|  1 | SIMPLE      | ue    | NULL       | ref  | idx_user_id     | idx_user_id | 5       | falcon_loan.u.id |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+-----------------+-------------+---------+------------------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
```
可以看到，当使用索引作为条件查询时，需要扫描的行数降低到4982行，走了`idx_age`索引，查询效率提升
##### 5. 索引失效场景
我们将user_test表的nage_age的联合索引删掉，之后执行or条件查询。
```sql
mysql> drop index idx_name_age on user_test;
mysql> explain select * from user_test where age = 30 or name = 'Tom Jack';
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | user_test | NULL       | ALL  | idx_age       | NULL | NULL    | NULL | 498481 |    10.89 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.01 sec)
```
如上，虽然我们age字段有索引，但是name字段的索引我们已经删除，or条件会将左右两边的条件都执行，所以`possible_keys`显示`idx_age`，但是实际查询的时候还需要判断`name`的条件，所以实际并没有使用索引。优化方式为对`name`和`age`加联合索引。
##### 6. 派生表/临时表
使用临时表的查询效率是比较低的，因为临时表没有索引，所以一般都是全表扫描
```sql
mysql> EXPLAIN
    -> SELECT u.id, sub.max_md5
    -> FROM user_test u
    -> JOIN (
    ->     SELECT user_id, MAX(name_md5) AS max_md5
    ->     FROM user_extra
    ->     GROUP BY user_id
    -> ) sub ON u.id = sub.user_id;
+----+-------------+------------+------------+--------+---------------+-------------+---------+-------------+--------+----------+-------------+
| id | select_type | table      | partitions | type   | possible_keys | key         | key_len | ref         | rows   | filtered | Extra       |
+----+-------------+------------+------------+--------+---------------+-------------+---------+-------------+--------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL        | NULL    | NULL        | 497284 |   100.00 | Using where |
|  1 | PRIMARY     | u          | NULL       | eq_ref | PRIMARY       | PRIMARY     | 4       | sub.user_id |      1 |   100.00 | Using index |
|  2 | DERIVED     | user_extra | NULL       | index  | idx_user_id   | idx_user_id | 5       | NULL        | 497284 |   100.00 | NULL        |
+----+-------------+------------+------------+--------+---------------+-------------+---------+-------------+--------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)
```
特别解释下，`select_type = DERIVED` 代表使用了派生表，第一行的`table = <derived2>`代表使用了id=2的派生表，并且走了全表扫描。优化的时候可以将派生表改为join形式，避免派生表。