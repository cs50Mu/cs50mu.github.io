+++
title = "csapp exceptional control flow"
date = "2016-08-26"
slug = "2016/08/26/csapp-exceptional-control-flow"
Categories = []
+++

###Chapter 8 Exceptional Control Flow

从一上电开始，CPU就在不停得执行指令，从第k个指令转到k+1个指令执行，叫做指令转移(control transfer)，一堆指令转移的集合就叫控制流(flow of control or control flow of the process)。

最简单的控制流就是一条接着一条的执行，只要执行的指令在内存中都是毗邻的，不需要跳转。当然，由于程序逻辑上的一些需求（比如分支、函数调用、调用返回）不可避免的会产生跳转。

但还有其它一些跳转需求，是跟程序内部执行逻辑无关的，比如硬件时钟、磁盘IO、创建子进程等(a hardware timer goes off at regular intervals and must be dealt with. Packets arrive at the network adapter and must be stored in memory. Programs request data from a disk and then sleep until they are notified that the data are ready. Parent processes that create child processes must be notified when their children terminate.) 对于这些情形，现代操作系统也是通过跳转来应对的。这些跳转被统称为exceptional control flow (ECF). ECF可以发生在一个计算机系统的各个层面，比如硬件层面、操作系统层面、应用程序层面等。(Exceptional control flow occurs at all levels of a computer system. For example, at the hardware level, events detected by the hardware trigger abrupt control transfers to exception handlers. At the operating systems level, the kernel transfers control from one user process to another via context switches. At the application level, a process can send a signal to another process that abruptly transfers control to a signal handler in the recipient. An individual program can react to errors by sidestepping the usual stack discipline and making nonlocal jumps to arbitrary locations in other functions)

那为什么要理解ECF呢？

- 有助于理解重要的操作系统概念。ECF is the basic mechanism that operating systems use to implement I/O, processes, and virtual memory. Before you can really understand these important ideas, you need to understand ECF.
- 有助于理解应用程序是如何跟操作系统交互的。Applications request services from the operating system by using a form of ECF known as a trap or system call. For example, writing data to a disk, reading data from a network, creating a new process, and terminating the current process are all accomplished by application programs invoking system calls. Understanding the basic system call mechanism will help you understand how these services are provided to applications.
- 有助于理解并发(concurrency)。ECF is a basic mechanism for implementing concurrency in computer systems. An exception handler that interrupts the execution of an application program, processes and threads whose execution overlap in time, and a signal handler that interrupts the execution of an application program are all examples of concurrency in action. Understanding ECF is a first step to understanding concurrency.
有助于理解软件层面的异常(software exceptions)是如何工作的。Languages such as C++ and Java provide software exception mechanisms via try, catch, and throw statements. Software exceptions allow the program to make nonlocal jumps (i.e., jumps that violate the usual call/return stack discipline) in response to error conditions. Nonlocal jumps are a form of application-level ECF, and are provided in C via the setjmp and longjmp functions. Understanding these low-level functions will help you understand how higher-level software exceptions can be implemented.

####8.1 Exceptions
异常部分是由硬件实现的，部分是由操作系统实现的(Exceptions are a form of exceptional control flow that are implemented partly by the hardware and partly by the operating system.)。

An exception is an abrupt change in the control flow in response to some change in the processor’s state. The change in state is known as an event. The event might be directly related to the execution of the current instruction. For example, a virtual memory page fault occurs, an arithmetic overflow occurs, or an instruction attempts a divide by zero. On the other hand, the event might be unrelated to the execution of the current instruction. For example, a system timer goes off or an I/O request completes.（状态的改变我们称之为事件，事件可能跟正在执行的指令相关，比如内存页错误；也可能与当前执行的指令不相干，比如CPU时间到了。）

不管是哪种情况，都会跳到对应的异常处理程序那里执行。(In any case, when the processor detects that the event has occurred, it makes an indirect procedure call (the exception), through a jump table called an exception table, to an operating system subroutine (the exception handler) that is specifically designed to process this particular kind of event.)

异常处理程序处理完以后，可能会发生以下三种情况的一种：

- The handler returns control to the current instruction Icurr, the instruction that was executing when the event occurred.
- The handler returns control to Inext, the instruction that would have executed next had the exception not occurred.
- The handler aborts the interrupted program.

#####Classes of Exceptions
异常可以被分为4类：interrupts, traps, faults, and aborts.
- Interrupts

Interrupts occur asynchronously as a result of signals from I/O devices that are external to the processor. Hardware interrupts are asynchronous in the sense that they are not caused by the execution of any particular instruction. Exception handlers for hardware interrupts are often called interrupt handlers.

这里讲到的asynchronous和synchronous的意思：Asynchronous exceptions occur as a result of events in I/O devices that are external to the processor. Synchronous exceptions occur as a direct result of executing an instruction.

interrupt的处理过程：

I/O devices such as network adapters, disk controllers, and timer chips trigger interrupts by signaling a pin on the processor chip and placing onto the system bus the exception number that identifies the device that caused the interrupt.

After the current instruction finishes executing, the processor notices that the interrupt pin has gone high, reads the exception number from the system bus, and then calls the appropriate interrupt handler. When the handler returns, it returns control to the next instruction (i.e., the instruction that would have followed the current instruction in the control flow had the interrupt not occurred). The effect is that the program continues executing as though the interrupt had never happened.

- Traps and System Calls

Traps

Traps are intentional exceptions that occur as a result of executing an instruction. Like interrupt handlers, trap handlers return control to the next instruction. The most important use of traps is to provide a procedure-like interface between user programs and the kernel known as a system call.

User programs often need to request services from the kernel such as reading a file (read), creating a new process (fork), loading a new program (execve), or terminating the current process (exit). To allow controlled access to such kernel services, processors provide a special “syscall n” instruction that user programs can execute when they want to request service n. Executing the syscall instruction causes a trap to an exception handler that decodes the argument and calls the appropriate kernel routine.

From a programmer’s perspective, a system call is identical to a regular function call. However, their implementations are quite different. Regular functions run in user mode, which restricts the types of instructions they can execute, and they access the same stack as the calling function. A system call runs in kernel mode, which allows it to execute instructions, and accesses a stack defined in the kernel. 

Faults

Faults result from error conditions that a handler might be able to correct. When a fault occurs, the processor transfers control to the fault handler. If the handler is able to correct the error condition, it returns control to the faulting instruction, thereby reexecuting it. Otherwise, the handler returns to an abort routine in the kernel that terminates the application program that caused the fault. 出现Fault后，要么经Fault handler处理后，原来的指令重新跑；要么直接Abort程序。

A classic example of a fault is the page fault exception, which occurs when an instruction references a virtual address whose corresponding physical page is not resident in memory and must therefore be retrieved from disk. As we will see in Chapter 9, a page is a contiguous block (typically 4 KB) of virtual memory. The page fault handler loads the appropriate page from disk and then returns control to the instruction that caused the fault. When the instruction executes again, the appropriate physical page is resident in memory and the instruction is able to run to completion without faulting.

Aborts

Aborts result from unrecoverable fatal errors, typically hardware errors such as parity errors that occur when DRAM or SRAM bits are corrupted. Abort handlers never return control to the application program. The handler returns control to an abort routine that terminates the application program. Abort handler不会把控制权返回给应用程序了。

#### Processes 进程

Exceptions are the basic building blocks that allow the operating system to provide the notion of a process, one of the most profound and successful ideas in computer science. 有了异常才能谈进程，异常机制是操作系统实现进程的基础。

When we run a program on a modern system, we are presented with the illusion that our program is the only one currently running in the system. Our program appears to have exclusive use of both the processor and the memory. The processor appears to execute the instructions in our program, one after the other, without interruption. Finally, the code and data of our program appear to be the only objects in the system’s memory. These illusions are provided to us by the notion of a process. 进程是个抽象概念。

Each program in the system runs in the context of some process. The context consists of the state that the program needs to run correctly. This state includes **the program’s code and data stored in memory, its stack, the contents of its general- purpose registers, its program counter, environment variables, and the set of open file descriptors.** 解释了进程上下文都包括哪些东西。

进程给应用程序提供了两个关键抽象：
>An independent logical control flow that provides the illusion that our pro- gram has exclusive use of the processor.
>
>A private address space that provides the illusion that our program has exclu- sive use of the memory system.

Logical Control Flow

The single physical control flow of the processor is partitioned into logical flows, one for each process. 

Concurrent Flows

A logical flow whose execution overlaps in time with another flow is called a concurrent flow, and the two flows are said to run concurrently. More precisely, flows X and Y are concurrent with respect to each other if and only if X begins after Y begins and before Y finishes, or Y begins after X begins and before X finishes.  并发并不是我们平常理解的那个意思，只要两个进程的逻辑流的执行时间有重叠，就算并发了。

Notice that the idea of concurrent flows is independent of the number of processor cores or computers that the flows are running on. If two flows overlap in time, then they are concurrent, even if they are running on the same processor. However, we will sometimes find it useful to identify a proper subset of concurrent flows known as parallel flows. If two flows are running concurrently on different processor cores or computers, then we say that they are parallel flows, that they are running in parallel, and have parallel execution. 解了我许久的一个困惑，那就是并发和并行到底是什么意思？？这里讲的就很清楚了，并发(concurrency)跟CPU核数是无关的，它只跟逻辑流的执行时间有关系，只要两个进程逻辑流的执行时间有重叠，那么他们就是并发的，即使它们是运行在同一个核上。并行(parallel)是并发的子集了，就是说，如果两个进程已经是并发了，而且还是运行在不同的核上或计算机上，那它们就是并行的了。

Private Address Space 私有内存空间

A process provides each program with its own private address space. This space is private in the sense that a byte of memory associated with a particular address in the space cannot in general be read or written by any other process.

User and Kernel Modes 用户模式和内核模式

In order for the operating system kernel to provide an airtight process abstraction, the processor must provide a mechanism that restricts the instructions that an application can execute, as well as the portions of the address space that it can access.

Processors typically provide this capability with a mode bit in some control register that characterizes the privileges that the process currently enjoys. When the mode bit is set, the process is running in kernel mode (sometimes called supervisor mode). A process running in kernel mode can execute any instruction in the instruction set and access any memory location in the system.

When the mode bit is not set, the process is running in user mode. A process in user mode is not allowed to execute privileged instructions that do things such as halt the processor, change the mode bit, or initiate an I/O operation. Nor is it allowed to directly reference code or data in the kernel area of the address space. Any such attempt results in a fatal protection fault. User programs must instead access kernel code and data indirectly via the system call interface.

A process running application code is initially in user mode. The only way for the process to change from user mode to kernel mode is via an exception such as an interrupt, a fault, or a trapping system call. When the exception occurs, and control passes to the exception handler, the processor changes the mode from user mode to kernel mode. The handler runs in kernel mode. When it returns to the application code, the processor changes the mode from kernel mode back to user mode.

Context Switches 上下文切换

The operating system kernel implements multitasking using a higher-level form of exceptional control flow known as a context switch. The context switch mecha- nism is built on top of the lower-level exception mechanism. 上下文切换也是基于异常做的。

The kernel maintains a context for each process. The context is the state that the kernel needs to restart a preempted process. It consists of the values of objects such as the general purpose registers, the floating-point registers, the program counter, user’s stack, status registers, kernel’s stack, and various kernel data structures such as a page table that characterizes the address space, a process table that contains information about the current process, and a file table that contains information about the files that the process has opened.

At certain points during the execution of a process, the kernel can decide to preempt the current process and restart a previously preempted process. This decision is known as scheduling, and is handled by code in the kernel called the scheduler. When the kernel selects a new process to run, we say that the kernel has scheduled that process. After the kernel has scheduled a new process to run, it preempts the current process and transfers control to the new process using a mechanism called a context switch that (1) saves the context of the current process, (2) restores the saved context of some previously preempted process, and (3) passes control to this newly restored process.

A context switch can occur while the kernel is executing a system call on behalf of the user. If the system call blocks because it is waiting for some event to occur, then the kernel can put the current process to sleep and switch to another process. For example, if a read system call requires a disk access, the kernel can opt to perform a context switch and run another process instead of waiting for the data to arrive from the disk. Another example is the sleep system call, which is an explicit request to put the calling process to sleep. In general, even if a system call does not block, the kernel can decide to perform a context switch rather than return control to the calling process.

A context switch can also occur as a result of an interrupt. For example, all systems have some mechanism for generating periodic timer interrupts, typically every 1 ms or 10 ms. Each time a timer interrupt occurs, the kernel can decide that the current process has run long enough and switch to a new process.

Creating and Terminating Processes 进程的创建和销毁

From a programmer’s perspective, we can think of a process as being in one of three states，进程的三种状态：

>Running. The process is either executing on the CPU or is waiting to be executed and will eventually be scheduled by the kernel. 注意这个running不一定是说正在跑，只是一个可调度的状态，叫ready更好一点。
>
>Stopped. The execution of the process is suspended and will not be scheduled. A process stops as a result of receiving a SIGSTOP, SIGTSTP, SIGTTIN, or SIGTTOU signal, and it remains stopped until it receives a SIGCONT signal, at which point it can begin running again. 不可调度状态，暂停了，但还能恢复
>
>Terminated. The process is stopped permanently. A process becomes termi- nated for one of three reasons: (1) receiving a signal whose default action is to terminate the process, (2) returning from the main routine, or (3) calling the exit function  永久结束了

Fork

A parent process creates a new running child process by calling the fork function. fork有以下特点：

    #include "csapp.h"

    int main()
    {
        pid_t pid;
        int x = 1;

        pid = Fork(); if(pid==0){ /*Child*/
            printf("child : x=%d\n", ++x);
            exit(0); }
        /* Parent */
        printf("parent: x=%d\n", --x);
        exit(0); }
    }

    output:
    unix> ./fork 
    parent: x=0 
    child : x=2

>Call once, return twice. The fork function is called once by the parent, but it returns twice: once to the parent and once to the newly created child. This is fairly straightforward for programs that create a single child. But programs with multiple instances of fork can be confusing and need to be reasoned about carefully.

>Concurrent execution. The parent and the child are separate processes that run concurrently. The instructions in their logical control flows can be inter- leaved by the kernel in an arbitrary way. When we run the program on our system, the parent process completes its printf statement first, followed by the child. However, on another system the reverse might be true. In general, as programmers we can never make assumptions about the interleaving of the instructions in different processes.

>Duplicate but separate address spaces. If we could halt both the parent and the child immediately after the fork function returned in each process, we would see that the address space of each process is identical. Each process has the same user stack, the same local variable values, the same heap, the same global variable values, and the same code. Thus, in our example program, local variable x has a value of 1 in both the parent and the child when the fork function returns in line 8. However, since the parent and the child are separate processes, they each have their own private address spaces. Any subsequent changes that a parent or child makes to x are private and are not reflected in the memory of the other process. This is why the variable x has different values in the parent and child when they call their respective printf statements.

>Shared files. When we run the example program, we notice that both parent and child print their output on the screen. The reason is that the child inherits all of the parent’s open files. When the parent calls fork, the stdout file is open and directed to the screen. The child inherits this file and thus its output is also directed to the screen.

Reaping Child Processes

When a process terminates for any reason, the kernel does not remove it from the system immediately. Instead, the process is kept around in a terminated state until it is reaped by its parent. When the parent reaps the terminated child, the kernel passes the child’s exit status to the parent, and then discards the terminated process, at which point it ceases to exist. A terminated process that has not yet been reaped is called a zombie. 已经是terminated状态但还没有被reaped的process叫僵尸程序

If the parent process terminates without reaping its zombie children, the kernel arranges for the init process to reap them. The init process has a PID of 1 and is created by the kernel during system initialization. Long-running programs such as shells or servers should always reap their zombie children. Even though zombies are not running, they still consume system memory resources. 父进程没收割的僵尸process会被init进程代为收割。

####Signals 信号

A signal is a small message that notifies a process that an event of some type has occurred in the system.

Each signal type corresponds to some kind of system event. **Low-level hard-ware exceptions are processed by the kernel’s exception handlers and would not normally be visible to user processes. Signals provide a mechanism for exposing the occurrence of such exceptions to user processes.** For example, if a process attempts to divide by zero, then the kernel sends it a SIGFPE signal (number 8). If a process executes an illegal instruction, the kernel sends it a SIGILL signal (number 4). If a process makes an illegal memory reference, the kernel sends it a SIGSEGV signal (number 11). Other signals correspond to higher-level soft- ware events in the kernel or in other user processes. For example, if you type a ctrl-c (i.e., press the ctrl key and the c key at the same time) while a process is running in the foreground, then the kernel sends a SIGINT (number 2) to the foreground process. A process can forcibly terminate another process by sending it a SIGKILL signal (number 9). When a child process terminates or stops, the kernel sends a SIGCHLD signal (number 17) to the parent. 信号为底层硬件异常与应用程序之间搭了一座桥梁。

信号的发和收

>Sending a signal. The kernel sends (delivers) a signal to a destination process by updating some state in the context of the destination process. The signal is delivered for one of two reasons: (1) The kernel has detected a system event such as a divide-by-zero error or the termination of a child process. (2) A process has invoked the kill function (discussed in the next section) to explicitly request the kernel to send a signal to the destination process. A process can send a signal to itself.
>
>Receiving a signal. A destination process receives a signal when it is forced by the kernel to react in some way to the delivery of the signal. The process can either ignore the signal, terminate, or catch the signal by executing a user-level function called a signal handler. Receipt of a signal triggers a control transfer to a signal handler. After it finishes processing, the handler returns control to the interrupted program. 接收信号也会打断正常的逻辑流。

A signal that has been sent but not yet received is called a pending signal. At any point in time, there can be **at most one pending signal of a particular type.** If a process has a pending signal of type k, then any subsequent signals of type k sent to that process are not queued; **they are simply discarded.**

A process can selectively **block the receipt of certain signals.** When a signal is blocked, it can be delivered, but the resulting pending signal will not be received until the process unblocks the signal. For each process, the kernel maintains the set of pending signals in the **pending bit vector**, and the set of blocked signals in the **blocked bit vector**. The kernel sets bit k in pending whenever a sig- nal of type k is delivered and clears bit k in pending whenever a signal of type k is received.

进程接收信号的流程

When the kernel is returning from an exception handler and is ready to pass control to process p, it checks the set of unblocked pending signals (pending & ~blocked) for process p. If this set is empty (the usual case), then the kernel passes control to the next instruction (Inext) in the logical control flow of p.

However, if the set is nonempty, then the kernel chooses some signal k in the set (typically the smallest k) and forces p to receive signal k. The receipt of the signal triggers some action by the process. Once the process completes the action, then control passes back to the next instruction (Inext ) in the logical control flow of p. Each signal type has a predefined default action, which is one of the following: 1) The process terminates. 2) The process terminates and dumps core. 3) The process stops until restarted by a SIGCONT signal. 4) The process ignores the signal. 一般收到信号后的默认行为是可以改的，但SIGSTOP和SIGKILL是不允许改的。

Signal Handling Issues 信号处理存在的问题

>Pending signals are blocked. Unix signal handlers typically block pending signals of the type currently being processed by the handler. For example, suppose a process has caught a SIGINT signal and is currently running its SIGINT handler. If another SIGINT signal is sent to the process, then the SIGINT will become pending, but will not be received until after the handler returns. 在等待的信号会被阻塞住，要等前面的信号处理完才能处理它。
>
>Pending signals are not queued. There can be at most one pending signal of any particular type. Thus, if two signals of type k are sent to a destination process while signal k is blocked because the destination process is currently executing a handler for signal k, then the second signal is simply discarded; it is not queued. The key idea is that the existence of a pending signal merely indicates that at least one signal has arrived. 最多只能pending一个信号，再多的信号就直接被抛弃了。
>
>System calls can be interrupted. System calls such as read, wait, and accept that can potentially block the process for a long period of time are called slow system calls. On some systems, slow system calls that are interrupted when a handler catches a signal do not resume when the signal handler returns, but instead return immediately to the user with an error condition and errno set to EINTR. 当收到信号后，就算系统调用也会被打断，关键是在某些系统上被打断的系统调用在signal handler返回后不会自动重启。

Explicitly Blocking and Unblocking Signals  显式block和unblock信号

Applications can explicitly block and unblock selected signals using the sigproc- mask function, The sigprocmask function changes the set of currently blocked signals.

Synchronizing Flows to Avoid Nasty Concurrency Bugs

并发容易引发bug, 并且很难定位。

The problem of how to program concurrent flows that read and write the same storage locations has challenged generations of computer scientists. In general,**the number of potential interleavings of the flows is exponential in the number of instructions. Some of those interleavings will produce correct answers, and others will not.** The fundamental problem is to somehow synchronize the concurrent flows so as to allow the largest set of feasible interleavings such that each of the feasible interleavings produces a correct answer.

Such errors are enormously difficult to debug because it is often impossible to test every interleaving. You may run the code a billion times without a problem, but then the next test results in an interleaving that triggers the race.

####Nonlocal Jumps

C provides a form of user-level exceptional control flow, called a nonlocal jump, that transfers control directly from one function to another currently executing function without having to go through the normal call-and-return sequence. Non- local jumps are provided by the setjmp and longjmp functions.

The setjmp function saves the current calling environment in the env buffer, for later use by longjmp, and returns a 0. The calling environment includes the program counter, stack pointer, and general purpose registers.

The longjmp function restores the calling environment from the env buffer and then triggers a return from the most recent setjmp call that initialized env. The setjmp then returns with the nonzero return value retval.

The interactions between setjmp and longjmp can be confusing at first glance. The setjmp function is called once, but returns multiple times: once when the setjmp is first called and the calling environment is stored in the env buffer, and once for each corresponding longjmp call. On the other hand, the longjmp function is called once, but never returns.  调用关系确实有点乱啊。

An important application of nonlocal jumps is to permit an immediate return from a deeply nested function call, usually as a result of detecting some error condition. If an error condition is detected deep in a nested function call, we can use a nonlocal jump to return directly to a common localized error handler instead of laboriously unwinding the call stack.  应用之一：从很深的调用嵌套中直接跳到表层，而不是一层一层的解嵌套跳出(unwind the entire stack)。

Another important application of nonlocal jumps is to branch out of a signal handler to a specific code location, rather than returning to the instruction that was interrupted by the arrival of the signal. 应用之一：从signal handler里跳出来。

The exception mechanisms provided by C++ and Java are higher-level, more-structured versions of the C setjmp and longjmp functions. You can think of a catch clause inside a try statement as being akin to a setjmp function. Similarly, a throw statement is similar to a longjmp function. C++和Java中的exception是基于setjmp和longjmp实现的。
