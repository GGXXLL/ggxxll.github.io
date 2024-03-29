---
title: 'Golang GMP'
tags:
    - Golang
date: '2022-11-11'
categories:
    - 后端
draft: false
---

## 简介

CPU 线程切换的开销是影响系统处理性能的一大障碍，Go 提供了一种机制，可以在用户空间实现任务的切换，上下文切换成本更小，可以达到使用较少的线程数量实现较大并发的能力，即 GMP 模型

- G（Goroutine）: 即Golang协程，协程是一种用户态线程，比内核态线程更加轻量。使用 `go` 关键字可以创建一个 Golang 协程
- M（Machine）: 即工作线程。实质上实现业务逻辑的载体
- P（Processor）: 处理器。是 Golang 内部的定义，非 CPU。包含运行 Golang 代码的必要资源，也有调度 Goroutine 的能力


![](img/gmp.webp)


1. 全局队列（Global Queue）：存放等待运行的 G。
2. P 的本地队列：同全局队列类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。新建 G时，G优先加入到 P 的本地队列，如果队列满了，则会把本地队列中一半的 G 移动到全局队列。
3. P 列表：所有的 P 都在程序启动时创建，并保存在数组中，最多有 GOMAXPROCS(可配置) 个。
4. M：线程想运行任务就得获取 P，从 P 的本地队列获取 G，P 队列为空时，M 也会尝试从全局队列拿一批 G 放到 P 的本地队列，或从其他 P 的本地队列偷一半放到自己 P 的本地队列。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。

## 关于P和M，G的个数问题
1. G 的数量
- 无限制，理论上受内存的影响，创建一个 G 的初始栈大小为2-4K，配置一般的机器也能简简单单开启数十万个 Goroutine ，而且Go语言在 G 退出的时候还会把 G 清理之后放到 P 本地或者全局的闲置列表 gFree 中以便复用。
2. P 的数量
- 由启动时环境变量 `GOMAXPROCS` 或者是由 `runtime.GOMAXPROCS()` 决定。
3. M 的数量

- go 语言本身的限制：go 程序启动时，会设置 M 的最大数量，默认 10000。但是内核很难支持这么多的线程数，所以这个限制可以忽略。
- runtime/debug 中的 SetMaxThreads 函数，设置 M 的最大数量
- 一个 M 阻塞了，会创建新的 M。M 与 P 的数量没有绝对关系，一个 M 阻塞，P 就会去创建或者切换另一个 M，所以，即使 P 的默认数量是 1，也有可能会创建很多个 M 出来。

## P和M何时被创建
1. P 何时创建：在确定了 P 的最大数量 n 后，运行时系统会根据这个数量创建 n 个 P。
2. M 何时创建：没有足够的 M 来关联 P 并运行其中的可运行的 G。比如所有的 M 此时都阻塞住了，而 P 中还有很多就绪任务，就会去寻找空闲的 M，而没有空闲的，就会去创建新的 M。

## Goroutine 调度流程
![](img/gmp-go.webp)

从上图我们可以分析出几个结论：

1. 我们通过 `go func ()` 来创建一个 goroutine；
2. 有两个存储 G 的队列，一个是局部调度器 P 的本地队列、一个是全局 G 队列。新创建的 G 会先保存在 P 的本地队列中，如果 P 的本地队列已经满了就会保存在全局的队列中；
3. G 只能运行在 M 中，一个 M 必须持有一个 P，M 与 P 是 1：1 的关系。M 会从 P 的本地队列弹出一个可执行状态的 G 来执行，如果 P 的本地队列为空，就会想其他的 MP 组合偷取一个可执行的 G 来执行；
4. 一个 M 调度 G 执行的过程是一个循环机制；
5. 当 M 执行某一个 G 时候如果发生了 syscall 或则其余阻塞操作，M 会阻塞，如果当前有一些 G 在执行，runtime 会把这个线程 M 从 P 中摘除 (detach)，然后再创建一个新的操作系统的线程 (如果有空闲的线程可用就复用空闲线程) 来服务于这个 P；
6. 当 M 系统调用结束时候，这个 G 会尝试获取一个空闲的 P 执行，并放入到这个 P 的本地队列。如果获取不到 P，那么这个线程 M 变成休眠状态， 加入到空闲线程中，然后这个 G 会被放入全局队列中。

调度器的生命周期流程：
![](img/gmp-life.webp)


## 特殊的 M0 和 G0

> M0 是启动程序后的编号为 0 的主线程，这个 M 对应的实例会在全局变量 runtime.m0 中，不需要在 heap 上分配，M0 负责执行初始化操作和启动第一个 G， 在之后 M0 就和其他的 M 一样了。

> G0 是每次启动一个 M 都会第一个创建的 gourtine，G0 仅用于负责调度的 G，G0 不指向任何可执行的函数，每个 M 都会有一个自己的 G0。在调度或系统调用时会使用 G0 的栈空间，全局变量的 G0 是 M0 的 G0。


## 调度器的设计策略

1. 复用线程：避免频繁的创建、销毁线程，而是对线程的复用。
- work stealing机制

    当本线程无可运行的 G 时，尝试从其他线程绑定的 P 偷取 G，而不是销毁线程。
- hand off机制

    当本线程因为 G 进行系统调用阻塞时，线程释放绑定的 P，把 P 转移给其他空闲的线程执行。

2. 利用并行：`GOMAXPROCS`设置 P 的数量，最多有`GOMAXPROCS`个线程分布在多个 CPU 上同时运行。`GOMAXPROCS`也限制了并发的程度，比如`GOMAXPROCS` = 核数/2，则最多利用了一半的CPU核进行并行。
3. 抢占：在 coroutine 中要等待一个协程主动让出 CPU 才执行下一个协程，在 Go 中，一个 goroutine 最多占用 CPU 10ms，防止其他 goroutine 被饿死，这就是 goroutine 不同于 coroutine 的一个地方。
4. 全局 G 队列：在新的调度器中依然有全局 G 队列，但功能已经被弱化了，当 M 执行 work stealing 从其他 P 偷不到 G 时，它可以从全局 G 队列获取 G。



## 注意事项
1. Goroutine 本质上没有数量限制，最大数量取决于内存。
2. 对每一个使用标准编译器编译的Go程序，在运行时刻，每一个协程将维护一个栈（stack）。 一个栈是一个预申请的内存段，它做为一个内存池供某些内存块从中开辟。 在 Go 1.12 之前没有最大限制；在 Go 1.14 ~ 1.19 版本之前，一个栈的初始尺寸总是 2KiB。 从 1.19 版本开始，栈的初始尺寸是自适应的。 每个栈的尺寸在协程运行的时候将按照需要增长和收缩。 栈的最小尺寸为 2KiB。

（注意：Go 运行时维护着一个协程栈的最大尺寸限制，此限制为全局的。 如果一个协程在增长它的栈的时候超过了此限制，整个程序将崩溃。 对于目前的官方标准 Go 工具链 1.19 版本，此最大限制的默认值在 64 位系统上为 1GB，在32位系统上为 250MB。 我们可以在运行时刻调用 `runtime/debug` 标准库包中的 `SetMaxStack` 来修改此值。 另外请注意，当前的官方标准编译器实现中，实际上允许的协程栈的最大尺寸为不超过最大尺寸限制的2的幂。 所以对于默认设置，实际上允许的协程栈的最大尺寸在64 位系统上为 512MiB，在 32 位系统上为 128MiB。）