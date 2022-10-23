---
title:  mysql 的 order by 和 limit 语句导致慢 sql
tags:
- Mysql
  date: 2022-09-19
  categories:
- 数据库
---

### 问题

`order by` 和 `limit` 造成优化器选择索引错误

### 测试表
```sql
CREATE DATABASE `app`;
USE app;
```

```sql
CREATE TABLE `users` (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `age` int DEFAULT NULL,
  `created_at` timestamp NOT NULL,
  `deleted_at` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `users_age_IDX` (`age`) USING BTREE,
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 填充测试数据
```sql
DROP PROCEDURE IF EXISTS addData;

DELIMITER $$
$$
CREATE PROCEDURE addData(IN n int)
BEGIN
	DECLARE i int default 1;
	while (i < n) DO
		INSERT INTO users(name, age, created_at, deleted_at) VALUES(CONCAT('name',i), i, now(), null);
		set i=i  |  1;
End while;
END$$
DELIMITER ;
```

```sql
CALL addData(400)
```

### 查询 sql

1. 以下查询中使用的索引正确使用了 `users_age_IDX`。
```sql
explain SELECT * from users u where age > 1000  limit 1;
explain SELECT * from users u where age > 1000  order by id desc limit 2;
```
| id  | select_type | table | partitions | type  | possible_keys | key     | key_len | ref | rows | filtered | Extra                            |  
|---|-------------|-------|------------|-------|------------------------------------|---------|---------|-----|------|----------|----------------------------------|  
| 1 | SIMPLE      |  u      |              |  range  |  users_age_IDX,users_deleted_at_IDX  |  users_age_IDX  |  5        |       |  1988  |     100.0  |  Using index condition; Using where  |  

2. 这个查询中使用的索引是主键, 数据量
```sql
explain SELECT * from users u where age > 100 order by id desc limit 1;
```
| id  | select_type | table | partitions | type  | possible_keys | key     | key_len | ref | rows | filtered | Extra                            |  
|---|-------------|-------|------------|-------|------------------------------------|---------|---------|-----|------|----------|----------------------------------|  
| 1   | SIMPLE   | u  | | index | users_age_IDX,users_deleted_at_IDX | PRIMARY | 4   |   | 65   | 15.33    | Using where; Backward index scan |

### 原因

MySQL 优化器认为在 `limit` 值较小的情况下，走主键索引能够更快的找到那一条数据，并且如果走联合索引需要扫描索引后进行排序，而主键索引天生有序，所以优化器综合考虑，走了主键索引。


