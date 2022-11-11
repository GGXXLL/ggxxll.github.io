---
title: 'Redis Zset'
tags:
    - Redis
date: '2022-11-11'
categories:
    - 数据库
draft: true
---

### 编码选择

有序集合对象的编码可以是 ziplist 或者 skiplist。

使用 ziplist：
- 元素数量小于128个
- 所有member的长度都小于64字节

使用 skiplist：
- 不能满足上面两个条件的使用 skiplist 编码。以上两个条件也可以通过Redis配置文件 `zset-max-ziplist-entries` 选项和 `zset-max-ziplist-value` 进行修改
- 对于一个 `REDIS_ENCODING_ZIPLIST` 编码的 Zset，只要满足以上任一条件，则会被转换为 `REDIS_ENCODING_SKIPLIST` 编码

### ziplist

#### 介绍
ziplist 编码的 Zset 使用紧挨在一起的压缩列表节点来保存，第一个节点保存 member，第二个保存 score。ziplist 内的集合元素按 score 从小到大排序，其实质是一个双向链表。虽然元素是按 score 有序排序的， 但对 ziplist 的节点指针只能线性地移动，所以在 `REDIS_ENCODING_ZIPLIST` 编码的 Zset 中， 查找某个给定元素的复杂度为 O(N)。

#### 结构

各个部分在内存上是前后相邻的并连续的，每一部分作用如下：

- zlbytes： 存储一个无符号整数，固定四个字节长度（32bit），用于存储压缩列表所占用的字节（也包括 zlbytes 本身占用的4个字节），当重新分配内存的时候使用，不需要遍历整个列表来计算内存大小。
- zltail： 存储一个无符号整数，固定四个字节长度（32bit），表示 ziplist 表中最后一项（entry）在 ziplist 中的偏移字节数。 zltail 的存在，使得我们可以很方便地找到最后一项（不用遍历整个ziplist），从而可以在ziplist尾端快速地执行 push 或 pop 操作。
- zllen： 压缩列表包含的节点个数，固定两个字节长度（16bit）， 表示ziplist中数据项（entry）的个数。由于zllen字段只有16bit，所以可以表达的最大值为2^16-1。

注意点：如果ziplist中数据项个数超过了16bit能表达的最大值，ziplist仍然可以表示。ziplist是如何做到的？

如果 zllen 小于等于2^16-2（也就是不等于2^16-1），那么 zllen 就表示ziplist中数据项的个数；否则，也就是 zllen 等于16bit全为1的情况，那么 zllen 就不表示数据项个数了，这时候要想知道ziplist中数据项总数，那么必须对ziplist从头到尾遍历各个数据项，才能计数出来。

entry，表示真正存放数据的数据项，长度不定。一个数据项（entry）也有它自己的内部结构。

zlend， ziplist最后1个字节，值固定等于255，其是一个结束标记。


### skiplist

#### 介绍

skiplist 编码的 Zset 底层为一个被称为 zset 的结构体，这个结构体中包含一个字典和一个跳跃表。跳跃表按 score 从小到大保存所有集合元素，查找时间复杂度为平均 O(logN)，最坏 O(N) 。字典则保存着从 member 到 score 的映射，这样就可以用 O(1) 的复杂度来查找 member 对应的 score 值。虽然同时使用两种结构，但它们会通过指针来共享相同元素的 member 和 score，因此不会浪费额外的内存。

#### 详解

跳表(skip List)是一种随机化的数据结构，基于并联的链表，实现简单，插入、删除、查找的复杂度均为O(logN)。简单说来跳表也是链表的一种，只不过它在链表的基础上增加了跳跃功能，正是这个跳跃的功能，使得在查找元素时，跳表能够提供O(logN)的时间复杂度。

### Q&A 

1. Redis为什么用skiplist而不用平衡树？

这里从内存占用、对范围查找的支持和实现难易程度这三方面总结的原因。

- 也不是非常耗费内存，实际上取决于生成层数函数里的概率 p，取决得当的话其实和平衡树差不多。
- 因为有序集合经常会进行 ZRANGE 或 ZREVRANGE 这样的范围查找操作，跳表里面的双向链表可以十分方便地进行这类操作。
- 实现简单，ZRANK 操作还能达到 o(logn) 的时间复杂度。