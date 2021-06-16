+++
title = "thrift server error caused by aliyun health check"
date = "2017-01-14"
slug = "2017/01/14/thrift-server-error-caused-by-aliyun-health-check"
Categories = []
+++

近期遇到一个thrift server方面的问题，记录一下。

### Background

server做的事情其实很简单，就是一个图片上传服务，别人把图片传过来，服务器上传到阿里云的云上，再返回一个图片地址。

问题暴露出来的现象是：这个服务每隔一段时间就会“挂掉”，重启服务后会好那么一段时间然后又会“挂掉”。客户端调用返回`TSocket: Could read 4 bytes from ....`，但去线上ps会发现，server进程其实是在的，
也就是说服务器其实只是不响应了，并没有真正挂掉。

### First guess

分析后台日志发现，大量重复的报错信息`Connection reset by peer`，以及偶尔的错误信息`too many open files`（其实，这个偶尔也只是猜测，因为并没有完整的确认过），当时看到这个日志后，就猜测
原因是：请求太多，导致该程序打开的文件描述符超过限制，从而拒绝服务。由于是线上服务，出现问题时为了及时恢复服务，当时就重启了服务，因此也并没有验证这个猜测。但看了下操作系统对单个程序
打开文件数的限制:

    $ ulimit -n 
    65535

基于too many open files这个错误加上这个thrift server使用的ThreadPoolServer类型，google了一番，找到几个类似问题的帖子：

- [http://blog.csdn.net/heavendai/article/details/8614941](http://blog.csdn.net/heavendai/article/details/8614941)
- [http://web.archive.org/web/20110103083546/http://blog.rushcj.com/2010/12/20/thrift-close-wait/](http://web.archive.org/web/20110103083546/http://blog.rushcj.com/2010/12/20/thrift-close-wait/)
- [http://blog.csdn.net/hwz119/article/details/1611182](http://blog.csdn.net/hwz119/article/details/1611182)

里面讲到，ThreadPoolServer来说，它使用的是定长线程池来服务的，当并发太多时使得现存的线程数无法满足要求时，就会出现很多`CLOSE_WAIT`状态的连接，最终会
把当前程序的文件描述符占满，从而出现`too many open files`的错误。正好这个服务用的线程是默认的，只有10个，当时就认为问题就在这里了。

### Second guess

然后第二天又问了下服务调用方，其实这个服务的使用频率很低，只有在用户更改头像的时候才会调一下，按理说不会有太多并发量，所以这样看来，上面的猜测就不成立了。然后去线上看了下，该服务一共部署了两台机器，
发现其中一台机器上的服务又“挂了”，抓住这个好机会，看了下这个服务的socket连接情况：

    # 先找到程序的进程id
    $ ps aux | grep 'xxx'
    # 然后再用lsof看
    $ lsof -i -P | grep 28719
    python  28719  www    4u  IPv4 261027973      0t0  TCP *:9455 (LISTEN)
    python  28719  www    5u  IPv4 261328458      0t0  TCP 10.171.20.131:9455->100.109.225.0:45587 (ESTABLISHED)
    python  28719  www    6u  IPv4 261327693      0t0  TCP 10.171.20.131:9455->100.109.221.128:23814 (ESTABLISHED)
    python  28719  www    7u  IPv4 261429153      0t0  TCP 10.171.20.131:9455->100.109.224.0:17384 (ESTABLISHED)
    python  28719  www    8u  IPv4 261717598      0t0  TCP 10.171.20.131:9455->100.109.225.0:23881 (ESTABLISHED)
    python  28719  www    9u  IPv4 261429722      0t0  TCP 10.171.20.131:9455->100.109.225.128:19752 (ESTABLISHED)
    python  28719  www   10u  IPv4 261470638      0t0  TCP 10.171.20.131:9455->100.109.220.128:14370 (ESTABLISHED)
    python  28719  www   11u  IPv4 261456259      0t0  TCP 10.171.20.131:9455->100.109.224.128:24267 (ESTABLISHED)
    python  28719  www   12u  IPv4 261714765      0t0  TCP 10.171.20.131:9455->100.109.220.0:8860 (ESTABLISHED)
    python  28719  www   13u  IPv4 261470701      0t0  TCP 10.171.20.131:9455->100.109.221.0:42186 (ESTABLISHED)
    python  28719  www   14u  IPv4 261717599      0t0  TCP 10.171.20.131:9455->100.109.224.0:3487 (ESTABLISHED)
    python  28719  www   98u  IPv4 261718019      0t0  TCP 10.171.20.131:9455->100.109.220.0:37454 (ESTABLISHED)
    python  28719  www  827u  IPv4 261722045      0t0  TCP 10.171.20.131:9455->10.44.155.227:24038 (CLOSE_WAIT)
    python  28719  www  914u  IPv4 261722497      0t0  TCP 10.171.20.131:9455->10.172.224.17:35876 (CLOSE_WAIT)
    python  28719  www 1309u  IPv4 261724582      0t0  TCP 10.171.20.131:9455->10.172.142.166:52953 (CLOSE_WAIT)
    python  28719  www 1329u  IPv4 261724675      0t0  TCP 10.171.20.131:9455->10.44.155.227:52921 (CLOSE_WAIT)
    python  28719  www 1672u  IPv4 261726365      0t0  TCP 10.171.20.131:9455->10.44.191.118:7070 (CLOSE_WAIT)
    python  28719  www 1694u  IPv4 261726531      0t0  TCP 10.171.20.131:9455->10.170.210.248:39802 (CLOSE_WAIT)

可以看到`CLOSE_WAIT`状态的连接并不多，但此时这台机器上部署的服务已经不能用了。同时又去现在还可用的机器上看了下，发现只有状态为`ESTABLISHED`的连接，并没有状态为`CLOSE_WAIT`的连接，
同时发现日志在不停的报错：`Connection reset by peer`。但并没人反馈服务不可用啊，看了下图片上传还是可用的，正百思不得其解中，运维哥哥说，这不会是阿里云的健康检查导致的吧！于是，
暂停了一下健康检查，果然日志的报错停止了。

然后我的猜测又变成了：由于阿里云的持续不断的健康检查（2秒一次）把服务检查挂了。。显然，运维是不会认可这个猜测的，他们说报这个错是正常的，别的服务也有这个问题，只要不理会它就行了。
说实话，光看这个，确实无法断定服务挂是由于健康检查的原因。况且我也查了下阿里云所谓的健康检查，找到了如下官方资料：

- [https://help.aliyun.com/knowledge_list/39451.html?spm=5176.7739464.6.753.FE0164](https://help.aliyun.com/knowledge_list/39451.html?spm=5176.7739464.6.753.FE0164)
- [https://help.aliyun.com/knowledge_detail/39455.html?spm=5176.7839451.2.4.KKMkQd](https://help.aliyun.com/knowledge_detail/39455.html?spm=5176.7839451.2.4.KKMkQd)
- [https://help.aliyun.com/knowledge_detail/39464.html?spm=5176.7839451.2.13.KKMkQd](https://help.aliyun.com/knowledge_detail/39464.html?spm=5176.7839451.2.13.KKMkQd)

看来阿里云官方确认认为这个错误是正常的。

第二天，又去线上看了下，发现某一台机器上的服务又“挂了”，看了下这次的“车祸现场”，注意到了一点上次没有注意的细节：

- 有11个状态为`ESTABLISHED`的连接
- 这个11个状态为`ESTABLISHED`的连接的另一边都是来自阿里云健康检查的网段ip
- 此时，如果调用该服务会返回错误，而且会增加一条状态为`CLOSE_WAIT`的socket连接
- 对于目前可用的机器上的服务，没有状态为`CLOSE_WAIT`的连接，但有几个状态为`ESTABLISHED`的连接，而且尽管没有人调用它，状态为`ESTABLISHED`的连接数在慢慢增加。

由此，可以基本断定，罪魁祸首就是阿里云的健康检查了。健康检查把当前服务的可用连接数（10个线程只能同时服务10个连接）用完了，并且没有释放，再进来的连接只能wait了。那么现在唯一的问题
就是：Why？阿里云的健康检查为什么会对thrift服务造成这样的结果？我还没有找到原因。从阿里云的官方文档中看到，所谓的TCP健康检查，是对指定ip的指定端口进行了三次握手然后发送了RST断开了
连接，没有看到他们说会造成连接一直不释放的情况。

google到一篇阿里云上部署thrift server出问题的文章，里面也提到了健康检查：

- [http://www.concurrent.work/one-java-thrift-case-caused-by-default-parameters/](http://www.concurrent.work/one-java-thrift-case-caused-by-default-parameters/)

### 反思

- 不要急于下结论。先把日志完整分析下，服务的使用场景也要了解下，再下结论，而不要看到一点问题就瞎猜。
- 需要了解下网络方面的基础知识了。
