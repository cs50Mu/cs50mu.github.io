+++
title = "thrift source code walkthrough"
date = "2016-08-19"
slug = "2016/08/19/thrift-source-code-walkthrough"
Categories = []
+++

准备学习一下thrift的源码，看的是Python版的，以下所有分析都是基于0.9.0版本的thrift

thrift整个逻辑结构是分层的，类似于网络模型的分层，从下到上依次为Transport层，它封装了底层的socket；Protocol层在Transport层的基础上实现了传输协议；

### Transport
先看是如何用的，thrift client端的正确打开方式是:

```python
# Make socket
transport = TSocket.TSocket('localhost', 9090)

# Buffering is critical. Raw sockets are very slow
transport = TTransport.TBufferedTransport(transport)

# Wrap in a protocol
protocol = TBinaryProtocol.TBinaryProtocol(transport)

# Create a client to use the protocol encoder
client = Calculator.Client(protocol)

# Connect!
transport.open()

client.ping()
```
先看transport的初始化，其实是创建了一个socket，我们去源码里看一下到底是如何创建的，定位到`thrift.transport.TSocket`的TSocket类：

```python
class TSocket(TSocketBase):
  """Socket implementation of TTransport base."""

  def __init__(self, host='localhost', port=9090, unix_socket=None):
    """Initialize a TSocket

    @param host(str)  The host to connect to.
    @param port(int)  The (TCP) port to connect to.
    @param unix_socket(str)  The filename of a unix socket to connect to.
                             (host and port will be ignored.)
    """
    self.host = host
    self.port = port
    self.handle = None
    self._unix_socket = unix_socket
    self._timeout = None

  def setHandle(self, h):
    self.handle = h

  def isOpen(self):
    return self.handle is not None

  def setTimeout(self, ms):
    if ms is None:
      self._timeout = None
    else:
      self._timeout = ms / 1000.0

    if self.handle is not None:
      self.handle.settimeout(self._timeout)

  def open(self):
    try:
      res0 = self._resolveAddr()
      for res in res0:       #   res is of structure:   (family, socktype, proto, canonname, sockaddr)
        self.handle = socket.socket(res[0], res[1])    #  make a socket
        self.handle.settimeout(self._timeout)
        try:
          self.handle.connect(res[4])      #    connect
        except socket.error, e:
            if res is not res0[-1]:    #   if it's not the last, continue; or else raise exception
            continue
          else:
            raise e
        break                     #    if there if one socket that can be connected, then we are happy
    except socket.error, e:
      if self._unix_socket:
        message = 'Could not connect to socket %s' % self._unix_socket
      else:
        message = 'Could not connect to %s:%d' % (self.host, self.port)
      raise TTransportException(type=TTransportException.NOT_OPEN,
                                message=message)

  def read(self, sz):
    try:
      buff = self.handle.recv(sz)
    except socket.error, e:
      if (e.args[0] == errno.ECONNRESET and
          (sys.platform == 'darwin' or sys.platform.startswith('freebsd'))):
        # freebsd and Mach don't follow POSIX semantic of recv
        # and fail with ECONNRESET if peer performed shutdown.
        # See corresponding comment and code in TSocket::read()
        # in lib/cpp/src/transport/TSocket.cpp.
        self.close()
        # Trigger the check to raise the END_OF_FILE exception below.
        buff = ''
      else:
        raise
    if len(buff) == 0:
      raise TTransportException(type=TTransportException.END_OF_FILE,
                                message='TSocket read 0 bytes')
    return buff

  def write(self, buff):
    if not self.handle:
      raise TTransportException(type=TTransportException.NOT_OPEN,
                                message='Transport not open')
    sent = 0
    have = len(buff)
    while sent < have:
      plus = self.handle.send(buff)
      if plus == 0:
        raise TTransportException(type=TTransportException.END_OF_FILE,
                                  message='TSocket sent 0 bytes')
      sent += plus
      buff = buff[plus:]

  def flush(self):
    pass
```
比较重要的方法是open、read和write。open会初始化一个socket并且connect；read和write分别封装了底层socket库的recv和send方法。

再回到使用的角度上，注意它在初始的transport上又包了一个TBufferedTransport，这又是干嘛呢？正如注释中指出的那样，是给原始的socket接口包了一个buffer，
这样会减少对socket的读写，效率会高些，依然看下源码，定位到thrift.transport.TTransport的TBufferedTransport类：

```python
class TBufferedTransport(TTransportBase, CReadableTransport):
  """Class that wraps another transport and buffers its I/O.

  The implementation uses a (configurable) fixed-size read buffer
  but buffers all writes until a flush is performed.
  """
  DEFAULT_BUFFER = 4096

  def __init__(self, trans, rbuf_size=DEFAULT_BUFFER):
    self.__trans = trans
    self.__wbuf = StringIO()
    self.__rbuf = StringIO("")
    self.__rbuf_size = rbuf_size

  def isOpen(self):
    return self.__trans.isOpen()

  def open(self):
    return self.__trans.open()

  def close(self):
    return self.__trans.close()

  def read(self, sz):
    ret = self.__rbuf.read(sz)     #  默认从buffer读，buffer里有数据的话直接返回
    if len(ret) != 0:
      return ret

    self.__rbuf = StringIO(self.__trans.read(max(sz, self.__rbuf_size)))   # 否则的话就从socket读出buffer-size大小的数据缓存在buffer里
    return self.__rbuf.read(sz)                                           # 在从buffer里返回要求大小的数据量

  def write(self, buf):
    self.__wbuf.write(buf)       # 写操作的话就是一直往buffer里写，并不自动flush

  def flush(self):
    out = self.__wbuf.getvalue()
    # reset wbuf before write/flush to preserve state on underlying failure
    self.__wbuf = StringIO()
    self.__trans.write(out)
    self.__trans.flush()

  # Implement the CReadableTransport interface.
  @property
  def cstringio_buf(self):
    return self.__rbuf

  def cstringio_refill(self, partialread, reqlen):
    retstring = partialread
    if reqlen < self.__rbuf_size:
      # try to make a read of as much as we can.
      retstring += self.__trans.read(self.__rbuf_size)

    # but make sure we do read reqlen bytes.
    if len(retstring) < reqlen:
      retstring += self.__trans.readAll(reqlen - len(retstring))

    self.__rbuf = StringIO(retstring)
    return self.__rbuf
```

再来看看给server端用的socket

```python
class TServerSocket(TSocketBase, TServerTransportBase):
  """Socket implementation of TServerTransport base."""

  def __init__(self, host=None, port=9090, unix_socket=None):
    self.host = host
    self.port = port
    self._unix_socket = unix_socket
    self.handle = None

  def listen(self):
    res0 = self._resolveAddr()
    for res in res0:
      if res[0] is socket.AF_INET6 or res is res0[-1]:
        break

    # We need remove the old unix socket if the file exists and
    # nobody is listening on it.
    if self._unix_socket:
      tmp = socket.socket(res[0], res[1])
      try:
        tmp.connect(res[4])
      except socket.error, err:
        eno, message = err.args
        if eno == errno.ECONNREFUSED:
          os.unlink(res[4])

    self.handle = socket.socket(res[0], res[1])
    self.handle.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    if hasattr(self.handle, 'settimeout'):
      self.handle.settimeout(None)
    self.handle.bind(res[4])
    self.handle.listen(128)

  def accept(self):     #  accept会把返回的socket再用TSocket包一下
    client, addr = self.handle.accept()
    result = TSocket()
    result.setHandle(client)
    return result
```
### Protocol
这个模块定义的是传输协议，代码都放在protocol目录下。关于这一部分的作用，官网已经说的很清楚了，直接引用过来：

>The Protocol abstraction defines a mechanism to map in-memory data structures to a wire-format. In other words, a protocol specifies how datatypes use the underlying Transport to encode/decode themselves. Thus the protocol implementation governs the encoding scheme and is responsible for (de)serialization. Some examples of protocols in this sense include JSON, XML, plain text, compact binary etc.

这个模块的接口如下：

>writeMessageBegin(name, type, seq)    
>writeMessageEnd()    
>writeStructBegin(name)   
>writeStructEnd()   
>writeFieldBegin(name, type, id)     
>writeFieldEnd()      
>writeFieldStop()     
>writeMapBegin(ktype, vtype, size)     
>writeMapEnd()      
>writeListBegin(etype, size)     
>writeListEnd()     
>writeSetBegin(etype, size)      
>writeSetEnd()       
>writeBool(bool)       
>writeByte(byte)       
>writeI16(i16)       
>writeI32(i32)      
>writeI64(i64)      
>writeDouble(double)        
>writeString(string)       
>
>name, type, seq = readMessageBegin()         
>                  readMessageEnd()         
>name = readStructBegin()       
>       readStructEnd()          
>name, type, id = readFieldBegin()       
>                 readFieldEnd()        
>k, v, size = readMapBegin()       
>             readMapEnd()       
>etype, size = readListBegin()       
>              readListEnd()      
>etype, size = readSetBegin()       
>              readSetEnd()      
>bool = readBool()        
>byte = readByte()      
>i16 = readI16()       
>i32 = readI32()      
>i64 = readI64()      
>double = readDouble()      
>string = readString()        

一个可能的使用例子：

写

```python
  def send_addPlan(self, header, plan):
    self._oprot.writeMessageBegin('addPlan', TMessageType.CALL, self._seqid)
    args = addPlan_args()
    args.header = header
    args.plan = plan
    args.write(self._oprot)
    self._oprot.writeMessageEnd()
    self._oprot.trans.flush()
```
读

```python
  def recv_addPlan(self, ):
    (fname, mtype, rseqid) = self._iprot.readMessageBegin()
    if mtype == TMessageType.EXCEPTION:
      x = TApplicationException()
      x.read(self._iprot)
      self._iprot.readMessageEnd()
      raise x
    result = addPlan_result()
    result.read(self._iprot)
    self._iprot.readMessageEnd()
    if result.success != None:
      return result.success
    if result.e != None:
      raise result.e
    raise TApplicationException(TApplicationException.MISSING_RESULT, "addPlan failed: unknown result");
```

先看write，它首先会调用writeMessageBegin方法，然后写入相应的内容，最后调用writeMessageEnd方法来结束写操作。

```python
  def writeMessageBegin(self, name, type, seqid):
    if self.strictWrite:
      self.writeI32(TBinaryProtocol.VERSION_1 | type)
      self.writeString(name)
      self.writeI32(seqid)
    else:
      self.writeString(name)
      self.writeByte(type)
      self.writeI32(seqid)
```

调用writeMessageBegin的时候分为两种情况，严格写和普通写。严格写的时候需要先写入版本号和消息的类型，然后是消息名称、消息序列号。普通写只要依次写入消息名称、消息类型
和消息序列号即可。这其实只是相当于把header信息（也就是元信息）写进去了，写完这些以后才会写具体的数据，最后调用一下writeMessageEnd表示写消息结束（这个方法根据各个具体的协议会有不同的
实现，像在TBinaryProtocol里这个方法其实是空的，什么都不做）。

再看read，与写的过程类似，不过是反的，会先调用readMessageBegin，它会返回一个(name, type, seqid)的三元组，标识了收到的这条消息的名称、类型和序列号。

```python
  def readMessageBegin(self):
    sz = self.readI32()
    if sz < 0:
      version = sz & TBinaryProtocol.VERSION_MASK
      if version != TBinaryProtocol.VERSION_1:
        raise TProtocolException(
          type=TProtocolException.BAD_VERSION,
          message='Bad version in readMessageBegin: %d' % (sz))
      type = sz & TBinaryProtocol.TYPE_MASK
      name = self.readString()
      seqid = self.readI32()
    else:
      if self.strictRead:
        raise TProtocolException(type=TProtocolException.BAD_VERSION,
                                 message='No protocol version header')
      name = self.trans.readAll(sz)
      type = self.readByte()
      seqid = self.readI32()
    return (name, type, seqid)
```

类似的，读出header信息后，会继续读出具体的返回数据，视情况决定是否返回exception。

这一部分的分析有一个巨牛的资源，写的非常清楚：   
[由浅入深了解Thrift（二）——Thrift的工作原理](http://houjixin.blog.163.com/blog/static/35628410201501654039437/)


### Processor
这一块儿的代码是由thrift compiler自动生成的

### Server
