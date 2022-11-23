---
title: 'MongoDB 索引'
tags:
    - MongoDB
date: '2022-11-23'
categories:
draft: false
---

## 索引操作

```mongo
// 创建索引
db.userinfos.createIndex({age:-1})

// 查看userinfos中的所有索引
db.userinfos.getIndexes()

// 删除特定一个索引
db.userinfos.dropIndex({name:1,age:-1})

// 删除所有的索引(主键索引_id不会被删除)
db.userinfos.dropIndexes()
```

## 分类

### 单键索引

单键索引(Single Field Indexes)顾名思义就是单个字段作为索引列，mongoDB 的所有 collection 默认都有一个单键索引 _id，我们也可以对一些经常作为过滤条件的字段设置索引，如给 age 字段添加一个索引，语法十分简单：

```
// 为 age 添加升序索引
db.userinfos.createIndex({age:1})

// 嵌套字段添加索引
db.userinfos.createIndex({"ename.firstname":1})
```

### 复合索引

复合索引(Compound Indexes)指一个索引包含多个字段，用法和单键索引基本一致。使用复合索引时要注意字段的顺序，如下添加一个 name 和 age 的复合索引，name 正序，age 倒序，document 首先按照 name 正序排序，然后 name 相同的 document 按 age 进行倒序排序。mongoDB 中一个复合索引最多可以包含 32 个字段。

```
//添加复合索引，name正序，age倒序
db.userinfos.createIndex({"name":1,"age":-1}) 

//过滤条件为name，或包含name的查询会使用索引(索引的第一个字段)
db.userinfos.find({name:'张三'}).explain()
db.userinfos.find({name:"张三",level:10}).explain()
db.userinfos.find({name:"张三",age:23}).explain()

//查询条件为age时，不会使用上边创建的索引,而是使用的全表扫描
db.userinfos.find({age:23}).explain()
```


### 多键索引

多键索引(mutiKey Indexes)是建在数组上的索引，在 mongoDB 的 document 中，有些字段的值为数组，多键索引就是为了提高查询这些数组的效率。看一个栗子：准备测试数据，classes 集合中添加两个班级，每个班级都有一个 students 数组，如下：
```
db.classes.insertMany([
     {
         "classname":"class1",
         "students":[{name:'jack',age:20},
                    {name:'tom',age:22},
                    {name:'lilei',age:25}]
      },
      {
         "classname":"class2",
         "students":[{name:'lucy',age:20},
                    {name:'jim',age:23},
                    {name:'jarry',age:26}]
      }]
  )

db.classes.createIndex({'students.age':1})
```


### 哈希索引

哈希索引(hashed Indexes)就是将 field 的值进行 hash 计算后作为索引，其强大之处在于实现 O(1) 查找，当然用哈希索引最主要的功能也就是实现定值查找，对于经常需要排序或查询范围查询的集合不要使用哈希索引。

## 属性

### 唯一索引
唯一索引(unique indexes)用于为collection添加唯一约束，即强制要求collection中的索引字段没有重复值。添加唯一索引的语法：

```
//在userinfos的name字段添加唯一索引
db.userinfos.createIndex({name:1},{unique:true})
```

### 局部索引

局部索引(Partial Indexes)顾名思义，只对collection的一部分添加索引。创建索引的时候，根据过滤条件判断是否对document添加索引，对于没有添加索引的文档查找时采用的全表扫描，对添加了索引的文档查找时使用索引。使用方法也比较简单：

```
//userinfos集合中age>25的部分添加age字段索引
db.userinfos.createIndex(
    {age:1},
    { partialFilterExpression: { age:{$gt: 25 }}}
)

//查询age<25的document时，因为age<25的部分没有索引，会全表扫描查找(stage:COLLSCAN)
db.userinfos.find({age:23})

//查询age>25的document时，因为age>25的部分创建了索引，会使用索引进行查找(stage:IXSCAN)
db.userinfos.find({age:26})
```

### 稀疏索引
稀疏索引(sparse indexes)在有索引字段的 document 上添加索引，如在 address 字段上添加稀疏索引时，只有 document 有 address 字段时才会添加索引。而普通索引则是为所有的 document 添加索引，使用普通索引时如果 document没有索引字段的话，设置索引字段的值为 null。

稀疏索引的创建方式如下，当document包含address字段时才会创建索引：
```
db.userinfos.createIndex({address:1},{sparse:true})
```

### TTL索引
TTL索引(TTL indexes)是一种特殊的单键索引，用于设置 document 的过期时间，mongoDB 会在document 过期后将其删除，TTL 非常容易实现类似缓存过期策略的功能。我们看一个使用 TTL 索引的栗子：

```
//添加测试数据
db.logs.insertMany([
       {_id:1,createtime:new Date(),msg:"log1"},
       {_id:2,createtime:new Date(),msg:"log2"},
       {_id:3,createtime:new Date(),msg:"log3"},
       {_id:4,createtime:new Date(),msg:"log4"}
       ])
       //在createtime字段添加TTL索引，过期时间是120s
       db.logs.createIndex({createtime:1}, { expireAfterSeconds: 120 })


//logs中的document在创建后的120s后过期，会被mongoDB自动删除
```

注意：
- TTL索引只能设置在 date 类型字段(或者包含 date 类型的数组)上，过期时间为字段值 + exprireAfterSeconds；
- document过期时不一定就会被立即删除，因为 mongoDB 执行删除任务的时间间隔是60s；
- capped Collection 不能设置TTL索引，因为 mongoDB 不能主动删除 capped Collection 中的 document。