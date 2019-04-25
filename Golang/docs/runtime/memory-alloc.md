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
    //并且，`0`表示已分配，`1`表示未分配；比如，如果8位，已分配两个，则allocCache当前位`00111111`
    //如果再分配一个，变为`00011111`
    allocCache uint64 
    nelems uintptr // 当前span中的object数量
    allocBits *gcBits //uint8类型的数组，数组中的每个值表示多个位置的object的分配情况
    gcmarkBits *gcBits
    elemsize uintptr //span中可分配的object的size
    allocCount uint16 //span中已分配的object的个数
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
            span := c.alloc[sizeClass, noscan] //根据size class 和noscan参数获取cache中对应的span的头指针
            v := nextFreeFast(span)// 快速找到可分配的object的位置
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





### 参考

- [Go语言内存分配器设计](http://skoo.me/go/2013/10/08/go-memory-manage-system-design)
- [图解TCMalloc](https://zhuanlan.zhihu.com/p/29216091)
- [简单易懂的 Go 内存分配原理解读](https://yq.aliyun.com/articles/652551)
- [图解 Go 内存分配器](https://www.infoq.cn/article/IEhRLwmmIM7-11RYaLHR)
- [golang内存分配](https://studygolang.com/articles/5790)