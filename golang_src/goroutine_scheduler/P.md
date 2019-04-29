# P

我理解的P是一个任务池，里面包含多个任务(G),程序启动时初始化N个池子，用户go生成协程会放入池子，等待M的调用。

## P的基本结构

```
type p struct {
    lock mutex
    
    id          int32
    status      uint32 // one of pidle/prunning/...
    link        puintptr        //  这个link是用来查看有g的队列列表的 p -> nextp -> nextp
    
    
    // Queue of runnable goroutines. Accessed without lock.
    runqhead uint32
    runqtail uint32
    runq     [256]guintptr    //  p 的local p队列， 256items的array。通过head和tail操作，表现为一个环形buffer
    
    // runnext, if non-nil, is a runnable G that was ready'd by
    // the current G and should be run next instead of what's in
    // runq if there's time remaining in the running G's time
    // slice. It will inherit the time left in the current time
    // slice. If a set of goroutines is locked in a
    // communicate-and-wait pattern, this schedules that set as a
    // unit and eliminates the (potentially large) scheduling
    // latency that otherwise arises from adding the ready'd
    // goroutines to the end of the run queue.
    runnext guintptr
    
    // Available G's (status == Gdead)
    gfree    *g
    gfreecnt int32	
}	
```

重点说一下runq，runq是p的一个local队列，是一个环形buffer。通过go 语法将协程与p绑定流程：
+ 如果put的时候设置了next为true， 将p的next置换出来(如果有的话)
+ 检查环形buffer是否已满(len=256),未满，加到尾部
+ 已满了，试图加到global队列，这里是调用runqputslow
+ 不断的retry， 是谁在消费队列？？？？ 猜测是一读一写线程使用了lockfree

```
// runqput tries to put g on the local runnable queue.
// If next is false, runqput adds g to the tail of the runnable queue.
// If next is true, runqput puts g in the _p_.runnext slot.
// If the run queue is full, runnext puts g on the global queue.
// Executed only by the owner P.
func runqput(_p_ *p, gp *g, next bool) {
    if randomizeScheduler && next && fastrand()%2 == 0 {
        next = false
    }
    
    if next {
    retryNext:
        oldnext := _p_.runnext
        if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
            goto retryNext
        }
        if oldnext == 0 {
            return
        }
        // Kick the old runnext out to the regular run queue.
        gp = oldnext.ptr()
    }
    
    retry:
    h := atomic.Load(&_p_.runqhead) // load-acquire, synchronize with consumers
    t := _p_.runqtail
    if t-h < uint32(len(_p_.runq)) {
        _p_.runq[t%uint32(len(_p_.runq))].set(gp)
        atomic.Store(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
        return
    }
    if runqputslow(_p_, gp, h, t) {
        return
    }
    // the queue is not full, now the put above must succeed
    goto retry
}

```


procresize是设置进程processor的函数，我们看一下他的初始化流程： 

```
// Change number of processors. The world is stopped, sched is locked.
// gcworkbufs are not being modified by either the GC or
// the write barrier code.
// Returns list of Ps with local work, they need to be scheduled by the caller.
func procresize(nprocs int32) *p {
    old := gomaxprocs   // 在osinit()函数中默认为cpu的数量
    
    // Grow allp if necessary.
    if nprocs > int32(len(allp)) {
        // Synchronize with retake, which could be running
        // concurrently since it doesn't run on a P.
        lock(&allpLock)     // 因为allp slice是一个全局变量，所有的M都可能去访问，这里需要lock
        if nprocs <= int32(cap(allp)) {
            allp = allp[:nprocs]
        } else {
            nallp := make([]*p, nprocs)
            // Copy everything up to allp's cap so we
            // never lose old allocated Ps.
            copy(nallp, allp[:cap(allp)])
            allp = nallp
        }
        unlock(&allpLock)
    }
    
    // initialize new P's
    for i := int32(0); i < nprocs; i++ {
        pp := allp[i]
        if pp == nil {
            pp = new(p)
            pp.id = i
            pp.status = _Pgcstop
            pp.sudogcache = pp.sudogbuf[:0]
            for i := range pp.deferpool {
                pp.deferpool[i] = pp.deferpoolbuf[i][:0]
            }
            pp.wbBuf.reset()
            atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
        }
    }
    
    // free unused P's， 因为p数量调整了，所以把多余的p上面的g提出来放到global的前面
    for i := nprocs; i < old; i++ {
        p := allp[i]
        
        // move all runnable goroutines to the global queue
        for p.runqhead != p.runqtail {
            // pop from tail of local queue
            p.runqtail--
            gp := p.runq[p.runqtail%uint32(len(p.runq))].ptr()
            // push onto head of global queue
            globrunqputhead(gp)
        }
        if p.runnext != 0 {
            globrunqputhead(p.runnext.ptr())
            p.runnext = 0
        }
        
        for i := range p.sudogbuf {
            p.sudogbuf[i] = nil
        }
        p.sudogcache = p.sudogbuf[:0]
        for i := range p.deferpool {
            for j := range p.deferpoolbuf[i] {
                p.deferpoolbuf[i][j] = nil
            }
            p.deferpool[i] = p.deferpoolbuf[i][:0]
        }
        gfpurge(p)
        
        p.gcAssistTime = 0
        p.status = _Pdead
        // can't free P itself because it can be referenced by an M in syscall
    }
    
    // Trim allp. 调整一下allp，跟nprocs数量一致
    if int32(len(allp)) != nprocs {
        lock(&allpLock)
        allp = allp[:nprocs]
        unlock(&allpLock)
    }	
    
    // 解绑超出调整后的nprocs大小的p
    _g_ := getg()
    if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs { // 正常范围内，可以继续使用
        // continue to use the current P
        _g_.m.p.ptr().status = _Prunning
    } else {// 超过了nprocs大小，跟M解绑， 重新绑定allp[0],为什么要这样呢？
        // release the current P and acquire allp[0]
        if _g_.m.p != 0 {
            _g_.m.p.ptr().m = 0
        }
        _g_.m.p = 0   // 解绑当前p
        _g_.m.mcache = nil
        p := allp[0]
        p.m = 0  // allp[0]跟原来的m解绑
        p.status = _Pidle
        acquirep(p)  // allp[0]绑定当前m
    }
    
    
    // 给每个非空的p重新分配m， 最终返回一个runnablePs的链表回去
    var runnablePs *p
    for i := nprocs - 1; i >= 0; i-- {
        p := allp[i]
        if _g_.m.p.ptr() == p { // 当前m，已经有p在运行这个g了，可以continue
            continue
        }
        p.status = _Pidle
        if runqempty(p) {
            pidleput(p)  // 将 p 放到全局调度器的 pidle 队列中
        } else {
            p.m.set(mget())
            p.link.set(runnablePs)
            runnablePs = p
        }
    }
		
}
```

总结一下： procresize
函数将allp调整为对应nprocs长度，如果有多余的p，则将p上面的g分配到global队列，解绑m和p的关系。
调整完毕后，还会重新分配m，只要p中有g，则会分配一个新的m。具体是什么原因还需要继续探索？