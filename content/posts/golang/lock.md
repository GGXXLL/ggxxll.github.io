---
title: 'Golang sync Lock'
tags:
    - Golang
date: '2022-11-12'
categories:
draft: false
---

## Mutex

Go 语言的 sync.Mutex 由两个字段 state 和 sema 组成。其中 state 表示当前互斥锁的状态，而 sema 是用于控制锁状态的信号量。
```go
type Mutex struct {
    state int32
    sema  uint32
}
```
上述两个加起来只占 8 字节空间的结构体表示了 Go 语言中的互斥锁。

### 状态

互斥锁的状态比较复杂，如下图所示，最低三位分别表示 mutexLocked、mutexWoken 和 mutexStarving，剩下的位置用来表示当前有多少个 Goroutine 在等待互斥锁的释放：

在默认情况下，互斥锁的所有状态位都是 0，int32 中的不同位分别表示了不同的状态：

- mutexLocked：表示互斥锁的锁定状态；
- mutexWoken：表示从正常模式被从唤醒；
- mutexStarving：当前的互斥锁进入饥饿状态；
- waitersCount：当前互斥锁上等待的 Goroutine 个数；

## RWMutex

```
type RWMutex struct {
    w           Mutex
    writerSem   uint32
    readerSem   uint32
    readerCount int32
    readerWait  int32
}
```

- w：复用互斥锁提供的能力
- writerSem：写等待读
- readerSem：读等待写
- readerCount：存储了当前正在执行的读操作数量
- readerWait：表示当写操作被阻塞时等待的读操作个数

## WaitGroup

```go
type WaitGroup struct {
    noCopy noCopy
    state1 [3]uint32
}
```
- noCopy：保证自身不会被开发者通过再赋值的方式拷贝；
- state1：存储着状态和信号量；

## Once
```go
type Once struct {
    done uint32
    m    Mutex
}
```

## Cond

```go
type Cond struct {
    noCopy  noCopy
    L       Locker
    notify  notifyList
    checker copyChecker
}

```
- noCopy：用于保证结构体不会在编译期间拷贝
- copyChecker：用于禁止运行期间发生的拷贝
- L：用于保护内部的 notify 字段，Locker 接口类型的变量
- notify：一个 Goroutine 的链表，它是实现同步机制的核心结构

```go
type notifyList struct {
    wait uint32
    notify uint32

    lock mutex
    head *sudog
    tail *sudog
}
```

在 `sync.notifyList` 结构体中，head 和 tail 分别指向的链表的头和尾，wait 和 notify 分别表示当前正在等待的和已经通知到的 Goroutine 的索引。
