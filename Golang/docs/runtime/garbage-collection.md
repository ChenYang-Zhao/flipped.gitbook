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


### Golang的三色标记法

Golang的标记-清理gc算法使用了三色标记法，大致含义是：

- 最开始所有的对象都是白色；
- 从gc root扫描所有的可达对象，标记为灰色，并入队；
- 循环从队列取出对象并标记为黑色，将其饮用对象标记为灰色，并入队；知道队列为空；
- 标记完成后，此时的黑色对象是gc后继续使用的对象，白色对象是可以清理的对象；

标记过程和用户代码会一起执行，标记阶段用户代码对内存的修改会改变对象之间的引用关系，为了保证标记结果的正确性，golang引入了写屏障；简单来说就是在标记阶段用户代码对内存的改动都会被写屏障截获，并将修改后的对象之间的引用关系重新标记并入队；


### 核心逻辑实现

#### gc启动 

```Go
func gcStart(trigger gcTrigger) {
    // 循环清理未清理的span，直到全部清理完成
    for trigger.test() && sweepone() != ^uintptr(0) {
        ... //清理计数
    }
    semacquire(&work.startSema) //获取gc phase从off到mark/mark termination的锁
    if !trigger.test(){ //拿到锁之后，再次检查phase转换的条件是否满足
        semarelease(&work.startSema)
        return
    }
    //检查本次gc是否是强制发起的
    work.userForced = trigger.kind == gcTriggerAlways || trigger.kind == gcTriggerCycle 
    mode := gcBackgroundMode //gc 和sweep都并发，正常模式
    if debug.gcstoptheworld == 1 {
        mode = gcForceMode //强制触发，gc阶段STW，并发sweep
    }else if debug.gcstoptheworld == 2{
        mode = gcForceBlockMode // gc和sweep都是STW，block模式
    }

    semacquire(&worldsema) //获取stop the world，STW的信号量

    // 检查所有P的mcache是否都已经flush过, flush mcache的过程在acquirep中触发
    for _, p := range allp{
         fg := p.mcache.flushGen; fg != _mheap.sweepgen {
             throw("p mcache not flushed")
         }
    }
    
    gcBgMarkStartWorkers() //开启后台的mark workers
    gcResetMarkState //

    systemstack(stopTheWorldWithSema) //stop the world
    // 开始扫描之前，确保所有的sweep都是swept状态
    systemstack(func(){
        finishsweep_m()
    })

    clearpools()

    gcController.startCycle() //开始gc
    work.heapGoal = memstats.next_gc //设置gc的目标

    if mode != gcBackgroundMode {
        schedEnableUser(false) //停止用户goroutine的调度
    }
    
    //以下开始进入mark阶段
    setGCPhase(_GCmark)

    gcBgMarkPrepare()
    gcMarkRootPrepare() //将root scan job（stack，globals）入队

    gcMarkTinyAllocs() //将tiny block上的object置为gray

    systemstack(func(){
        now = startTheWorldWithSema()
    })

    if mode != gcBackgroundMode{
        Goshed()
    }

    semrelease(&work.startSema)
}
```


### 参考

- [Getting to Go: The Journey of Go's Garbage Collector](https://blog.golang.org/ismmkeynote)
- [图解Golang的GC算法](https://juejin.im/post/5c8525666fb9a049ea39c3e6)
- [Golang源码探索(三) GC的实现原理](https://www.cnblogs.com/zkweb/p/7880099.html)
- [Go 垃圾回收](https://ninokop.github.io/2017/12/07/Go-%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6/)