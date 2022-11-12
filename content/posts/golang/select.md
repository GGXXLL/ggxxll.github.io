---
title: 'Golang Select'
tags:
    - Golang
date: '2022-11-12'
categories:
draft: false
---
## 数据结构
select 在 Go 语言的源代码中不存在对应的结构体，但是我们使用 runtime.scase 结构体表示 select 控制结构中的 case：

```go
type scase struct {
    c    *hchan         // chan
    elem unsafe.Pointer // data element
}
```

因为非默认的 case 中都与 Channel 的发送和接收有关，所以 runtime.scase 结构体中也包含一个 runtime.hchan 类型的字段存储 case 中使用的 Channel。

## 原理

select 结构的执行过程与实现原理，首先在编译期间，Go 语言会对 select 语句进行优化，它会根据 select 中 case 的不同选择不同的优化路径：

- 空的 select 语句会被转换成调用 `runtime.block` 直接挂起当前 Goroutine；
- 如果 select 语句中只包含一个 case，编译器会将其转换成 `if ch == nil { block }; n;` 表达式；
    - 首先判断操作的 Channel 是不是空的；
    - 然后执行 case 结构中的内容；
- 如果 select 语句中只包含两个 case 并且其中一个是 default，那么会使用 `runtime.selectnbrecv` 和 `runtime.selectnbsend`非阻塞地执行收发操作；
- 在默认情况下会通过 `runtime.selectgo` 获取执行 case 的索引，并通过多个 `if` 语句执行对应 case 中的代码；

在编译器已经对 select 语句进行优化之后，Go 语言会在运行时执行编译期间展开的 `runtime.selectgo `函数，该函数会按照以下的流程执行：

随机生成一个遍历的轮询顺序 pollOrder 并根据 Channel 地址生成锁定顺序 lockOrder；
- 根据 pollOrder 遍历所有的 case 查看是否有可以立刻处理的 Channel；
    - 如果存在，直接获取 case 对应的索引并返回；
    - 如果不存在，创建 `runtime.sudog` 结构体，将当前 Goroutine 加入到所有相关 Channel 的收发队列，并调用 `runtime.gopark` 挂起当前 - Goroutine 等待调度器的唤醒；
- 当调度器唤醒当前 Goroutine 时，会再次按照 lockOrder 遍历所有的 case，从中查找需要被处理的 `runtime.sudog` 对应的索引；

select 关键字是 Go 语言特有的控制结构，它的实现原理比较复杂，需要编译器和运行时函数的通力合作。