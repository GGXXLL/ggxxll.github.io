---
title: 'Golang Slice'
tags:
    - Golang
date: '2022-11-11'
categories:
    - 后端
draft: false
---

## 简介

切片（slice）是 Golang 中一种比较特殊的数据结构，这种数据结构更便于使用和管理数据集合。切片是围绕动态数组的概念构建的，可以按需自动增长和缩小。

切片的动态增长是通过内置函数 `append()` 来实现的，这个函数可以快速且高效地增长切片，也可以通过对切片再次切割，缩小一个切片的大小。

因为切片的底层也是在连续的内存块中分配的，所以切片还能获得索引、迭代以及为垃圾回收优化的好处。

## 实现
```golang
type slice struct {
  array unsafe.Pointer // 指向底层数组的指针
  len   int // 长度
  cap   int // 容量
}
```

## nil 和 空切片

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

空切片与 `nil` 切片的区别:
- `nil 切片 == nil` , 而`空切片 != nil` ，在使用切片进行逻辑运算时尽量不要使用空切片
- 空切片指针指向一个特殊的 `zerobase` 地址，而  `nil`  为 0
- 在 JSON 序列化有区别： `nil`  切片为 `{"values":null}`, 而空切片为 `{"value" []}`


## 追加

对于 append 向 slice 添加元素的步骤：

- 加入 slice 容量够用，则追加新元素进去，`slice.len++`，返回原来的slice。
- 当原容量不够，则 slice 先扩容，扩容之后得到新的 slice，将元素追加进新的 slice，`slice.len++`，返回新的 slice。

```go
func main(){
	a := []int{1, 2, 3, 4, 5}
	b := a[:3]
	// b 作为 a 的子切片，两者指向的是用一个底层数组，所以 cap 相同，只是限制了 b 的 len
	t.Log(a, len(a), cap(a)) // [1 2 3 4 5] 5 5
	t.Log(b, len(b), cap(b)) // [1 2 3] 3 5
	// 这种情况下，a b 会互相影响
	a[0] = 0
	b[1] = 1
	t.Log(a, len(a), cap(a)) // [0 1 3 4 5] 5 5
	t.Log(b, len(b), cap(b)) // [0 1 3] 3 5

	// 在未产生扩容的情况下，b 的 append 操作会覆盖 a 中的值
	// 如下：a[3] 被覆盖为 6 了
	b = append(b, 6)
	t.Log(a, len(a), cap(a)) // [0 1 3 6 5] 5 5
	t.Log(b, len(b), cap(b)) // [0 1 3 6] 4 5


	// 产生扩容，b 指向的底层数组是新的数组了，a b 不再互相影响
	b = append(b, 7, 8, 9, 10)
	t.Log(a, len(a), cap(a)) // [0 1 3 6 5] 5 5
	t.Log(b, len(b), cap(b)) // [0 1 3 6 7 8 9 10] 8 10

	a[0] = -1
	b[1] = -2
	t.Log(a, len(a), cap(a)) // [-1 1 3 6 5] 5 5
	t.Log(b, len(b), cap(b)) // [0 -2 3 6 7 8 9 10] 8 10
}
```

## 扩容

### go1.15.15

`go1.15`，`go1.16` 和 `go1.17` 版本的扩容的边界条件都是 1024， 但是 `go1.15` 基于 `slice.len` 判断的，`go1.16` 和 `go1.17`版本是基于 `slice.cap` 容量判断的；

扩容的基本原则：
1. 如果要扩容的新容量已经超过旧容量的 2 倍，那么就直接使用新容量。
2. 否则：
- 如果当前切片的长度小于 1024 就会将容量翻倍；
- 否则就会每次增加 25% 的容量，直到新容量大于期望容量。

```golang
//go1.15.6 源码 src/runtime/slice.go
func growslice(et *_type, old slice, cap int) slice {
	// 省略部分判断代码
    // 计算扩容部分
    // 其中，cap : 所需容量，newcap : 最终申请容量
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 { // 1.15 之后使用 old.cap
			newcap = doublecap
		} else {
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
 
	//省略部分判断代码
}
```


### go1.18

在 `go1.18` 之后， 扩容的基本原则：
1. 如果新容量已经超过旧容量的 2 倍，那么就直接使用新容量。
2. 否则：
- 如果旧容量小于 256，则新的扩容会是原来的 2 倍。
- 否则使用计算公式 `newcap += (newcap + 3 * 256) / 4`，直到新容量大于期望容量：从小切片的 2 倍增长过渡到大切片的 1.25 倍增长，这个公式在两者之间给出了平滑的过渡。

扩容函数 [growslice](https://github.com/golang/go/blob/43456202a1e55da55666fac9d56ace7654a65b64/src/runtime/slice.go)
```go
func growslice(et *_type, old slice, cap int) slice {
	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
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
			for 0 < newcap && newcap < cap {
				newcap += (newcap + 3*threshold) / 4
			}
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
	...
}
```

### go1.20

在 `go1.19` 之后，扩容参数变更为 `newLen`， 扩容的基本原则：
1. 如果 `newLen` 已经超过 `oldCap` 的 2 倍，那么就直接使用 `newLen` 作为新容量。
2. 否则：
- 如果 `oldCap` 小于 256，则新的扩容会 `oldCap * 2`。
- 否则使用计算公式 `newcap += (newcap + 3 * 256) / 4`，直到新容量大于期望容量：从小切片的 2 倍增长过渡到大切片的 1.25 倍增长，这个公式在两者之间给出了平滑的过渡。


扩容函数 [growslice](https://github.com/golang/go/blob/cf93b25366aa418dea3eea49a7b85447631c2a1d/src/runtime/slice.go)
```golang
// arguments:
//
//	oldPtr = slice.array 指针
//	newLen = oldLen + num
//	oldCap = 原始 slice 的容量
//	num = 要添加的元素个数
//	et = element type
func growslice(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice {
	oldLen := newLen - num
	if newLen < 0 {
		panic(errorString("growslice: len out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve oldPtr in this case.
		return slice{unsafe.Pointer(&zerobase), newLen, newLen}
	}

	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		newcap = newLen
	} else {
		const threshold = 256
		if oldCap < threshold {
			newcap = doublecap
		} else {
			for 0 < newcap && newcap < newLen {
				newcap += (newcap + 3*threshold) / 4
			}
			if newcap <= 0 {
				newcap = newLen
			}
		}
	}
	...
}
```
