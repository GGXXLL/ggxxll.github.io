---
title: Postgresql查询表的大小
tags: 
  - Postgresql
date: 2022-08-24
categories:
  - 数据库
---

原贴: https://www.cnblogs.com/winkey4986/p/6433704.html

--数据库中单个表的大小（不包含索引）
```sql
select pg_size_pretty(pg_relation_size('表名'));
```

--查出所有表（包含索引）并排序
```sql
SELECT table_schema || '.' || table_name AS table_full_name, pg_size_pretty(pg_total_relation_size('"' || table_schema || '"."' || table_name || '"')) AS size
FROM information_schema.tables
ORDER BY
pg_total_relation_size('"' || table_schema || '"."' || table_name || '"') DESC limit 20
```

--查出表大小按大小排序并分离data与index
```sql
SELECT
table_name,
pg_size_pretty(table_size) AS table_size,
pg_size_pretty(indexes_size) AS indexes_size,
pg_size_pretty(total_size) AS total_size
FROM (
SELECT
table_name,
pg_table_size(table_name) AS table_size,
pg_indexes_size(table_name) AS indexes_size,
pg_total_relation_size(table_name) AS total_size
FROM (
SELECT ('"' || table_schema || '"."' || table_name || '"') AS table_name
FROM information_schema.tables
) AS all_tables
ORDER BY total_size DESC
) AS pretty_sizes
```