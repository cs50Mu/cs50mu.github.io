+++
title = "csapp system level io"
date = "2016-08-28"
slug = "2016/08/28/csapp-system-level-io"
Categories = []
+++

###概览
是什么？

Input/output (I/O) is the process of copying data between main memory and external devices such as disk drives, terminals, and networks. An input operation copies data from an I/O device to main memory, and an output operation copies data from memory to a device.

为什么？

All language run-time systems provide higher-level facilities for performing I/O. For example, ANSI C provides the standard I/O library, with functions such as printf and scanf that perform buffered I/O. The C++ language provides similar functionality with its overloaded << (“put to”) and >> (“get from”) operators. On Unix systems, these higher-level I/O functions are implemented using system-level Unix I/O functions provided by the kernel. **Most of the time, the higher-level I/O functions work quite well and there is no need to use Unix I/O directly. So why bother learning about Unix I/O?**

- Understanding Unix I/O will help you understand other systems concepts. I/O is integral to the operation of a system, and because of this we often encounter circular dependences between I/O and other systems ideas.

- Sometimes you have no choice but to use Unix I/O. There are some important cases where using higher-level I/O functions is either impossible or inappro- priate. For example, the standard I/O library provides no way to access file metadata such as file size or file creation time. Further, there are problems with the standard I/O library that make it risky to use for network programming.

###Unix I/O

A Unix file is a sequence of m bytes.

All I/O devices, such as networks, disks, and terminals, are modeled as files, and all input and output is performed by reading and writing the appropriate files. This elegant mapping of devices to files allows the Unix kernel to export a simple, low-level application interface, known as Unix I/O, that enables all input and output to be performed in a uniform and consistent way. 所有的io设备都被抽象成文件，对设备的输入输出操作被抽象成对文件的读和写操作。

针对文件的基本操作：

- Opening files. An application announces its intention to access an I/O device by asking the kernel to open the corresponding file. The kernel returns a small nonnegative integer, called a descriptor, that identifies the file in all subsequent operations on the file. The kernel keeps track of all information about the open file. The application only keeps track of the descriptor.

Each process created by a Unix shell begins life with three open files: standard input (descriptor 0), standard output (descriptor 1), and standard error (descriptor 2). The header file `<unistd.h>` defines constants `STDIN_ FILENO`, `STDOUT_FILENO`, and `STDERR_FILENO`, which can be used instead of the explicit descriptor values. 进程从出生自带三个打开的文件：标准输入、标准输出和标准错误输出。

- Changing the current file position. The kernel maintains a file position k, ini- tially 0, for each open file. The file position is a byte offset from the beginning of a file. An application can set the current file position k explicitly by per- forming a seek operation.

- Reading and writing files. A read operation copies n > 0 bytes from a file to memory, starting at the current file position k, and then incrementing k by n. Given a file with a size of m bytes, performing a read operation when k ≥ m triggers a condition known as end-of-file (EOF), which can be detected by the application. There is no explicit “EOF character” at the end of a file. 在文件中并不存在EOF这个字符，EOF是由操作系统触发的。

- Closing files. When an application has finished accessing a file, it informs the kernel by asking it to close the file. The kernel responds by freeing the data structures it created when the file was opened and restoring the descriptor to a pool of available descriptors. When a process terminates for any reason, the kernel closes all open files and frees their memory resources.

读或者写函数有时候返回的字节数要比你要求的要少，这并不代表有错，出现这种情形的原因有以下几点(In some situations, read and write transfer fewer bytes than the application requests. Such short counts do not indicate an error. They occur for a number of reasons)：

- Encountering EOF on reads. Suppose that we are ready to read from a file that contains only 20 more bytes from the current file position and that we are reading the file in 50-byte chunks. Then the next read will return a short count of 20, and the read after that will signal EOF by returning a short count of zero. 读的时候碰到EOF了。

- Reading text lines from a terminal. If the open file is associated with a terminal (i.e., a keyboard and display), then each read function will transfer one text line at a time, returning a short count equal to the size of the text line(A text line is a sequence of ASCII characters terminated by a newline character.). 从终端读的时候，是按行返回的，用户输入了几个字符就会返回几个。

- Reading and writing network sockets. If the open file corresponds to a network socket (Section 11.3.3), then internal buffering constraints and long network delays can cause read and write to return short counts. Short counts can also occur when you call read and write on a Unix pipe, an interprocess communication mechanism that is beyond our scope. 从socket返回时，由于buffer或者网络延迟。

In practice, you will never encounter short counts when you read from disk files except on EOF, and you will never encounter short counts when you write to disk files. However, if you want to build robust (reliable) network applications such as Web servers, then you must deal with short counts by repeatedly calling read and write until all requested bytes have been transferred. 实际在从硬盘文件中读的时候，除了碰到EOF其它时候都不应该遇到返回字节数少的情况；在往硬盘文件中写的时候，任何时候都不应该出现这种情况。


