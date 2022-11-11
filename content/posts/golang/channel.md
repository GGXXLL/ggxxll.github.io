---
title: 'Golang Channel'
tags:
date: '2022-11-11'
categories:
draft: false
---

## channel 结构

### hchan
```go
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    elemtype *_type // element type
    sendx    uint   // send index
    recvx    uint   // receive index
    recvq    waitq  // list of recv waiters
    sendq    waitq  // list of send waiters

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}

type waitq struct {
    first *sudog
    last  *sudog
}
```
- dataqsize是buffer的大小，也就是make(chan T, N)中的N。
- elemsize是channel中单个元素的大小
- buf是带缓冲的channel(buffered channel)中用来保存数据的循环队列
- closed表示channel是否关闭。0->打开，1->关闭
- sendx和recvx表示循环队列接受和发送数据的下标
- recvq和sendq是保存阻塞的Goroutine的等待队列，recvq保存读取数据而阻塞的Goroutine,sendq保存写入数据而阻塞的Goroutine
- lock是在每个读写操作对channel的锁

### waitq


## select 操作

select 的使用例子:
```go
func TestSelect(t *testing.T) {
    cInt := make(chan int, 5)
    cString := make(chan string, 5)
    select {
    case msg := <-cInt:
        fmt.Println("receive msg", msg)
    case msg := <-cString:
        fmt.Println("receive msg", msg)
    default:
        fmt.Println("no msg received")
    }
}
```

1. 操作是互斥的，使所有需要获得参与 select 的 channel 的锁，先按 hchane 的地址排序，然后顺序获得锁，所以并不是同时获得所有参与 select 的 channel 的锁。

在 scases 数组上的每一个scase结构体，包含着当前的操作类型，和正在操作的channel。
```go
type scase struct {
    c           *hchan         // chan
    elem        unsafe.Pointer // data element
    kind        uint16
    pc          uintptr // race pc (for race detector / msan)
    releasetime int64
}
```
kind 是当前操作类型的case,可以是CaseRecv, CaseSend 或者 CaseDefault.
2. select的选取顺序是，以伪随机数的方法，将参与 select 的 channel 顺序打乱。所以 select 的顺序和程序声明的顺序是不一样的。

- 只要有不阻塞的 channel，select就会返回，不会等待每一个参与 select 的 channel 都才准备好。
- 如果当前没有 channle 回应，并且没有 default 语句，当前的g就会对应的等待队列悬停