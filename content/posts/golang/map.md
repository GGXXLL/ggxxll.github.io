---
title:  Golang Map
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
   count     int    // map中的元素个数， golang 中调用 len(map) 的时候直接返回该字段
   flags     uint8  // 状态标记位，通过与定义的枚举值进行&操作可以判断当前是否处于这种状态
   B         uint8  // 2^B 表示 buckets 的数量
   noverflow uint16 // overflow bucket 的数量的近似数，当其 B>=16 时为近似值
   hash0     uint32 // 是哈希的种子。在创建map时 fastrand 函数生成，并在调用哈希函数时作为参数传入

   buckets    unsafe.Pointer // 指向buckets数组的指针，数组大小为2^B，如果元素个数为0，它为nil。
   oldbuckets unsafe.Pointer // 如果发生扩容，oldbuckets 是指向老的buckets数组的指针，老的 buckets 数组大小是新的 buckets 的1/2。非扩容状态下，它为nil
   nevacuate  uintptr        // 当桶进行调整时指示的搬迁进度，地址小于当前指针的 bucket 已经迁移完成

   extra *mapextra // 这个字段是为了优化GC扫描而设计的
}

// A bucket for a Go map.
type bmap struct {
    tophash [bucketCnt]uint8
}
```
![](img/hmap-and-buckets.webp)

如上图所示哈希表 `runtime.hmap` 的桶是 `runtime.bmap`。每一个 `runtime.bmap` 都能存储 8 个键值对，当哈希表中存储的数据过多，单个桶已经装满时就会使用 `extra.nextOverflow` 中桶存储溢出的数据。

上述两种不同的桶在内存中是连续存储的，我们在这里将它们分别称为正常桶和溢出桶，上图中黄色的 `runtime.bmap` 就是正常桶，绿色的 `runtime.bmap` 是溢出桶，溢出桶是在 Go 语言还使用 C 语言实现时使用的设计，由于它能够减少扩容的频率所以一直使用至今。

桶的结构体 `runtime.bmap` 在 Go 语言源代码中的定义只包含一个简单的 `tophash` 字段，`tophash` 存储了键的哈希的高 8 位，通过比较不同键的哈希的高 8 位可以减少访问键值对次数以提高性能：

在运行期间，`runtime.bmap` 结构体其实不止包含 `tophash` 字段，因为哈希表中可能存储不同类型的键值对，而且 Go 语言也不支持泛型，所以键值对占据的内存空间大小只能在编译时进行推导。`runtime.bmap` 中的其他字段在运行时也都是通过计算内存地址的方式访问的，所以它的定义中就不包含这些字段，不过我们能根据编译期间的 `cmd/compile/internal/gc.bmap` 函数重建它的结构：

```go
type bmap struct { 
    topbits  [8]uint8 // 键的哈希值的高 8 位
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr        // 内存对齐使用（新版已移除）
    overflow uintptr        // 存放了所指向的溢出桶的地址。当 bucket 的8个key 存满了之后，该字段指向溢出桶
}
```

### hmap
map 的底层结构是 hmap，hmap 包含若干个结构为 bmap 的 bucket 数组，每个 bucket 底层都采用链表结构。

### bmap
bmap 就是我们常说的“桶”的底层数据结构， 一个桶中可以存放最多 8 个 key/value.

bmap 中 key 和 value 是各自放在一起的 `bmap.keys` 和 `bmap.values`，并不是 `key/value/key/value/...` 这样的形式。源码里说明这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间。

> 使用 `key/value/key/value/...` 结构，如果存储的键和值的类型不同，在内存中布局中所占字节不同的话，就需要对齐。比如说存储一个 `map[int64]int8` 类型的字典。

### mapextra

```go
// mapextra holds fields that are not present on all maps.
type mapextra struct {
    // 如果 key 和 value 都不包含指针，并且可以被 inline(<=128 字节)
    // 就使用 hmap 的 extra 字段 来存储 overflow buckets，这样可以避免 GC 扫描整个 map
    // 然而 bmap.overflow 也是个指针，这时候我们只能把这些 overflow 的指针
    // 都放在 hmap.extra.overflow 和 hmap.extra.oldoverflow 中了
    // overflow 包含的是 hmap.buckets 的 overflow 的 buckets
    // oldoverflow 包含扩容时的 hmap.oldbuckets 的 overflow 的 bucket
    overflow    *[]*bmap
    oldoverflow *[]*bmap
    
    // 指向空闲的 overflow bucket 的指针
    nextOverflow *bmap
}

```

### 操作

#### 查找 Key

使用创建 map 时生成的 hash 种子 `hash0`，调用 hash 函数得到 key 的 哈希值。 然后使用哈希值的后 B 位确定 桶序号，再使用前 8 位确定 key 在桶中的位置。

注意：对于高低位的选择，该操作的实质是取余，但是取余开销很大，在实际代码实现中采用的是位操作。以下是tophash的实现代码：
```go
func tophash(hash uintptr) uint8 {
    top := uint8(hash >> (sys.PtrSize*8 - 8))
    if top < minTopHash {
        top += minTopHash
    }
    return top
}
```

源码实现：
```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // 如果开启了竞态检测 -race
    if raceenabled && h != nil {
        callerpc := getcallerpc()
        pc := funcPC(mapaccess1)
        racereadpc(unsafe.Pointer(h), callerpc, pc)
        raceReadObjectPC(t.key, key, callerpc, pc)
    }
    // 如果开启了memory sanitizer -msan
    if msanenabled && h != nil {
        msanread(key, t.key.size)
    }
    // 如果map为空或者元素个数为0，返回零值
    if h == nil || h.count == 0 {
        if t.hashMightPanic() {
            t.hasher(key, 0) // see issue 23734
        }
        return unsafe.Pointer(&zeroVal[0])
    }
    // 注意，这里是按位与操作
    // 当h.flags对应的值为hashWriting（代表有其他goroutine正在往map中写key）时，那么位计算的结果不为0，因此抛出以下错误。
    // 这也表明，go的map是非并发安全的
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
    // 不同类型的key，会使用不同的hash算法，可详见src/runtime/alg.go中typehash函数中的逻辑
    hash := t.hasher(key, uintptr(h.hash0))
    m := bucketMask(h.B)
    // 按位与操作，找到对应的bucket
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
    // 如果oldbuckets不为空，那么证明map发生了扩容
    // 如果有扩容发生，老的buckets中的数据可能还未搬迁至新的buckets里
    // 所以需要先在老的buckets中找
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() {
            m >>= 1
        }
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        // 如果在oldbuckets中tophash[0]的值，为evacuatedX、evacuatedY，evacuatedEmpty其中之一
        // 则evacuated()返回为true，代表搬迁完成。
        // 因此，只有当搬迁未完成时，才会从此oldbucket中遍历
        if !evacuated(oldb) {
            b = oldb
        }
    }
    // 取出当前key值的tophash值
    top := tophash(hash)
    // 以下是查找的核心逻辑
    // 双重循环遍历：外层循环是从桶到溢出桶遍历；内层是桶中的cell遍历
    // 跳出循环的条件有三种：第一种是已经找到key值；第二种是当前桶再无溢出桶；
    // 第三种是当前桶中有cell位的tophash值是emptyRest，这个值在前面解释过，它代表此时的桶后面的cell还未利用，所以无需再继续遍历。
bucketloop:
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
            // 判断tophash值是否相等
            if b.tophash[i] != top {
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
            }
            // 因为在bucket中key是用连续的存储空间存储的，因此可以通过bucket地址+数据偏移量（bmap结构体的大小）+ keysize的大小，得到k的地址
            // 同理，value的地址也是相似的计算方法，只是再要加上8个keysize的内存地址
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey() {
                k = *((*unsafe.Pointer)(k))
            }
            // 判断key是否相等
            if t.key.equal(key, k) {
                e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                if t.indirectelem() {
                    e = *((*unsafe.Pointer)(e))
                }
                return e
            }
        }
    }
    // 所有的bucket都未找到，则返回零值
    return unsafe.Pointer(&zeroVal[0])
}
```

整个查找过程优先在 oldbuckets 里面找(如果存在 oldbuckets 的话)，找完再去新 bmap 里面找。

有人可能会有疑问，为何这里要加入 tophash 多一次比较呢？

tophash 的引入是为了加速查找的。由于它只存了 hash 值的高 8 位，比查找完整的 64 位要快很多。通过比较高 8 位，迅速找到高 8 位一致 hash 值的索引，接下来再进行一次完整的比较，如果还一致，那么就判定找到该 key 了。

如果找到了 key 就返回对应的 value。如果没有找到，还会继续去 overflow 桶继续寻找，直到找到最后一个桶，如果还没有找到就返回对应类型的零值。

#### 插入 Key

插入 key 的过程和查找 key 的过程大体一致。

有几点不同，需要注意：

1. 如果找到要插入的 key ，只需要直接更新对应的 value 值就好了。
2. 如果没有在 bmap 中没有找到待插入的 key ，这么这时分几种情况。
- bmap 中还有空位，在遍历 bmap 的时候预先标记空位，一旦查找结束也没有找到 key，就把 key 放到预先遍历时候标记的空位上。
- bmap中已经没有空位了。这个时候 bmap 装的很满了。此时需要检查一次最大负载因子是否已经达到了。如果达到了，立即进行扩容操作。扩容以后在新桶里面插入 key，流程和上述的一致。如果没有达到最大负载因子，那么就在新生成一个 bmap，并把前一个 bmap 的 overflow 指针指向新的 bmap。
3. 在扩容过程中，oldbucket 是被冻结的，查找 key 时会在oldbucket 中查找，但不会在 oldbucket 中插入数据。如果在 oldbucket 是找到了相应的 key，做法是将它迁移到新 bmap 后加入 evalucated 标记。

#### 删除 Key

删除操作主要流程和查找 key 流程也差不多，找到对应的 key 以后，如果是指针指向原来的 key，就把指针置为 nil，如果是值就清空它所在的内存。还要清理 tophash 里面的值，最后把 hmap 的 count 减 1。

如果在扩容过程中，删除操作会在扩容以后在新的 bmap 里面执行删除。

查找的过程依旧会一直遍历到链表的最后一个 bmap 桶。

## 特性

### 引用类型

map是个指针，底层指向hmap，所以是个引用类型。

golang 有三个常用的高级类型 slice、map、channel，它们都是引用类型，当引用类型作为函数参数时，可能会修改原内容数据。

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

golang中map是一个 kv 对集合。底层使用 hash table，用链表来解决冲突 ，出现冲突时，不是每一个 key 都申请一个结构通过链表串起来，
而是以 bmap 为最小粒度挂载，一个 bmap 可以放 8 个 kv。

在哈希函数的选择上，会在程序启动时，检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。

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
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
    // If the threshold is too low, we do extraneous work.
    // If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
    // "too many" means (approximately) as many overflow buckets as regular buckets.
    // See incrnoverflow for more details.
    if B > 15 {
        B = 15
    }
    // The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
    return noverflow >= uint16(1)<<(B&15)
}
```

1. 装载因子超过阈值  

    源码里定义的阈值是 6.5 (`loadFactorNum/loadFactorDen`)，是经过测试后取出的一个比较合理的因子。
我们知道，每个 bucket 有 8 个空位，在没有溢出，且所有的桶都装满了的情况下，装载因子算出来的结果是 8。因此当装载因子超过 6.5 时，
表明很多 bucket 都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的。  

    对于条件 1，元素太多，而 bucket 数量太少，很简单：将 B 加 1，bucket 最大数量(2^B)直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了。  

    注意，这时候元素都在老 bucket 里，还没迁移到新的 bucket 来。新 bucket 只是最大数量变为原来最大数量的 2 倍(2^B * 2) 。  

2. overflow 的 bucket 数量过多  
    在装载因子比较小的情况下，这时候 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，
即 map 里元素总数少，但是 bucket 数量多（真实分配的 bucket 数量多，包括大量的 overflow bucket）。  

    不难想像造成这种情况的原因：不停地插入、删除元素。先插入很多元素，导致创建了很多 bucket，但是装载因子达不到第 1 点的临界值，未触发扩容来缓解这种情况。之后，删除元素降低元素总数量，再插入很多元素，导致创建很多的 overflow bucket，但就是不会触发第 1 点的规定，你能拿我怎么办？

    overflow bucket 数量太多，导致 key 会很分散，查找插入效率低得吓人，因此出台第 2 点规定。

    这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难。

    解决办法就是开辟一个新 bucket 空间，
    将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。这样，原来，在 overflow bucket 中的 key 可以移动到 bucket 中来。节省了空间，提高 bucket 利用率，map 的查找和插入效率自然就会提升。

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

上面说的 `hashGrow()` 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。真正搬迁 buckets 的动作在 `growWork()` 函数中，而调用 `growWork()` 函数的动作是在 `mapassign` 和 `mapdelete` 函数中。也就是插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。先检查 oldbuckets 是否搬迁完毕，具体来说就是检查 oldbuckets 是否为 nil。


## 迁移

`hashGrow` 操作算是扩容之前的准备工作，实际拷贝的过程在 `evacuate` 中。

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

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
    b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
    // 在准备扩容之前桶的个数
    newbit := h.noldbuckets()
    alg := t.key.alg
    if !evacuated(b) {
        // TODO: reuse overflow buckets instead of using new ones, if there
        // is no iterator using the old buckets.  (If !oldIterator.)

        var (
            x, y   *bmap          // 在新桶里面 低位桶和高位桶
            xi, yi int            // key 和 value 值的索引值分别为 xi ， yi 
            xk, yk unsafe.Pointer // 指向 x 和 y 的 key 值的指针 
            xv, yv unsafe.Pointer // 指向 x 和 y 的 value 值的指针  
        )
        // 新桶中低位的一些桶
        x = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
        xi = 0
        // 扩容以后的新桶中低位的第一个 key 值
        xk = add(unsafe.Pointer(x), dataOffset)
        // 扩容以后的新桶中低位的第一个 key 值对应的 value 值
        xv = add(xk, bucketCnt*uintptr(t.keysize))
        // 如果不是等量扩容
        if !h.sameSizeGrow() {
            y = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
            yi = 0
            yk = add(unsafe.Pointer(y), dataOffset)
            yv = add(yk, bucketCnt*uintptr(t.keysize))
        }
        // 依次遍历溢出桶
        for ; b != nil; b = b.overflow(t) {
            k := add(unsafe.Pointer(b), dataOffset)
            v := add(k, bucketCnt*uintptr(t.keysize))
            // 遍历 key - value 键值对
            for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
                top := b.tophash[i]
                if top == empty {
                    b.tophash[i] = evacuatedEmpty
                    continue
                }
                if top < minTopHash {
                    throw("bad map state")
                }
                k2 := k
                // key 值如果是指针，则取出指针里面的值
                if t.indirectkey {
                    k2 = *((*unsafe.Pointer)(k2))
                }
                useX := true
                if !h.sameSizeGrow() {
                    // 如果不是等量扩容，则需要重新计算 hash 值，不管是高位桶 x 中，还是低位桶 y 中
                    hash := alg.hash(k2, uintptr(h.hash0))
                    if h.flags&iterator != 0 {
                        if !t.reflexivekey && !alg.equal(k2, k2) {
                            // 如果两个 key 不相等，那么他们俩极大可能旧的 hash 值也不相等。
                            // tophash 对要迁移的 key 值也是没有多大意义的，所以我们用低位的 tophash 辅助扩容，标记一些状态。
                            // 为下一个级 level 重新计算一些新的随机的 hash 值。以至于这些 key 值在多次扩容以后依旧可以均匀分布在所有桶中
                            // 判断 top 的最低位是否为1
                            if top&1 != 0 {
                                hash |= newbit
                            } else {
                                hash &^= newbit
                            }
                            top = uint8(hash >> (sys.PtrSize*8 - 8))
                            if top < minTopHash {
                                top += minTopHash
                            }
                        }
                    }
                    useX = hash&newbit == 0
                }
                if useX {
                    // 标记低位桶存在 tophash 中
                    b.tophash[i] = evacuatedX
                    // 如果 key 的索引值到了桶最后一个，就新建一个 overflow
                    if xi == bucketCnt {
                        newx := h.newoverflow(t, x)
                        x = newx
                        xi = 0
                        xk = add(unsafe.Pointer(x), dataOffset)
                        xv = add(xk, bucketCnt*uintptr(t.keysize))
                    }
                    // 把 hash 的高8位再次存在 tophash 中
                    x.tophash[xi] = top
                    if t.indirectkey {
                        // 如果是指针指向 key ，那么拷贝指针指向
                        *(*unsafe.Pointer)(xk) = k2 // copy pointer
                    } else {
                        // 如果是指针指向 key ，那么进行值拷贝
                        typedmemmove(t.key, xk, k) // copy value
                    }
                    // 同理拷贝 value
                    if t.indirectvalue {
                        *(*unsafe.Pointer)(xv) = *(*unsafe.Pointer)(v)
                    } else {
                        typedmemmove(t.elem, xv, v)
                    }
                    // 继续迁移下一个
                    xi++
                    xk = add(xk, uintptr(t.keysize))
                    xv = add(xv, uintptr(t.valuesize))
                } else {
                    // 这里是高位桶 y，迁移过程和上述低位桶 x 一致，下面就不再赘述了
                    b.tophash[i] = evacuatedY
                    if yi == bucketCnt {
                        newy := h.newoverflow(t, y)
                        y = newy
                        yi = 0
                        yk = add(unsafe.Pointer(y), dataOffset)
                        yv = add(yk, bucketCnt*uintptr(t.keysize))
                    }
                    y.tophash[yi] = top
                    if t.indirectkey {
                        *(*unsafe.Pointer)(yk) = k2
                    } else {
                        typedmemmove(t.key, yk, k)
                    }
                    if t.indirectvalue {
                        *(*unsafe.Pointer)(yv) = *(*unsafe.Pointer)(v)
                    } else {
                        typedmemmove(t.elem, yv, v)
                    }
                    yi++
                    yk = add(yk, uintptr(t.keysize))
                    yv = add(yv, uintptr(t.valuesize))
                }
            }
        }
        // Unlink the overflow buckets & clear key/value to help GC.
        if h.flags&oldIterator == 0 {
            b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
            // Preserve b.tophash because the evacuation
            // state is maintained there.
            if t.bucket.kind&kindNoPointers == 0 {
                memclrHasPointers(add(unsafe.Pointer(b), dataOffset), uintptr(t.bucketsize)-dataOffset)
            } else {
                memclrNoHeapPointers(add(unsafe.Pointer(b), dataOffset), uintptr(t.bucketsize)-dataOffset)
            }
        }
    }

    // Advance evacuation mark
    if oldbucket == h.nevacuate {
        h.nevacuate = oldbucket + 1
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
                h.extra.overflow[1] = nil
            }
            h.flags &^= sameSizeGrow
        }
    }
}
```

如果未迁移完毕，赋值/删除的时候，扩容完毕后（预分配内存），不会马上就进行迁移。而是采取增量扩容的方式，当有访问到具体 bukcet 时，才会逐渐的进行迁移（将 oldbucket 迁移到 bucket）

### 迁移函数

`hmap.nevacuate` 标识的是当前的进度，如果都搬迁完，应该和 2^B 的长度是一样的
在 evacuate 方法实现是把这个位置对应的 bucket，以及其冲突链上的数据都转移到新的 bucket 上。

1. 先要判断当前 bucket 是不是已经转移。 (oldbucket 标识需要搬迁的 bucket 对应的位置)。转移的判断直接通过 tophash 就可以，判断 tophash 中第一个 hash 值即可
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

4. 如果当前搬迁的 bucket 和 总体搬迁的 bucket 的位置是一样的，我们需要更新总体进度的标记 nevacuate
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
- map的迁移是逐步进行的，在每次赋值时，会做至少一次迁移工作
- map中删除 key，有可能导致出现很多空的 kv，这会导致迁移操作，如果可以避免，尽量避免
- 扩容分为增量扩容和等量扩容。增量扩容，会增加桶的个数（增加一倍），把原来一个桶中的 keys 被重新分配到两个桶中。等量扩容，不会更改桶的个数，只是会将桶中的数据变得紧凑。不管是增量扩容还是等量扩容，都需要创建新的桶数组，并不是原地操作的。
- map中定义了2的B次方个桶，每个桶中能够容纳8个key。根据key的不同哈希值，将其散落到不同的桶中。哈希值的低位（哈希值的后B个bit位）决定桶序号，高位（哈希值的前8个bit位）标识同一个桶中的不同 key。

### 参考

- https://www.cnblogs.com/cnblogs-wangzhipeng/p/13292524.html
- https://halfrost.com/go_map_chapter_one/
- https://zhuanlan.zhihu.com/p/273666774