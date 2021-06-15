+++
title = "异步IO的威力"
date = "2019-11-19"
slug = "2019/11/19/the-power-of-asyncio"
Categories = ["Python", "asyncio"]
+++

### what

so, what's the problem? 

- 压测目标有提升
- 之前的压测没有考虑三方服务慢返回(通常为200ms左右)的问题，如果当前的压测仍用同步worker会死的佷惨

### how

结合项目的实际情况，问题转化为如何在`Python + Django 1.8 + Gunicorn`这一组合下使用异步IO。

首先，需要让Gunicorn使用异步worker：

```sh
gunicorn demo.wsgi --workers 3 -k gevent
```
其次，安装异步worker所需的依赖以及使用与gevent兼容的mysql client库：

```sh
# cat requirements.txt

...
gevent==1.4.0
PyMySQL==0.9.3
...
```
最后，压测标识的处理。之前项目使用的是进程worker，收到压测请求后，是放在threadLocal中的；现在换用异步worker后，worker的scope不再是进程和线程级别的，而是协程级别的。协程级别的local实现也有很多，我们直接使用了werkzeug的实现替换掉了当前的threadLocal

使用同样的硬件资源情况，在请求外部资源耗时200ms的场景下，结果对比：

使用同步worker

```sh
Running 20s test @ http://payment2-asyncio.test.svc.luojilab.dc
  10 threads and 40 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.03s   119.85ms   1.92s    92.64%
    Req/Sec     6.45      5.33    30.00     69.53%
  Latency Distribution
     50%    1.01s
     75%    1.05s
     90%    1.09s
     99%    1.67s
  761 requests in 20.10s, 756.54KB read
Requests/sec:     37.86
Transfer/sec:     37.64KB
```

使用gevent worker

```sh
Running 20s test @ http://payment2-asyncio.test.svc.luojilab.dc
  10 threads and 40 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   395.56ms   55.53ms 672.69ms   71.48%
    Req/Sec    10.74      5.78    30.00     62.64%
  Latency Distribution
     50%  387.63ms
     75%  424.52ms
     90%  468.89ms
     99%  564.88ms
  2002 requests in 20.08s, 1.94MB read
Requests/sec:     99.70
Transfer/sec:     99.12KB
```

不仅qps有了近3倍的提升，而且耗时也明显好很多了。

### why

我想从两个方面来阐述：

- 异步IO的底层原理
- 具体到Python + Gevent + gunicorn的原理

#### 操作系统的I/O模型

我们首先要区分这几种基本的I/O模型，这样才可能理解后续的问题。

对一个socket的读取过程一般分为两个阶段：

- 等待数据准备好（数据到达网卡以及从网卡buffer复制到内核buffer）
- 数据从内核buffer复制到应用进程的buffer

根据系统调用在两个阶段的不同行为，可以定义下面5种I/O模型：

> Blocking I/O Model / 阻塞 I/O 模型

这是大家最熟悉的 I/O 模型，也是socket的默认行为，在两个阶段都是阻塞的

![](https://i.imgur.com/uAxGXLe.png)

> Nonblocking I/O Model / 非阻塞 I/O 模型

我们可以将socket设置为非阻塞的模式，意思是，告诉操作系统，当应用发起一个I/O请求时，如果这个请求无法立即返回（即数据还没准备好），直接返回一个特定的错误就好了，不要让应用一直等待。

这种模型看似是将主动权放到了应用这边，但也意味着应用需要不停的轮询操作系统来确认数据是否已经准备好了。

![](https://i.imgur.com/Yq9Kiib.png)

> I/O Multiplexing Model / I/O 多路复用模型 

先发起一个阻塞的系统调用(select / poll / epoll)，当有数据可读后，它会返回，然后应用继续发起recvfrom调用开始读取数据。这种模型乍一看并没有多少优势，反而还比阻塞的模型多了一次系统调用，但它的优点是可以一次监控多个socket（阻塞模型中一次只能等待一个socket）

![](https://i.imgur.com/vsCsyoE.png)

> Signal-Driven I/O Model / 信号驱动 I/O 模型

在这个模型下，我们可以向操作系统注册一个信号处理器（signal handler），当数据准备好后，操作系统会发送一个信号来通知我们，此时我们可以开始发起recvfrom来读取数据。注意，第一个阶段是非阻塞的，第二个阶段是阻塞的

![](https://i.imgur.com/CFr5zb1.png)

> Asynchronous I/O Model / 异步I/O模型

这个模型下，两个阶段的调用都是阻塞的。应用只需发起一个`aio_read`的调用，然后等着操作系统的通知即可，如果收到了通知，说明此时数据一定是准备好的。

这种模型看起来是最理想的，但不幸的是并没有多少操作系统支持这种 I/O 模型

![](https://i.imgur.com/vOkfZam.png)

小结：

根据 POSIX 的定义：

- 同步I/O操作(synchronous I/O operation)指本次I/O操作会导致请求方阻塞，直至本次I/O操作完成
- 异步I/O操作(asynchronous I/O operation)指本次I/O操作不会阻塞请求方

上述列出的5种I/O模型，前4种都是同步I/O操作，只有最后一种属于异步I/O操作

#### epoll

epoll是linux平台上的 I/O 多路复用模型的实现中的佼佼者。它的性能很好，可以支持大量的并发连接，现在高性能的webserver一般底层都有epoll的功劳（比如Nginx）。

我们首先来看一下它的基本用法，它对外只暴露了3个接口：

```sh
int epoll_create(int size)；
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```
其中，

- `epoll_create`用于创建一个epoll实例，后续的操作都是基于它；
- `epoll_ctl`用于告诉epoll你想关心哪些文件描述符的哪种行为（比如可写、可读）；
    - epfd即为`epoll_create`返回的epoll实例
    - op为要对文件描述符监听类别，分为新增（`EPOLL_CTL_ADD`）、删除（`EPOLL_CTL_DEL`）和修改（`EPOLL_CTL_MOD`）
    - fd即想要监听的文件描述符
    - `epoll_event`主要用于表示想关注这个fd的哪些状态，比如可写、可读等
- `epoll_wait`是等待I/O事件发生，它是一个阻塞请求，但它只要返回了文件描述符，那这些文件描述符就一定是ready的（即数据已准备好，可立即读写，不会阻塞）
    - events 是调用方初始化后传入的，如果有ready的文件描述符，epoll会写入到这里，调用方在`epoll_wait`返回后处理events即可
    - timeout，等待的时候可以指定一个超时时间，表示最多只等待这么长时间，若此期间没有任何I/O 事件发生也会返回

一个简单的使用示例，伪代码如下：

```python
epfd = epoll_init1(0);
event.events = EPOLLET | EPOLLIN;
event.data.fd = serverfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, serverfd, &event);
// 主循环
while(true) {
    // 等待事件
    count = epoll_wait(epfd, &events, MAXEVENTS, timeout);
    // 遍历ready的事件
    for(i = 0; i < count; ++i) {
        if(events[i].events & EPOLLERR || events[i].events & EPOLLHUP)
            // 处理错误
        if(events[i].data.fd == serverfd)
            // 为接入的连接注册事件
        else if(events[i].events & EPOLLIN)
            // 处理可读的缓冲区
            read(events[i].data.fd, buf, len);
            // 向epoll注册一个该fd可写的“关注点”
            event.events = EPOLLET | EPOLLOUT;
            event.data.fd = events[i].data.fd;
            epoll_ctl(epfd, EPOLL_CTL_MOD, events[i].data.fd, &event);
        else
            // 处理可写的缓冲区
            write(events[i].data.fd, buf, len);
            // 后续可以关闭fd或者MOD至EPOLLOUT
    }
}
```
值得注意的是，里面有一个loop forever的循环，因为需要不停的通过发起`epoll_wait`调用来询问操作系统哪些fd是ready状态了，以便进行处理。记住这一点，我们后面还会提到这个循环。

下面简单说一下epoll为什么效率这么高，要想知道epoll为啥快就得先知道它的先驱者们（select
 / poll）为啥慢，有对比才有伤害嘛。
 
 select 和 poll 的实现原理基本是一样的，细节略有差别。它们的基本思想是调用方提供一批文件描述符给它，假设为n个，它帮你遍历一遍，逐个查看一下这个文件描述符是否可写可读，所以它的实现复杂度是O(n)的，连接数少的时候还好，当连接数数量上来的时候，效率就很差了。
 
 再说说epoll，它在内部维护了一个就绪列表（ready list），代表了当前可读可写的文件描述符集合，因此当应用调用`epoll_wait`来问当前有哪些文件描述符可用时，它直接返回这个list即可，一般在同一时间可读可写的I/O数量是很少的，因此它的时间复杂度是O(当前可读可写的文件描述符数量)，甚至可以说是O(1)，当然是非常快的。
 
 我们可以再追问一句，这个就绪列表是如何实现的呢？它是如何能够反映某一个时刻的文件描述符的可用情况的呢？这里就要讲到epoll的另一个函数`epoll_ctl`，记得我们上面说它的作用是用于告诉epoll我们关心哪些文件描述符的可读可写，guess what？当你在调用这个函数的时候，epoll会向内核注册一个回调函数，当某个我们关心的文件描述符变为可读 / 可写状态时，内核就通过这个回调函数讲这个文件描述符写入就绪列表了，可见它的实现机制是事件驱动的，这也是epoll名字的由来（event poll）

参考：

- [从源码角度看Golang的TCP Socket(epoll)实现](https://studygolang.com/articles/22460)
- [深入理解IO复用之epoll](https://zhuanlan.zhihu.com/p/87843750?utm_source=wechat_session&utm_medium=social&utm_oi=26858074669056) 非常好
- [源码解读epoll内核机制](http://gityuan.com/2019/01/06/linux-epoll/)  代码讲的6
- [论epoll的实现](http://www.hulkdev.com/posts/epoll-io)
- [epoll的那些事](http://linbo.github.io/2019/03/01/epoll-fundamental)
- [Linux epoll 详解](http://blog.lucode.net/linux/epoll-tutorial.html)
- [谈谈epoll实现原理](http://luodw.cc/2016/01/24/epoll/)

#### event loop（libev / libuv）

记得我们上面在介绍epoll时说，它的使用一般是通过一个循环（loop），libev就是这样一种事件循环库，它帮用户屏蔽了很多细节，用户只需要告诉它想要关心的文件描述符，然后启动一个事件循环等待回调即可。

来自 libev 官方文档的介绍：

> Libev is an event loop: you register interest in certain events (such as a file descriptor being readable or a timeout occurring), and it will manage these event sources and provide your program with events.
> 
> To do this, it must take more or less **complete control over your process** (or thread) by executing the event loop handler, and will then **communicate events via a callback mechanism**.

可以看到，它不仅仅支持I/O事件，更重要的是它支持跨平台使用，支持了 Linux 平台的 epoll 和 BSD 平台上的kqueue，使用它开发的应用可以在无需更改代码的情况下在几个平台之间无缝切换。

参考：

- [libev 使用简介](https://jin-yang.github.io/post/linux-libev.html)
- [事件驱动异步IO的真正奥秘之Libuv入门](https://juejin.im/post/5d412865e51d4561e84fcba7)
- [libev 介绍](https://www.jianshu.com/p/2c78f7ec7c7f)
- [使用 libevent 和 libev 提高网络应用性能](https://www.ibm.com/developerworks/cn/aix/library/au-libev/index.html)
- [libev官方文档](https://metacpan.org/pod/distribution/EV/libev/ev.pod)

#### 协程 / coroutine

简单的说，协程就是可以暂时中断，之后可以再继续执行的程序。它跟线程、进程是一个概念上的东西，但又有很大的区别：

- 线程的上下文切换成本很高，但协程之间的切换成本很低，轻易可以产生大量的协程
- 协程的切换必须发生在一个线程内，不能跨线程切换
- 线程的切换是由操作系统来控制的，不受程序员控制，即它是抢占式的；而协程的切换是由程序员控制的

一个简单的例子来了解下协程是个什么东西：

```
import time

def consumer():
    r = ''
    while True:
        n = yield r
        if not n:
            return
        print('[CONSUMER] Consuming %s...' % n)
        time.sleep(1)
        r = '200 OK'

def produce(c):
    c.next()
    n = 0
    while n < 3:
        n = n + 1
        print('[PRODUCER] Producing %s...' % n)
        r = c.send(n)
        print('[PRODUCER] Consumer return: %s' % r)
    c.close()

if __name__=='__main__':
    c = consumer()
    produce(c)
```

执行结果如下：

```sh
[PRODUCER] Producing 1...
[CONSUMER] Consuming 1...
[PRODUCER] Consumer return: 200 OK
[PRODUCER] Producing 2...
[CONSUMER] Consuming 2...
[PRODUCER] Consumer return: 200 OK
[PRODUCER] Producing 3...
[CONSUMER] Consuming 3...
[PRODUCER] Consumer return: 200 OK
```

可以看到，我们实现了简单的生产者消费者模式，但我们没有使用多线程，而是在一个线程内使用协程的模式实现的，程序的控制权在生产者和消费者之间轮流切换

参考：

- [淺談coroutine與gevent](http://blog.ez2learn.com/2010/07/17/talk-about-coroutine-and-gevent/)
- [Coroutine](https://en.wikipedia.org/wiki/Coroutine)

#### Greenlet

Python2 只支持简单的协程机制(semicoroutine)，Greenlet是一个第三方的 Python 协程实现，它实现了完善的协程机制。

来自官方文档的一个简单例子：

```python
from greenlet import greenlet

def test1():
    print(12)
    gr2.switch()
    print(34)

def test2():
    print(56)
    gr1.switch()
    print(78)

gr1 = greenlet(test1)
gr2 = greenlet(test2)
gr1.switch()

# 输出结果
12
56
34
# 注意，78并没有机会输出
```

参考：

- [greenlet: Lightweight concurrent programming](https://greenlet.readthedocs.io/en/latest/)

#### gevent

讲完了上面的铺垫，终于可以说说我们今天的主角了。

来自官网的介绍：

> gevent is a **coroutine-based** Python networking library that uses greenlet to provide a high-level synchronous API on top of the libev or libuv event loop.

它是构建于协程(greenlet)、事件循环(libev / libuv)之上的一个网络库，可以让你用同步的写法来做异步的事情（而不是js中的那种变态的callback hell写法）。那么问题来了，它是怎么做到的呢？

- 基于协程，可以在用户态控制协程的切换（而不是由操作系统强制切换）
- 基于事件循环，从而能够快速感知I/O事件
- 基于上述两者将Python标准库中所有涉及到I/O的库全部重写了一遍（所有对外API保持不变，这是关键），然后提供了一个monkey patch方法可以让用户很便利的一键替换

那么重写的库的逻辑是什么呢？无非是当遇到I/O需要等待时，就将控制流切换到别的协程上，等I/O准备好后再伺机切换回来，程序的执行模式大概是这个样子的：

![](https://i.imgur.com/ElFdb46.png)


参考：

- [gevent For the Working Python Developer](https://sdiehl.github.io/gevent-tutorial/)
- [gevent: the Good, the Bad, the Ugly](https://engineering.mixpanel.com/2010/10/29/gevent-the-good-the-bad-the-ugly/)
- [Why Zapier Doesn't Use Gevent (Yet)](https://zapier.com/engineering/why-zapier-doesnt-use-gevent-yet/)
- [Blocking IO in Gunicorn Gevent Workers](https://tech.wayfair.com/2018/07/blocking-io-in-gunicorn-gevent-workers/)
- [Django, fast: part 2](https://blog.mirumee.com/django-fast-part-2-d73a4ecd61f3)
- [Squeezing every drop of performance out of a Django app on Heroku](https://medium.com/@bfirsh/squeezing-every-drop-of-performance-out-of-a-django-app-on-heroku-4b5b1e5a3d44)
- [Scaling Django with gevent](https://www.slideshare.net/mahendram/scaling-django-with-gevent)

#### gunicorn 的异步 worker

在计算机科学中有一句“名言”，没有什么问题不能通过增加一层封装来解决的，如果没解决，那就再封装一层... 所以，欢迎来到我们的最后一层封装，有了它，我们的工作进一步简化成“通过参数指定一个异步worker类型来启动gunicorn”即可享受异步IO的好处。那么，它做了什么呢？

- 帮我们执行了monkey patch替换掉了标准库中的阻塞实现（是的，使用gunicorn后你甚至都不需要自己monkey patch）
- 启动了一个协程池来处理请求

#### “题外话”

话题跟当前主题不太相关，但也是有关系的，我想再多说几句关于使用wrk的一些心得

> wrk 是什么？

首先明确一点，wrk是压力测试工具，其目的是验证出系统支持的最大QPS，不是为了模拟现实流量的。（wrk is not tool for emulate real load from humans, 100 connections doesn't mean 100 users (browsers). No one can push F5 key 10000 times a second. But wrk can send 10000 requests on 1 connection.）

> 压测时，c 和 t 参数各该如何设置？

这个问题曾经困扰了我好久.. 在项目的github issues里也充斥了此类疑问。在围观了各路大神的回复后，有如下结论：

t（线程数）应当根据压测机器的配置来设置（比如`2*core + 1`)，而c（连接数）则根据被压测接口的表现，从一个初始值开始慢慢提升，直至被压测接口的qps无法再提升为止

> wrk 的机制（原理）？

底层也使用了异步IO，不过是借用的Redis实现的event loop，没有使用常用的libev等库。它会启动 t 个线程，每个线程中开 c / t = m 个连接，然后不停的在一个连接中发request、接response。是的，它利用了 http 的 keep alive特性，不会不停的新建和关闭连接。
