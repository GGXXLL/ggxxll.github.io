---
title: 'Golang Recover'
tags:
    - Golang
date: '2022-11-20'
categories:
draft: false
---

内置`recover`函数必须在合适的位置调用才能发挥作用；否则，它的调用相当于空操作。 比如，在下面这个程序中，没有一个`recover`函数调用恢复了恐慌`bye`。

```go
package main

func main() {
	defer func() {
		defer func() {
			recover() // 空操作
		}()
	}()
	defer func() {
		func() {
			recover() // 空操作
		}()
	}()
	func() {
		defer func() {
			recover() // 空操作
		}()
	}()
	func() {
		defer recover() // 空操作
	}()
	func() {
		recover() // 空操作
	}()
	recover()       // 空操作
	defer recover() // 空操作
	panic("bye")
}
```
我们已经知道下面这个`recover`调用是有作用的。
```go 
package main

func main() {
	defer func() {
		recover() // 将恢复恐慌"byte"
	}()

	panic("bye")
}
```

在下面的情况下，`recover`函数调用的返回值为`nil`：
- 传递给相应`panic`函数调用的实参为`nil`；
- 当前协程并没有处于恐慌状态；
- `recover`函数并未直接在一个延迟函数调用中调用。

一个`recover`调用只有在它的直接外层调用（即`recover`调用的父调用）是一个延迟调用，并且此延迟调用（即父调用）的直接外层调用（即`recover`调用的爷调用）和当前协程中最新产生并且尚未恢复的恐慌相关联时才起作用。 

一个有效的`recover`调用将最新产生并且尚未恢复的恐慌和与此恐慌相关联的函数调用（即爷调用）剥离开来，并且返回当初传递给产生此恐慌的`panic`函数调用的参数。