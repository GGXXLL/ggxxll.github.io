---
title: 'Golang Channel'
tags:
date: '2022-11-11'
categories:
draft: false
---

## 数据结构

Go 语言的 Channel 在运行时使用 runtime.hchan 结构体表示。我们在 Go 语言中创建新的 Channel 时，实际上创建的都是如下所示的结构：

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

    lock mutex
}

type waitq struct {
    first *sudog
    last  *sudog
}
```
- qcount：channel 中的元素个数
- dataqsize：buffer 的大小，也就是make(chan T, N)中的N。
- elemsize：channel中单个元素的大小
- elemtype：channel中元素的类型
- buf：带缓冲的channel(buffered channel)中用来保存数据的循环队列
- closed：channel是否关闭，0->打开，1->关闭
- sendx和recvx：循环队列接受和发送数据的下标
- recvq和sendq：保存阻塞中的 Goroutine 的等待队列，recvq 保存读取数据而阻塞 的Goroutine,sendq 保存写入数据而阻塞的 Goroutine
- lock：在每个读写操作对 channel 的锁

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