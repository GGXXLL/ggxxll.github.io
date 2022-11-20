---
title: 'Golang Select'
tags:
    - Golang
date: '2022-11-12'
categories:
draft: false
---
## 数据结构
select 在 Go 语言的源代码中不存在对应的结构体，但是我们使用 `runtime.scase` 结构体表示 select 控制结构中的 case：

```go
type scase struct {
    c    *hchan         // chan
    elem unsafe.Pointer // data element
}

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
```

因为非默认的 case 中都与 Channel 的发送和接收有关，所以 `runtime.scase` 结构体中也包含一个 `runtime.hchan` 类型的字段存储 case 中使用的 Channel。


## selectgo

### 初始化

`runtime.selectgo` 函数首先会进行执行必要的初始化操作并决定处理 case 的两个顺序 — 轮询顺序 `pollOrder` 和加锁顺序 `lockOrder`：

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    // scase 的数量最大为 65536 
    cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
    order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))
    
    ncases := nsends + nrecvs
    scases := cas1[:ncases:ncases]
    pollorder := order1[:ncases:ncases]
    lockorder := order1[ncases:][:ncases:ncases]

    norder := 0
    for i := range scases {
        cas := &scases[i]
    }

    for i := 1; i < ncases; i++ {
        j := fastrandn(uint32(i + 1))
        pollorder[norder] = pollorder[j]
        pollorder[j] = uint16(i)
        norder++
    }
    pollorder = pollorder[:norder]
    lockorder = lockorder[:norder]

    // 根据 Channel 的地址排序确定加锁顺序
    ...
    sellock(scases, lockorder)
    ...
}
```

轮询顺序 `pollOrder` 和加锁顺序 `lockOrder` 分别是通过以下的方式确认的：
- 轮询顺序：通过 `runtime.fastrandn` 函数引入随机性；
- 加锁顺序：按照 Channel 的地址排序后确定加锁顺序；

**随机的轮询顺序可以避免 Channel 的饥饿问题，保证公平性；而根据 Channel 的地址顺序确定加锁顺序能够避免死锁的发生**。这段代码最后调用的 `runtime.sellock` 会按照之前生成的加锁顺序锁定 select 语句中包含所有的 Channel。

### 循环

当我们为 `select` 语句锁定了所有 Channel 之后就会进入 `runtime.selectgo` 函数的主循环，它会分三个阶段查找或者等待某个 Channel 准备就绪：

1. 查找是否已经存在准备就绪的 Channel，即可以执行收发操作；
2. 将当前 Goroutine 加入 Channel 对应的收发队列上并等待其他 Goroutine 的唤醒；
3. 当前 Goroutine 被唤醒之后找到满足条件的 Channel 并进行处理；

`runtime.selectgo` 函数会根据不同情况通过 `goto` 语句跳转到函数内部的不同标签执行相应的逻辑，其中包括：

- `bufrecv`：可以从缓冲区读取数据；
- `bufsend`：可以向缓冲区写入数据；
- `recv`：可以从休眠的发送方获取数据；
- `send`：可以向休眠的接收方发送数据；
- `rclose`：可以从关闭的 Channel 读取 EOF；
- `sclose`：向关闭的 Channel 发送数据；
- `retc`：结束调用并返回；

我们先来分析循环执行的第一个阶段，查找已经准备就绪的 Channel。循环会遍历所有的 case 并找到需要被唤起的 runtime.sudog 结构，在这个阶段，我们会根据 case 的四种类型分别处理：

1. 当 case 不包含 Channel 时
    - 这种 case 会被跳过；
2. 当 case 会从 Channel 中接收数据时
    - 如果当前 Channel 的 `sendq` 上有等待的 Goroutine，就会跳到 `recv` 标签并从缓冲区读取数据后将等待 Goroutine 中的数据放入到缓冲区中相同的位置；
    - 如果当前 Channel 的缓冲区不为空，就会跳到 `bufrecv` 标签处从缓冲区获取数据；
    - 如果当前 Channel 已经被关闭，就会跳到 `rclose` 做一些清除的收尾工作；
3. 当 case 会向 Channel 发送数据时
    - 如果当前 Channel 已经被关，闭就会直接跳到 `sclose` 标签，触发 `panic` 尝试中止程序；
    - 如果当前 Channel 的 `recvq` 上有等待的 Goroutine，就会跳到 `send` 标签向 Channel 发送数据；
    - 如果当前 Channel 的缓冲区存在空闲位置，就会将待发送的数据存入缓冲区；
4. 当 select 语句中包含 `default` 时
    - 表示前面的所有 case 都没有被执行，这里会解锁所有 Channel 并返回，意味着当前 select 结构中的收发都是非阻塞的；

第一阶段的主要职责是查找所有 case 中是否有可以立刻被处理的 Channel。无论是在等待的 Goroutine 上还是缓冲区中，只要存在数据满足条件就会立刻处理，如果不能立刻找到活跃的 Channel 就会进入循环的下一阶段，按照需要将当前 Goroutine 加入到 Channel 的 `sendq` 或者 `recvq` 队列中：
```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    ...
    gp = getg()
    nextp = &gp.waiting
    for _, casei := range lockorder {
        casi = int(casei)
        cas = &scases[casi]
        c = cas.c
        sg := acquireSudog()
        sg.g = gp
        sg.c = c

        if casi < nsends {
            c.sendq.enqueue(sg)
        } else {
            c.recvq.enqueue(sg)
        }
    }

    gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)
    ...
}
```

除了将当前 Goroutine 对应的 `runtime.sudog` 结构体加入队列之外，这些结构体都会被串成链表附着在 Goroutine 上。在入队之后会调用 `runtime.gopark` 挂起当前 Goroutine 等待调度器的唤醒。

当 select 中的一些 Channel 准备就绪之后，当前 Goroutine 就会被调度器唤醒。这时会继续执行 `runtime.selectgo` 函数的第三部分，从 `runtime.sudog` 中读取数据：

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    ...
    sg = (*sudog)(gp.param)
    gp.param = nil

    casi = -1
    cas = nil
    sglist = gp.waiting
    for _, casei := range lockorder {
        k = &scases[casei]
        if sg == sglist {
            casi = int(casei)
            cas = k
        } else {
            c = k.c
            if int(casei) < nsends {
                c.sendq.dequeueSudoG(sglist)
            } else {
                c.recvq.dequeueSudoG(sglist)
            }
        }
        sgnext = sglist.waitlink
        sglist.waitlink = nil
        releaseSudog(sglist)
        sglist = sgnext
    }

    c = cas.c
    goto retc
    ...
}
```

第三次遍历全部 case 时，我们会先获取当前 Goroutine 接收到的参数 `sudog` 结构，我们会依次对比所有 case 对应的 `sudog` 结构找到被唤醒的 case，获取该 case 对应的索引并返回。

由于当前的 select 结构找到了一个 case 执行，那么剩下 case 中没有被用到的 `sudog` 就会被忽略并且释放掉。为了不影响 Channel 的正常使用，我们还是需要将这些废弃的 `sudog` 从 Channel 中出队。

当我们在循环中发现缓冲区中有元素或者缓冲区未满时就会通过 `goto` 关键字跳转到 `bufrecv` 和 `bufsend` 两个代码段，这两段代码的执行过程都很简单，它们只是向 Channel 中发送数据或者从缓冲区中获取新数据：

```go
bufrecv:
    recvOK = true
    qp = chanbuf(c, c.recvx)
    if cas.elem != nil {
        typedmemmove(c.elemtype, cas.elem, qp)
    }
    typedmemclr(c.elemtype, qp)
    c.recvx++
    if c.recvx == c.dataqsiz {
        c.recvx = 0
    }
    c.qcount--
    selunlock(scases, lockorder)
    goto retc

bufsend:
    typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
    c.sendx++
    if c.sendx == c.dataqsiz {
        c.sendx = 0
    }
    c.qcount++
    selunlock(scases, lockorder)
    goto retc
```


这里在缓冲区进行的操作和直接调用 `runtime.chansend` 和 `runtime.chanrecv` 差不多，上述两个过程在执行结束之后都会直接跳到 `retc` 字段。

两个直接收发 Channel 的情况会调用运行时函数 `runtime.send` 和 `runtime.recv`，这两个函数会与处于休眠状态的 Goroutine 打交道：
```go
recv:
    recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
    recvOK = true
    goto retc

send:
    send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
    goto retc
```

不过如果向关闭的 Channel 发送数据或者从关闭的 Channel 中接收数据，情况就稍微有一点复杂了：

- 从一个关闭 Channel 中接收数据会直接清除 Channel 中的相关内容；
- 向一个关闭的 Channel 发送数据就会直接 `panic` 造成程序崩溃：

```go
rclose:
    selunlock(scases, lockorder)
    recvOK = false
    if cas.elem != nil {
        typedmemclr(c.elemtype, cas.elem)
    }
    goto retc

sclose:
    selunlock(scases, lockorder)
    panic(plainError("send on closed channel")
```

## 总结

select 结构的执行过程与实现原理，首先在编译期间，Go 语言会对 select 语句进行优化，它会根据 select 中 case 的不同选择不同的优化路径：

1. 空的 select 语句会被转换成调用 `runtime.block` 直接挂起当前 Goroutine；
    ```go
    func block() {
        gopark(nil, nil, waitReasonSelectNoCases, traceEvGoStop, 1)
    }
    ```
2. 如果 select 语句中只包含一个 case，编译器会将其转换成 `if ch == nil { block }; n;` 表达式；
    - 首先判断操作的 Channel 是不是空的；
    - 然后执行 case 结构中的内容；
3. 如果 select 语句中只包含两个 case 并且其中一个是 `default`，那么会使用 `runtime.selectnbrecv` 和 `runtime.selectnbsend`非阻塞地执行收发操作；
4. 在默认情况下会通过 `runtime.selectgo` 获取执行 case 的索引，并通过多个 `if` 语句执行对应 case 中的代码；

在编译器已经对 select 语句进行优化之后，Go 语言会在运行时执行编译期间展开的 `runtime.selectgo `函数，该函数会按照以下的流程执行：

1. 随机生成一个遍历的轮询顺序 `pollOrder` 并根据 Channel 地址生成锁定顺序 `lockOrder`；
2. 根据 `pollOrder` 遍历所有的 case 查看是否有可以立刻处理的 Channel；
    - 如果存在，直接获取 case 对应的索引并返回；
    - 如果不存在，创建 `runtime.sudog` 结构体，将当前 Goroutine 加入到所有相关 Channel 的收发队列，并调用 `runtime.gopark` 挂起当前 Goroutine 等待调度器的唤醒；
3. 当调度器唤醒当前 Goroutine 时，会再次按照 `lockOrder` 遍历所有的 case，从中查找需要被处理的 `runtime.sudog` 对应的索引；

select 关键字是 Go 语言特有的控制结构，它的实现原理比较复杂，需要编译器和运行时函数的通力合作。


