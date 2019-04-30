# M

M可以认为是对操作系统线程的一个封装，先来看一下m的结构。

## M的基本结构
```
type m struct {
    // g0 , 属于m的调度协程，其stack值要比普通的(2k)要大。
    g0      *g     // goroutine with scheduling stack
    
    // tls是线程的local变量，用来保存各个协程切换时的上下文
    tls           [6]uintptr   // thread-local storage (for x86 extern register)
    mstartfn      func()       // 创建时传入，在m start的时候调用
    curg          *g       // current running goroutine
    
    p             puintptr // attached p for executing go code (nil if not executing go code)
    nextp         puintptr	// 需要继续研究作用
    
    locks         int32
    
    alllink       *m // on allm
    schedlink     muintptr
    
    createstack   [32]uintptr    // stack that created this thread.	
    
    thread        uintptr // thread handle
    freelink      *m      // on sched.freem
    
    mOS	
}	

其中mOS的定义如下：
type mOS struct {
    initialized bool
    mutex       pthreadmutex
    cond        pthreadcond
    count       int
}

可以明显看出，mOS主要是用来做数据同步用的，使用了操作系统的mutex和条件变量
```

M的创建：

```
// Create a new m. It will start off with a call to fn, or else the scheduler.
// fn needs to be static and not a heap allocated closure.
// May run with m.p==nil, so write barriers are not allowed.
//go:nowritebarrierrec
func newm(fn func(), _p_ *p) {
	mp := allocm(_p_, fn)
	mp.nextp.set(_p_)
	mp.sigmask = initSigmask
	newm1(mp)
}
```

通过newm创建一个m实例，其中allocm是具体创建m的方法，给m分配stack，初始化一些操作，我们看一下它的定义：

```
// Allocate a new m unassociated with any thread.
// Can use p for allocation context if needed.
// fn is recorded as the new m's m.mstartfn.
//
// This function is allowed to have write barriers even if the caller
// isn't because it borrows _p_.
//
//go:yeswritebarrierrec
func allocm(_p_ *p, fn func()) *m {
	_g_ := getg()
	_g_.m.locks++ // disable GC because it can be called from sysmon
	if _g_.m.p == 0 {  // 当前协程所在的m没有绑定p，临时绑定这个参数p
		acquirep(_p_) // temporarily borrow p for mallocs in this function
	}

	mp := new(m)
	mp.mstartfn = fn   // 设置m初始化时需要调用的func

	// In case of cgo or Solaris or Darwin, pthread_create will make us a stack.
	// Windows and Plan 9 will layout sched stack on OS stack.
	if iscgo || GOOS == "solaris" || GOOS == "windows" || GOOS == "plan9" || GOOS == "darwin" {
		mp.g0 = malg(-1)
	} else {
		mp.g0 = malg(8192 * sys.StackGuardMultiplier) // g0的stack比普通的要大
	}
	mp.g0.m = mp   // 新的m要跟g0绑定

	if _p_ == _g_.m.p.ptr() {   // 如果传参_p_是当前m绑定的p，需要将当前m和p解绑
		releasep()
	}
	_g_.m.locks--
	if _g_.m.locks == 0 && _g_.preempt { // restore the preemption request in case we've cleared it in newstack
		_g_.stackguard0 = stackPreempt
	}

	return mp
}
```
简单来说，allocm如其名，就是新建m，给一个p绑定，并设置一些初始化的数据。

继续来看newm1(mp)这个函数:

```
func newm1(mp *m) {
	execLock.rlock() // Prevent process clone.
	newosproc(mp)
	execLock.runlock()
}
```

这个里面啥都没做，直接调用了newosproc(mp):

```
// May run with m.p==nil, so write barriers are not allowed.
//go:nowritebarrierrec
func newosproc(mp *m) {
	stk := unsafe.Pointer(mp.g0.stack.hi)
	if false {
		print("newosproc stk=", stk, " m=", mp, " g=", mp.g0, " id=", mp.id, " ostk=", &mp, "\n")
	}

	// Initialize an attribute object.
	var attr pthreadattr
	var err int32
	err = pthread_attr_init(&attr)
	if err != 0 {
		write(2, unsafe.Pointer(&failthreadcreate[0]), int32(len(failthreadcreate)))
		exit(1)
	}

	// Set the stack size we want to use.  64KB for now.
	// TODO: just use OS default size?
	const stackSize = 1 << 16
	if pthread_attr_setstacksize(&attr, stackSize) != 0 {
		write(2, unsafe.Pointer(&failthreadcreate[0]), int32(len(failthreadcreate)))
		exit(1)
	}
	//mSysStatInc(&memstats.stacks_sys, stackSize) //TODO: do this?

	// Tell the pthread library we won't join with this thread.
	if pthread_attr_setdetachstate(&attr, _PTHREAD_CREATE_DETACHED) != 0 {
		write(2, unsafe.Pointer(&failthreadcreate[0]), int32(len(failthreadcreate)))
		exit(1)
	}

	// Finally, create the thread. It starts at mstart_stub, which does some low-level
	// setup and then calls mstart.
	var oset sigset
	sigprocmask(_SIG_SETMASK, &sigset_all, &oset)
	err = pthread_create(&attr, funcPC(mstart_stub), unsafe.Pointer(mp))
	sigprocmask(_SIG_SETMASK, &oset, nil)
	if err != 0 {
		write(2, unsafe.Pointer(&failthreadcreate[0]), int32(len(failthreadcreate)))
		exit(1)
	}
}
```

newosproc这个函数就是熟悉的系统调用啦，通过pthread_create创建一个系统线程，将mstart_stub函数作为线程入口，mp作为参数：

```
// mstart_stub is the first function executed on a new thread started by pthread_create.
// It just does some low-level setup and then calls mstart.
// Note: called with the C calling convention.
TEXT runtime·mstart_stub(SB),NOSPLIT,$0
	// The value at SP+4 points to the m.
	// We are already on m's g0 stack.

	// Save callee-save registers.
	SUBL	$16, SP
	MOVL	BP, 0(SP)
	MOVL	BX, 4(SP)
	MOVL	SI, 8(SP)
	MOVL	DI, 12(SP)

	MOVL	SP, AX       // hide argument read from vet (vet thinks this function is using the Go calling convention)
	MOVL	20(AX), DI   // m
	MOVL	m_g0(DI), DX // g

	// Initialize TLS entry.
	// See cmd/link/internal/ld/sym.go:computeTLSOffset.
	MOVL	DX, 0x18(GS)

	// Someday the convention will be D is always cleared.
	CLD

	CALL	runtime·mstart(SB)

	// Restore callee-save registers.
	MOVL	0(SP), BP
	MOVL	4(SP), BX
	MOVL	8(SP), SI
	MOVL	12(SP), DI

	// Go is all done with this OS thread.
	// Tell pthread everything is ok (we never join with this thread, so
	// the value here doesn't really matter).
	XORL	AX, AX

	ADDL	$16, SP
	RET
```

看一下mstart_stub的定义，其实就是做了一些初始化的工作，然后调用mstart函数，开始我们的m调用。
