## Go内存分配机制

```Go
func mallocgc(type, size) pointer {
    m := aquirem // 获得M
    m.mallocing=1 //lock
    c := gomcache() // 获取当前goroutine的cache
    
    if size <= maxSmallSize {
        if size <= maxTinySize && notContainsPointer {
            //tiny object alloc
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
            //normal small object alloc
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




### 参考

- [Go语言内存分配器设计](http://skoo.me/go/2013/10/08/go-memory-manage-system-design)
- [图解TCMalloc](https://zhuanlan.zhihu.com/p/29216091)
- [简单易懂的 Go 内存分配原理解读](https://yq.aliyun.com/articles/652551)
- [图解 Go 内存分配器](https://www.infoq.cn/article/IEhRLwmmIM7-11RYaLHR)
- [golang内存分配](https://studygolang.com/articles/5790)