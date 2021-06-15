+++
title = "socks代理和http代理的区别"
date = "2014-10-05"
slug = "2014/10/05/socks-proxy-and-http-proxy"
Categories = []
+++

一直傻傻分不清楚，都是因为平时很少接触，虽然翻墙常用，但也只是按教程一步一步来，并未知其所以然。

之前一直听别人说socks代理多么多么好，自己只是模模糊糊半懂不懂，现在终于切身体会到了一个好处，用socks代理后就没有证书问题的困扰了～～之前用的GoAgent使用的是http代理，每次访问https网站时，总是提示证书有问题，导入GoAgent的自带证书还是有问题，这下用了shadowsocks后一切都清净了。。为什么socks代理就这么牛逼呢？因为socks代理更底层，只负责发送接收数据，至于你是什么http ftp tcp udp它根本就不care。

- socks代理更底层，是在会话层；而http代理是在应用层。因此socks代理可以代理一切客户端的连接，而http代理只能代理使用http协议的客户端
- 由于更底层，不需要处理高级协议的细节，所以socks代理更快
- socks协议分v4和v5版本，v4只能代理tcp协议，而v5什么协议都可以代理 

