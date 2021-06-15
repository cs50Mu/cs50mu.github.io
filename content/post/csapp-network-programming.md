+++
title = "csapp network programming"
date = "2016-08-29"
slug = "2016/08/29/csapp-network-programming"
Categories = []
+++

###The Client-Server Programming Model

Every network application is based on the client-server model. With this model, an application consists of a server process and one or more client processes. A server manages some resource, and it provides some service for its clients by manipulating that resource. 

###Networks

Clients and servers often run on separate hosts and communicate using the hard- ware and software resources of a computer network. 

To a host, **a network is just another I/O device that serves as a source and sink for data.** An adapter plugged into an expansion slot on the I/O bus provides the physical interface to the network. Data received from the network is copied from the adapter across the I/O and memory buses into memory, typically by a DMA transfer. Similarly, data can also be copied from memory to the network.

###The Socket Interface

Internet clients and servers communicate by sending and receiving streams of bytes over connections. A connection is point-to-point in the sense that it connects a pair of processes. It is full-duplex in the sense that data can flow in both directions **at the same time.** And it is reliable in the sense that—barring some catastrophic failure such as a cable cut by the proverbial careless backhoe operator—the stream of bytes sent by the source process is eventually received by the destination process in the same order it was sent.

A socket is an end point of a connection. Each socket has a corresponding socket address that consists of an Internet address and a 16-bit integer port, and is denoted by address:port. The port in the client’s socket address is assigned automatically by the kernel when the client makes a connection request, and is known as an ephemeral port. However, the port in the server’s socket address is typically some well-known port that is associated with the service. For example, Web servers typically use port 80, and email servers use port 25. 客户端的端口是由内核自动指定的，服务器的端口是事先手动定义好的。

A connection is uniquely identified by the socket addresses of its two end-points. This pair of socket addresses is known as a socket pair. 一个connection由两端的socket唯一确定。

The sockets interface is a set of functions that are used in conjunction with the Unix I/O functions to build network applications.

###Web Servers

Web clients and servers interact using a text-based application-level protocol known as HTTP (Hypertext Transfer Protocol). HTTP is a simple protocol. A Web client (known as a browser) opens an Internet connection to a server and requests some content. The server responds with the requested content and then closes the connection. The browser reads the content and displays it on the screen.

What distinguishes Web services from conventional file retrieval services such as FTP? The main difference is that Web content can be written in a language known as HTML (Hypertext Markup Language). An HTML program (page) contains instructions (tags) that tell the browser how to display various text and graphical objects in the page. HTTP服务跟其它web服务的区别

####Web Content

To Web clients and servers, content is a sequence of bytes with an associated MIME (Multipurpose Internet Mail Extensions) type. Web servers provide content to clients in two different ways:

- Fetch a disk file and return its contents to the client. The disk file is known as static content and the process of returning the file to the client is known as serving static content.

- Run an executable file and return its output to the client. The output produced by the executable at run time is known as dynamic content, and the process of running the program and returning its output to the client is known as serving dynamic content.


