+++
title = "csapp machine level representation of programs"
date = "2016-09-03"
slug = "2016/09/03/csapp-machine-level-representation-of-programs"
draft = true
Categories = []
+++

Computers execute machine code, sequences of bytes encoding the low-level op- erations that manipulate data, manage memory, read and write data on storage devices, and communicate over networks. A compiler generates machine code through a series of stages, based on the rules of the programming language, the instruction set of the target machine, and the conventions followed by the operat- ing system. The gcc C compiler generates its output in the form of assembly code, a textual representation of the machine code giving the individual instructions in the program. gcc then invokes both an assembler and a linker to generate the exe- cutable machine code from the assembly code.

When programming in a high-level language such as C, and even more so in Java, we are shielded from the detailed, machine-level implementation of our pro- gram. In contrast, when writing programs in assembly code (as was done in the early days of computing) a programmer must specify the low-level instructions the program uses to carry out a computation. Most of the time, it is much more produc- tive and reliable to work at the higher level of abstraction provided by a high-level language. The type checking provided by a compiler helps detect many program errors and makes sure we reference and manipulate data in consistent ways. With modern, optimizing compilers, the generated code is usually at least as efficient as what a skilled, assembly-language programmer would write by hand. Best of all, a program written in a high-level language can be compiled and executed on a number of different machines, whereas assembly code is highly machine specific.

Relative to the computations expressed in the C code, optimizing compilers can rearrange execution order, eliminate unneeded computations, replace slow operations with faster ones, and even change recursive computations into iterative ones. Under- standing the relation between source code and the generated assembly can often be a challenge—it’s much like putting together a puzzle having a slightly differ- ent design than the picture on the box. It is a form of reverse engineering—trying to understand the process by which a system was created by studying the system and working backward. 编译器优化、逆向工程

###Program Encodings

####Machine-Level Code

computer systems employ several different forms of abstraction, hiding details of an implementation through the use of a simpler, abstract model. 

Two of these are especially important for machine-level programming. First, the format and behavior of a machine-level program is de- fined by the instruction set architecture, or “ISA,” defining the processor state, the format of the instructions, and the effect each of these instructions will have on the state.  一个是指令集

Second, the memory addresses used by a machine-level program are vir- tual addresses, providing a memory model that appears to be a very large byte array. The actual implementation of the memory system involves a combination of multiple hardware memories and operating system software 还有一个是虚拟地址

The assembly-code representation is very close to machine code. Its main feature is that it is in a more readable textual format, as compared to the binary format of machine code. Being able to understand assembly code and how it relates to the original C code is a key step in understanding how computers execute programs. 汇编语言跟机器语言基本一样

IA32 machine code differs greatly from the original C code. Parts of the processor state are visible that normally are hidden from the C programmer（机器语言跟C语言的区别，就是更底层,下面这些在汇编中都是可见的，也意味着你要自己处理和关心）:

- The program counter (commonly referred to as the “PC,” and called %eip in IA32) indicates the address in memory of the next instruction to be executed.

- The integer register file contains eight named locations storing 32-bit values. 数据寄存器

- The condition code registers hold status information about the most recently executed arithmetic or logical instruction. These are used to implement con- ditional changes in the control or data flow, such as is required to implement if and while statements. 状态寄存器

- A set of floating-point registers store floating-point data. 浮点寄存器

Whereas C provides a model in which objects of different data types can be declared and allocated in memory, machine code views the memory as simply a large, byte-addressable array. Aggregate data types in C such as arrays and structures are represented in machine code as contiguous collections of bytes. Even for scalar data types, assembly code makes no distinctions between signed or unsigned integers, between different types of pointers, or even between pointers and integers. 机器语言没有数据类型，更不用说数据结构的概念了。

Due to its origins as a 16-bit architecture that expanded into a 32-bit one, Intel uses the term “word” to refer to a 16-bit data type. Based on this, they refer to 32- bit quantities as “double words.” They refer to 64-bit quantities as “quad words.” Most instructions we will encounter operate on bytes or double words. 由于历史原因，16位称为一个字，32位为double words，64位为quad words

####Accessing Information

the low-order 2 bytes of the first four registers can be independently read or written by the byte operation instructions. This feature was provided in the 8086 to allow backward compatibility to the 8008 and 8080—two 8-bit microprocessors that date back to 1974. When a byte instruction updates one of these single-byte “register elements,” the remaining 3 bytes of the register do not change. Similarly, the low-order 16 bits of each register can be read or written by word operation instructions. This feature stems from IA32’s evolutionary heritage as a 16-bit microprocessor and is also used when operating on integers with size designator short. 开始的4个寄存器的低端的8位和16位可以被单独使用。

Operand Specifiers  这应该是指寻址方式了

 Source values can be given as constants or read from registers or memory. Results can be stored in either registers or memory. Thus, the different operand possibilities can be classified into three types. 主要分三类，The first type, immediate, is for constant values.（直接存取），开头用`$`，比如`$0x1F`；The second type, register, denotes the contents of one of the registers，寄存器方式，当然是直接用寄存器的名字来表示了，比如`%eax`；The third type of operand is a memory reference, in which we access some memory location according to a computed address, often called the effective ad- dress. 内存寻址，there are many different addressing modes allowing dif- ferent forms of memory references. 内存寻址的种类很多。

####Data Movement Instructions

就是各种mov pop push类的指令了，其中pop和push指令是用来操作程序栈的(program stack)

With IA32, the program stack is stored in some region of memory. 注意，栈是放在内存里的。

###Arithmetic and Logical Operations

The operations are divided into four groups: load effective address, unary, binary, and shifts. Binary operations have two operands, while unary operations have one operand. 计算和逻辑类指令，分为4类

####Load Effective Address

The load effective address instruction leal is actually a variant of the movl instruc- tion. It has the form of an instruction that reads from memory to a register, but it does not reference memory at all. Its first operand appears to be a memory refer- ence, but instead of reading from the designated location, the instruction copies the effective address to the destination. 跟mov干的事差不多。The load effective address (leal) instruction is commonly used to perform simple arithmetic. 可以用来做简单计算(这个貌似mov干不了)

####Unary and Binary Operations

对应于C语言中的`++`和`+=`类操作

####Shift Operations

左移、右移操作

###Control

####Condition Codes 状态码

In addition to the integer registers, the CPU maintains a set of single-bit condition code registers describing attributes of the most recent arithmetic or logical opera- tion. These registers can then be tested to perform conditional branches. The most useful condition codes are:

    CF: Carry Flag. The most recent operation generated a carry out of the most significant bit. Used to detect overflow for unsigned operations.
    ZF: Zero Flag. The most recent operation yielded zero.
    SF: Sign Flag. The most recent operation yielded a negative value.
    OF: Overflow Flag. The most recent operation caused a two’s-complement overflow—either negative or positive.

这些状态是由算术和逻辑运算指令设置的。除此之外，cmp和test指令也可以设置状态码

####Accessing the Condition Codes

Rather than reading the condition codes directly, there are three common ways of using the condition codes: (1) we can set a single byte to 0 or 1 depending on some combination of the condition codes, (2) we can conditionally jump to some other part of the program, or (3) we can conditionally transfer data.  condition codes的用法，并不是直接用。
