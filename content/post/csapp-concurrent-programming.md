+++
title = "csapp concurrent programming"
date = "2016-08-29"
slug = "2016/08/29/csapp-concurrent-programming"
Categories = []
+++


Applications that use application-level concurrency are known as concurrent programs. Modern operating systems provide three basic approaches for building concurrent programs:

- Processes. With this approach, each logical control flow is a process that is scheduled and maintained by the kernel. Since processes have separate virtual address spaces, flows that want to communicate with each other must use some kind of explicit interprocess communication (IPC) mechanism.

- I/O multiplexing.This is a form of concurrent programming where **applications explicitly schedule their own logical flows in the context of a single process**. Logical flows are modeled as state machines that the main program explicitly transitions from state to state as a result of data arriving on file descriptors. Since the program is a single process, all flows share the same address space.

- Threads. Threads are logical flows that run in the context of a single process and are scheduled by the kernel. You can think of threads as a hybrid of the other two approaches, scheduled by the kernel like process flows, and sharing the same virtual address space like I/O multiplexing flows.

### IO之殇

Access to the filesystem or the network are really long operations from the perspective of the CPU. See the numbers from Jeffrey Dean’s talk Stanford CS295 class lecture, Spring, 2007.

    Operation     Latency
    L1 cache reference     0.5 ns
    Branch mispredict     5 ns
    L2 cache reference     7 ns
    Mutex lock/unlock     25 ns
    Main memory reference     100 ns
    Compress 1K bytes with Zippy     3,000 ns
    Send 2K bytes over 1 Gbps network     20,000 ns
    Read 1 MB sequentially from memory     250,000 ns
    Round trip within same datacenter     500,000 ns
    Disk seek     10,000,000 ns
    Read 1 MB sequentially from disk     20,000,000 ns
    Send packet CA->Netherlands->CA     150,000,000 ns

Blocking I/Os (or synchronous I/Os) will tie up the system resources as the waiting processes/threads cannot be used for something else. And from the CPU perspective, any I/O which is not done from/to memory takes ages. Fortunately the system itself is not stuck while I/Os are happening. The OS is going to preempt (i.e. interrupt) the process waiting for an I/O, allowing the CPU to be used by another process. But **this costs another context switch and meanwhile I/O intensive applications will spend most of their time… waiting!**

对于multi-process/threading来说

So, the more processes there are, the more they will compete for the CPU. **The more I/Os the application is doing, the more context switches there are, amplified by the number of process/threads the application is made of.** At some point, no matter how good the operating system is, it is going to become overwhelmed. It will spend most of its time switching contexts and have many processes/threads waiting either for I/O or to acquire the CPU. This basically means that **in such a model, the scalability is not at all linear to the number of processes/threads up to the CPU limit. The capacity gain of adding a process/thread significantly decreases with the number of active processes/threads.**

### Concurrent Programming with Processes

Pros and Cons of Processes

Processes have a clean model for sharing state information between parents and children: file tables are shared and user address spaces are not. Having separate address spaces for processes is both an advantage and a disadvantage. It is im- possible for one process to accidentally overwrite the virtual memory of another process, which eliminates a lot of confusing failures—an obvious advantage.

On the other hand, separate address spaces make it more difficult for pro- cesses to share state information. To share information, they must use explicit IPC (interprocess communications) mechanisms. (See Aside.) Another disadvan- tage of process-based designs is that they tend to be slower because the overhead for process control and IPC is high.

### Concurrent Programming with I/O Multiplexing

The select function manipulates sets of type `fd_set`, which are known as descriptor sets. Logically, we think of a descriptor set as a bit vector of size n: bn−1,...,b1,b0

Each bit bk corresponds to descriptor k. Descriptor k is a member of the descriptor set if and only if bk = 1.

the select function takes two inputs: a descriptor set (fdset) called the read set, and the cardinality (n) of the read set (actually the maximum cardinality of any descriptor set). **The select function blocks until at least one descriptor in the read set is ready for reading. A descriptor k is ready for reading if and only if a request to read 1 byte from that descriptor would not block.**select函数也是会阻塞的。

Question: IO多路复用是如何实现并发的效果的？

本质上是Event Driven Model来实现并发的，I/O Multiplexing只是必不可少的一环，有了它，使用Event Driven Model才成为可能。

我是这样理解的：在一个事件循环（event loop）中，不断的调用select来返回当前可读、可写的descriptor，然后做相应的处理，一直循环往复。在这个过程中的一个关键是，只有select操作是阻塞的，一旦它有返回，后面的操作就一定不是阻塞的，所以就不会有无谓的时间浪费在等待读和等待写上面。如果CPU的执行速度够快，那么从一个使用者的角度来看的话，就会看到并发的效果（而实质上这些“并发”的连接是按顺序被处理的）。这个道理现在看来，跟单核CPU上的多进程、多线程是一个道理（CPU-time multiplexing）。感觉实现并发效果的关键是：select的这个过程要足够快，然后对select返回的descriptor的处理也要快，否则就不会有并发的效果了。

#### Pros and Cons of I/O Multiplexing
- One advantage is that event-driven designs give programmers more control over the behavior of their programs than process-based designs. For example, we can imagine writing an event-driven concurrent server that gives preferred service to some clients, which would be difficult for a concurrent server based on processes. 程序员对自己的程序有更多的控制，比如优先提供某个服务，而不是像基于进程的并发那样，完全交给操作系统来决定。

- Another advantage is that an event-driven server based on I/O multiplexing runs in the context of a single process, and thus every logical flow has access to the entire address space of the process.This makes it easy to share data between flows. A related advantage of running as a single process is that you can debug your concurrent server as you would any sequential program, using a familiar debugging tool such as gdb. Finally, event-driven designs are often significantly more efficient than process-based designs because they do not require a process context switch to schedule a new flow. 由于是单进程，所以共享更方便；debug难度也比多进程要低；资源消耗也比多进程低（因为没有context switch）

- A significant disadvantage of event-driven designs is coding complexity. 缺点就是代码的复杂度上来了。

- Another significant disadvantage of event-based designs is that they cannot fully utilize multi-core processors. 还有一个缺点是不能利用多核

#### Event Driven Programming又是什么呢？
常常看到event driven和I/O Multiplexing这两个概念在一起。我是这样理解的，通过I/O Multiplexing可以提供event driven programming中的事件。

In an event model, everything runs in one process, one thread. Instead of spawning a new process/thread for each connection request, a event is emitted and the appropriate callback for that event is invoked. Once an event is treated, the process is ready to treat another event.

Such a model is particularly interesting if most of the activities can be turned into events. This becomes a really good concurrency and high-performance model when any I/Os (not just network I/O as is the most common in existing frameworks) are events. It is based on event patterns such as the Reactor or the Proactor which are patterns for Concurrent, Parallel, and Distributed Systems; documents from Douglas C. Schmidt. This event-driven concurrency model is superior to the traditional multithreaded/multi-process one: the memory footprint is drastically reduced, the CPU is better used and more clients can be served concurrently out of a single machine.

#### The Event Loop
To some extent, one can consider the event-driven approach being very similar to cooperative multitasking but at the application level. Event-driven applications are themselves multiplexing CPU time between clients.

There is obviously a risk with this; the same that with cooperative multitasking in fact. A risk which explains why at the OS level, preemptive multitasking is used. **If the process at some point can block for one client, then it will block all the other clients.** For example, in cooperative multitasking a non-responding process would make the system hang (Remember Windows before Windows 95 or Mac OS before Mac OS X ? )

In event-driven model, all the events are treated by a gigantic loop know as the event-loop. The event-loop is going to get from a queue the next event to process and will dispatch it the corresponding handler. Anyone blocking the event-loop will prevent the other events from being processed. So in Node (and in all event-driven framework) the golden rule is **“DO NOT BLOCK THE EVENT LOOP”.** Everything has to be non-blocking. And Node is particularly good at this because all the API it exposes is non-blocking (with the exception of some file system operations which come in two flavors: asynchronous and synchronous).

#### 参考
- [Node-Event-driven programming](http://www.baloo.io/blog/2013/11/30/node-event-driven-programming/)

### Concurrent Programming with Threads

A thread is a logical flow that runs in the context of a process. modern systems also allow us to write programs that have multiple threads running concurrently in a single process. The threads are scheduled automatically by the kernel. **Each thread has its own thread context, including a unique integer thread ID (TID), stack, stack pointer, program counter, general-purpose registers, and condition codes.** All threads running in a process share the entire virtual address space of that process.

#### Thread Execution Model

**Each process begins life as a single thread called the main thread.** At some point, the main thread creates a peer thread, and from this point in time the two threads run concurrently. Eventually, control passes to the peer thread via a context switch, because the main thread executes a slow system call such as read or sleep, or because it is interrupted by the system’s interval timer. The peer thread executes for a while before control passes back to the main thread, and so on.

Thread execution differs from processes in some important ways. Because a thread context is much smaller than a process context, a thread context switch is faster than a process context switch(由于thread context要比process context要小，所以上下文切换要比进程快). Another difference is that threads, unlike processes, are not organized in a rigid parent-child hierarchy. The threads associated with a process form a pool of peers, independent of which threads were created by which other threads. The main thread is distinguished from other threads only in the sense that it is always the first thread to run in the process. The main impact of this notion of a pool of peers is that a thread can kill any of its peers, or wait for any of its peers to terminate. Further, each peer can read and write the same shared data.(进程内的线程之间没有父子的继承关系，都是平等的，一个线程可以kill或者wait其它任何线程，主线程跟其它线程的唯一区别是它总是一个第一个被创建的)

#### Terminating Threads

A thread terminates in one of the following ways:

- The thread terminates implicitly when its top-level thread routine returns. 上层的thread返回退出了（看这的意思感觉又是有层级关系了，主线程退出了，由它发起的其它线程也会terminates implicitly

- The thread terminates explicitly by calling the "pthread exit" function. If the main thread calls "pthread exit", it waits for all other peer threads to terminate, and then terminates the main thread and the entire process with a return value of `thread_return`.

- Some peer thread calls the Unix exit function, which terminates the process and all threads associated with the process. 通过结束整个进程

- Another peer thread terminates the current thread by calling the "pthread cancel" function with the ID of the current thread. 被其它线程kill

#### Detaching Threads

At any point in time, a thread is joinable or detached. A joinable thread can be reaped and killed by other threads. Its memory resources (such as the stack) are not freed until it is reaped by another thread. In contrast, a detached thread cannot be reaped or killed by other threads. Its memory resources are freed automatically by the system when it terminates. detached状态的thread在退出后会被操作系统自动回收。

By default, threads are created joinable. In order to avoid memory leaks, each joinable thread should either be explicitly reaped by another thread, or detached by a call to the "pthread detach" function. 线程默认都是joinable的

#### Shared Variables in Threaded Programs

From a programmer’s perspective, one of the attractive aspects of threads is the ease with which multiple threads can share the same program variables. However, this sharing can be tricky. In order to write correctly threaded programs, **we must have a clear understanding of what we mean by sharing and how it works.**

Threads Memory Model

A pool of concurrent threads runs in the context of a process. Each thread has its own separate thread context, which includes a thread ID, stack, stack pointer, program counter, condition codes, and general-purpose register values. Each thread shares the rest of the process context with the other threads. **This includes the entire user virtual address space, which consists of read-only text (code), read/write data, the heap, and any shared library code and data areas. The threads also share the same set of open files.**

In an operational sense, **it is impossible for one thread to read or write the register values of another thread.** On the other hand, **any thread can access any location in the shared virtual memory.** If some thread modifies a memory location, then every other thread will eventually see the change if it reads that location. Thus, registers are never shared, whereas virtual memory is always shared.

#### Synchronizing Threads with Semaphores

Shared variables can be convenient, but they introduce the possibility of nasty **synchronization errors**

Semaphores

Semaphores provide a convenient way to ensure mutually exclusive access to shared variables. The basic idea is to associate a semaphore s, initially 1, with each shared variable (or related set of shared variables) and then surround the corresponding critical section with P (s) and V (s) operations.

- P (s): If s is nonzero, then P decrements s and returns immediately. If s is zero, then suspend the thread until s becomes nonzero and the process is restarted by a V operation. After restarting, the P operation decrements s and returns control to the caller.

- V (s): The V operation increments s by 1. If there are any threads blocked at a P operation waiting for s to become nonzero, then the V operation restarts exactly one of these threads, which then completes its P operation by decrementing s.

The test and decrement operations in P occur indivisibly, in the sense that once the semaphore s becomes nonzero, the decrement of s occurs without in- terruption. The increment operation in V also occurs indivisibly, in that it loads, increments, and stores the semaphore without interruption.  注意，关键是P或者V操作都是原子操作，不可分割，这是能够实现锁的关键。

The definitions of P and V ensure that a running program can never enter a state where a properly initialized semaphore has a negative value. This property, known as the semaphore invariant, provides a powerful tool for controlling the trajectories of concurrent programs

Synchronizing Threads with Semaphores

A semaphore that is used in this way to protect shared variables is called a binary semaphore because its value is always 0 or 1. **Binary semaphores whose purpose is to provide mutual exclusion are often called mutexes.** Performing a P operation on a mutex is called locking the mutex. Similarly, **performing the V operation is called unlocking the mutex. A thread that has locked but not yet unlocked a mutex is said to be holding the mutex. **

Using Semaphores to Schedule Shared Resources

Another important use of semaphores, besides providing mutual exclusion, is to **schedule accesses to shared resources.** In this scenario, a thread uses a semaphore operation to notify another thread that some condition in the program state has become true. Two classical and useful examples are the producer-consumer and readers-writers problems.

Producer-Consumer Problem

A producer and consumer thread share a bounded buffer with n slots. The producer thread repeatedly produces new items and inserts them in the buffer. The consumer thread repeat- edly removes items from the buffer and then consumes (uses) them. Variants with multiple producers and consumers are also possible.

Since inserting and removing items involves updating shared variables, we must guarantee mutually exclusive access to the buffer. But guaranteeing mutual exclusion is not sufficient. We also need to schedule accesses to the buffer. If the buffer is full (there are no empty slots), then the producer must wait until a slot becomes available. Similarly, if the buffer is empty (there are no available items), then the consumer must wait until an item becomes available. 不仅要保证对共享变量的独享读写，而且还有保证先后顺序，必须要先生产再消费。

#### Other Concurrency Issues

You probably noticed that life got much more complicated once we were asked to synchronize accesses to shared data. Synchronization is a fundamentally difficult problem that raises issues that simply do not arise in ordinary sequential programs.

Thread Safety

A function is said to be thread-safe if and only if it will always produce correct results when called repeatedly from multiple concurrent threads. If a function is not thread-safe, then we say it is thread-unsafe.

四类线程非安全的函数：

- Functions that do not protect shared variables. This class of thread-unsafe function is relatively easy to make thread-safe: protect the shared variables with synchronization operations such as P and V . An advantage is that it does not require any changes in the calling program. A disadvantage is that the synchronization operations will slow down the function. 解决方案，加锁。

- Functions that keep state across multiple invocations. A pseudorandom number generator is a simple example of this class of thread-unsafe function. The rand function is thread-unsafe because the result of the current invocation depends on an intermediate result from the previous iteration. When we call rand repeatedly from a single thread after seeding it with a call to srand, we can expect a repeatable sequence of numbers. However, this assumption no longer holds if multiple threads are calling rand. The only way to make a function such as rand thread-safe is to rewrite it so that it does not use any static data, relying instead on the caller to pass the state information in arguments. The disadvantage is that the programmer is now forced to change the code in the calling routine as well. 解决方案是不用static，而是通过caller传参数。

- Functions that return a pointer to a static variable. Some functions, such as ctime and gethostbyname, compute a result in a static variable and then return a pointer to that variable. If we call such functions from concurrent threads, then disaster is likely, as results being used by one thread are silently overwritten by another thread.

- Functions that call thread-unsafe functions. If a function f calls a thread-unsafe function g, is f thread-unsafe? It depends. If g is a class 2 function that relies on state across multiple invocations, then f is also thread- unsafe and there is no recourse short of rewriting g. However, if g is a class 1 or class 3 function, then f can still be thread-safe if you protect the call site and any resulting shared data with a mutex. 一个调用了线程非安全函数的函数是否是线程安全的呢？ 这得看情况。

Reentrancy 可重入性

There is an important class of thread-safe functions, known as reentrant functions, that are characterized by the property that they do not reference any shared data when they are called by multiple threads. 可重入函数是线程安全函数的子集，也就是说可重入函数一定是线程安全的，反之不成立。

Reentrant functions are typically more efficient than nonreentrant thread- safe functions because they require no synchronization operations.

Races  竞争

A race occurs when the correctness of a program depends on one thread reaching point x in its control flow before another thread reaches point y. Races usually occur because programmers assume that threads will take some particular trajec- tory through the execution state space, forgetting the golden rule that threaded programs must work correctly for any feasible trajectory.

The scary thing is that whether we get the correct answer depends on how the kernel sched- ules the execution of the threads. On our system it fails, but on other systems it might work correctly, leaving the programmer blissfully unaware of a serious bug. 此类bug不易复现，因而很难修复

Deadlocks 死锁

Semaphores introduce the potential for a nasty kind of run-time error, called deadlock, where a collection of threads are blocked, waiting for a condition that will never be true.

Deadlock is an especially difficult issue because it is not always predictable. Some lucky execution trajectories will skirt the deadlock region, while others will be trapped by it. The implications for a programmer are scary. You might run the same program 1000 times without any problem, but then the next time it deadlocks.Or the program might work fine on one machine but deadlock on another. Worst of all, the error is often not repeatable because different executions have different trajectories.
