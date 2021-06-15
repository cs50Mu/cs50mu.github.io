+++
title = "「译」Asynchronous Python and Databases"
date = "2020-01-06"
slug = "2020/01/06/asynchronous-python-and-databases"
Categories = ["translation", "Python", "asyncio"]
+++

翻译一篇 SQLAlchemy 作者的博文，原文：[Asynchronous Python and Databases](https://techspot.zzzeek.org/2015/02/15/asynchronous-python-and-databases/)

这篇文章给异步IO泼了点冷水，引导我们用正确的态度来看待这个技术(其实是任何技术)，不要过度“神化”它，我受益匪浅。

===================== 译文开始 =======================

异步编程的话题很难用几句话来说清。这些年来，这个话题也越来越复杂，而我基本也没有涉猎这个领域。但由于我在Python领域做了很多与关系型数据库交互相关的工作，经常要回答很多有关异步IO和数据库编程的问题，比如跟SQLAlchemy或者Openstack相关的。

不想看长篇大论的，我先简单说一下我的想法：我认为Python的asyncio库设计的非常巧妙、很有前途、使用起来也很有意思。它的代码组织的也足够好，让SQLAlchemy在一定程度上兼容它是可行的。鉴于asyncio已经进入Python标准库，我有兴趣在未来将它引入SQLAlchemy。

尽管我上面说的，我还是认为异步编程只是一种潜在的可用途径，我们不应该把它当成一种万能药，遇到问题就想通过它来解决，除非我们正在写一个HTTP或者聊天服务器（在这种情况下，你需要同时维持大量的慢请求或者空闲TCP连接）。在标准的CRUD模式下的数据库交互代码，从来都没必要使用asyncio，而且若使用的话几乎一定会影响性能（相对于同步io和threading模式）。那些为了所谓的“正确”而对asyncio的吹捧论断是经不起推敲的。

把我的观点说清楚后，让我们开始具体阐述吧

### 什么是Asynchronous IO？

Asynchronous IO是一种获取并发效果的方式，它通过允许进程/线程在IO期间可以继续执行（而不是等待IO完成）来实现。为了实现这种效果，IO函数需要被设置成非阻塞模式（socket的默认模式是阻塞的），因而这些IO函数可以在真正的IO操作完成之前就立即返回。它的实现原理是这样的：通常在一个循环中不断调用一个轮询系统（通常依赖操作系统不同而不同，比如linux下的epoll）来查询当前关注的一批文件描述符是否有数据可读、可写的，若有，则处理它们，当处理完成后，执行权又回到了这个事件循环中来查询下一批可读可写的文件描述符，以此往复。

当一堆线程都在等待一个socket返回结果时是很没有效率的，这时候非阻塞IO就发挥出了它的优势。当你需要监听非常多的TCP socket，并且这些socket大部分时间都在“睡觉”或者是很慢的返回 - 最好的例子就是聊天服务器或者类似的消息系统，在这种场景下，同时会有非常多的连接存在，但同一时间只有很少的连接会有数据交互，当一个连接上有数据出现时，我们称之为一个IO事件。

近几年，asynchronous IO被成功应用于HTTP服务器和应用中。使用asynchronous IO可以让服务器只用单线程就能处理大量的HTTP连接，特别地，慢HTTP客户端再也不会影响其它正常的客户端的请求。非阻塞的web服务器，比如nginx，在实践中已被证明能很好的工作。

### 异步IO和脚本语言

在脚本语言中，异步io编程主要是通过事件循环来实现的，经典用法是设置一个回调函数，当对应的IO请求的数据已经准备好后会调用这个回调函数。由于事件循环可以提供进程/线程调度的功能，在脚本语言中通常没有必要使用线程。实际上，在代码里混合使用多线程阻塞IO编程和事件循环式的非阻塞IO编程会比较尴尬，因为它们使用的是不同的编程范式。

异步io和事件循环的关系，加上它在服务器端编程因能以直观的方式来提供并发的能力，让它在Javascript语言里特别流行。Javascript是一种用于浏览器上的客户端脚本语言。浏览器，像其它图形应用一样，基本上是一个事件机，事件机做的是，对用户发起的事件（按下按钮、移动鼠标等）做出响应。因此，Javascript里使用回调函数是很平常的事情，而且，直到最近它才支持了多线程。

前端开发者们一直以来都很习惯这些客户端的回调用法，他们开始将回调用于网络事件中，一位新的玩家即将登台，这将把已经火热的Javascript推向新的高潮...

### 服务器

早在Node.js[之前](https://docs.oracle.com/cd/E19957-01/816-6411-10/getstart.htm#1015788)就有尝试想将Javascript用于服务端编程。不过，Node.js成功的一个关键原因在于在它发布之时已经有大量的经验丰富的Javascript程序员，并且它也完全拥抱了客户端Javascript开发者已非常熟悉的事件驱动编程范式。

为了推广，Node.js遵循了一个原则：非阻塞IO模式不仅仅应当被用在大量睡眠或慢请求这样的场景上，更应当做为一个事实上的服务器端编程的[标准](https://www.youtube.com/watch?v=bzkRVzciAZg)来推行。这意味着任何类型的网络IO都应该使用非阻塞IO的方式，这当然包括数据库连接了 —— 每个进程通常只维持了少量的(10 - 50)这种连接，而且已经被"池化"，因此因TCP创建连接带来的开销影响会很小，而且，对于一个设计良好的数据库来说，一次数据库请求的时间通常是非常快和可预测的 —— 无论从哪个方面来看，数据库连接的场景都不是一开始引入非阻塞IO所要解决的问题，或者说这个场景跟非阻塞IO所要解决的是恰恰相反的。Postgresql数据库在libpq里支持一个异步命令行API，不过这个API主要是为[图形应用](http://www.postgresql.org/docs/8.3/static/libpq-async.html)准备的。

node.js已经受益于一个高性能的[JIT引擎](https://code.google.com/p/v8/)了，因此尽管在数据库连接的场景中使用非阻塞IO并不“合适”，但用起来效果还可以。

### 线程幽灵

早在node.js将大量的客户端Javascript开发者转变为服务端异步开发者之前，多线程编程模型已经受到一些[学术理论家](http://www.eecs.berkeley.edu/Pubs/TechRpts/2006/EECS-2006-1.html)的抱怨：多线程编程容易产生非确定性的程序，而异步编程，由于使用了一种新的编程范式（事件驱动），迅速被用来攻击多线程编程模型的缺点，主要有两点：一个是由于线程的创建和维持的代价比较大，使得它非常不适合需要同时维持成千上万连接的场景；另一个是多线程的代码很难写，且运行结果是非确定性的。在Python世界中，由于GIL一直以来的问题，async模型做为一个“救世主”的形象，相对于其它地方更容易在这里生根发芽。

### How do you like your Spaghetti? / 你有多喜欢你的意大利面？

node.js的回调风格的代码以及其它异步风格的范式被认为是有问题的，稍微复杂一点的逻辑如果使用回调风格来组织的话会让代码变得非常繁琐和晦涩难懂，这种代码通常被称为回调面条([callback spaghetti](https://www.google.com/search?q=node.js+spaghetti+code&ie=utf-8&oe=utf-8#q=node.js+spaghetti))。回调风格的代码到底是一坨屎还是一朵花，这个问题曾经是21世纪最大的争论之一，不过很庆幸我不需要参与其中，因为异步社区已一致确认回调风格的代码是前者并且已经采取措施来改进它这方面的问题。

在Python中，有一种方法可以让你在不使用回调的情况下使用异步IO编程模型，那就是由[eventlet](http://eventlet.net/)和[gevent](http://www.gevent.org/)提供的所谓“隐式异步IO”。它们通过将IO函数设置为非阻塞模式，启动一批并发的协程（[green threads](http://en.wikipedia.org/wiki/Green_threads)），然后依赖底层的事件驱动库（比如[libev](http://libev.schmorp.de/)）来根据IO事件的触发来调度这批协程工作。“隐式异步IO”的好处是你原来的同步代码基本是不用动的，只需稍微改动几行代码就可以将你的程序切换到异步模式（当然，肯定是有一些特殊情况造成的小坑，这个另当别论）

与隐式异步IO相对的是，非常有前途的由Python官方提供的异步库[asyncio](https://www.python.org/dev/peps/pep-3156/)（本文一开始提到过），现在已进入Python 3标准库了。asyncio库给Python带来了标准的“futures”和[协程](http://en.wikipedia.org/wiki/Coroutine)的概念，通过它们我们可以写出跟传统的阻塞代码类似的非阻塞的代码，而在那些需要非阻塞IO操作时会明显区分出来（而不是像gevent/eventlet一样无法区分）。

### SQLAlchemy? Asyncio? 确定吗？

现在既然asyncio已经进入Python标准库了，是该整合一下之前的老代码了。因为asyncio依旧兼容着之前的返回值和异常机制，跑起来一个asyncio版本的SQLAlchemy没那么难，可能需要几个外部模块使用async results重新实现一下SQLAlchemy Core和ORM的关键方法，大部分代码都不用动。这不再意味着SQLAlchemy的完全重写，并且关于异步的模块可能会独立于主逻辑（并不会污染主逻辑）。整个过程会比较麻烦但 是可行的。

不过，你真的想要一个异步模式的SQLAlchemy吗？

### Taking Async Web Scale

下面让我们来具体说说，nodejs和asyncio到底哪里做错了，尤其是在与数据库交互时所犯的错误

#### 问题一 —— 作为性能上魔术仙尘的异步范式

许多（当然不是全部）node.js社区和Python社区的人声称异步编程范式在几乎所有场景下的并发性能都是很好的。特别地，有一种声音认为显式异步的模式，比如asyncio，的“上下文切换”的代价非常小，再加上Python的那个”臭名昭著“的GIL，他们认为asyncio一定会比使用线程并发的模式更快 —— 至少不会慢。因此，所有的web应用应当尽快换成异步的模式，而且是所有的地方都换，从HTTP请求到数据库请求，这样的话没有任何代价就能提升服务器性能。

我下面仅仅针对数据库请求的情况来阐述，对于HTTP server或者 chat server，使用asyncio可能效果会很好，因为它可以同时维持很多慢连接，但对于本地数据库请求，情况不是这样的。

##### 1. Python相对于你的数据库来说，非常、~~非常~~慢

更新：Reddit上的Riddlerforce发现了这一节的论述有点问题，即：我不是基于网络连接来测试的，我已经基于网络连接重新做了测试并将结果做了更新，结论是一样的，只是不像一开始那样夸张。

让我们先看一下异步编程有很大[优势](http://blog.kgriffs.com/2012/09/18/demystifying-async-io.html)的场景，[I/O密集型](http://en.wikipedia.org/wiki/I/O_bound)应用：

> I/O密集指的是这样一种场景，在这种场景下，由于等待输入输出操作完成所耗费的时间占了整个计算时间的大部分。

经常遇到的一个很大的误解是：在一个主要与数据库交互的Python应用中，大部分时间是花费在了它跟数据库的通信上。这个观点在编译型语言上，比如C甚至Java，或许成立，但在Python里不成立。Python 相对于上述编译语言非常慢，尽管PyPy的速度是一个很大的进步了，在简单的CRUD场景（意思是非OLAP类型的重查询，并且网络延迟也很低的情况）下，Python的速度仍然无法跟数据库相提并论。在我为Openstack所做的针对PyMySQL的[评估](https://wiki.openstack.org/wiki/PyMySQL_evaluation)中可以看到，纯C写的数据库驱动和纯Python写的驱动，它们的速度相差很大，仅仅考虑驱动的话，Python写的驱动比C写的慢一个数量级。尽管在网络上的耗费会使得CPU和IO的比例更平均，但由Python数据库驱动所耗费的CPU时间仍然比网络IO要多一倍，这还是在没有任何其它数据库抽象层、业务逻辑和呈现逻辑的情况下。

这个[脚本](https://techspot.zzzeek.org/files/2015/mysql_speed_test.py)，由Openstack的入口代码改造而来，发起了一堆INSERT和SELECT请求数据库驱动，除此之外基本没有别的Python代码

MySQL-Python，一个纯C写的数据库驱动，在走网络的情况下，有如下表现：

```
DBAPI (cProfile):  <module 'MySQLdb'>
     47503 function calls in 14.863 seconds
DBAPI (straight time):  <module 'MySQLdb'>, total seconds 12.962214
```

PyMySQL，一个纯Python写的数据库驱动，大概要慢30%：

```
DBAPI (cProfile):  <module 'pymysql'>
     23807673 function calls in 21.269 seconds
DBAPI (straight time):  <module 'pymysql'>, total seconds 17.699732
```

如果不走网络的话（本地直连），PyMySQL要比MySQLdb慢一个数量级：

```
DBAPI:  <module 'pymysql'>, total seconds 9.121727

DBAPI:  <module 'MySQLdb'>, total seconds 1.025674
```

为了看清楚IO操作在整个过程中占多大比例，我们使用[RunSnakeRun](http://www.vrplumber.com/programming/runsnakerun/)生成了两张PyMySQL的图，一张是直接请求本地数据库，另一张是走网络请求数据库。这个比例在走网络的时候没本地直连那么夸张，但即使走网络的情况下IO操作也仅占总时间的三分之一，其它三分之二的时间都花在Python处理sql执行返回的结果上。而且，请记住，这仅仅是数据库驱动，一个真实的应用还会有数据库抽象层、业务逻辑和其它展示逻辑。

![](https://i.imgur.com/arlwrwD.png)

本地连接 - 明显不是IO密集的

![](https://i.imgur.com/rqoSRP4.png)

走网络 - 没走本地连接时那么夸张，但仍然不是IO密集的（24秒的总执行时间中，操作socket的时间只占8.7秒）

让我们总结一下，在Python里，请求数据库的操作不是一个IO密集型操作（除非你在做大量复杂的查询或者查询会返回很大的数据量，而在高性能应用中肯定要避免这种情况；又或者除非你的应用处在一个很慢的网络环境中）。当我们跟数据库交互时，几乎总是会使用某种类型的连接池，所以创建连接的额外开销已经大大缓解。在正常的网络环境下，数据库的写入和查询速度是很快的。Python本身的开销，仅仅是通过网络封送消息并生成结果集，就给CPU带来了很多工作量，这抵消了非阻塞IO所具有的任何独特的吞吐量优势。 真实应用围绕数据库操作还有大量其它操作，花在CPU上的比例只会增加。

##### 2. AsyncIO使用了吸引人的但相对来说低效的Python范式

asyncio的亮点是引入了`@asyncio.coroutine`这个装饰器，它的作用是将看似同步的代码逻辑推迟到其它协程中执行。这个实现的关键之处是引入了一个新的语法形式：`yield from`，它的作用是会暂时将函数的执行停止在那一点，将执行权交给其它协程执行，直到事件循环再次将执行权交给它后再次**继续执行**。这是一个很棒的创意，而且它其实用Python已有的`yield`语句也可以做到，不过用`yield from`可以让你继续写return语句，这样可以跟之前的非协程代码保持相对的一致。

```python
@asyncio.coroutine
def some_coroutine():
    conn = yield from db.connect()
    return conn
```

这个语法非常好，我很喜欢，不过它增加了函数调用的额外开销，我曾在twitter上[发过](https://twitter.com/zzzeek/status/563865362996133889)一个简单的demo来说明这个问题，下面就是这段demo：

```python
    """One function calls another normal function, which returns a value."""

    def foo():
        return 5

    def bar():
        f1 = foo()
        return f1

    return bar

def return_with_generator():
    """One function calls another coroutine-like function,
    which returns a value."""

    def decorate_to_return(fn):
        def decorate():
            it = fn()
            try:
                x = next(it)
            except StopIteration as y:
                return y.args[0]
        return decorate

    @decorate_to_return
    def foo():
        yield from range(0)
        return 5

    def bar():
        f1 = foo()
        return f1

    return bar

return_with_normal = return_with_normal()
return_with_generator = return_with_generator()

import timeit

print(timeit.timeit("return_with_generator()",
    "from __main__ import return_with_generator", number=10000000))
print(timeit.timeit("return_with_normal()",
    "from __main__ import return_with_normal", number=10000000))
```

结果如下，`yield from + StopIteration`的版本的耗时是正常的函数调用的6倍左右

```
yield from: 12.52761328802444
normal: 2.110536064952612
```

针对这个结果，许多人反驳我说，“那又怎么样呢？你的数据库请求的操作占用的时间更多”。但请注意，我们现在不是在讨论一种优化现有代码的方案，而是如何避免将已经很好的代码变得更慢.. PyMySQL的例子应该能说明仅仅在纯Python驱动内，Python自身的额外开销就增加得很快。不过，这个回答可能还是不那么令人信服。

所以，我准备了一个详尽的测试用例，通过这个测试用例我们可以看到Python中的传统线程、asyncio以及gevent形式的异步io这三者之间的对比。我们会使用[`psycopg2`](http://initd.org/psycopg/)、[`aiopg`](https://github.com/aio-libs/aiopg)和[`psycogreen`](https://bitbucket.org/dvarrazzo/psycogreen)。

这个测试用例具体做的是，将几百万条记录尽可能快地插入到Postgresql数据库中，使用上述的三种方式，当然三种方式使用的SQL是完全一样的，这样在保持其它变量一样的情况下，我们就能看出是否是GIL拖慢了我们的程序，又或者asyncio是否能大幅提升我们程序的性能。在这个测试用例中，我没有限制可以同时使用的数据库连接数，在我的测试中，最高曾用到过350个并发连接，如果在实际应用中你这么做的话，相信我，你会被DBA骂死的。

测试的结果放在这个[README](https://bitbucket.org/zzzeek/bigdata/)的最后了，包含了在不同的机器、不同的条件下的测试结果。在所有的场景下，性能最好的是当一台机器运行测试代码，去请求运行在另一个机器上的Postgresql数据库时，不过几乎在我运行的每个场景中，不管是在Mac只启动15个线程/协程还是在Linux上启动350个线程/协程，使用线程的代码都比用asyncio要快的多（甚至包括启动350个线程的场景，这挺令我吃惊的），而且使用线程的代码通常也会比使用gevent要快一些（虽然没有asyncio那么夸张）。以下是在我的Linux笔记本上运行120个线程/协程/数据库连接请求另一台Mac笔记本上的Postgresql数据库的测试结果：

```
Python2.7.8 threads (22k r/sec, 22k r/sec)
Python3.4.1 threads (10k r/sec, 21k r/sec)
Python2.7.8 gevent (18k r/sec, 19k r/sec)
Python3.4.1 asyncio (8k r/sec, 10k r/sec)
```

从上面的数据可以看出，对于第一个测试项（第一列）使用asyncio的代码比其它方式要慢多了（Python 3.4 在线程和协程似乎都有点问题），对于第二个测试项（第二列），asyncio的方式也比Python 2.7和Python 3.4下的线程方式要慢一倍。甚至当启动350个并发连接时（通常我们不会在一个单核CPU上启动这么多线程）asyncio的效率也不如线程。即使是使用非常快的、纯C写的psycopg2驱动，仅aiopg库的额外开销以及使用psycopg2的异步库在Python中接收轮询结果的动作，都足以拖慢脚本运行速度。

记住，我甚至没有试图要证明asyncio要比线程慢很多，而只是想说明asyncio并没有（像某些人说的）那样比线程快。跑出来的这个测试结果展示出来的差异比我预期的要大的多。我们也看到了，gevent作为一种相当高效的异步io的方式，还是比线程要慢一些（但没慢太多），这也证实了异步IO范式在这种场景中并不是必然比线程快的，而且从asyncio要比gevent慢很多这个结果可以看出，正是由于asyncio以及其它Python结构的开销再加上本来效率就不高的基于IO的上下文切换机制拖慢了asyncio的性能。

#### 问题二 —— “异步范式会让写代码更简单”

这是“魔术仙尘”硬币的另一面。这一论断在“线程是不好”的断言之上更进一步，它指出，如果一个程序用到了线程，比如，你写了一个WSGI web应用，而碰巧又使用线程池的形式运行在`mod_wsgi`下，那你就是在做“多线程编程”，危险程度不亚于这个[多线程编程练习](http://www.cs.cmu.edu/afs/cs/academic/class/15492-f07/www/pthreads.html#PITFALLS)，完全无视了这个事实：你的WSGI应用根本不应该有一点进程共享的逻辑。但不不，你就是在做多线程编程，线程是不好的，你应该立马停止。

“线程是不好的”的论断有一个有趣的转折（哈！），它被显式异步的拥护者用来批判隐式异步技术（比如gevent）。Glyph的这篇[帖子](https://glyph.twistedmatrix.com/2014/02/unyielding.html)很好的印证了这一点。他们的论证思路是这样的，如果同意“多线程编程是不好的”这一观点，那么使用隐式模式的异步编程同样也是不好的，因为不管怎么说，隐式模式异步的代码看起来跟多线程编程一样，而且因为IO操作可能发生在代码的任何地方，这种模式跟多线程编程一样是非确定性的。我碰巧也同意这一点，是的，gevent类的协程并发机制并没有比多线程好，不比它差就不错了。一个原因是，Python多线程编程中出现的并发问题比较“轻微”，因为有GIL（尽管我们非常不喜欢它），它让各种非常危险的操作，比如向列表中添加元素，变得安全。但用协程的话，你可以很容易地启动成百上千的协程，也很容易遇到通常在有GIL保护下多线程编程碰不到的[奇怪问题](https://github.com/PyMySQL/PyMySQL/issues/275)。

顺便说一句，Glyph给了“魔术仙尘”派一巴掌：

> 不幸的是，经常有人在安利“异步系统”时过分强调异步系统因某种可疑的优化因而能获得相对于多线程模式更好的并发能力，却忽略了我上面谈到的多线程编程模型本身存在的问题。这么来看待“异步”的话，也就不难理解他们会把所有4个选项都放在一起了。

> 这些年来我一直很内疚，内疚在之前说过的一些话，比如之前我说：一个系统如果使用Twisted实现会比使用线程的性能更好。在很多方面，这个话没错，不过：

> 1. 上面只是一个理解猜测，现实中优化性能是一个复杂的工程，涉及到方方面面，

> 2. 在真实应用中，“上下文切换”基本不会成为瓶颈，

> 3. 上面的话“避重就轻”，其实事件驱动编程的最大优势提现在写更大体量的应用程序上，这个“体量”既指的是项目的代码量大，也指并发使用的用户量大。

人们在说到使用asyncio会让你的程序里的bug更少时会援引Glyph的这篇帖子，但同时又会承诺使用asyncio也会带来更高的性能（因“某种原因”选择性忽视Glyph这篇帖子中的某些内容）

Glyph对他的两个观点做了清晰、完美的阐述，即我们既应该用非阻塞IO而且应该用显式地使用它。不过，他的论证过程跟异步IO最初想解决的问题（处理大量并发慢连接的场景）没有一点关系，相反，他说的是事件循环的本质以及这一新的并发模型（避免了把系统级的上下文切换直接暴露给用户）是如何出现的。

我们写了那么久的回调代码，现在使用asyncio终于又可以写正常一点的代码了，这种方式应当仍然需要程序员在代码中明确指出哪些函数会发生IO操作，让我们举一个例子：

```python
def transfer(amount, payer, payee, server):
    if not payer.sufficient_funds_for_withdrawal(amount):
        raise InsufficientFunds()
    log("{payer} has sufficient funds.", payer=payer)
    payee.deposit(amount)
    log("{payee} received payment", payee=payee)
    payer.withdraw(amount)
    log("{payer} made payment", payer=payer)
    server.update_balances([payer, payee])
```

从多线程的角度来看，这段代码存在并发的问题：当两个线程同时执行`transfer`函数时，它们可能都从付款人账户扣款成功而不会报错，从而造成多扣的问题

对应的显式异步的版本如下：

```python
@coroutine
def transfer(amount, payer, payee, server):
    if not payer.sufficient_funds_for_withdrawal(amount):
        raise InsufficientFunds()
    log("{payer} has sufficient funds.", payer=payer)
    payee.deposit(amount)
    log("{payee} received payment", payee=payee)
    payer.withdraw(amount)
    log("{payer} made payment", payer=payer)
    yield from server.update_balances([payer, payee])
```

现在，在最后有一个`yield from`，那我们知道只有执行到最后那一句的时候程序的执行权才会切换到其它协程，并发执行`payer.withdraw()`的两个协程，一个协程在执行到最后一句之前，另一个协程没有机会执行。

然后他据此提出为啥隐式异步的模式（gevent）也是不够的，从上面的代码我们可以看出，因为`payee.deposit()`和`payer.withdraw()`没有做`yield from`，因此我们确定在未来的迭代中这两个函数中不会出现IO操作，因而也不会出现并发调用`transfer()`的情况。

（说个题外话，其实我不太明白，从“我们必须写下`yield from`，这样我们才知道我们正在干什么”这个角度来说，为什么我们非得需要一个真正的、结构上的程序语言结构（`yield from`），而不是，比如，一个代码注释，然后这个注释可以被一个兼容gevent/eventlet协程的语法检查器(linter)所识别，然后这个`yield from`的事情就可以通过语法检查器来完成，而不用新增一个语法，这样的话，既不会对周边的三方库造成影响，也不会产生因显式异步而带来的额外开销。当然，这是另一个话题了。）

抛开显式异步的风格问题，它还有以下两个缺点：

一个是，asyncio让你可以很容易的写`yield from`语句，这使得它所声称的“可以让你少犯错”的好处大打折扣。Hacker News上的一个读者针对“显式异步的代码更容易debug”的论断做了如下评论：

> 这话的意思基本上是，“我需要在代码中显式表示出上下文切换来，如果不能的话，那这个代码的可读性就非常差”

> 我认为这是典型的稻草人论证，作者所列举的多线程代码的那些问题是所有重入代码都会出现的问题，不管它是多线程还是单线程。如果你的函数不小心调用了另一个函数，而这个函数又调用了最初的那个函数，那就会出现一样的问题

> 但是，你猜怎么样，这种情况发生的概率很低，大部分代码不是可重入的，大部分状态不是共享的。

> 对于那些并发的、有相互调用关系的代码，你必须得认真仔细考虑一下才行，到处写满`yield from`并不会解决问题。

> 在实际中，最终你会在你的代码中写满`yield from`，结果就是你会认为“哈，看起来在哪个地方都有可能切到别的协程执行了”，而这正是你一开始想解决的问题。

从我的压测代码中可以看到，这最后一点说的真是没错！下面的代码片段来自多线程版本：

```python
cursor.execute(
    "select id from geo_record where fileid=%s and logrecno=%s",
    (item['fileid'], item['logrecno'])
)
row = cursor.fetchone()
geo_record_id = row[0]

cursor.execute(
    "select d.id, d.index from dictionary_item as d "
    "join matrix as m on d.matrix_id=m.id where m.segment_id=%s "
    "order by m.sortkey, d.index",
    (item['cifsn'],)
)
dictionary_ids = [
    row[0] for row in cursor
]
assert len(dictionary_ids) == len(item['items'])

for dictionary_id, element in zip(dictionary_ids, item['items']):
    cursor.execute(
        "insert into data_element "
        "(geo_record_id, dictionary_item_id, value) "
        "values (%s, %s, %s)",
        (geo_record_id, dictionary_id, element)
    )
```

以下代码来自asyncio版本：

```python
yield from cursor.execute(
    "select id from geo_record where fileid=%s and logrecno=%s",
    (item['fileid'], item['logrecno'])
)
row = yield from cursor.fetchone()
geo_record_id = row[0]

yield from cursor.execute(
    "select d.id, d.index from dictionary_item as d "
    "join matrix as m on d.matrix_id=m.id where m.segment_id=%s "
    "order by m.sortkey, d.index",
    (item['cifsn'],)
)
rows = yield from cursor.fetchall()
dictionary_ids = [row[0] for row in rows]

assert len(dictionary_ids) == len(item['items'])

for dictionary_id, element in zip(dictionary_ids, item['items']):
    yield from cursor.execute(
        "insert into data_element "
        "(geo_record_id, dictionary_item_id, value) "
        "values (%s, %s, %s)",
        (geo_record_id, dictionary_id, element)
    )
```

注意到它们看起来基本是一样的吗？代码中有没有`yield from`并没有改变我的代码要表达的意图 —— 这是因为在无聊的与数据库交互的代码中，我们基本上都是在做一些顺序查询。

不管这个想法是否吸引人，实际上它并没多大意义 —— 在程序中使用async或者互斥锁等其它方式来控制并发在任何场景下都是不够的。相反，在真实的无聊的数据库交互代码中，有一个流程是我们绝对总是要做的，那就是：

### 通过数据库的ACID来控制并发，而不是通过进程同步机制

如果我们正在操作的对象是关系型数据库，那不管我们是使用多线程，或是隐式协程，或是显式协程并且用类似`yield from`标识出了所有可能产生并发竟态的地方，关系都不大。特别是在当今世界，几乎所有的东西都是运行在分布式集群中，那些学院理论学家关于多线程的非确定性的论断只是冰山一角，在分布式集群中并发的情况会发生在完全不同的进程中，非确定性是一定存在的。

在与数据库交互的代码中，你只有一种技术可以用来确保正确的并发，那就是通过数据库提供的事务特性（ACID）。不幸的是，它们并不是“开箱即用”的，尽管有许多出色的工具可以帮助你指引正确的方向。

从数据库角度来说，上面所有的`transfer()`例子都是错误的。下面是一个正确的版本：

```python
def transfer(amount, payer, payee, server):
    with transaction.begin():
        if not payer.sufficient_funds_for_withdrawal(amount, lock=True):
            raise InsufficientFunds()
        log("{payer} has sufficient funds.", payer=payer)
        payee.deposit(amount)
        log("{payee} received payment", payee=payee)
        payer.withdraw(amount)
        log("{payer} made payment", payer=payer)
        server.update_balances([payer, payee])
```

看到区别了吗？在上面的代码中，我们使用了事务。先通过`SELECT`查询付款人的账户余额然后再使用自动提交来修改他们的余额的做法是完全错误的。我们必须确保我们使用了某种锁机制来获取这个值，因而从我们读取这个值到我们修改它这期间，其它进程不可能基于一个“旧”值来做修改。我们可能使用`SELECT .. FOR UPDATE`来锁住我们想要修改的这一行记录；或者我们可能使用“读提交”的隔离级别结合一个版本计数器来做一个乐观锁的方案。但无论我们采取哪种方案，这都跟我们要采取的单进程的并发机制（多线程、协程等）都没有关系，因为我们这里说的并发问题涉及的是完全独立的进程之间的交互。

### 总结！

请注意，我说这些不是为了让你不要用asyncio。我认为这个库写的很好，也非常好用，也非常有兴趣把它整合进SQLAlchemy，因为我相信不管别人怎么说大家还是想要asyncio版本的SQLAlchemy...

我的观点是，当与数据库交互时，使用asyncio相对于传统的多线程范式并没有优势，而且你可能还会观察到一个小到中量的性能上的下降（而不是提升）。这是我的许多同事所熟知的，但是最近我仍然不得不争论这一点。

如果既想在接收web请求时利用非阻塞io的优势，又不想在业务逻辑里写满显式的`yield from`，那么可以考虑结合使用nginx和uWsgi。
