---
title: 'Golang sync.Map'
tags:
    - Golang
date: '2022-11-11'
categories:
draft: false
---


## 简介

一般情况下解决并发读写 map 的思路是加一把大锁，或者把一个 map 分成若干个小 map，对 key 进行哈希，只操作相应的小 map。前者锁的粒度比较大，影响效率；后者实现起来比较复杂，容易出错。

而使用 sync.Map 之后，对 map 的读写，不需要加锁。并且它通过空间换时间的方式，使用 read 和 dirty 两个 map 来进行读写分离，降低锁时间来提高效率。

sync.Map 适用于读多写少的场景。对于写多的场景，会导致 read map 缓存失效，需要加锁，导致冲突变多；而且由于未命中 read map 次数过多，导致 dirty map 提升为 read map，这是一个 O(N) 的操作，会进一步降低性能。

## 源码解析

```go
type Map struct {
	mu Mutex
	read atomic.Value // readOnly
	dirty map[any]*entry
	misses int
}


// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
	m       map[any]*entry
	amended bool // true if the dirty map contains some key not in m.
}

// expunged is an arbitrary pointer that marks entries which have been deleted
// from the dirty map.
var expunged = unsafe.Pointer(new(any))

// An entry is a slot in the map corresponding to a particular key.
type entry struct {
	p unsafe.Pointer // *any
}

```

- 互斥量 mu 保护 read 和 dirty。

- read 是 `atomic.Value` 类型，可以并发地读。但如果需要更新 read，则需要加锁保护。对于 read 中存储的 entry 字段，可能会被并发地 CAS 更新。但是如果要更新一个之前已被删除的 entry，则需要先将其状态从 expunged 改为 nil，再拷贝到 dirty 中，然后再更新。
- dirty 是一个非线程安全的原始 map。包含新写入的 key，并且包含 read 中的所有未被删除的 key。这样，可以快速地将 dirty 提升为 read 对外提供服务。如果 dirty 为 nil，那么下一次写入时，会新建一个新的 dirty，这个初始的 dirty 是 read 的一个拷贝，但除掉了其中已被删除的 key。
- 每当从 read 中读取失败，都会将 misses 的计数值加 1，当加到一定阈值以后，需要将 dirty 提升为 read，以期减少 miss 的情形。

read map 和 dirty map 的存储方式是不一致的。前者使用 `atomic.Value`，后者只是单纯的使用 map。

原因是 read map 使用 lock free 操作，必须保证 load/store 的原子性；而 dirty map 的 load+store 操作是由 lock（就是 mu）来保护的。


entry 对象是一个指针，指向 value。看来，read 和 dirty 各自维护一套 key，key 指向的都是同一个 value。也就是说，只要修改了这个 entry，对 read 和 dirty 都是可见的。这个指针的状态有三种：
- nil: 说明这个键值对已被删除，并且 `m.dirty == nil`，或 `m.dirty[k]` 指向该 entry。
- expunged: 说明这条键值对已被删除，并且 `m.dirty != nil`，且 m.dirty 中没有这个 key。
- 正常：指向一个正常的值，表示实际 `any` 的地址，并且被记录在 `m.read.m[key]` 中。如果这时 `m.dirty != nil`，那么它也被记录在 `m.dirty[key]` 中。两者实际上指向的是同一个值。

当删除 key 时，并不实际删除。一个 entry 可以通过原子地（CAS 操作）设置 p 为 nil 被删除。如果之后创建 m.dirty，nil 又会被原子地设置为 expunged，且不会拷贝到 dirty 中。

如果 p 不为 expunged，和 entry 相关联的这个 value 可以被原子地更新；如果 p == expunged，那么仅当它初次被设置到 m.dirty 之后，才可以被更新。

![](img/syncmap.webp)


## Load

```go
func (m *Map) Load(key any) (value any, ok bool) {
    // 因read只读，线程安全，优先读取
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    
    // 如果read没有，并且dirty有新数据，那么去dirty中查找
    if !ok && read.amended {
        m.mu.Lock()
        // 双重检查（原因是前文的if判断和加锁非原子的，害怕这中间发生故事）
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        
        // 如果read中还是不存在，并且dirty中有新数据
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // m计数+1
            m.missLocked()
        }
        
        m.mu.Unlock()
    }
    
    if !ok {
        return nil, false
    }
    return e.load()
}

func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    
    // 将dirty置给read，因为穿透概率太大了(原子操作，耗时很小)
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}

func (e *entry) load() (value any, ok bool) {
	p := atomic.LoadPointer(&e.p)
	if p == nil || p == expunged {
		return nil, false
	}
	return *(*any)(p), true
}

```


## Delete

```go
func (m *Map) Delete(key any) {
    // 读出read，断言为readOnly类型
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    // 如果read中没有，并且dirty中有新元素，那么就去dirty中去找。这里用到了amended，当read与dirty不同时为true，说明dirty中有read没有的数据。
    if !ok && read.amended {
        m.mu.Lock()
        // 再检查一次，因为前文的判断和锁不是原子操作，防止期间发生了变化。
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        
        if !ok && read.amended {
            // 直接删除
            delete(m.dirty, key)
        }
        m.mu.Unlock()
    }
    
    if ok {
        // 如果read中存在该key，则将该value 赋值nil（采用标记的方式删除！）
        e.delete()
    }
}

func (e *entry) delete() (hadValue bool) {
    for {
    	// 数据的指针
        p := atomic.LoadPointer(&e.p)
        if p == nil || p == expunged {
            return false
        }
        
        // 原子操作
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return true
        }
    }
}
```


## Store
```go
func (m *Map) Store(key, value any) {
    // 如果m.read存在这个key，并且没有被标记删除，则尝试更新。
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }
    
    // 如果read不存在或者已经被标记删除
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
   
    if e, ok := read.m[key]; ok { // read 存在该key
    // 如果entry被标记expunge，则表明dirty没有key，可添加入dirty，并更新entry。
        if e.unexpungeLocked() { 
            // 加入dirty中，这儿是指针
            m.dirty[key] = e
        }
        // 更新value值
        e.storeLocked(&value) 
        
    } else if e, ok := m.dirty[key]; ok { // dirty 存在该key，更新
        e.storeLocked(&value)
        
    } else { // read 和 dirty都没有
        // 如果read与dirty相同，则触发一次dirty刷新（因为当read重置的时候，dirty已置为nil了）
        if !read.amended { 
            // 将read中未删除的数据加入到dirty中
            m.dirtyLocked() 
            // amended标记为read与dirty不相同，因为后面即将加入新数据。
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value) 
    }
    m.mu.Unlock()
}

// 将read中未删除的数据加入到dirty中
func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }
    
    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[any]*entry, len(read.m))
    
    // 遍历read。
    for k, e := range read.m {
        // 通过此次操作，dirty中的元素都是未被删除的，可见标记为expunged的元素不在dirty中！！！
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

// 判断entry是否被标记删除，并且将标记为nil的entry更新标记为expunge
func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    
    for p == nil {
        // 将已经删除标记为nil的数据标记为expunged
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}

// 对entry尝试更新 （原子cas操作）
func (e *entry) tryStore(i *any) bool {
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
        return false
    }
    for {
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
    }
}

// read里 将标记为expunge的更新为nil
func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

// 更新entry
func (e *entry) storeLocked(i *any) {
    atomic.StorePointer(&e.p, unsafe.Pointer(i))
}

```

