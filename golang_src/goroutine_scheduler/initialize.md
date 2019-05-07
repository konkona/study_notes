# Initialize

了解GPM的基本结构之后，来学习一下进程的初始化操作

```go
// _rt0_amd64 is common startup code for most amd64 systems when using
// internal linking. This is the entry point for the program from the
// kernel for an ordinary -buildmode=exe program. The stack holds the
// number of arguments and the C-style argv.
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ	0(SP), DI	// argc
	LEAQ	8(SP), SI	// argv
	JMP	runtime·rt0_go(SB)

```

通过go
build生成的二进制文件入口就是上面的这个_rt0_amd64，这个函数将参数放入寄存器之后跳转到rt0_go，看一下rt0_go的定义：

```go
TEXT runtime·rt0_go(SB),NOSPLIT,$0
	// copy arguments forward on an even stack
	MOVQ	DI, AX		// argc
	MOVQ	SI, BX		// argv

	// 重点看一下jmp ok 之后的代码，这里就是各种初始化操作
ok:
	// set the per-goroutine and per-mach "registers"
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX
	MOVQ	CX, g(BX)
	LEAQ	runtime·m0(SB), AX

	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)

	CLD				// convention is D is always left cleared
	CALL	runtime·check(SB)

	MOVL	16(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	24(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	CALL	runtime·args(SB)
	CALL	runtime·osinit(SB)
	CALL	runtime·schedinit(SB)

	// create a new goroutine to start program
	MOVQ	$runtime·mainPC(SB), AX		// entry
	PUSHQ	AX
	PUSHQ	$0			// arg size
	CALL	runtime·newproc(SB)
	POPQ	AX
	POPQ	AX

	// start this M
	CALL	runtime·mstart(SB)

	CALL	runtime·abort(SB)	// mstart should never return
	RET

	// Prevent dead-code elimination of debugCallV1, which is
	// intended to be called by debuggers.
	MOVQ	$runtime·debugCallV1(SB), AX
	RET

```

看一下几个CALL调用都做了什么？

```go
// BSD interface for threading.
func osinit() {
	// pthread_create delayed until end of goenvs so that we
	// can look at the environment first.

	// 获取cpu的数量，初始化的时候把这个值作为allp的个数
	ncpu = getncpu()
	physPageSize = getPageSize()
}
```

```go
// The bootstrap sequence is:
//
//	call osinit
//	call schedinit
//	make & queue new G
//	call runtime·mstart
//
// The new G calls runtime·main.
func schedinit() {
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}	
}
```

schedinit函数主要做sched的初始化工作，通过procresize([procresize](./P.md))函数生成ncpu个p，并放到全局调度器的
pidle 队列中.

newproc函数([newproc](./G.md))新起一个协程runtime.main,并通过runqput加入到本地local
p中，请注意，目前为止，还只有一个主线程，所以后面runtime.main协程将会在主线程中执行。

跟踪一下runtime.main的调用时机，查看mstart函数定义： 

```go
func mstart() {
	_g_ := getg()

	osStack := _g_.stack.lo == 0
	if osStack {
		// Initialize stack bounds from system stack.
		// Cgo may have left stack size in stack.hi.
		// minit may update the stack bounds.
		size := _g_.stack.hi
		if size == 0 {
			size = 8192 * sys.StackGuardMultiplier
		}
		_g_.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
		_g_.stack.lo = _g_.stack.hi - size + 1024
	}
	
	// Initialize stack guards so that we can start calling
	// both Go and C functions with stack growth prologues.
	_g_.stackguard0 = _g_.stack.lo + _StackGuard
	_g_.stackguard1 = _g_.stackguard0
	mstart1()	
}	
```

mstart主要是做一下stack的初始化，当前是在g0协程。然后再调用mstart1()。

```
func mstart1() {
	_g_ := getg()
	
	// Record the caller for use as the top of stack in mcall and
	// for terminating the thread.
	// We're never coming back to mstart1 after we call schedule,
	// so other calls can reuse the current frame.
	save(getcallerpc(), getcallersp())	

	// 如果有注册初始化函数，在m运行前先调用
	if fn := _g_.m.mstartfn; fn != nil {
		fn()
	}	
	
	if _g_.m.helpgc != 0 {
		_g_.m.helpgc = 0
		stopm()
	} else if _g_.m != &m0 {
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
	
	// 开始调度任务
	schedule()	
}	
```

mstart1()是我们要重点分析的函数了，首先看一下

```
save(getcallerpc(), getcallersp())

// getcallerpc 会返回caller的pc地址，就是程序入口地址
// getcallersp 则返回caller的sp地址，就是程序的栈顶
getcallerpc()

//  再看看save的定义

// save updates getg().sched to refer to pc and sp so that a following
// gogo will restore pc and sp.
//
// save must not have write barriers because invoking a write barrier
// can clobber getg().sched.
//
//go:nosplit
//go:nowritebarrierrec
func save(pc, sp uintptr) {
	_g_ := getg()

	_g_.sched.pc = pc
	_g_.sched.sp = sp
	_g_.sched.lr = 0
	_g_.sched.ret = 0
	_g_.sched.g = guintptr(unsafe.Pointer(_g_))
	// We need to ensure ctxt is zero, but can't have a write
	// barrier here. However, it should always already be zero.
	// Assert that.
	if _g_.sched.ctxt != nil {
		badctxt()
	}
}
``` 

save函数保存了当前caller的入口地址和栈顶值，是用来干嘛的呢？ 继续往下看

