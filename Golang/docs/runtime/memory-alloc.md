## Go内存分配机制



### 数据模型


#### mspan

```Go
type gcBits uint8

type mspan struct {
    next *mspan
    prev *mspan
    list *mspanList
    startAddr uintptr
    npages uintptr
    //介于0和nelems之间的值，标识下次分配object时扫描allocBits下标的开始位置；如果freeindex==nelems, 表示当前span没有可分配的object
    //每次分配成功，会修改该值
    freeindex uintptr 
    //allocBits从freeindex处开始的缓存，它的最低位与freeindex所在的位置对齐; 
    //并且，`0`表示已分配，`1`表示未分配，与allocBits相反；比如，如果8位，已分配两个，则allocCache当前位`00111111`
    //如果再分配一个，变为`00011111`
    allocCache uint64 
    nelems uintptr // 当前span中的object数量
    allocBits *gcBits //uint8类型的数组，数组中的每个值表示多个位置的object的分配情况
    gcmarkBits *gcBits
    elemsize uintptr //span中可分配的object的size
    allocCount uint16 //span中已分配的object的个数
} 
```

#### mcentral

```Go
type mcentral struct {
    lock mutex //锁，由于mcentral对象是所有goroutine共享的，分配必须加锁
    spanclass spanClass //此mcentral对应的class对象，每个class对应一个分配的object的大小
    nonempty mSpanList  //还有可分配空间的span列表
    empty mSpanList //没有可分配空间的span列表
    nmalloc uint64 // 从次mcentral分配出去的object的累计个数，假设所有已被mcache缓存的span中所有的object都已分配
}
```


### 流程

#### 对象分配逻辑

```Go
func mallocgc(type, size) pointer {
    m := aquirem // 获得M
    m.mallocing=1 //lock
    c := gomcache() // 获取当前goroutine的cache
    noscan := type == nil || (type.kind != pointerKind)

    if size <= maxSmallSize {
        if size <= maxTinySize && noscan {
            //tiny object 分配
            if c.tinyBlock != nil{
                //直接在cache的tiny block上分配
                m.mallocing = 0 //unlock
                releasem(m) //释放M
                return
            }
            span := c.alloc[tinySpanClass] //获取cache 上tiny class的头span指针
            v := nextFreeFast(span)// 快速查询，获取可以分配的object的位置
            if v == 0 {
                //快读查询失败，常规分配
                v = c.nextFree(tinySpanClass)
            }
            x := unsafe.Pointer(v)
        }else {
            //正常 small object 分配
            sizeClass := size_to_class[size]//根据object的size获取class
            size = class_to_size[sizeClass] //获取真实要分配的memory的大小，大于等于请求大小
            span := c.alloc[sizeClass, noscan] //根据size class 和noscan参数获取cache中对应的可分配span的指针
            v := nextFreeFast(span)// 在spanClass对应的span列表的第一个span上快速找到可分配的object的位置
            if v == 0 {
                v = nextFree(span)// 快速分配失败，常规分配到可用的object位置
            }
            x := unsafe.Pointer(v)
        }
    } else {
        //大对象分配
        x := largeAlloc(size, needzero, noscan) //大对象直接在heap里分配
    }

    m.mallocing = 0 //unlock
    releasem(m) //释放M
    //handle memory profile ...

    return x
}

```

#### nextFreeFast 逻辑

在当前allocCache中分配

```Go
//return the next free object if one is quickly available
//otherwise, return 0
func nextFreeFast(*span) gclinkptr {
    theBit := Ctz64(span.allocCache) //从地位往高位查找allocCache中从低位开始0的个数, 如果allocCache中仍然有obejct可以分配，此方法一定返回0
    if theBit<64 {
        result := span.freeindex + theBit //预计可分配的object的位置
        if result < span.nelems {
            newFreeindex = result + 1 
            if newFreeindex % 64 == 0 && newFreeindex != span.nelems{
                //如果newFreeindex是64的倍数，则只可能只剩最后一个object可以分配，否则分配失败
                return 0
            }
            span.allocCache >>= uint(theBit+1)//更新allocCache,如果分配成功，等同于右移一位
            span.freeindex = newFreeindex
            span.allocCount++
            return gclinkptr(result*span.elemsize+span.base()) //返回分配的object的指针
        }
        //位置超出span中object的个数，快速分配失败
    }

    return 0 //快速分配失败
}

```

#### mcache.nextFree 逻辑

nextFreeFast分配失败，即当前span的allocCache上分配失败时调用此方法，此方法在分配时会根据分配情况重现填充可分配span的allocCache，甚至申请新的span，此方法理论上一定会成功

```Go
func (*mcache) nextFree(spanClass) (v gclinkptr, resultSpan *mspan, shouldhelpgc bool) {
    shouldhelpgc = false
    s := mcache.alloc[spanClass] //获取mcache中spanClass对应的可分配的span
    freeindex := s.nextFreeIndex() //在当前span中找到可分配的object的位置
    if freeindex == s.nelems{
        //当前span已满
        mcache.refill()//重新申请新的span
        shouldhelpgc = true //标记当前分配是一次重量级分配
        s := mcache.alloc[spanClass] //重新获取可以分配的span
        freeindex := s.nextFreeIndex() //获取新的span上可以分配的object的位置
    }
    v = gclinkptr(freeindex*s.elemsize+s.base()) //获取分配的object的指针
    return
}
```

#### mcache.nextFreeIndex()逻辑

从span.freeindex查询下一个可分配的object的位置，并返回；如果返回结果==span.nelems，表示当前span已经没有可分配的空间

```Go
func (s *mspan) nextFreeIndex() uintptr {
    if s.freeindex ==  s.nelems {
        return s.freeindex //当前span已满，没有可分配的空间
    }

    bitIndex := Ctz64(s.allocCache) //返回从低位开始连续0的个数
    for bitIndex == 64 {
        //当前allocCache全部已分配，调整s.freeindex 和s.allocCache,直到找到一个有空闲空间的，或者return s.nelems，表示当前span已满
        s.freeindex = （s.freeindex+64）&^(64-1) //move freeindex
        if s.freeindex >= s.nelems {
            return s.nelems //当前span已满
        }
        s.refillAllocCache() //重新填充s.alloCache，基本逻辑是从allocBits中取出从freeindex对应的字节开始的后面8个字节的值，并取反^
        bitIndex = Ctz64(s.allocCache) 
    }
    result := s.freeindex + uintptr(bitIndex) //计算可分配的object在s.allocBits中的位置
    if result >= s.nelems {
        return s.nelems //当前span已满
    }
    s.allocCache >>= uint(bitIndex+1) //将allocCache中对应的位置标记为已分配
    s.freeindex = result + 1 //重新计算span中freeindex的位置，指向下一个可分配的object

    if s.freeindex % 64 == 0 && s.freeindex != s.nelems {
        //当前freeindex已经移动到下一个allocCache的首位，填充下一个allocCache
        s.refillAllocCache()
    }
    return result
}

```

#### mcache.refill逻辑

mcache中没有可分配的span（所有span没有可分配的object），申请新的span

```Go
func (c *mcache) refill(spanClass) {
    s := c.alloc[spanClass] //获取mcache的当前span
    if s.allocCount != s.nelems {
        throw(error) //如果当前span未满，报错
    }
    atomic.Store(s.sweepgen, mheap_.sweepgen) //更新当前span的sweep，表示此span不再被缓存
    //spanClass对应的mcentral中获取一个可以分配的新span
    //此时s不再指向mcache的当前span，也意味着将这个已经没有可用空间的span还给了mcentral
    s = mheap_.central[spanClass].mcentral.cacheSpan()
    if s == nil { //如果mcentral没有返回可用的span，表示内存已不足，报错
        throw("out of memory")
    }
    s.sweepgen = mheap_.sweepgen+3 //将新分配的span标记为已经被缓存，阻止下次sweep周期到时sweep这个span
    c.alloc[spaClass] = s //将分配到的新span缓存到mcache；所以，通常mcache的alloc中每个spanClass对应的span最多只有一个
}

```

#### mcentral.cacheSpan逻辑

从central中分配新的span

```Go
func (c *mcentral) cacheSpan() *mspan {
    spanBytes := uintptr(class_to_allocpages[c.spanclass.sizeClass()]) * _PageSize //计算要分配的span的字节数
    lock(c.lock)  //加锁

retry:
    var s *mspan
    for s=c.nonempty.first; s!= nil; s=s.next{
        if s.sweepgen == spanNeedSweeping {
            s.sweepgen = spanHasSweept //将span置为已经sweept,原子操作
            c.nonempty.remove(s) //从mcentral的nonempty列表中摘除该span
            c.empty.insertBack(s) //将span插入mcentral的empty列表中
            unlock(c.lock) //解锁
            s.sweep(true) //清理该span
            goto havespan
        }
        if s.sweepgen == spanHasSweept { //该span已经被sweep， 跳过
            continue
        }
        //走到这，当前span可以分配并且不需要sweep，分配该span
        c.nonempty.remove(s) //从mcentral的nonempty列表中摘除该span
        c.empty.insertBack(s) //将span插入mcentral的empty列表中
        unlock(c.lock) //解锁
        goto havespan
    }

    //到这，nonempty中没有可以分配的span，便利empty span列表，看能否找到一个可以清除一些空间的span
    for s=c.empty.first; s != nil; s=s.next {
        if s.sweepgen==spanNeedSweeping {
            s.sweepgen = spanHasSweept //将span置为已经sweept，原子操作
            //得到一个需要清理的span，清理它看能否释放一些空间
            c.empty.remove(s) //将该span查到队尾
            c.empty.insertBack(s)
            unlock(c.lock)
            s.sweep(true)//清理该span
            freeindex := s.nextFreeIndex() //找到一个可以分配的object的位置
            if freeindex != s.nelems { 
                //该位置的object可以分配，返回该span
                s.freeindex = freeindex
                goto havespan
            }       
            lock(c.lock)
            goto retry     
        }
        if s.sweepgen == spanHasSweept {
            continue
        }
        //到这，s后面的span要么是正在sweeping的，要么是已经sweept的，mcentral当前不能满足分配要求，循环退出
        break
    }
    unlock(c.lock)
    s = c.grow() //从heap申请新的span补充该mcentral
    if s == nil {
        return nil //分配失败
    }
    //从heap得到新的span，插入mcentral的empty列表中
    lock(c.lock)
    c.empty.insertBack(s)
    unlock(c.lock)

havespan: //到这，s是一个被插入到empty列表尾部的有剩余空间的span
    //根据span的freeindex重置该span的allocCache
    freeByteBase := s.freeindex &^ (64-1)
    whichByte := freeByteBase / 8
    s.refillAllocCache(whichByte) //初始化s的allocCache
    s.allocCache >>= s.freeindex % 64 //调整allocCache，使allocCache的最低位对齐freeindex
    return s
}
```



### 参考

- [Go语言内存分配器设计](http://skoo.me/go/2013/10/08/go-memory-manage-system-design)
- [图解TCMalloc](https://zhuanlan.zhihu.com/p/29216091)
- [简单易懂的 Go 内存分配原理解读](https://yq.aliyun.com/articles/652551)
- [图解 Go 内存分配器](https://www.infoq.cn/article/IEhRLwmmIM7-11RYaLHR)
- [golang内存分配](https://studygolang.com/articles/5790)