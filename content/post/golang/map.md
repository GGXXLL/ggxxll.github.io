---
title:  Map 探究
tags:
  - Golang
date: 2022-08-25
categories:
  - 后端
---

## 基础知识

### map的概念

map的直观意思是映射，是<key, value> 对组成的抽象结构，且 key 不会重复。map 的底层实现一般有两种：
1. 搜索树（search tree），天然有序，平均/最差的写入/查找复杂度是O(logN)
2. 哈希表（hash table），无序存储，平均的写入/查找复杂度是O(1)，最差是O(N)

## 底层

### 源码

在 golang 中，[map](https://github.com/golang/go/blob/master/src/runtime/map.go) 的底层实现是哈希表：
```go
const (
    bucketCntBits = 3
    bucketCnt     = 1 << bucketCntBits
)


type hmap struct {
   count     int    // map中存入元素的个数， golang 中调用 len(map) 的时候直接返回该字段
   flags     uint8  // 状态标记位，通过与定义的枚举值进行&操作可以判断当前是否处于这种状态
   B         uint8  // 2^B 表示 bucket 的数量， B 表示取hash后多少位来做 bucket 的分组
   noverflow uint16 // overflow bucket 的数量的近似数
   hash0     uint32 // hash seed （hash 种子） 一般是一个素数

   buckets    unsafe.Pointer // 共有2^B个 bucket，但是如果没有元素存入，这个字段可能为nil
   oldbuckets unsafe.Pointer // 在扩容期间，将旧的 bucket 数组放在这里， 新 bucket 会是这个的两倍大; 非扩容状态，值为 nil
   nevacuate  uintptr        // 表示已经完成扩容迁移的bucket的指针， 地址小于当前指针的 bucket 已经迁移完成

   extra *mapextra // 为了优化GC扫描而设计的。当 key 和 value 均不包含指针，并且都可以inline时使用。extra是指向 mapextra 类型的指针
}

// A bucket for a Go map.
type bmap struct {
    tophash [bucketCnt]uint8
}

// 编译期间会给它加料，动态地创建一个新的结构：
type bmap struct {
    // 长度为8的数组，用来快速定位 key 是否在桶里
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr        // 内存对齐使用，可能不需要
    overflow uintptr        // 当 bucket 的8个key 存满了之后
}
```

### hmap
map 的底层结构是 hmap，hmap 包含若干个结构为 bmap 的 bucket 数组，每个 bucket 底层都采用链表结构。

### bmap
bmap 就是我们常说的“桶”的底层数据结构， 一个桶中可以存放最多 8 个 key/value, map 使用 hash 函数 得到 hash 值决定分配到哪个桶， 
然后又会根据 hash 值的高 8 位来寻找放在桶的哪个位置。具体的 map 的组成结构如下图所示：

bmap 中 key 和 value 是各自放在一起的，并不是 key/value/key/value/... 这样的形式。源码里说明这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间。

### mapextra

```go
// mapextra holds fields that are not present on all maps.
type mapextra struct {
    // 如果 key 和 value 都不包含指针，并且可以被 inline(<=128 字节)
    // 就使用 hmap 的 extra 字段 来存储 overflow buckets，这样可以避免 GC 扫描整个 map
    // 然而 bmap.overflow 也是个指针。这时候我们只能把这些 overflow 的指针
    // 都放在 hmap.extra.overflow 和 hmap.extra.oldoverflow 中了
    // overflow 包含的是 hmap.buckets 的 overflow 的 buckets
    // oldoverflow 包含扩容时的 hmap.oldbuckets 的 overflow 的 bucket
    overflow    *[]*bmap
    oldoverflow *[]*bmap
    
    // 指向空闲的 overflow bucket 的指针
    nextOverflow *bmap
}

```


## 特性

### 引用类型

map是个指针，底层指向hmap，所以是个引用类型。
golang 有三个常用的高级类型slice、map、channel，它们都是引用类型，当引用类型作为函数参数时，可能会修改原内容数据。
golang 中没有引用传递，只有值和指针传递。所以 map 作为函数实参传递时本质上也是值传递，只不过因为 map 底层数据结构是通过指针指向实际的元素存储空间，
在被调函数中修改 map，对调用者同样可见，所以 map 作为函数实参传递时表现出了引用传递的效果。
因此，传递 map 时，如果想修改map的内容而不是map本身，函数形参无需使用指针。

```go
func TestSliceFn(t *testing.T) {
    m := map[string]int{}
    t.Log(m, len(m))
    // map[a:1]
    mapAppend(m, "b", 2)
    t.Log(m, len(m))
    // map[a:1 b:2] 2
}

func mapAppend(m map[string]int, key string, val int) {
    m[key] = val
}
```

### 非线程安全

map默认是并发不安全的，原因如下：

Go 官方在经过了长时间的讨论后，认为 Go map 更应适配典型使用场景（不需要从多个 goroutine 中进行安全访问），而不是为了小部分情况（并发访问），导致大部分程序付出加锁代价（性能），决定了不支持。

更改 map 时会检查 hmap 的标志位 flags。如果 flags 的写标志位此时被置 1 了，说明有其他协程在执行“写”操作，进而导致程序 panic。这也说明了 map 对协程是不安全的。

### 共享存储空间
map 底层数据结构是通过指针指向实际的元素存储空间 ，这种情况下，对其中一个map的更改，会影响到其他map
```go
func TestMapShareMemory(t *testing.T) {
    m1 := map[string]int{}
    m2 := m1
    m1["a"] = 1
    t.Log(m1, len(m1))
    // map[a:1] 1
    t.Log(m2, len(m2))
    // map[a:1]
}
```
### 哈希冲突

golang中map是一个kv对集合。底层使用hash table，用链表来解决冲突 ，出现冲突时，不是每一个key都申请一个结构通过链表串起来，
而是以bmap为最小粒度挂载，一个bmap可以放8个kv。在哈希函数的选择上，会在程序启动时，检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。

### 遍历无序

map 在没有被修改的情况下，使用 range 多次遍历 map 时输出的 key 和 value 的顺序可能不同。这是 Go 语言的设计者们有意为之，
在每次 range 时的顺序被随机化，旨在提示开发者们，Go 底层实现并不保证 map 遍历顺序稳定，请大家不要依赖 range 遍历结果顺序。


## 扩容

### 扩容条件

再来说触发 map 扩容的时机：在向 map 插入新 key 的时候，会进行条件检测，符合下面这 2 个条件，就会触发扩容：
```go
// If we hit the max load factor or we have too many overflow buckets,
// and we're not already in the middle of growing, start growing.
if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
    hashGrow(t, h)
    goto again // Growing the table invalidates everything, so try again
}
```

1. 装载因子超过阈值  

    源码里定义的阈值是 6.5 (loadFactorNum/loadFactorDen)，是经过测试后取出的一个比较合理的因子。
我们知道，每个 bucket 有 8 个空位，在没有溢出，且所有的桶都装满了的情况下，装载因子算出来的结果是 8。因此当装载因子超过 6.5 时，
表明很多 bucket 都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的。  

    对于条件 1，元素太多，而 bucket 数量太少，很简单：将 B 加 1，bucket 最大数量(2^B)直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了。  

    注意，这时候元素都在老 bucket 里，还没迁移到新的 bucket 来。新 bucket 只是最大数量变为原来最大数量的 2 倍(2^B * 2) 。  

2. overflow 的 bucket 数量过多  

    在装载因子比较小的情况下，这时候 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，
即 map 里元素总数少，但是 bucket 数量多（真实分配的 bucket 数量多，包括大量的 overflow bucket）。  

    不难想像造成这种情况的原因：不停地插入、删除元素。先插入很多元素，导致创建了很多 bucket，但是装载因子达不到第 1 点的临界值，未触发扩容来缓解这种情况。
之后，删除元素降低元素总数量，再插入很多元素，导致创建很多的 overflow bucket，但就是不会触发第 1 点的规定，你能拿我怎么办？
overflow bucket 数量太多，导致 key 会很分散，查找插入效率低得吓人，因此出台第 2 点规定。

    这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难。

    对于条件 2，其实元素没那么多，但是 overflow bucket 数特别多，说明很多 bucket 都没装满。解决办法就是开辟一个新 bucket 空间，
将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。这样，原来，在 overflow bucket 中的 key 可以移动到 bucket 中来。
结果是节省空间，提高 bucket 利用率，map 的查找和插入效率自然就会提升。

### 扩容函数
```go
func hashGrow(t *maptype, h *hmap) {
    // If we've hit the load factor, get bigger.
    // Otherwise, there are too many overflow buckets,
    // so keep the same number of buckets and "grow" laterally.
    bigger := uint8(1)
    if !overLoadFactor(h.count+1, h.B) {
        bigger = 0
        h.flags |= sameSizeGrow
    }
    oldbuckets := h.buckets
    newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

    flags := h.flags &^ (iterator | oldIterator)
    if h.flags&iterator != 0 {
        flags |= oldIterator
    }
    // commit the grow (atomic wrt gc)
    h.B += bigger
    h.flags = flags
    h.oldbuckets = oldbuckets
    h.buckets = newbuckets
    h.nevacuate = 0
    h.noverflow = 0

    if h.extra != nil && h.extra.overflow != nil {
        // Promote current overflow buckets to the old generation.
        if h.extra.oldoverflow != nil {
            throw("oldoverflow is not nil")
        }
        h.extra.oldoverflow = h.extra.overflow
        h.extra.overflow = nil
    }
    if nextOverflow != nil {
        if h.extra == nil {
            h.extra = new(mapextra)
        }
        h.extra.nextOverflow = nextOverflow
    }

    // the actual copying of the hash table data is done incrementally
    // by growWork() and evacuate().
}
```

由于 map 扩容需要将原有的 key/value 重新搬迁到新的内存地址，如果有大量的 key/value 需要搬迁，会非常影响性能。因此 Go map 的扩容采取了一种称为“渐进式”的方式，原有的 key 并不会一次性搬迁完毕，每次最多只会搬迁 2 个 bucket。

上面说的 hashGrow() 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。真正搬迁 buckets 的动作在 growWork() 函数中，而调用 growWork() 函数的动作是在 mapassign 和 mapdelete 函数中。也就是插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。先检查 oldbuckets 是否搬迁完毕，具体来说就是检查 oldbuckets 是否为 nil。


## 迁移

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
    // make sure we evacuate the oldbucket corresponding
    // to the bucket we're about to use
    evacuate(t, h, bucket&h.oldbucketmask())

    // evacuate one more oldbucket to make progress on growing
    if h.growing() {
        evacuate(t, h, h.nevacuate)
    }
}
```

### 迁移条件

如果未迁移完毕，赋值/删除的时候，扩容完毕后（预分配内存），不会马上就进行迁移。而是采取增量扩容的方式，当有访问到具体 bukcet 时，才会逐渐的进行迁移（将 oldbucket 迁移到 bucket）

### 迁移函数

hmap.nevacuate 标识的是当前的进度，如果都搬迁完，应该和2^B的长度是一样的
在evacuate 方法实现是把这个位置对应的 bucket，以及其冲突链上的数据都转移到新的 bucket 上。

1. 先要判断当前 bucket 是不是已经转移。 (oldbucket 标识需要搬迁的 bucket 对应的位置)。转移的判断直接通过tophash 就可以，判断 tophash 中第一个 hash 值即可
   ```go
   var (
       emptyOne       = 1 // this cell is empty
       minTopHash     = 5 // minimum tophash for a normal filled cell.
   )
   
   
   func evacuated(b *bmap) bool {
       h := b.tophash[0]
       return h > emptyOne && h < minTopHash
   }
   ```
2. 如果没有被转移，那就要迁移数据了。数据迁移时，可能是迁移到大小相同的 buckets 上，也可能迁移到2倍大的 buckets 上。这里 xy 都是标记目标迁移位置的标记：x 标识的是迁移到相同的位置，y 标识的是迁移到2倍大的位置上。我们先看下目标位置的确定：
   
   ```go
   var xy [2]evacDst
   x := &xy[0]
   x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
   x.k = add(unsafe.Pointer(x.b), dataOffset)
   x.e = add(x.k, bucketCnt*uintptr(t.keysize))

   if !h.sameSizeGrow() {
       // Only calculate y pointers if we're growing bigger.
       // Otherwise GC can see bad pointers.
       y := &xy[1]
       y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
       y.k = add(unsafe.Pointer(y.b), dataOffset)
       y.e = add(y.k, bucketCnt*uintptr(t.keysize))
   }
   ```

3. 确定bucket位置后，需要按照kv 一条一条做迁移。

4. 如果当前搬迁的bucket 和 总体搬迁的bucket的位置是一样的，我们需要更新总体进度的标记 nevacuate
   ```go
   func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
       h.nevacuate++
       // Experiments suggest that 1024 is overkill by at least an order of magnitude.
       // Put it in there as a safeguard anyway, to ensure O(1) behavior.
       stop := h.nevacuate + 1024
       if stop > newbit {
           stop = newbit
       }
       for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
           h.nevacuate++
       }
       if h.nevacuate == newbit { // newbit == # of oldbuckets
           // Growing is all done. Free old main bucket array.
           h.oldbuckets = nil
           // Can discard old overflow buckets as well.
           // If they are still referenced by an iterator,
           // then the iterator holds a pointers to the slice.
           if h.extra != nil {
               h.extra.oldoverflow = nil
           }
           h.flags &^= sameSizeGrow
       }
   }
   ```

## 总结

- map是引用类型
- map遍历是无序的
- map是非线程安全的
- map的哈希冲突解决方式是链表法
- map的扩容不是一定会新增空间，也有可能是只是做了内存整理
- map的迁移是逐步进行的，在每次赋值时，会做至少一次迁移工作
- map中删除key，有可能导致出现很多空的kv，这会导致迁移操作，如果可以避免，尽量避免