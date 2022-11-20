---
title: 'Golang Channel'
tags:
    - Golang
date: '2022-11-11'
categories:
    - 后端
draft: false
---

## 数据结构

Go 语言的 Channel 在运行时使用 `runtime.hchan` 结构体表示。我们在 Go 语言中创建新的 Channel 时，实际上创建的都是如下所示的结构：

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
- `qcount`：元素个数
- `dataqsize`：buffer 的大小，也就是make(chan T, N)中的N。
- `elemsize`：单个元素的大小
- `elemtype`：元素的类型
- `buf`：缓冲区数据指针
- `closed`：channel是否关闭，0->打开，1->关闭
- `sendx`：发送操作处理到的位置
- `recvx`：接收操作处理到的位置
- `recvq`：读取数据而阻塞的 Goroutine 双向链表
- `sendq`：写入数据而阻塞的 Goroutine 双向链表
- `lock`：锁，保证并发安全

## 初始化

当我们在代码里面通过 `make(chan int, 1)`, 实际调用了 `runtime.makechan`:

```go
func makechan(t *chantype, size int) *hchan {
  elem := t.elem
  // 判断 元素类型的大小
  if elem.size >= 1<<16 {
    throw("makechan: invalid channel element type")
  }
  // 判断对齐限制
  if hchanSize%maxAlign != 0 || elem.align > maxAlign {
    throw("makechan: bad alignment")
  }
  // 判断 size 非负 和 是否大于 maxAlloc 限制
  mem, overflow := math.MulUintptr(elem.size, uintptr(size))
  if overflow || mem > maxAlloc-hchanSize || size < 0 {
    panic(plainError("makechan: size out of range"))
  }
  var c *hchan
  switch {
  case mem == 0: // 无缓冲区，即 make 没设置大小
    c = (*hchan)(mallocgc(hchanSize, nil, true)) 
    c.buf = c.raceaddr()
  case elem.ptrdata == 0:  // 数据类型不包含指针
    c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
    c.buf = add(unsafe.Pointer(c), hchanSize)
  default:  // 如果包含指针
    // Elements contain pointers.
    c = new(hchan)
    c.buf = mallocgc(mem, elem, true)
  }
  c.elemsize = uint16(elem.size)
  c.elemtype = elem
  c.dataqsiz = uint(size)
  if debugChan {
    print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
  }
  return c
}
```

创建channel分为三种情况:

- 无缓冲，只会为 `runtime.hchan` 分配一段内存空间

- 有缓冲，且channel的类型不包含指针，此时 `buf` 为 `hchanSize+元素大小*元素个数` 的连续内存

- 有缓冲，且channel的类型包含指针，则不能简单的根据元素的大小去申请内存，需要根据元素去分配内存


## 发送数据

我们在这里可以简单梳理和总结一下使用 `ch <- i` 表达式向 Channel 发送数据时遇到的几种情况：

1. 如果当前 Channel 的 `recvq` 上存在已经被阻塞的 Goroutine，那么会直接将数据发送给当前 Goroutine 并将其设置成下一个运行的 Goroutine；
2. 如果 Channel 存在缓冲区并且其中还有空闲的容量，我们会直接将数据存储到缓冲区 `sendx` 所在的位置上；
3. 如果不满足上面的两种情况，会创建一个 `runtime.sudog` 结构并将其加入 Channel 的 `sendq` 队列中，当前 Goroutine 也会陷入阻塞等待其他的协程从 Channel 接收数据；

发送数据的过程中包含几个会触发 Goroutine 调度的时机：

1. 发送数据时发现 Channel 上存在等待接收数据的 Goroutine，立刻设置处理器的 `runnext` 属性，但是并不会立刻触发调度；
2. 发送数据时并没有找到接收方并且缓冲区已经满了，这时会将自己加入 Channel 的 `sendq` 队列并调用 `runtime.goparkunlock` 触发 Goroutine 的调度让出处理器的使用权；

## 接收数据

我们梳理一下从 Channel 中接收数据时可能会发生的五种情况：

1. 如果 Channel 为空，那么会直接调用 `runtime.gopark` 挂起当前 Goroutine；
2. 如果 Channel 已经关闭并且缓冲区没有任何数据，`runtime.chanrecv` 会直接返回；
3. 如果 Channel 的 `sendq` 队列中存在挂起的 Goroutine，会将 `recvx` 索引所在的数据拷贝到接收变量所在的内存空间上并将 `sendq` 队列中 Goroutine 的数据拷贝到缓冲区；
4. 如果 Channel 的缓冲区中包含数据，那么直接读取 `recvx` 索引对应的数据；
在默认情况下会挂起当前的 Goroutine，将 `runtime.sudog` 结构加入 `recvq` 队列并陷入休眠等待调度器的唤醒；

我们总结一下从 Channel 接收数据时，会触发 Goroutine 调度的两个时机：

1. 当 Channel 为空时；
2. 当缓冲区中不存在数据并且也不存在数据的发送者时；

## 关闭通道

1. 当 Channel 是一个空指针或者已经被关闭时，Go 语言运行时都会直接崩溃并抛出异常。
2. 然后将 `recvq` 和 `sendq` 两个队列中的数据加入到 Goroutine 列表 gList 中，与此同时该函数会清除所有 `runtime.sudog` 上未被处理的元素。
3. 最后会为所有被阻塞的 Goroutine 调用 `runtime.goready` 触发调度。

```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}

	c.closed = 1

	var glist gList

	// release all readers
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```


## 引用

[Golang 深度剖析 -- channel的底层实现](https://zhuanlan.zhihu.com/p/264305133)
[Go 语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)