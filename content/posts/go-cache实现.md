---
title: 使用 map 设计缓存
tags: 
  - Golang
date: 2022-08-24
categories:
  - 后端
---

## 题一

- 问题1： 会输出什么？为什么？
- 问题2： 应该如何修改？

```go
package main

import "fmt"

func main() {
	cache := Cache{}
    fmt.Println(cache.Get("foo"))

	cache.Set("foo", "bar")

	fmt.Println(cache.Get("foo"))
}

type Cache struct {
	data map[string]interface{}
}

func (c Cache) Get(s string) (interface{},bool) {
	v, ok := c.data[s]
	return v, ok
}

func (c Cache) Set(s string, v interface{}) {
	c.data[s] = v
}
```

## 题二

- 问题1：会输出什么？为什么？
- 问题2：应该如何修改？

```go
package main

import "fmt"

func main() {
	cache := Cache{}
	cache.Add(1)
	fmt.Println(cache.List())
}

type Cache struct {
	data []interface{}
}

func (c Cache) Add(s interface{}) {
	c.data = append(c.data, s)
}

func (c Cache) List() []interface{} {
	return c.data
}
```

## 题三
- 问题1：为什么题一需要初始化data，题二不需要?
- 问题2：实现这个Cache还需要什么补充的吗?

## 题一解
- 问题1：
    - Get 返回 `nil, false`
    - Set 会报空指针错误
- 问题2：
    - Cache 需要初始化 data

## 题二解
- 问题1：
    - 输出空数组
- 问题2：
    - 修改 `func (c Cache) Add` 为 `func (c *Cache) Add`

## 题三解
- 问题1：因为 map 是引用传递，数组是值传递
- 问题2：需要添加锁 `sync.Mutex` ，并将 `Cache` 的方法改为指针调用