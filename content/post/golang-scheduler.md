---
title: "Golang调度器源码分析"
date: 2021-06-15T19:02:04+08:00
Categories: ["Golang"]
---

内核对系统线程的调度简单的归纳为：在执行操作系统代码时，内核调度器按照一定的算法挑选出一个线程并把该线程保存在内存之中的寄存器的值放入CPU对应的寄存器从而恢复该线程的运行。

万变不离其宗，系统线程对goroutine的调度与内核对系统线程的调度原理是一样的，实质都是**通过保存和修改CPU寄存器的值来达到切换线程/goroutine的目的。**

### 数据结构

**g**

为了实现对goroutine的调度，需要引入一个数据结构来保存CPU寄存器的值以及goroutine的其它一些状态信息，在Go语言调度器源代码中，这个数据结构是一个名叫g的结构体，它保存了goroutine的所有信息，该结构体的每一个实例对象都代表了一个goroutine，调度器代码可以通过g对象来对goroutine进行调度，当goroutine被调离CPU时，调度器代码负责把CPU寄存器的值保存在g对象的成员变量之中，当goroutine被调度起来运行时，调度器代码又负责把g对象的成员变量所保存的寄存器的值恢复到CPU的寄存器。

```go
// 前文所说的g结构体，它代表了一个goroutine
type g struct {
    // Stack parameters.
    // stack describes the actual stack memory: [stack.lo, stack.hi).
    // stackguard0 is the stack pointer compared in the Go stack growth prologue.
    // It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
    // stackguard1 is the stack pointer compared in the C stack growth prologue.
    // It is stack.lo+StackGuard on g0 and gsignal stacks.
    // It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
 
    // 记录该goroutine使用的栈
    stack       stack   // offset known to runtime/cgo
    // 下面两个成员用于栈溢出检查，实现栈的自动伸缩，抢占调度也会用到stackguard0
    stackguard0 uintptr // offset known to liblink
    stackguard1 uintptr // offset known to liblink

    ......
 
    // 此goroutine正在被哪个工作线程执行
    m              *m      // current m; offset known to arm liblink
    // 保存调度信息，主要是几个寄存器的值
    sched          gobuf
 
    ......
    // schedlink字段指向全局运行队列中的下一个g，
    //所有位于全局运行队列中的g形成一个链表
    schedlink      guintptr

    ......
    // 抢占调度标志，如果需要抢占调度，设置preempt为true
    preempt        bool       // preemption signal, duplicates stackguard0 = stackpreempt

   ......
}
```

以下两个结构是被包含在g之内的：

**stack**

主要用来记录goroutine所使用的栈的信息，包括栈顶和栈底位置

```go
// Stack describes a Go execution stack.
// The bounds of the stack are exactly [lo, hi),
// with no implicit data structures on either side.
//用于记录goroutine使用的栈的起始和结束位置
type stack struct {  
    lo uintptr    // 栈顶，指向内存低地址
    hi uintptr    // 栈底，指向内存高地址
}
```

**gobuf**

用于保存goroutine的调度信息，主要包括CPU的几个寄存器的值

```go
type gobuf struct {
    // The offsets of sp, pc, and g are known to (hard-coded in) libmach.
    //
    // ctxt is unusual with respect to GC: it may be a
    // heap-allocated funcval, so GC needs to track it, but it
    // needs to be set and cleared from assembly, where it's
    // difficult to have write barriers. However, ctxt is really a
    // saved, live register, and we only ever exchange it between
    // the real register and the gobuf. Hence, we treat it as a
    // root during stack scanning, which means assembly that saves
    // and restores it doesn't need write barriers. It's still
    // typed as a pointer so that any other writes from Go get
    // write barriers.
    sp   uintptr  // 保存CPU的rsp寄存器的值
    pc   uintptr  // 保存CPU的rip寄存器的值
    g    guintptr // 记录当前这个gobuf对象属于哪个goroutine
    ctxt unsafe.Pointer
 
    // 保存系统调用的返回值，因为从系统调用返回之后如果p被其它工作线程抢占，
    // 则这个goroutine会被放入全局运行队列被其它工作线程调度，其它线程需要知道系统调用的返回值。
    ret  sys.Uintreg  
    lr   uintptr
 
    // 保存CPU的rip寄存器的值
    bp   uintptr // for GOEXPERIMENT=framepointer
}
```

**schedt**

用来保存调度器的状态信息和goroutine的全局运行队列

要实现对goroutine的调度，仅仅有g结构体对象是不够的，至少还需要一个存放所有（可运行）goroutine的容器，便于工作线程寻找需要被调度起来运行的goroutine，于是Go调度器又引入了schedt结构体，一方面用来保存调度器自身的状态信息，另一方面它还拥有一个用来保存goroutine的运行队列。因为每个Go程序只有一个调度器，所以在每个Go程序中schedt结构体只有一个实例对象，该实例对象在源代码中被定义成了一个共享的全局变量，这样每个工作线程都可以访问它以及它所拥有的goroutine运行队列，我们称这个运行队列为全局运行队列。

```go
type schedt struct {
    // accessed atomically. keep at top to ensure alignment on 32-bit systems.
    goidgen  uint64
    lastpoll uint64

    lock mutex

    // When increasing nmidle, nmidlelocked, nmsys, or nmfreed, be
    // sure to call checkdead().

    // 由空闲的工作线程组成链表
    midle        muintptr // idle m's waiting for work
    // 空闲的工作线程的数量
    nmidle       int32    // number of idle m's waiting for work
    nmidlelocked int32    // number of locked m's waiting for work
    mnext        int64    // number of m's that have been created and next M ID
    // 最多只能创建maxmcount个工作线程
    maxmcount    int32    // maximum number of m's allowed (or die)
    nmsys        int32    // number of system m's not counted for deadlock
    nmfreed      int64    // cumulative number of freed m's

    ngsys uint32 // number of system goroutines; updated atomically

    // 由空闲的p结构体对象组成的链表
    pidle      puintptr // idle p's
    // 空闲的p结构体对象的数量
    npidle     uint32
    nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.

    // Global runnable queue.
    // goroutine全局运行队列
    runq     gQueue
    runqsize int32

    ......

    // Global cache of dead G's.
    // gFree是所有已经退出的goroutine对应的g结构体对象组成的链表
    // 用于缓存g结构体对象，避免每次创建goroutine时都重新分配内存
    gFree struct {
        lock          mutex
        stack        gList // Gs with stacks
        noStack   gList // Gs without stacks
        n              int32
    }
 
    ......
}

```

**p**

既然说到全局运行队列，读者可能猜想到应该还有一个局部运行队列。确实如此，因为全局运行队列是每个工作线程都可以读写的，因此访问它需要加锁，然而在一个繁忙的系统中，加锁会导致严重的性能问题。于是，调度器又为每个工作线程引入了一个私有的局部goroutine运行队列，工作线程优先使用自己的局部运行队列，只有必要时才会去访问全局运行队列，这大大减少了锁冲突，提高了工作线程的并发性。在Go调度器源代码中，**局部运行队列被包含在p结构体的实例对象之中**，每一个运行着go代码的工作线程都会与一个p结构体的实例对象关联在一起。

p结构体用于保存工作线程执行go代码时所必需的资源，比如goroutine的运行队列，内存分配用到的缓存等等。

```go
type p struct {
    lock mutex

    status       uint32 // one of pidle/prunning/...
    link            puintptr
    schedtick   uint32     // incremented on every scheduler call
    syscalltick  uint32     // incremented on every system call
    sysmontick  sysmontick // last tick observed by sysmon
    m                muintptr   // back-link to associated m (nil if idle)

    ......

    // Queue of runnable goroutines. Accessed without lock.
    //本地goroutine运行队列
    runqhead uint32  // 队列头
    runqtail uint32     // 队列尾
    runq     [256]guintptr  //使用数组实现的循环队列
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
    gFree struct {
        gList
        n int32
    }

    ......
}
```

**m**

除了上面介绍的g、schedt和p结构体，**Go调度器源代码中还有一个用来代表工作线程的m结构体，每个工作线程都有唯一的一个m结构体的实例对象与之对应**，m结构体对象除了记录着工作线程的诸如栈的起止位置、当前正在执行的goroutine以及是否空闲等等状态信息之外，还通过指针维持着与p结构体的实例对象之间的绑定关系。于是，通过m既可以找到与之对应的工作线程正在运行的goroutine，又可以找到工作线程的局部运行队列等资源。

```go
type m struct {
    // g0主要用来记录工作线程使用的栈信息，在执行调度代码时需要使用这个栈
    // 执行用户goroutine代码时，使用用户goroutine自己的栈，调度时会发生栈的切换
    g0      *g     // goroutine with scheduling stack

    // 通过TLS实现m结构体对象与工作线程之间的绑定
    tls           [6]uintptr   // thread-local storage (for x86 extern register)
    mstartfn      func()
    // 指向工作线程正在运行的goroutine的g结构体对象
    curg          *g       // current running goroutine
 
    // 记录与当前工作线程绑定的p结构体对象
    p             puintptr // attached p for executing go code (nil if not executing go code)
    nextp         puintptr
    oldp          puintptr // the p that was attached before executing a syscall
   
    // spinning状态：表示当前工作线程正在试图从其它工作线程的本地运行队列偷取goroutine
    spinning      bool // m is out of work and is actively looking for work
    blocked       bool // m is blocked on a note
   
    // 没有goroutine需要运行时，工作线程睡眠在这个park成员上，
    // 其它线程通过这个park唤醒该工作线程
    park          note
    // 记录所有工作线程的一个链表
    alllink       *m // on allm
    schedlink     muintptr

    // Linux平台thread的值就是操作系统线程ID
    thread        uintptr // thread handle
    freelink      *m      // on sched.freem

    ......
}
```


下面是g、p、m和schedt之间的关系图：

![](images/g-p-m.jpg)

上图中圆形图案代表g结构体的实例对象，三角形代表m结构体的实例对象，正方形代表p结构体的实例对象，其中红色的g表示m对应的工作线程正在运行的goroutine，而灰色的g表示处于运行队列之中正在等待被调度起来运行的goroutine。

从上图可以看出，每个m都绑定了一个p，每个p都有一个私有的本地goroutine队列，m对应的线程从本地和全局goroutine队列中获取goroutine并运行之。

> 一个问题，工作线程是如何与m绑定的？

**重要的全局变量**

```go
allgs     []*g     // 保存所有的g
allm       *m    // 所有的m构成的一个链表，包括下面的m0
allp       []*p    // 保存所有的p，len(allp) == gomaxprocs

ncpu             int32   // 系统中cpu核的数量，程序启动时由runtime代码初始化
gomaxprocs int32   // p的最大值，默认等于ncpu，但可以通过GOMAXPROCS修改

sched      schedt     // 调度器结构体对象，记录了调度器的工作状态

m0  m       // 代表进程的主线程
g0   g        // m0的g0，也就是m0.g0 = &g0
```

### 实现上下文切换的关键流程

**区分g0与其它g**

每个`m`都会关联一个`g0`，在执行runtime调度代码时会切换到`g0`栈

**区分m0和其它m**

只有主线程（程序启动后创建的第一个线程）对应着 `m0`，其它后续启动的线程对应着普通的`m`

**调度器初始化**

此阶段会：

- 初始化全局变量`g0`，g0主要是提供一个栈供runtime代码执行
- 初始化m0，将主线程与m0绑定（通过tls 线程本地存储机制）
- 创建所有的p（有几个核创建几个p）

**创建main goroutine**

这是程序启动后创建的第一个goroutine。这个goroutine将要执行的第一个函数是`runtime.main`，在这个过程中，会分配好这个goruotine的栈（是从堆上分配的）

**调度main goroutine**

此阶段完成从g0到main goroutine的切换（所谓context switch）

代码位置：runtime/proc.go

`mstart --> mstart1 --> schedule --> execute --> gogo`

实现切换的汇编代码：

```asm
# func gogo(buf *gobuf)

# restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $16-8
    #buf = &gp.sched
    MOVQ  buf+0(FP), BX   # BX = buf
  
    #gobuf->g --> dx register
    MOVQ  gobuf_g(BX), DX  # DX = gp.sched.g
  
    #下面这行代码没有实质作用，检查gp.sched.g是否是nil，如果是nil进程会crash死掉
    MOVQ  0(DX), CX   # make sure g != nil
  
    get_tls(CX) 
  
    #把要运行的g的指针放入线程本地存储，这样后面的代码就可以通过线程本地存储
    #获取到当前正在执行的goroutine的g结构体对象，从而找到与之关联的m和p
    MOVQ  DX, g(CX)
  
    #把CPU的SP寄存器设置为sched.sp，完成了栈的切换
    MOVQ  gobuf_sp(BX), SP  # restore SP
  
    #下面三条同样是恢复调度上下文到CPU相关寄存器
    MOVQ  gobuf_ret(BX), AX
    MOVQ  gobuf_ctxt(BX), DX
    MOVQ  gobuf_bp(BX), BP
  
    #清空sched的值，因为我们已把相关值放入CPU对应的寄存器了，不再需要，这样做可以少gc的工作量
    MOVQ  $0, gobuf_sp(BX)  # clear to help garbage collector
    MOVQ  $0, gobuf_ret(BX)
    MOVQ  $0, gobuf_ctxt(BX)
    MOVQ  $0, gobuf_bp(BX)
  
    #把sched.pc值放入BX寄存器
    MOVQ  gobuf_pc(BX), BX
  
    #JMP把BX寄存器的包含的地址值放入CPU的IP寄存器，于是，CPU跳转到该地址继续执行指令，
    JMP BX
```

**普通goroutine的退出**

退出时需要把cpu控制权通过`schedule`来交给runtime

普通goroutine在执行完成时，会返回到`runtime·goexit`这个函数，然后会继续调用：

`runtime·goexit --> goexit1 --> runtime·mcall --> goexit0 --> schedule`

mcall函数比较重要，它完成了普通goroutine到runtime的切换：

```asm
# func mcall(fn func(*g))
# Switch to m->g0's stack, call fn(g).
# Fn must never return. It should gogo(&g->sched)
# to keep running g.
# mcall的参数是一个指向funcval对象的指针
TEXT runtime·mcall(SB), NOSPLIT, $0-8
    #取出参数的值放入DI寄存器，它是funcval对象的指针，此场景中fn.fn是goexit0的地址
    MOVQ  fn+0(FP), DI

    get_tls(CX)
    MOVQ  g(CX), AX # AX = g，本场景g 是 g2

    #mcall返回地址放入BX
    MOVQ  0(SP), BX# caller's PC

    #保存g2的调度信息，因为我们要从当前正在运行的g2切换到g0
    MOVQ  BX, (g_sched+gobuf_pc)(AX)   #g.sched.pc = BX，保存g2的rip
    LEAQ  fn+0(FP), BX # caller's SP  
    MOVQ  BX, (g_sched+gobuf_sp)(AX)  #g.sched.sp = BX，保存g2的rsp
    MOVQ  AX, (g_sched+gobuf_g)(AX)   #g.sched.g = g
    MOVQ  BP, (g_sched+gobuf_bp)(AX)  #g.sched.bp = BP，保存g2的rbp

    # switch to m->g0 & its stack, call fn
    #下面三条指令主要目的是找到g0的指针
    MOVQ  g(CX), BX         #BX = g
    MOVQ  g_m(BX), BX    #BX = g.m
    MOVQ  m_g0(BX), SI   #SI = g.m.g0

    #此刻，SI = g0， AX = g，所以这里在判断g 是否是 g0，如果g == g0则一定是哪里代码写错了
    CMPQ  SI, AX# if g == m->g0 call badmcall
    JNE  3(PC)
    MOVQ  $runtime·badmcall(SB), AX
    JMP  AX

    #把g0的地址设置到线程本地存储之中
    MOVQ  SI, g(CX)

    #恢复g0的栈顶指针到CPU的rsp积存，这一条指令完成了栈的切换，从g的栈切换到了g0的栈
    MOVQ  (g_sched+gobuf_sp)(SI), SP# rsp = g0->sched.sp

    #AX = g
    PUSHQ  AX   #fn的参数g入栈
    MOVQ  DI, DX   #DI是结构体funcval实例对象的指针，它的第一个成员才是goexit0的地址
    MOVQ  0(DI), DI   #读取第一个成员到DI寄存器
    CALL  DI   #调用goexit0(g)
    POPQ  AX
    MOVQ  $runtime·badmcall2(SB), AX
    JMP  AX
    RET
```

### 调度时机、策略

什么时候会发生调度？ 如何选择下一个要执行的goroutine？主动调度？被动调度？

#### 使用什么策略来挑选下一个进入运行的goroutine？



```go
// runtime/proc.go

// One round of scheduler: find a runnable goroutine and execute it.
// Never returns.
func schedule() {
    _g_ := getg()   //_g_ = m.g0

    ......

    var gp *g

    ......
   
    if gp == nil {
    // Check the global runnable queue once in a while to ensure fairness.
    // Otherwise two goroutines can completely occupy the local runqueue
    // by constantly respawning each other.
       //为了保证调度的公平性，每个工作线程每进行61次调度就需要优先从全局运行队列中获取goroutine出来运行，
       //因为如果只调度本地运行队列中的goroutine，则全局运行队列中的goroutine有可能得不到运行
        if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
            lock(&sched.lock) //所有工作线程都能访问全局运行队列，所以需要加锁
            gp = globrunqget(_g_.m.p.ptr(), 1) //从全局运行队列中获取1个goroutine
            unlock(&sched.lock)
        }
    }
    if gp == nil {
        //从与m关联的p的本地运行队列中获取goroutine
        gp, inheritTime = runqget(_g_.m.p.ptr())
        if gp != nil && _g_.m.spinning {
            throw("schedule: spinning with local work")
        }
    }
    if gp == nil {
        //如果从本地运行队列和全局运行队列都没有找到需要运行的goroutine，
        //则调用findrunnable函数从其它工作线程的运行队列中偷取，如果偷取不到，则当前工作线程进入睡眠，
        //直到获取到需要运行的goroutine之后findrunnable函数才会返回。
        gp, inheritTime = findrunnable() // blocks until work is available
    }

    ......

    //当前运行的是runtime的代码，函数调用栈使用的是g0的栈空间
    //调用execte切换到gp的代码和栈空间去运行
    execute(gp, inheritTime)  
}
```

本地队列是一个由runq、runqhead和runqtail这三个成员组成的一个无锁循环队列，需要学习下它是怎么实现的：

```go
// Get g from local runnable queue.
// If inheritTime is true, gp should inherit the remaining time in the
// current time slice. Otherwise, it should start a new time slice.
// Executed only by the owner P.
func runqget(_p_ *p) (gp *g, inheritTime bool) {
    // If there's a runnext, it's the next G to run.
    //从runnext成员中获取goroutine
    for {
        //查看runnext成员是否为空，不为空则返回该goroutine
        next := _p_.runnext  
        if next == 0 {
            break
        }
        if _p_.runnext.cas(next, 0) {
            return next.ptr(), true
        }
    }

    //从循环队列中获取goroutine
    for {
        h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
        t := _p_.runqtail
        if t == h {
            return nil, false
        }
        gp := _p_.runq[h%uint32(len(_p_.runq))].ptr()
        if atomic.CasRel(&_p_.runqhead, h, h+1) { // cas-release, commits consume
            return gp, false
        }
    }
}
```

无锁队列可参考：[无锁队列的实现](https://coolshell.cn/articles/8239.html)

#### 什么时候会发生调度？/ 调度时机

- goroutine执行某个操作因条件不满足需要等待而发生的调度，比如channel阻塞；

以读取channel阻塞为例，channel的读取操作会被编译器转化成对runtime.chanrecv1函数的调用：

```go
  // entry points for <- c from compiled code
  //go:nosplit
  func chanrecv1(c *hchan, elem unsafe.Pointer) {
      chanrecv(c, elem, true)
  }
```

chanrecv首先会判断channel是否有数据可读，如果有数据则直接读取并返回，但如果没有数据，则需要把当前goroutine挂入channel的读取队列之中并调用gopark函数阻塞该goroutine.

gopark则通过调用`mcall`从当前main goroutine切换到g0去执行`park_m`函数，可以看到`park_m`做的事情是解除当前正在运行的g后再调用`schedule`将控制权交给runtime：

```go
  // park continuation on g0.
func park_m(gp *g) {
    _g_ := getg()

      if trace.enabled {
          traceGoPark(_g_.m.waittraceev, _g_.m.waittraceskip)
      }

      casgstatus(gp, _Grunning, _Gwaiting)
      dropg()

      if fn := _g_.m.waitunlockf; fn != nil {
          ok := fn(gp, _g_.m.waitlock)
          _g_.m.waitunlockf = nil
          _g_.m.waitlock = nil
          if !ok {
              if trace.enabled {
                  traceGoUnpark(gp, 2)
              }
              casgstatus(gp, _Gwaiting, _Grunnable)
              execute(gp, true) // Schedule it back, never returns.
          }
      }
      schedule()
  }
```
- goroutine主动调用Gosched()函数让出CPU而发生的调度；

示例代码：

```go
package main

import (
    "runtime"
    "sync"
)

const N = 1

func main() {
    var wg sync.WaitGroup
 
    wg.Add(N)
    for i := 0; i < N; i++ {
        go start(&wg)
    }

    wg.Wait()
}

func start(wg *sync.WaitGroup) {
    for i := 0; i < 1000 * 1000 * 1000; i++ {
        runtime.Gosched()
    }

    wg.Done()
}
```

主动调度**完全是用户代码自己控制的**，我们根据代码就可以预见什么地方一定会发生调度，从主动调度的入口函数`Gosched()`开始分析：

```go
// Gosched yields the processor, allowing other goroutines to run. It does not
// suspend the current goroutine, so execution resumes automatically.
func Gosched() {
    checkTimeouts() //amd64 linux平台空函数
   
    //切换到当前m的g0栈执行gosched_m函数
    mcall(gosched_m)
    //再次被调度起来则从这里开始继续运行
}

// Gosched continuation on g0.
func gosched_m(gp *g) {
      if trace.enabled {
          traceGoSched()
      }
      goschedImpl(gp)
  }

func goschedImpl(gp *g) {
  status := readgstatus(gp)
  if status&^_Gscan != _Grunning {
      dumpgstatus(gp)
      throw("bad g status")
  }
  casgstatus(gp, _Grunning, _Grunnable)
  // 解除绑定关系
  dropg()
  lock(&sched.lock)
  // 放到全局队列中
  globrunqput(gp)
  unlock(&sched.lock)

  // 交出控制权
  schedule()
}
```

参考：[Go语言调度器之主动调度(20)](https://mp.weixin.qq.com/s?__biz=MzU1OTg5NDkzOA==&mid=2247483828&idx=1&sn=96efd1306fca9c524e440396bc61e0d8&scene=19#wechat_redirect)


- goroutine运行时间太长或长时间处于系统调用之中而被调度器剥夺运行权而发生的调度（抢占调度）

sysmon系统监控线程会定期（10毫秒）通过retake函数对goroutine发起抢占请求：

```go
// forcePreemptNS is the time slice given to a G before it is
// preempted.
const forcePreemptNS = 10 * 1000 * 1000 // 10ms

func retake(now int64) uint32 {
    n := 0
    // Prevent allp slice changes. This lock will be completely
    // uncontended unless we're already stopping the world.
    lock(&allpLock)
    // We can't use a range loop over allp because we may
    // temporarily drop the allpLock. Hence, we need to re-fetch
    // allp each time around the loop.
    for i := 0; i < len(allp); i++ { //遍历所有的P
        _p_ := allp[i]
        if _p_ == nil {
            // This can happen if procresize has grown
            // allp but not yet created new Ps.
            continue
        }
       
        //_p_.sysmontick用于sysmon线程记录被监控p的系统调用时间和运行时间
        pd := &_p_.sysmontick
        s := _p_.status
        if s == _Psyscall { //P处于系统调用之中，需要检查是否需要抢占
            // Retake P from syscall if it's there for more than 1 sysmon tick (at least 20us).
            t := int64(_p_.syscalltick)
            if int64(pd.syscalltick) != t {
                pd.syscalltick = uint32(t)
                pd.syscallwhen = now
                continue
            }
            // On the one hand we don't want to retake Ps if there is no other work to do,
            // but on the other hand we want to retake them eventually
            // because they can prevent the sysmon thread from deep sleep.
            if runqempty(_p_) &&  atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
                continue
            }
            // Drop allpLock so we can take sched.lock.
            unlock(&allpLock)
            // Need to decrement number of idle locked M's
            // (pretending that one more is running) before the CAS.
            // Otherwise the M from which we retake can exit the syscall,
            // increment nmidle and report deadlock.
            incidlelocked(-1)
            if atomic.Cas(&_p_.status, s, _Pidle) {
                if trace.enabled {
                    traceGoSysBlock(_p_)
                    traceProcStop(_p_)
                }
                n++
                _p_.syscalltick++
                handoffp(_p_)
            }
            incidlelocked(1)
            lock(&allpLock)
        } else if s == _Prunning { //P处于运行状态，需要检查其是否运行得太久了
            // Preempt G if it's running for too long.
            //_p_.schedtick：每发生一次调度，调度器++该值
            t := int64(_p_.schedtick)
            if int64(pd.schedtick) != t {
                //监控线程监控到一次新的调度，所以重置跟sysmon相关的schedtick和schedwhen变量
                pd.schedtick = uint32(t)
                pd.schedwhen = now
                continue
            }
           
            //pd.schedtick == t说明(pd.schedwhen ～ now)这段时间未发生过调度，
            //所以这段时间是同一个goroutine一直在运行，下面检查一直运行是否超过了10毫秒
            if pd.schedwhen+forcePreemptNS > now {
                //从某goroutine第一次被sysmon线程监控到正在运行一直运行到现在还未超过10毫秒
                continue
            }
            //连续运行超过10毫秒了，设置抢占请求
            preemptone(_p_)
        }
    }
    unlock(&allpLock)
    return uint32(n)
}
```

思路是，轮询所有的`p`，若其状态为`_Psyscall`或`_Prunning`，则检查它的运行时间是否大于等于10ms，若是则设置此刻正在运行的g的`抢占标志`，抢占标志的设置是通过`preemptone`函数，可以看到只是设置了两个字段，`gp.preempt`和`gp.stackguard0`， 其中`gp.stackguard0`字段非常重要，后面对于抢占的响应关键是靠它：

```go
func preemptone(_p_ *p) bool {
    mp := _p_.m.ptr()
    if mp == nil || mp == getg().m {
        return false
    }
    //gp是被抢占的goroutine
    gp := mp.curg
    if gp == nil || gp == mp.g0 {
        return false
    }

    gp.preempt = true  //设置抢占标志

    // Every call in a go routine checks for stack overflow by
    // comparing the current stack pointer to gp->stackguard0.
    // Setting gp->stackguard0 to StackPreempt folds
    // preemption into the normal stack overflow check.
    //stackPreempt是一个常量0xfffffffffffffade，是非常大的一个数
    gp.stackguard0 = stackPreempt  //设置stackguard0使被抢占的goroutine去处理抢占请求
    return true
}
```

既然设置了一些标志，那一定需要对这些标志进行处理，那么在哪里呢？在编译的时候，编译器会在每个函数的开头和结尾加一点自己的“配料”（即所谓的函数序言prologue），我们来反编译一个简单的程序看下：

```go
package main

import "fmt"

func sum(a, b int) int {
    a2 := a * a
    b2 := b * b
    c := a2 + b2

    fmt.Println(c)

    return c
}

func main() {
    sum(1, 2)
}
```

将main函数反编译后：

```
     // rcx = g
  => 0x0000000000486a80 <+0>:   mov   %fs:0xfffffffffffffff8,%rcx
     // 将从内存中读取相对于g偏移16这个地址中的内容与rsp寄存器的值比较
     // 那么g偏移16的地址是放的什么东西呢？是g结构体的stackguard0字段
     0x0000000000486a89 <+9>:   cmp   0x10(%rcx),%rsp
     // jbe的意思是：无符号小于等于就跳转，因此这两行的意思是在比较栈顶寄存器rsp的值是否比stackguard0的值小，
     // 如果rsp的值更小，说明当前g的栈要用完了，有溢出风险，需要扩栈，
     // 而假设main goroutine被设置了抢占标志，那么rsp的值就会远远小于stackguard0，因为从上一节的分析我们知道sysmon监控线程在设置
     // 抢占标志时把需要被抢占的goroutine的stackguard0成员设置成了0xfffffffffffffade，而对于goroutine来说其rsp栈顶不可能这么大
     // 因此，当设置抢占标志时一定会发生跳转
     0x0000000000486a8d <+13>:  jbe   0x486abd <main.main+61>
     0x0000000000486a8f <+15>:  sub   $0x20,%rsp
     0x0000000000486a93 <+19>: mov   %rbp,0x18(%rsp)
     0x0000000000486a98 <+24>: lea   0x18(%rsp),%rbp
     0x0000000000486a9d <+29>: movq   $0x1,(%rsp)
     0x0000000000486aa5 <+37>: movq   $0x2,0x8(%rsp)
     0x0000000000486aae <+46>: callq   0x4869c0 <main.sum>
     0x0000000000486ab3 <+51>: mov   0x18(%rsp),%rbp
     0x0000000000486ab8 <+56>: add   $0x20,%rsp
     0x0000000000486abc <+60>: retq
     // 跳转到这里  
     0x0000000000486abd <+61>: callq  0x44ece0 <runtime.morestack_noctxt>
     // 执行完morestack_noctxt后，又会跳回去继续执行正常逻辑
     0x0000000000486ac2 <+66>: jmp   0x486a80 <main.main>
```

注意前三行和倒数两行，再看看`morestack_noctxt`做了什么：

```
// morestack but not preserving ctxt.
TEXT runtime·morestack_noctxt(SB),NOSPLIT,$0
    MOVL  $0, DX
    JMP  runtime·morestack(SB)


// Called during function prolog when more stack is needed.
//
// The traceback routines see morestack on a g0 as being
// the top of a stack (for example, morestack calling newstack
// calling the scheduler calling newm calling gc), so we must
// record an argument size. For that purpose, it has no arguments.
TEXT runtime·morestack(SB),NOSPLIT,$0-0
    ......
    get_tls(CX)
    MOVQ  g(CX), SI  # SI = g(main goroutine对应的g结构体变量)
    ......
    #SP栈顶寄存器现在指向的是morestack_noctxt函数的返回地址，
    #所以下面这一条指令执行完成后AX = 0x0000000000486ac2
    MOVQ  0(SP), AX

    #下面两条指令给g.sched.PC和g.sched.g赋值，我们这个例子g.sched.PC被赋值为0x0000000000486ac2，
    #也就是执行完morestack_noctxt函数之后应该返回去继续执行指令的地址。
    MOVQ  AX, (g_sched+gobuf_pc)(SI) #g.sched.pc = 0x0000000000486ac2
    MOVQ  SI, (g_sched+gobuf_g)(SI) #g.sched.g = g

    LEAQ  8(SP), AX  #main函数在调用morestack_noctxt之前的rsp寄存器

    #下面三条指令给g.sched.sp，g.sched.bp和g.sched.ctxt赋值
    MOVQ  AX, (g_sched+gobuf_sp)(SI)
    MOVQ  BP, (g_sched+gobuf_bp)(SI)
    MOVQ  DX, (g_sched+gobuf_ctxt)(SI)
    #上面几条指令把g的现场保存了起来，下面开始切换到g0运行

    #切换到g0栈，并设置tls的g为g0
    #Call newstack on m->g0's stack.
    MOVQ  m_g0(BX), BX
    MOVQ  BX, g(CX)  #设置TLS中的g为g0
    #把g0栈的栈顶寄存器的值恢复到CPU的寄存器，达到切换栈的目的，下面这一条指令执行之前，
    #CPU还是使用的调用此函数的g的栈，执行之后CPU就开始使用g0的栈了
    MOVQ  (g_sched+gobuf_sp)(BX), SP
    CALL  runtime·newstack(SB)
    CALL  runtime·abort(SB)// crash if newstack returns
    RET
```

`morestack_noctxt`函数执行的流程类似于前面我们分析过的mcall函数，首先保存调用morestack函数的goroutine（我们这个场景是main goroutine）的调度信息到对应的g结构的sched成员之中，然后切换到当前工作线程的g0栈继续执行newstack函数。

```
// runtime/stack.go

// Called from runtime·morestack when more stack is needed.
// Allocate larger stack and relocate to new stack.
// Stack growth is multiplicative, for constant amortized cost.
//
// g->atomicstatus will be Grunning or Gscanrunning upon entry.
// If the GC is trying to stop this g then it will set preemptscan to true.
//
// This must be nowritebarrierrec because it can be called as part of
// stack growth from other nowritebarrierrec functions, but the
// compiler doesn't check this.
//
//go:nowritebarrierrec
func newstack() {
    thisg := getg() // thisg = g0
    ......
    // 这行代码获取g0.m.curg，也就是需要扩栈或响应抢占的goroutine
    // 对于我们这个例子gp = main goroutine
    gp := thisg.m.curg
    ......
    // NOTE: stackguard0 may change underfoot, if another thread
    // is about to try to preempt gp. Read it just once and use that same
    // value now and below.
    //检查g.stackguard0是否被设置为stackPreempt
    preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt

    // Be conservative about where we preempt.
    // We are interested in preempting user Go code, not runtime code.
    // If we're holding locks, mallocing, or preemption is disabled, don't
    // preempt.
    // This check is very early in newstack so that even the status change
    // from Grunning to Gwaiting and back doesn't happen in this case.
    // That status change by itself can be viewed as a small preemption,
    // because the GC might change Gwaiting to Gscanwaiting, and then
    // this goroutine has to wait for the GC to finish before continuing.
    // If the GC is in some way dependent on this goroutine (for example,
    // it needs a lock held by the goroutine), that small preemption turns
    // into a real deadlock.
    if preempt {
        //检查被抢占goroutine的状态
        if thisg.m.locks != 0 || thisg.m.mallocing != 0 || thisg.m.preemptoff != "" ||  thisg.m.p.ptr().status != _Prunning {
            // Let the goroutine keep running for now.
            // gp->preempt is set, so it will be preempted next time.
            //还原stackguard0为正常值，表示我们已经处理过抢占请求了
            gp.stackguard0 = gp.stack.lo + _StackGuard
           
            //不抢占，调用gogo继续运行当前这个g，不需要调用schedule函数去挑选另一个goroutine
            gogo(&gp.sched) // never return
        }
    }

    //省略的代码做了些其它检查所以这里才有两个同样的判断

    if preempt {
        if gp == thisg.m.g0 {
            throw("runtime: preempt g0")
        }
        if thisg.m.p == 0 && thisg.m.locks == 0 {
            throw("runtime: g is running but p is not")
        }
        ......
        //下面开始响应抢占请求
        // Act like goroutine called runtime.Gosched.
        //设置gp的状态，省略的代码在处理gc时把gp的状态修改成了_Gwaiting
        casgstatus(gp, _Gwaiting, _Grunning)
       
        //调用gopreempt_m把gp切换出去
        gopreempt_m(gp) // never return
    }
    ......
}
```

`newstack` 里与抢占有关的代码如上，这里会检查抢占标志，若已设置这里会做真正的抢占动作，即 `gopreempt_m` --> `goschedImpl` --> `schedule`，最终将控制权交给runtime。

从函数命名上来看，其实抢占调度应该只算是它的一个“副业”，它的主业是扩栈，每次在函数的开头检查栈空间是否够用，若不够用则将栈扩大一倍（扩栈的代码在上面的代码里省略了）

下面再单独分析下系统调用的情况，它与正常执行时被抢占略微不同，我们先看看是如何进入系统调用的：

（怎么来找对应的代码呢？可以写一段包含系统调用的代码，比如`os.Open`，然后使用delve来下断点单步分析）

可以看到os.Open函数最终会调用到Syscall6函数（其它的系统调用不一定是这个函数，但类似）：

```asm
// syscall/asm_linux_amd64.s

// func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2, err uintptr)
TEXT ·Syscall6(SB), NOSPLIT, $0-80
    CALL  runtime·entersyscall(SB)

    #按照linux系统约定复制参数到寄存器并调用syscall指令进入内核
    MOVQ  a1+8(FP), DI
    MOVQ  a2+16(FP), SI
    MOVQ  a3+24(FP), DX
    MOVQ  a4+32(FP), R10
    MOVQ  a5+40(FP), R8
    MOVQ  a6+48(FP), R9
    MOVQ  trap+0(FP), AX#syscall entry，系统调用编号放入AX
    SYSCALL  #进入内核

    # 从内核返回，判断返回值，linux使用 -1 ~ -4095 作为错误码
    # 注意 JLS 是【无符号】小于或等于就跳转，若系统调用成功会
    # 返回 0，0一定小于 $0xfffffffffffff001，会跳到ok6继续
    # 执行
    CMPQ  AX, $0xfffffffffffff001
    JLS  ok6

    #系统调用返回错误，为Syscall6函数准备返回值
    MOVQ  $-1, r1+56(FP)
    MOVQ  $0, r2+64(FP)
    # 系统调用执行失败，错误码为负数，且放在AX中
    NEGQ  AX
    MOVQ  AX, err+72(FP)
    CALL  runtime·exitsyscall(SB)
    RET
ok6:      #系统调用返回成功
    MOVQ  AX, r1+56(FP)
    # DX其实是发起系统调用时传入的参数：a3
    MOVQ  DX, r2+64(FP)
    # 执行成功，所以err为零
    MOVQ  $0, err+72(FP)
    CALL  runtime·exitsyscall(SB)
    RET
```

可以看到，它在进入系统调用之前会先调用`runtime·entersyscall`，执行完成后会再调用`runtime·exitsyscall`

```go
// Standard syscall entry used by the go syscall library and normal cgo calls.
//go:nosplit
func entersyscall() {
    reentersyscall(getcallerpc(), getcallersp())
}

func reentersyscall(pc, sp uintptr) {
    _g_ := getg()  //执行系统调用的goroutine

    // Disable preemption because during this function g is in Gsyscall status,
    // but can have inconsistent g->sched, do not let GC observe it.
    _g_.m.locks++

    // Entersyscall must not call any function that might split/grow the stack.
    // (See details in comment above.)
    // Catch calls that might, by replacing the stack guard with something that
    // will trip any stack check and leaving a flag to tell newstack to die.
    _g_.stackguard0 = stackPreempt
    _g_.throwsplit = true

    // Leave SP around for GC and traceback.
    save(pc, sp)  //save函数分析过，用来保存g的现场信息，rsp, rbp, rip等等
    _g_.syscallsp = sp
    _g_.syscallpc = pc
    casgstatus(_g_, _Grunning, _Gsyscall) 
    ......
    _g_.m.syscalltick = _g_.m.p.ptr().syscalltick
    _g_.sysblocktraced = true
    _g_.m.mcache = nil
    pp := _g_.m.p.ptr()
    pp.m = 0  //p解除与m之间的绑定
    _g_.m.oldp.set(pp)   //把p记录在oldp中，等从系统调用返回时，优先绑定这个p
    _g_.m.p = 0  //m解除与p之间的绑定
    atomic.Store(&pp.status, _Psyscall)  //修改当前p的状态，sysmon线程依赖状态实施抢占
    .....
    _g_.m.locks--
}
```
`entersyscall` 直接调用了`reentersyscall`函数，`reentersyscall`首先把现场信息保存在当前g的sched成员中，然后解除m和p的绑定关系并设置p的状态为`_Psyscall`，前面我们已经看到sysmon监控线程需要依赖该状态实施抢占。继续分析sysmon监控线程中处理系统调用的部分代码：

```go
...
        if s == _Psyscall { //系统调用抢占处理
            // Retake P from syscall if it's there for more than 1 sysmon tick (at least 20us).
            //_p_.syscalltick用于记录系统调用的次数，主要由工作线程在完成系统调用之后++
            t := int64(_p_.syscalltick)
            if int64(pd.syscalltick) != t {
                //pd.syscalltick != _p_.syscalltick，说明已经不是上次观察到的系统调用了，
                //而是另外一次系统调用，所以需要重新记录tick和when值
                pd.syscalltick = uint32(t)
                pd.syscallwhen = now
                continue
            }
           
            //pd.syscalltick == _p_.syscalltick，说明还是之前观察到的那次系统调用，
            //计算这次系统调用至少过了多长时间了
           
            // On the one hand we don't want to retake Ps if there is no other work to do,
            // but on the other hand we want to retake them eventually
            // because they can prevent the sysmon thread from deep sleep.
            // 只要满足下面三个条件中的任意一个，则抢占该p，否则不抢占
            // 1. p的运行队列里面有等待运行的goroutine
            // 2. 没有无所事事的p
            // 3. 从上一次监控线程观察到p对应的m处于系统调用之中到现在已经超过10了毫秒
            if runqempty(_p_) &&  atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
                continue
            }
            // Drop allpLock so we can take sched.lock.
            unlock(&allpLock)
            // Need to decrement number of idle locked M's
            // (pretending that one more is running) before the CAS.
            // Otherwise the M from which we retake can exit the syscall,
            // increment nmidle and report deadlock.
            incidlelocked(-1)
            if atomic.Cas(&_p_.status, s, _Pidle) {
                ......
                _p_.syscalltick++
                handoffp(_p_)  //寻找一个新的m出来接管P
            }
...
```
可以看到最终会调用`handoffp`来将当前p移交给其它没有被阻塞的线程，具体看看`handoffp`的逻辑：

```go
// runtime/proc.go

// Hands off P from syscall or locked M.
// Always runs without a P, so write barriers are not allowed.
//go:nowritebarrierrec
func handoffp(_p_ *p) {
    // handoffp must start an M in any situation where
    // findrunnable would return a G to run on _p_.

    // if it has local work, start it straight away
    //运行队列不为空，需要启动m来接管
    if !runqempty(_p_) || sched.runqsize != 0 {
        startm(_p_, false)
        return
    }
    // if it has GC work, start it straight away
    //有垃圾回收工作需要做，也需要启动m来接管
    if gcBlackenEnabled != 0 && gcMarkWorkAvailable(_p_) {
        startm(_p_, false)
        return
    }
    // no local work, check that there are no spinning/idle M's,
    // otherwise our help is not required
    //所有其它p都在运行goroutine，说明系统比较忙，需要启动m
    if atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) == 0 &&  atomic.Cas(&sched.nmspinning, 0, 1) { // TODO: fast atomic
        startm(_p_, true)
        return
    }
    lock(&sched.lock)
    if sched.gcwaiting != 0 { //如果gc正在等待Stop The World
        _p_.status = _Pgcstop
        sched.stopwait--
        if sched.stopwait == 0 {
            notewakeup(&sched.stopnote)
        }
        unlock(&sched.lock)
        return
    }
    ......
    if sched.runqsize != 0 { //全局运行队列有工作要做
        unlock(&sched.lock)
        startm(_p_, false)
        return
    }
    // If this is the last running P and nobody is polling network,
    // need to wakeup another M to poll network.
    //不能让所有的p都空闲下来，因为需要监控网络连接读写事件
    if sched.npidle == uint32(gomaxprocs-1) && atomic.Load64(&sched.lastpoll) != 0 {
        unlock(&sched.lock)
        startm(_p_, false)
        return
    }
    pidleput(_p_)  //无事可做，把p放入全局空闲队列
    unlock(&sched.lock)
}
```
`handoffp`函数流程，它的主要任务是通过各种条件判断是否需要启动工作线程来接管当前的p，如果不需要则把p放入P的全局空闲队列。

最后我们看下当系统进程调用结束后，原来的goroutine是如何又被调度到cpu上的，从上面的`Syscall6`汇编代码中我们知道，最后调用到了`exitsyscall`：

```go
// runtime/proc.go

// The goroutine g exited its system call.
// Arrange for it to run on a cpu again.
// This is called only from the go syscall library, not
// from the low-level system calls used by the runtime.
//
// Write barriers are not allowed because our P may have been stolen.
//
//go:nosplit
//go:nowritebarrierrec
func exitsyscall() {
    _g_ := getg()
    ......
    oldp := _g_.m.oldp.ptr()  //进入系统调用之前所绑定的p
    _g_.m.oldp = 0
    if exitsyscallfast(oldp) {//因为在进入系统调用之前已经解除了m和p之间的绑定，所以现在需要绑定p
        //绑定成功，设置一些状态
        ......
       
        // There's a cpu for us, so we can run.
        _g_.m.p.ptr().syscalltick++  //系统调用完成，增加syscalltick计数，sysmon线程依靠它判断是否是同一次系统调用
        // We need to cas the status and scan before resuming...
        //casgstatus函数会处理一些垃圾回收相关的事情，我们只需知道该函数重新把g设置成_Grunning状态即可
        casgstatus(_g_, _Gsyscall, _Grunning)
        ......
        // 返回到用户代码继续执行
        return
    }
    ......
    _g_.m.locks--

    // Call the scheduler.
    //没有绑定到p，调用mcall切换到g0栈执行exitsyscall0函数
    mcall(exitsyscall0)
    ......
}
```
```go
// runtime/proc.go

// exitsyscall slow path on g0.
// Failed to acquire P, enqueue gp as runnable.
//
//go:nowritebarrierrec
func exitsyscall0(gp *g) {
    _g_ := getg()

    casgstatus(gp, _Gsyscall, _Grunnable)
   
    //当前工作线程没有绑定到p,所以需要解除m和g的关系
    dropg()
    lock(&sched.lock)
    var _p_ *p
    if schedEnabled(_g_) {
        _p_ = pidleget() //再次尝试获取空闲的p
    }
    if _p_ == nil { //还是没有空闲的p
        globrunqput(gp)  //把g放入全局运行队列
    } else if atomic.Load(&sched.sysmonwait) != 0 {
        atomic.Store(&sched.sysmonwait, 0)
        notewakeup(&sched.sysmonnote)
    }
    unlock(&sched.lock)
    if _p_ != nil {//获取到了p
        acquirep(_p_) //绑定p
        //继续运行g
        execute(gp, false) // Never returns.
    }
    if _g_.m.lockedg != 0 {
        // Wait until another thread schedules gp and so m again.
        stoplockedm()
        execute(gp, false) // Never returns.
    }
    stopm()  //当前工作线程进入睡眠，等待被其它线程唤醒
   
    //从睡眠中被其它线程唤醒，执行schedule调度循环重新开始工作
    schedule() // Never returns.
}
```
`exitsyscall`的基本思路是，先尝试获取一个p（优先尝试获取前面移交出去的p），若获取到了则直接返回到用户代码继续执行用户逻辑即可；否则调用mcall切换到g0栈执行`exitsyscall0`函数，
`exitsyscall0`还是会继续尝试获取空闲的p，若还是获取不到就会调用`stopm`将当前线程睡眠，等待被其它线程唤醒

参考：

- [抢占系统调用执行时间过长的goroutine](https://mp.weixin.qq.com/s?__biz=MzU1OTg5NDkzOA==&mid=2247483840&idx=1&sn=f2d7a78c190ff6ffce829c8938e50fe7&scene=19#wechat_redirect)
- [系统调用在 Golang 中的实践](https://wweir.cc/post/%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8%E5%9C%A8-golang-%E4%B8%AD%E7%9A%84%E5%AE%9E%E8%B7%B5/)
- [64-bit 快速系统调用](https://arthurchiao.art/blog/system-call-definitive-guide-zh/#42-64-bit-%E5%BF%AB%E9%80%9F%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8)

#### 工作线程的睡眠

工作线程在`findrunnable`中会尝试从其它p的队列中盗取g来做，若还是找不着的话就会进入睡眠状态（并不是像字面意思上理解的那样一直不停跑着spinning），工作线程的睡眠是通过调用`stopm`：

```go
  func stopm() {
      _g_ := getg()

      if _g_.m.locks != 0 {
          throw("stopm holding locks")
      }
      if _g_.m.p != 0 {
          throw("stopm holding p")
      }
      if _g_.m.spinning {
          throw("stopm spinning")
      }

      lock(&sched.lock)
      // 把m（就是自己）放入sched.midle空闲队列
      mput(_g_.m)
      unlock(&sched.lock)
      // 此处让m（就是自己）进入睡眠
      mPark()
      acquirep(_g_.m.nextp.ptr())
      _g_.m.nextp = 0
  }
```
继续看`mPark`：

```go
  // mPark causes a thread to park itself - temporarily waking for
  // fixups but otherwise waiting to be fully woken. This is the
  // only way that m's should park themselves.
  //go:nosplit
  func mPark() {
      g := getg()
      for {
          notesleep(&g.m.park)
          noteclear(&g.m.park)
          if !mDoFixup() {
              return
          }
      }
  }
  
  // linux平台上的notesleep实现是基于futex系统调用
  func notesleep(n *note) {
    gp := getg()
    if gp != gp.m.g0 {
        throw("notesleep not on g0")
    }
    ns := int64(-1)
    if *cgo_yield != nil {
        // Sleep for an arbitrary-but-moderate interval to poll libc interceptors.
        ns = 10e6
    }
    // futex有可能意外苏醒，用一个循环来确保我们
    // 只有在被其它线程主动唤醒时才醒过来（意外苏醒时，又会立即
    // 进入睡眠）
    for atomic.Load(key32(&n.key)) == 0 {
        gp.m.blocked = true
        futexsleep(key32(&n.key), 0, ns)
        if *cgo_yield != nil {
            asmcgocall(*cgo_yield, nil)
        }
        gp.m.blocked = false
    }
}
```

关于futex这个系统调用，可以参考：

- [futex(2) — Linux manual page](https://man7.org/linux/man-pages/man2/futex.2.html) 
- [Basics of Futexes](https://eli.thegreenplace.net/2018/basics-of-futexes/)
- [How different is a futex from mutex - conceptually and also implementation wise?](https://www.quora.com/How-different-is-a-futex-from-mutex-conceptually-and-also-implementation-wise)

futex是Fast userspace mutex的缩写，字面意思上看，是更快的用户态mutex锁，不过不能让这个名字给迷惑了，futex系统调用本身只能提供将线程睡眠和唤醒的作用，实际中使用要搭配原子操作（atomic operation）：

它相对于普通的mutex（锁）的一个优化是，基于一个观察：在大部分情况下，锁的获取是没有竞争的。那么，我们可以做的一个优化是，在获取锁的时候先尝试使用atomic operation，若成功了，则根本不需要系统调用了！（in most cases, locks are actually not contended. If a thread comes upon a free lock, locking it can be cheap because most likely no other thread is trying to lock it at the exact same time. So we can get by without a system call, attemping much cheaper atomic operations first [2]. There's a very high chance that the atomic instruction will succeed）

若atomic operation没成功，则说明已经有别的线程持有锁了，再调用futex来将线程睡眠。

在`parkM`这个场景，就是用到了futex的睡眠和唤醒的作用，并没有锁的概念。

#### 工作线程的唤醒与创建

在需要唤醒一个g来运行时（也即把g调度到cpu上运行），首先肯定是把g放到运行队列上（不管是localQ
还是globalQ），还会做的一步是，确认是否需要再启动额外的线程（比如，若当前所有的线程都因系统调用被hold住了）以及是否可以再启动额外的线程（比如，若当前已经启动了`GOMAXPROCS`个线程了），若需要，则会唤醒之前睡眠的线程或者启动新的线程。

```go
  // Mark gp ready to run.
  func ready(gp *g, traceskip int, next bool) {
      if trace.enabled {
          traceGoUnpark(gp, traceskip)
      }

      status := readgstatus(gp)

      // Mark runnable.
      _g_ := getg()
      mp := acquirem() // disable preemption because it can be holding p in a local var
      if status&^_Gscan != _Gwaiting {
          dumpgstatus(gp)
          throw("bad g->status in ready")
      }

      // status is Gwaiting or Gscanwaiting, make Grunnable and put on runq
      casgstatus(gp, _Gwaiting, _Grunnable)
      // 把g放到运行队列
      runqput(_g_.m.p.ptr(), gp, next)
      // 若需要，尝试唤醒线程或者启动新的线程
      wakep()
      releasem(mp)
  }
```

```go
  // Tries to add one more P to execute G's.
  // Called when a G is made runnable (newproc, ready).
  func wakep() {
      // 若没有空闲的p了，则直接返回
      if atomic.Load(&sched.npidle) == 0 {
          return
      }
      // be conservative about spinning threads
      // 若已经有其它线程在尝试“偷”任务了，也直接返回
      if atomic.Load(&sched.nmspinning) != 0 || !atomic.Cas(&sched.nmspinning, 0, 1) {
          return
      }
      startm(nil, true)
  }
```
下面是做真正的唤醒和新建线程的操作：

```go
// Schedules some M to run the p (creates an M if necessary).
// If p==nil, tries to get an idle P, if no idle P's does nothing.
// May run with m.p==nil, so write barriers are not allowed.
// If spinning is set, the caller has incremented nmspinning and startm will
// either decrement nmspinning or set m.spinning in the newly started M.
//go:nowritebarrierrec
func startm(_p_ *p, spinning bool) {
    lock(&sched.lock)
    if _p_ == nil { //没有指定p的话需要从p的空闲队列中获取一个p
        _p_ = pidleget() //从p的空闲队列中获取空闲p
        if _p_ == nil {
            unlock(&sched.lock)
            if spinning {
                // The caller incremented nmspinning, but there are no idle Ps,
                // so it's okay to just undo the increment and give up.
                //spinning为true表示进入这个函数之前已经对sched.nmspinning加了1，需要还原
                if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
                    throw("startm: negative nmspinning")
                }
            }
            return //没有空闲的p，直接返回
        }
    }
    mp := mget() //从m空闲队列中获取正处于睡眠之中的工作线程，所有处于睡眠状态的m都在此队列中
    unlock(&sched.lock)
    if mp == nil {
        //没有处于睡眠状态的工作线程
        var fn func()
        if spinning {
            // The caller incremented nmspinning, so set m.spinning in the new M.
            fn = mspinning
        }
        newm(fn, _p_) //创建新的工作线程
        return
    }
    if mp.spinning {
        throw("startm: m is spinning")
    }
    if mp.nextp != 0 {
        throw("startm: m has p")
    }
    if spinning && !runqempty(_p_) {
        throw("startm: p has runnable gs")
    }
    // The caller incremented nmspinning, so set m.spinning in the new M.
    mp.spinning = spinning
    mp.nextp.set(_p_)
   
    //唤醒处于休眠状态的工作线程
    notewakeup(&mp.park)
}
```

至此，处于休眠状态的线程被唤醒。

下面分析一下，若没有休眠的线程，如何调用`newm`创建新的线程：

```go
// runtime/proc.go

// Create a new m. It will start off with a call to fn, or else the scheduler.
// fn needs to be static and not a heap allocated closure.
// May run with m.p==nil, so write barriers are not allowed.
//go:nowritebarrierrec
func newm(fn func(), _p_ *p) {
    // 从堆上分配一个m结构体对象
    mp := allocm(_p_, fn)
    mp.nextp.set(_p_)
    ......
    newm1(mp)
}
```
```go
// runtime/proc.go
func newm1(mp *m) {
      //省略cgo相关代码.......
      execLock.rlock() // Prevent process clone.
      newosproc(mp)
      execLock.runlock()
}
```
```go
// runtime/os_linux.go

// May run with m.p==nil, so write barriers are not allowed.
//go:nowritebarrier
func newosproc(mp *m) {
    stk := unsafe.Pointer(mp.g0.stack.hi)                    
    ......
    // 通过调用clone系统调用来创建新线程
    // 注意传入的函数是`mstart`，它会在线程启动后首先执行
    ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0),         unsafe.Pointer(funcPC(mstart)))
    ......
}
//clone系统调用的Flags选项
cloneFlags = _CLONE_VM | /* share memory */ //指定父子线程共享进程地址空间
  _CLONE_FS | /* share cwd, etc */
  _CLONE_FILES | /* share fd table */
  _CLONE_SIGHAND | /* share sig handler table */
  _CLONE_SYSVSEM | /* share SysV semaphore undo lists (see issue #20763) */
  _CLONE_THREAD /* revisit - okay for now */  //创建子线程而不是子进程
```

`clone`的过程也很精彩

```asm
// runtime/sys_linux_amd64.s

// int32 clone(int32 flags, void *stk, M *mp, G *gp, void (*fn)(void));
TEXT runtime·clone(SB),NOSPLIT,$0
    MOVL  flags+0(FP), DI //系统调用的第一个参数
    MOVQ  stk+8(FP), SI   //系统调用的第二个参数
    MOVQ  $0, DX         //第三个参数
    MOVQ  $0, R10         //第四个参数

    // Copy mp, gp, fn off parent stack for use by child.
    // Careful: Linux system call clobbers CX and R11.
    MOVQ  mp+16(FP), R8
    MOVQ  gp+24(FP), R9
    MOVQ  fn+32(FP), R12

    MOVL  $SYS_clone, AX
    SYSCALL
    
    // In parent, return.
    CMPQ  AX, $0  #判断clone系统调用的返回值
    JEQ  3(PC) / #跳转到子线程部分
    MOVL  AX, ret+40(FP) #父线程需要执行的指令
    RET  #父线程需要执行的指令
    
    # In child, on new stack.
    #子线程需要继续执行的指令
    MOVQ  SI, SP  #设置CPU栈顶寄存器指向子线程的栈顶，这条指令看起来是多余的？内核应该已经把SP设置好了

    # If g or m are nil, skip Go-related setup.
    CMPQ  R8, $0    # m，新创建的m结构体对象的地址，由父线程保存在R8寄存器中的值被复制到了子线程
    JEQ  nog
    CMPQ  R9, $0    # g，m.g0的地址，由父线程保存在R9寄存器中的值被复制到了子线程
    JEQ  nog

    # Initialize m->procid to Linux tid
    MOVL  $SYS_gettid, AX  #通过gettid系统调用获取线程ID（tid）
    SYSCALL
    MOVQ  AX, m_procid(R8)  #m.procid = tid

    #Set FS to point at m->tls.
    #新线程刚刚创建出来，还未设置线程本地存储，即m结构体对象还未与工作线程关联起来，
    #下面的指令负责设置新线程的TLS，把m对象和工作线程关联起来
    LEAQ  m_tls(R8), DI  #取m.tls字段的地址
    CALL  runtime·settls(SB)

    #In child, set up new stack
    get_tls(CX)
    MOVQ  R8, g_m(R9)  # g.m = m
    MOVQ  R9, g(CX)      # tls.g = &m.g0
    CALL  runtime·stackcheck(SB)

nog:
    # Call fn
    CALL  R12  #这里调用mstart函数
    ......
```

回忆一下，mstart函数首先会去设置`m.g0`的`stackguard`成员，然后调用`mstart1()`函数把当前工作线程的`g0`的调度信息保存在`m.g0.sched`成员之中，最后通过调用`schedule`函数进入调度循环。

具体参考：[工作线程的唤醒及创建(19)](https://mp.weixin.qq.com/s?__biz=MzU1OTg5NDkzOA==&mid=2247483822&idx=1&sn=82e9328c3bb1d1153145261d932ab54b&scene=19#wechat_redirect)
