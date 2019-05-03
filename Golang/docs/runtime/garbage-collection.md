## Golang 垃圾回收

### GC触发条件

#### mallocgc触发

每次调用mallocgc是都会检查是否需要开始gc，首先如果mcache不能满足分配条件，调用refill从mcentral申请新的span时会将shouldhelpgc设置为true，此外，如果是大对象（大于32KB）的分配，直接设置shouldhelpgc为true；此后，在分配完成后检查是否需要开始一次gc； mallocgc方法中与gc相关的代码如下：

```Go
func mallocgc(...) {
    shouldhelpgc = false

    if size <= maxSmallSize{
        if size < maxTinySize {
            //tiny block 上分配失败，申请新的tiny block
            v := nextFreeFast()
            if v == nil {
                //快速分配失败，调用nextFree
                v, _, shouldhelpgc := c.nextFree()
            }
        }
        //small object alloc
        v := nextFreeFast()
            if v == nil {
                //快速分配失败，调用nextFree
                v, _, shouldhelpgc := c.nextFree()
            }
    }else {
        shouldhelpgc = true
        s = largeAlloc()
    }
    //分配成功后
    if gcphase != _GcOff {
        //gc的过程中分配对象，因为写屏障的存在，设置分配出的object为black
        gcmarknewobject()
    }

    if shouldhelpgc {
        if t := (gcTrigger{kind:gcTriggerHeap}); t.test() { //检查是否满足gc的触发条件，是则开始gc
            gcStart() 
        }
    }
}

func (t gcTrigger) test() bool {
    if !memtats.enablegc|| panicking!= 0 {
        return false
    }
    if t.kinfd == gcTriggerAlways {
        return true
    }
    if gcphase != _GCOff {
        return false
    }
    switch t.kind {
        case gcTriggerHeap:
            return memstats.heap_alive >= memstats.heap_trigger
        case gcTriggerTime:
            if gcpercent < 0 {
                return false
            }
            lastgc := memstats.last_gc_nanotime
            return lastgc != 0 && t.now-lastgc>forcegcperiod
        case gcTriggerCycle:
            return (t.n-work.cycles) > 0 
    }
    return true
}
```

### GC基本流程

Golang的gc采用并发的标记-清楚算法，并使用写屏障（write-barrier）保证gc时对内存的写操作不影响gc的正确性；GC的基本流程如下：

1. 执行 sweep termination：
   1. Stop the world，此时所有的P都会达到安全点（safe-point）；
   2. 清理所有未清理的span；此时，只有当本次gc循环是强制触发时才会存在未清理的span；
2. 执行标记流程：
   1. 设置gcphase _GCOff --> _GCPMark，开启写屏障，开启mutator assists将root mark jobs入队；在所有的P都开启写屏障之前，不会扫描任何对象；此时仍在STW中；
   2. start the world，从这一点开始，写屏障负责所有指针对象的改变和新增，新分配的对象直接被标记为黑色；
   3. 执行root marking jobs，包括扫描所有的stack、全局变量以及所有非heap中的内存区域；扫描goroutine的stack会停止它，扫描stack上所有的指针类型，再恢复goroutine的执行；
   4. 从gray object的work queue中出队objet，将其自身标记为黑色，然后将此object指向的其它未入队的对象标记为灰色，并入队work queue；
   5. 当work queue中不存在灰色对象时，gc进入到mark termination阶段；
3. 开始gc termination ：
   1. stop the world；
   2. 将gcphase 设置为_GCMarkTermination，disable mark workers和assists；
   3. 开始house keeping，比如将mcacheflush到mcentral；
4. gc开始清理（sweep）阶段：
   1. 将gcphase设置为_GCOff，设置sweep state，并关闭写屏障；
   2. start the world，从这一点开始，新分配的对象都是白色，如果需要的话，在使用之前分配新的已经清理过的span；
   3. gc开始在后台并发的清理span；
5. 如果新的内存分配到达了gc的阈值，开始新一轮的gc；

#### 并发清理

清理阶段和正常程序一起并发地执行，heap中的span被后台的清理routine一个一个地清理；在mark termination的STW阶段，所有的span都被标记为need sweeping。

当仍然存在没有被清理的span时，如果某个goroutine需要申请一个新的span，首先需要通过清理reclaim相同大小的内存空间；如果一个goroutine需要申请一个small-object span，则至少要清理出一个object的内存空间；如果某个goroutine需要申请一个large object span，则至少要在heap中清理出和申请的object相同page数的内存空间；

比如严格的保证不能对没有清理的span作任何操作，因为这样会破坏bitmap中的mark bits。GC阶段mcache中的所有内存都被flush近mcentral，然后所有的cache都是空的；如果之后一个mcache从mcantral申请了一个span，则必须要对其进行清理；

#### GC的频率

一次gc后，如果后续分配的内存达到当前使用内存的一定比例，就会导致再一次的GC；比例的数值可以通过环境变量GOGC来控制，默认为100，意味着当内存使用达到当前内存使用量的一倍时再次触发GC；



### 参考

- [Getting to Go: The Journey of Go's Garbage Collector](https://blog.golang.org/ismmkeynote)
- [图解Golang的GC算法](https://juejin.im/post/5c8525666fb9a049ea39c3e6)
- [Golang源码探索(三) GC的实现原理](https://www.cnblogs.com/zkweb/p/7880099.html)
- [Go 垃圾回收](https://ninokop.github.io/2017/12/07/Go-%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6/)