# G
在golang里面用户新起一个协程特别简单，使用go func语法糖即可。本文主要研究一下go
func到底做了什么。

## G的基本数据结构

```go
type g struct {
    stack       stack       // ⾃自定义栈。
    stackguard0 uintptr     // 栈溢出检查边界。默认  newg.stack.lo + _StackGuard
    sched          gobuf    // 执⾏现场。
    schedlink      guintptr // 链表。
}
```

其中stack是自己实现的一块堆栈，用来存储go协程运行时的数据，其实现如下 

```goregexp
type stack struct {
	lo uintptr      // 栈内存开始地址。
	hi uintptr      // 结束地址。 对应的0(SP)
}
```

sched是执行现场，其实现如下

```
type gobuf struct {
	sp   uintptr            // SP寄存器，用来存放栈顶指针  newg.stack.hi - totalSize， 其中totalSize是参数占用bytes+4个寄存器长度
	pc   uintptr            // PC寄存器，用来存放函数指令地址
	g    guintptr
	ctxt unsafe.Pointer
	ret  sys.Uintreg
	lr   uintptr
	bp   uintptr            // 函数空间的栈底

}
```

在使用go 语法糖的时候，其实是调用newproc来实现的，其实现如下

```
//go:nosplit      // 跳过stack split，禁止执行栈溢出检测
func newproc(siz int32, fn *funcval) {  //  siz  参数bytes， fn  函数地址
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg()
	pc := getcallerpc()
	systemstack(func() {
		newproc1(fn, (*uint8)(argp), siz, gp, pc) // 真正newproc的地方
	})
}
```

先简单分析一下systemstask函数

```
// systemstack runs fn on a system stack.
// If systemstack is called from the per-OS-thread (g0) stack, or
// if systemstack is called from the signal handling (gsignal) stack,
// systemstack calls fn directly and returns.
// Otherwise, systemstack is being called from the limited stack
// of an ordinary goroutine. In this case, systemstack switches
// to the per-OS-thread stack, calls fn, and switches back.
// It is common to use a func literal as the argument, in order
// to share inputs and outputs with the code around the call
// to system stack:

//go:noescape
func systemstack(fn func())
```

在C,C++中，当栈上的变量超过其生命周期后，会自动的释放掉。但是在go中，如果该变量在其他地方被引用，go会将该函数栈转移到堆中，逃逸就是指的这种行为。那么noescape就表示禁止逃逸。

systemstack函数如果是从每个os线程的g0或者是从signal
调用的，则fn会被直接调用并返回。否则，systemstack 将切换到 g0 栈上调用 fn 然后切换回来。


```
// Create a new g running fn with narg bytes of arguments starting
// at argp. callerpc is the address of the go statement that created
// this. The new g is put on the queue of g's waiting to run.
func newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr) {
    _g_ := getg()   // 获取当前g， 为什么不直接用callergp呢，因为上面说了systemstack函数可能是从普通协程调用的，所以会切换stack
    
    siz := narg
    siz = (siz + 7) &^ 7  // 8字节对齐
    
    _p_ := _g_.m.p.ptr()    // 找到当前的p，只要g在运行中，那么肯定有一个p跟它是绑定的，所以这里先找p
    newg := gfget(_p_)      // 获取可复用的g
    if newg == nil {
        newg = malg(_StackMin)  // 新建, 默认最小2048byte，受系统影响，该值可能会大于2k，最终是2的整数倍
        casgstatus(newg, _Gidle, _Gdead)   // 创建完成后设置_Gdead可以预防GC检查
        allgadd(newg) // 添加到全局变量，也就是global队列中，这里面加了锁，因为涉及到竞争
    }
    
    // 下面这几行是用来计算sp位置的, 最后的sp是去掉了参数空间，即入参在sp之上
    totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize // extra space in case of reads slightly beyond frame
    totalSize += -totalSize & (sys.SpAlign - 1)                  // align to spAlign
    sp := newg.stack.hi - totalSize
    spArg := sp
    
    // 如果参数bytes大于0，说明有参数存在，将参数入栈，由于栈是由高到低生长的，所以参数入栈之后占用空间[sp, newg.stack.hi]
    if narg > 0 {
        memmove(unsafe.Pointer(spArg), unsafe.Pointer(argp), uintptr(narg))
    }
    
    // 现在sp就是除去参数之后的栈顶了，用来存函数里面的各种变量
    
    // 保存现场， 设置sp, pc 到sched中
    memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
    newg.sched.sp = sp
    newg.stktopsp = sp
    newg.sched.pc = funcPC(goexit) + sys.PCQuantum // 为什么要加sys.PCQuantum呢？
    newg.sched.g = guintptr(unsafe.Pointer(newg))
    
    // 设置入口函数
    gostartcallfn(&newg.sched, fn)  // 调用 gostartcall函数
    
    newg.gopc = callerpc
    newg.ancestors = saveAncestors(callergp)
    newg.startpc = fn.fn
    
    
    // 初始化完成后， 设置状态为_Grunnable
    newg.gcscanvalid = false
    casgstatus(newg, _Gdead, _Grunnable)
    
    // 将⽣成的 G 对象放到 P 本地队列或全局队列。
    runqput(_p_, newg, true)
}

```

```
// adjust Gobuf as if it executed a call to fn with context ctxt
// and then did an immediate gosave.
func gostartcall(buf *gobuf, fn, ctxt unsafe.Pointer) {
	sp := buf.sp
	if sys.RegSize > sys.PtrSize {
		sp -= sys.PtrSize
		*(*uintptr)(unsafe.Pointer(sp)) = 0
	}
	sp -= sys.PtrSize // sp往上移一位 
	*(*uintptr)(unsafe.Pointer(sp)) = buf.pc // 即先前funcPC(goexit) + sys.PCQuantum的值，保证函数返回时一定会调用goexit
	buf.sp = sp   // 移动栈顶
	buf.pc = uintptr(fn)  //  真正的入口地址
	buf.ctxt = ctxt
}
```

在newproc1函数创建g的时候，设置了sched的pc值，为goexit +
sys.PCQuantum,看一下goexit的定义：

```
// The top-most function running on a goroutine
// returns to goexit+PCQuantum.
TEXT runtime·goexit(SB),NOSPLIT|NOFRAME,$0-0
	NOR	R0, R0	// NOP
	JAL	runtime·goexit1(SB)	// does not return
	// traceback from goexit1 must hit code range of goexit
	NOR	R0, R0	// NOP
```

可以看到goexit最终是调用goexit1的