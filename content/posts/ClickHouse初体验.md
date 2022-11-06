---
title:  Clickhouse初体验
tags: 
  - Clickhouse
date: 2022-08-24
categories:
  - 数据库
---

#### 概述

https://zhuanlan.zhihu.com/p/98135840

#### 链接

- [文档](https://clickhouse.tech/docs/zh/sql-reference/)
- [在线体验](https://clickhouse.tech/docs/zh/getting-started/playground/)

#### 数据库引擎

1. `Lazy` 

   - 会将最近一次查询的表保存在内存中, 过期时间为`expiration_time_in_seconds`. 
   - 仅适用于*Log表引擎. 
   - 如果表的数量较多, 数据量小, 且访问间隔长, 建议使用Lazy引擎数据库.

2. `Atomic` 

   - 不指定引擎时, 默认使用Atomic. 
   - 支持无阻塞删除和重命名表.
   - 默认会生成UUID, 不建议指定UUID
   - 不改变uuid和迁移表数据的话, 执行`rename table`可以立即生效
   - `DROP/DETACH TABLE`不会真正的删除表, 只是标记被删除了.  如果同步模式设置为`SYNC`, 在等待已有的查询/插入操作结束之后, 表会被立即删除.
   - 使用表引擎`ReplicatedMergeTree`, 不建议指定`path`和`replica name`
     - `ReplicatedReplacingMergeTree('/clickhouse/tables/{shard}/{database}/{table}', '{replica}')`

3. `Mysql`  

   - 远程**映射**Mysql的库表.  在clickhouse执行的查询等, 会被引擎转换, 发送到Mysql服务中. 
   - 不能执行 `RENAME`, `CREATE TABLE`, `ALTER`
   - 简单 `WHERE` 条款如 `=, !=, >, >=, <, <=` 当前在MySQL服务器上执行。其余的条件和 `LIMIT` 只有在对MySQL的查询完成后，才会在ClickHouse中执行采样约束。
   - 删除`DROP`和卸载`DETACH` 表，不会删除mysql中的表，且允许通过`ATTACH`再装载回来
   - 不支持的Mysql的数据类型会转换成`String`, 所有类型都支持`Nullable`

4. `MaterializeMySQL`

   - 使用MySQL中存在的所有表以及这些表中的所有数据创建ClickHouse数据库（即库级别的数据同步）。ClickHouse服务器用作MySQL副本。它读取binlog并执行DDL和DML查询。默认表引擎设置为`ReplacingMergeTree`

   - 使用表引擎 [ReplacingMergeTree](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replacingmergetree/) 时, 会添加虚拟列 `_sign` 和 `_version`

     - `_version` — Transaction counter. Type [UInt64]

     - `_sign` — Deletion mark. Type [Int8]. Possible values:
       - `1` — Row is not deleted,
       - `-1` — Row is deleted.
   
   - MySQL DDL查询将转换为相应的ClickHouse DDL查询（ALTER，CREATE，DROP，RENAME）。如果ClickHouse无法解析某些DDL查询，则该查询将被忽略。
   - 不支持 INSERT`, `DELETE 和 UPDATE操作. 复制时都视为INSERT操作, 会在数据上使用`_sign`标记.
     - INSERT, `1`
     - DELETE, `-1`
     - UPDATE, `1`或`-1`
   - 执行SELECT查询时
     - 不指定`_version`, 默认使用`FINAL`, 即`MAX(_version)`
     - 不指定`_sign`, 默认会限制查询条件`WHERE _sign=1`
   - 注意事项:
     - `_sign=-1`的数据是逻辑删除
     - 不支持update/delete
     - 复制过程很容易崩溃
     - 禁止手动操作数据库和表
     - 通过设置`optimize_on_insert`, 启用或禁用在插入之前进行数据转换，就像合并是在此块上完成的（根据表引擎）一样。
     - 不支持的Mysql数据类型会异常`Unhandled data type`并停止复制， 都支持`Nullable`
   
   5. PostgreSQL
      1. 远程链接PostgreSQL. 支持查询, 插入等操作. 支持修改表结构 (注意使用缓存时的情况: `DETACH` 和`ATTCH`可刷新缓存).
      2. 数据类型中仅`INTEGER`支持`Nullable`

#### 表引擎

1. [文档](https://clickhouse.tech/docs/zh/engines/table-engines/)
2. MergeTree
3. Kakfa，Mysql

##### 创建数据库

- ```sql
  CREATE DATABASE [IF NOT EXISTS] db_name [ON CLUSTER cluster] [ENGINE = engine(...)]
  ```

- ```sql
  CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
  (
      name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [COMMENT expr1] [TTL expr1],
      name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [COMMENT expr1] [TTL expr2],
      ...
      INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
      INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
  ) ENGINE = MergeTree()
  -- 指定数据排序使用字段，默认等同主键
  ORDER BY (expr1[, expr2,...])
  -- 指定数据分区使用的字段
  [PARTITION BY (expr1[, expr2,...])]
  -- 指定主键
  [PRIMARY KEY (expr1[, expr2,...])]
  -- 指定采样数据使用的字段，启用数据采样时，不会对所有数据执行查询，而只对特定部分数据（样本）执行查询
  [SAMPLE BY (expr1[, expr2,...])]
  [TTL expr 
      [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx' [, ...] ]
      [WHERE conditions] 
      [GROUP BY key_expr [SET v1 = aggr_func(v1) [, v2 = aggr_func(v2) ...]] ] 
  ] 
  [SETTINGS name=value, ...]
  ```

#### 字典

字典是一个映射 (键 -> 属性）, 是方便各种类型的参考清单。

ClickHouse支持一些特殊函数配合字典在查询中使用。 将字典与函数结合使用比将 `JOIN` 操作与引用表结合使用更简单、更有效。

- 内置字典：处理地理数据库
- 外部字典：自定义字典

#### 特殊查询

```sql
-- 计算pv值>20的记录条数 
select sum(pv>20) from test_all;
select count(*) from test_all where pv>20;

-- 计算pv>20的记录条数占比：0.5
select avg(pv>20) from test_all;
-- 计算pv在1,2，20这几个值中的记录条数的占比
select avg(pv in (1,2,2,20)) from test_all;

-- Date系列类型， 插入时可以混合时间戳和字符串插入
INSERT INTO dt Values (1546300800, 1), ('2019-01-01', 2);

```

#### 数据类型

- UUID
- FixedString
- Enum
- LowCardinality(T),  把其它数据类型转变为字典编码类型。如果该字段的值去重个数小于1W，性能优于普通类型。反之，性能下降。 通常， 一个字符串类型的Enum类型， 建议设置为LowCardinality， 更灵活和高效
- Array(T...)
- AggregateFunction， 适用于表引擎 AggregatingMergeTree
- Nested 嵌套字段， 不支持表引擎 MergeTree
- Tuple
- Nullable
- Domains： IPV4， IPV6
- Geo： **experimental feature**
  - Point ： 坐标，Tuple（x float64, y float64）
  - Ring： Array(Point)
  - Polygon： Array(Ring)
  - MultiPolygon：Array(Polygon)
- Map(key,value) **experimental feature**
- SimpleAggregateFunction 性能优于 AggregateFunction， 适用于表引擎 AggregatingMergeTree



#### 注意事项

1. 使用 `Nullable` 几乎总是对性能产生负面影响，在设计数据库时请记住这一点, 最好设置default值。
2. 删除表的时候， 会先在元数据标记被删除，exists、select、insert等会立即生效（表不存在）， 但是`create table`创建表等操作会抛异常（表存在）。 稍等几分钟，再新建表即可。
3. 删除表数据，应当操作本地表，操作分布式Distributed表会报错。
4. `ReplacingMergeTree`表引擎， 完全相同的数据会直接去重， 数据是在后台不确定的时间去合并去重的。不建议使用`OPTIMIZE`发起合并， 使用 `FINAL` 关键字进行查询，可以得到主键相同的、最新被插入的数据。
5. 那些有相同分区表达式值`PARTITION BY`的数据片段才会合并。这意味着 **你不应该用太精细的分区方案**（超过一千个分区）。否则，会因为文件系统中的文件数量过多和需要打开的文件描述符过多，导致 `SELECT` 查询效率不佳。可以根据`group by`语句来分析如何分区。绝对不能使用带有主键性质的字段做分区（比如唯一id）， 最好选择一个重复率高的字段（比如 日期， 渠道等）。
6. Alter语句DROP, MODIFY 时， 该字段不能是主键，会报错

#### 使用

```sql
create table test on cluster cluster1
(
    date Date,
    id   UInt32 default 0 comment '123',
    name String default '',
    pv   UInt32 default 0
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/{shard}/{database}/{table}', '{replica}')
PARTITION BY date
ORDER BY (date,id);
```

```sql
create table test_all on cluster cluster1
(
    date Date,
    id   UInt32 default 0 comment '123',
    name String default '',
    pv   UInt32 default 0
)
engine = Distributed('cluster1', 'wangzhuan', 'test', sipHash64(date));
```

```sql
-- 结果是一条数据
insert into test_all (date, id, name, pv) VALUES 
('2021-05-20',1,'a',10),
('2021-05-20',1,'a',10);
```

```sql
-- 结果是两条数据
insert into test_all (date, id, name, pv)
VALUES ('2021-05-20', 1, 'a', 10)
     , ('2021-05-20', 1, 'a', 20);
     
-- 结果是两条数据 30被20覆盖了， 
insert into test_all (date, id, name, pv)
VALUES ('2021-05-20', 1, 'a', 30)
     , ('2021-05-20', 1, 'a', 20);
     
     
-- 当然， 经过后台的数据合并之后， 应该是只剩1条数据 pv=30的
     
```

```sql
select * from test_all;

select * from test_all final;

select sum(pv) from test_all;

select sum(pv) from test_all final;
```

```sql
alter table test on cluster cluster1 delete where 1;

alter table test on cluster cluster1 delete where date='2021-05-20';
```

```sql
-- 仅当分区被使用到（PARTITION BY的字段的值超过一个），system.parts表里才会有数据
SELECT
    partition,
    name,
    active
FROM system.parts
WHERE table = 'test';

SELECT
    table,count(*)
FROM system.parts
group by table order by count(*) desc;
```

```sql
drop table test on cluster cluster1;
drop table test_all on cluster cluster1;
```

