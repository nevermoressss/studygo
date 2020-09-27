#### map的实现原理：hash
```text
hash表 O(1) 的读写性能非常优秀，提供了键值之间的映射。
hash 的性能好不好主要看2点 ：哈希函数和冲突解决方法
go中利用拉链法实现哈希表
装载因子：=元素数量 / 桶数量
```
hmap : runtime/map.hmap
```go
// A header for a Go map.
type hmap struct {
	// 当前的数量
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	// 当前bucket的数量 2^B
	B         uint8 
	// 溢出桶的大概数量
	noverflow uint16
	// hash 因子 range 的时候map会有随机性
	hash0     uint32 // hash seed
    // 当前的bucket
	buckets    unsafe.Pointer
	// 用于扩容的时候保存old bucket
	oldbuckets unsafe.Pointer
	// 迁移计数器
	nevacuate  uintptr 
    // 可选字段 存放溢出桶....
	extra *mapextra 
}
PS：buckets里的正常桶和extra里的溢出桶内存是连续的

```
新建一个map  
makemap ：runtime/makemap
```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
	// 计算make 需要的内存判断是否溢出或超出上限
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	// 调用fastrand 方法获取随机因子,也就是说map的随机性在make阶段就确定了
	h.hash0 = fastrand()
	
	// 计算B 
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B
	// B大于0 需要分配内存
	if h.B != 0 {
		var nextOverflow *bmap
		// 创建buckets
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```
makeBucketArray  
正常的bucket和溢出的bucket是一块连续的内存
```go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	// 计算桶的数量 2^B
	base := bucketShift(b)
	nbuckets := base
	// 对于B大于4的需要额外的溢出桶
	if b >= 4 {
		// 估算的溢出桶数量 等于 2^(B-4)
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		// 内存块的大小
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}
    // 如果dirtyalloc为nil 则需要重新分配
    // 否则可以复用一下
	if dirtyalloc == nil {
		// 申请内存
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		// 根据有没有指针来做不同的清楚工作
		if t.bucket.ptrdata != 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}
    // 含有溢出桶
	if base != nbuckets {
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```
map 的读取  
对于指定key的读取 在编译阶段会变成2个不同的函数
```go
v     := hash[key] // => v     := *mapaccess1(maptype, hash, &key)
v, ok := hash[key] // => v, ok := mapaccess2(maptype, hash, &key)
```
mapaccess1：runtime/mapaccess1
```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 开启了竞态检测
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	 //是否开启了msan，探测是否读未初始化的内存
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	// 如果map为空或者map没有数
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}
	// 如果读的时候map有在写入
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	//获取hash值
	hash := t.hasher(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	//拿到对应的桶
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	// 如果存在old buckets 就是触发了扩容 并且还没迁移完
	if c := h.oldbuckets; c != nil {
		// 判断新旧桶大小
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		// 要的数据是否在旧桶里
		if !evacuated(oldb) {
			b = oldb
		}
	}
	// 计算tophash（前8位）
	top := tophash(hash)
	// 遍历正常桶和对应bucket的溢出桶 tophash都不同就直接跳过
	// tophash相同就判断对应的key是否相同
	// 相同就可以返回了
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if t.key.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}
```
mapaccess2：runtime/mapaccess2  
和mapaccess1 的差別就是返回的時候增加了一个是否存在
```go
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if t.key.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
// ---------------------!!!!!! 区别在这里！！！！！--------------------//
				return e, true
			}
		}
	}
// ---------------------!!!!!! 区别在这里！！！！！--------------------//
	return unsafe.Pointer(&zeroVal[0]), false
```
写入
mapassign：runtime/mapassign
```go
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 写入空的map会触发panic
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	// 是否开启了竞态检测
	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	//是否开启了msan，探测是否读未初始化的内存
	if msanenabled {
		msanread(key, t.key.size)
	}
	// 并发写入
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	// 计算hash值
	hash := t.hasher(key, uintptr(h.hash0))

	// 把hashWriting 标记打上去flags
	h.flags ^= hashWriting
    // 如果buckets是空的，新建一个吧
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
	// 计算bucket位置
	bucket := hash & bucketMask(h.B)
	// 在迁移的过程
	if h.growing() {
		// 迁移我们需要的桶并且额外迁移一个桶
		growWork(t, h, bucket)
	}
	// 写入桶
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
	// 和读取同理，遍历桶的buckets 找到就覆盖，没找到追加
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if !t.key.equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			goto done
		}
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}
    // 如果达到最大负载因子或溢出桶过多，并且没有在迁移中，那就触发迁移
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		// 在去读一遍（迁移2个桶）
		goto again // Growing the table invalidates everything, so try again
	}

	if inserti == nil {
		// 对应bucket的桶都满了 分配个溢出桶
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}
	// 在插入位置存新的减值
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectelem() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

done:
	//在for循环里找到了会跳到这里直接设置key值
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	// 把hashWriting的标记去除
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
```
需要扩容的时候的操作
hashGrow：runtime/hashGrow
```go
func hashGrow(t *maptype, h *hmap) {
	// 默认扩容达到了bigger:1 翻倍
	bigger := uint8(1)
	// 如果是溢出桶过多
	if !overLoadFactor(h.count+1, h.B) {
		// 不扩容，减少溢出桶
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)
    // 置flags
	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// 赋值
	// commit the grow (atomic wrt gc)
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0
    // 移动overflow
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
删除
mapdelete：runtime/mapdelete
```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	// 是否开启了竞态检测
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapdelete)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	//是否开启了msan，探测是否读未初始化的内存
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return
	}
	// 是否有在写入
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
    // 计算hash
	hash := t.hasher(key, uintptr(h.hash0))
	// hashWriting 标记打上去
	h.flags ^= hashWriting
    // 计算bucket
	bucket := hash & bucketMask(h.B)
	// 是否在扩缩容
	if h.growing() {
		// 移动
		growWork(t, h, bucket)
	}
	// 对应的bmap
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b
	top := tophash(hash)
	// 去找 和 查找写入差不多的逻辑
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !t.key.equal(key, k2) {
				continue
			}
			// 找到了对应的key做清除操作
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.ptrdata != 0 {
				memclrHasPointers(k, t.key.size)
			}
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			if t.indirectelem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.elem.ptrdata != 0 {
				memclrHasPointers(e, t.elem.size)
			} else {
				memclrNoHeapPointers(e, t.elem.size)
			}
			b.tophash[i] = emptyOne
			if i == bucketCnt-1 {
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			for {
				b.tophash[i] = emptyRest
				if i == 0 {
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			break search
		}
	}
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}
```