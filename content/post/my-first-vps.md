+++
title = "捣鼓VPS翻墙"
date = "2014-07-14"
slug = "2014/07/14/my-first-vps"
Categories = ["vps", "GFW"]
+++

在学校论坛上看到有人介绍在vps上搭建shadowsocks服务器来翻墙，看到他推荐的vps很便宜，一年才20多，于是立马也想买一个了，其实以前就想折腾vps的，无奈找到的vps价格都太高，一个月要几百块神马的，像我这么小气的人肿么可能舍得。。不过，这个实在是太便宜了啊，虽然配置是最低的了，72M内存，2G硬盘，但一个月有100G的流量，对于翻墙来说够用了，果断注册下单购买，只支持PayPal支付，于是又注册PayPal，在这上面折腾了很久，一开始注册成中国版的了，结果提示无法跨境支付，放狗搜索后发现，需要注册国际版的，以为还要审核神马的，结果直接绑定银行卡后就可以支付了，很方便啊！

可是才用一天就被墙了！！我目前使用的是长城宽带的服务，ssh都无法登陆了，但后来发现使用学校自己的外网还是可以用的，学校的外网应该是用的联通的服务，后来我到V2EX上问了下发现有人用长城宽带的服务也有类似的问题，该死的长城宽带！

##更新（20141002）
不久前听人说Bandwagon出加州机房了，一直懒得弄，今天花时间看了下，果然有加州机房了，果断迁移过去，给换了个新ip，然后shadowsocks不能用了，一开始以为是shadowsocks服务器端没开，后来发现是服务器端的设置里ip也需要更新下才行。

正好记录下shadowsocks服务器端的配置，省得下次再忘。

- shadowsocks在服务器端是以daemon形式运行的。daemon脚本在`/etc/init.d/shadowsocks`，配置文件在`/etc/shadowsocks/config.json`
- 要启动/关闭/重启shadowsocks，只需在命令行执行`/etc/init.d/shadowsocks start/stop/restart`，它默认开始是自启动的。
- 刚刚才知道，原来shadowsocks是socks代理，因此可以无压力访问https网站，再也不会遇到GoAgent一样的证书问题了～～哈哈，真好

对了，火狐代理插件换成foxyproxy了，autoproxy已经好久不更新，基本无法使用了，之前一直迟疑不想换是因为听说foxyproxy配置比较麻烦，今天试了试才发现对于经常折腾的人来说根本就不是事～～ foxyproxy也支持订阅gfwlist列表，于是生活又美好起来了～～
##参考
- [shadowsocks-libev](https://github.com/madeye/shadowsocks-libev)
- [CentOS、Debian下搭建shadowsocks-libev服务端](http://www.lucong.com.cn/lulu/centos-debian-shadowsocks-libev.html)
