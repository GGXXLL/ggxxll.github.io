---
title: 'Golang Slice'
tags:
    - Golang
date: '2022-11-11'
categories:
    - 后端
draft: false
---

### 简介

切片(slice)是 Golang 中一种比较特殊的数据结构，这种数据结构更便于使用和管理数据集合。切片是围绕动态数组的概念构建的，可以按需自动增长和缩小。切片的动态增长是通过内置函数 `append()` 来实现的，这个函数可以快速且高效地增长切片，也可以通过对切片再次切割，缩小一个切片的大小。因为切片的底层也是在连续的内存块中分配的，所以切片还能获得索引、迭代以及为垃圾回收优化的好处。

### 实现
```golang
type slice struct {
  array unsafe.Pointer // 指向底层数组的指针
  len   int // 长度
  cap   int // 容量
}
```

### nil 和 空切片

nil 切片:
```golang
var nums []int
```

空切片：
```golang
//使用make
s1 := make([]int, 0)
//使用切片字面量
s2 := []int{}
```

空切片与nil切片的区别:
- nil切片=nil, 而空切片！=nil，在使用切片进行逻辑运算时尽量不要使用空切片
- 空切片指针指向一个特殊的 zerobase 地址，而 nil 为 0
- 在JSON序列化有区别：nil 切片为 `{"values":null}`, 而空切片为 `{"value" []}`


### 扩容

#### go1.15.6

扩容的基本原则：
- 如果 slice 的容量小于 1024，则新的扩容会是原来的 2 倍。
- 如果原来的 slice 的容量大于或者等于 1024，则新的扩容将扩大大于或者等于原来 1.25 倍。

```golang
//go1.15.6 源码 src/runtime/slice.go
func growslice(et *_type, old slice, cap int) slice {
	//省略部分判断代码
    //计算扩容部分
    //其中，cap : 所需容量，newcap : 最终申请容量
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
 
	//省略部分判断代码
}
```

#### go1.19

扩容的基本原则：
- 如果 slice 的容量小于 256，则新的扩容会是原来的 2 倍。
- 如果 slice 的容量大于 256，则使用计算公式 `newcap += (newcap + 3 * 256) / 4`。


```golang
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, abi.FuncPCABIInternal(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}
	if asanenabled {
		asanread(old.array, uintptr(old.len*int(et.size)))
	}

	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		const threshold = 256
		if old.cap < threshold {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				// Transition from growing 2x for small slices
				// to growing 1.25x for large slices. This formula
				// gives a smooth-ish transition between the two.
				newcap += (newcap + 3*threshold) / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
    ...
}
```


对于 append 向 slice 添加元素的步骤：

- 加入 slice 容量够用，则追加新元素进去，slice.len++，返回原来的slice。
- 当原容量不够，则 slice 先扩容，扩容之后得到新的 slice，将元素追加进新的 slice，slice.len++，返回新的 slice。