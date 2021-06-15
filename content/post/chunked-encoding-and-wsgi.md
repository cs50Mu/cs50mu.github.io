+++
title = "chunked encoding and wsgi"
date = "2017-10-15"
slug = "2017/10/15/chunked-encoding-and-wsgi"
draft = true
Categories = []
+++

记录一次问题定位过程。

一天，接到业务组的同学反馈：“你们的某个服务在仿真环境不可用了”。看到报错信息后我第一反应就是，是他们传递的参数有误，因为：1. 近期代码没有改动，只是由ECS
部署改为了Docker部署；2. 报错信息提示post的json数据有问题。

然而，跟业务组同学反复确认后发现，他们post的数据并没有错。。进一步定位后发现，原来不是post参数有误，而是应用里读取到的post data为空了！

跑到运维同学那里，进入容器环境用tcpdump抓了下包，发现进入应用框架(Django)之前post数据还是在的。。

```
tcpdump -l -s 0 -w - dst port 5000 | strings
```

抓包数据如下：

```
POST /api/purchases/ HTTP/1.1
Host: jccoin.dev.igetget.com
User-Agent: Go-http-client/1.1
Transfer-Encoding: chunked
Content-Type: application/json
Accept-Encoding: gzip
X-Forwarded-For: 10.30.47.54
Connection: close
{"purchase_no":"b7eb4vd86rijdi8i5sa0","category":2,"sys_code":"IGET","buyer_id":"21075408","device_type":"ANDROID","product_category":"53","product_id":"1","product_name":"
","coins_amount":335,"tag":"","sign":"ffc88ffc0b0aaf191d11b0909575f40c"}
```

但是，这怎么可能？！

切换回ECS环境后接口恢复正常了，于是在ECS环境重新抓包如下：

ECS环境上应用框架(Django)前有一个nginx，进入nginx前的包数据如下：

```
POST /api/purchases/ HTTP/1.1
Host: jccoin.dev.igetget.com
User-Agent: Go-http-client/1.1
Transfer-Encoding: chunked
Content-Type: application/json
Accept-Encoding: gzip
{"purchase_no":"b7ep9jt86rik99q3q6eg","category":2,"sys_code":"IGET","buyer_id":"645563","device_type":"ANDROID","product_category":"55","product_id":"2","product_name":"30
","coins_amount":48,"tag":"30","sign":"68fff62307beec83399933c5d2bd278d"}
```

从nginx出来，进入应用框架(Django)之前的包数据如下：

```
POST /api/purchases/ HTTP/1.0
Host: jccoin.dev.igetget.com
X-Real-IP: 10.30.47.54
X-Forwarded-For: 10.30.47.54
Connection: close
Content-Length: 263
User-Agent: Go-http-client/1.1
Content-Type: application/json
Accept-Encoding: gzip
{"purchase_no":"b7ep9jt86rik99q3q6eg","category":2,"sys_code":"IGET","buyer_id":"645563","device_type":"ANDROID","product_category":"55","product_id":"2","product_name":"30
","coins_amount":48,"tag":"30","sign":"68fff62307beec83399933c5d2bd278d"}
```

可以看到，经过nginx后数据由chunked变成了content-length。

从上面的情况猜测应用框架对chunked encoding支持有问题，一番google后发现果然如此:

- [https://stackoverflow.com/questions/12091067/handling-http-chunked-encoding-with-django](https://stackoverflow.com/questions/12091067/handling-http-chunked-encoding-with-django)
- [https://github.com/pallets/flask/issues/2229](https://github.com/pallets/flask/issues/2229)
- [http://mathslinux.org/?p=564](http://mathslinux.org/?p=564)
- [https://twitter.com/davidism/status/888426616602230784](https://twitter.com/davidism/status/888426616602230784)

- [http://lucumr.pocoo.org/2011/7/27/the-pluggable-pipedream/](http://lucumr.pocoo.org/2011/7/27/the-pluggable-pipedream/)
- [https://www.python.org/dev/peps/pep-0333/#handling-the-content-length-header](https://www.python.org/dev/peps/pep-0333/#handling-the-content-length-header)
- [http://rhodesmill.org/brandon/2013/chunked-wsgi/](http://rhodesmill.org/brandon/2013/chunked-wsgi/)

所以这么看来，Django这一类框架在前面挂一个nginx是必须的。
