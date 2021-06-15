+++
title = "在WR703N上刷openwrt"
date = "2013-06-21"
slug = "2013/06/21/flash-openwrt-in-wr703n"
Categories = ["WR703N", "路由器", "openwrt", "crontab"]
+++

买这个传说中很牛逼的路由器已经很久了，直到现在才想起来刷。。当初买它是因为一直用的一个无线路由老是罢工，一开始以为是学校网络的问题，后来发现不是，是路由器的问题，于是想刷个开源固件玩玩，可惜上网一搜发现这个路由器刷不了开源固件。。于是一番搜索后发现大家对现在这款（WR703N）路由的评价不错，好多用它来刷开源固件的，看得心痒痒啊，特别是这篇：[入手神器TL-WR703N一日谈](http://zihua.li/2012/03/about-tl-wr703n/)，害我直接又多买了个8G U盘！

刷机教程看了很多篇，不过最后主要是参照这篇来的：[在wr703n下安装配置openwrt、openvpn](http://www.imtxc.com/blog/2013/02/24/install-and-configure-openwrt-on-wr703n/)，写得很不错，我基本是按着他的步骤来的，一开始有一步没照着他的来，结果最后从USB启动后连接不上无线了。。。然后，晚上睡觉的时候突然想到会不会是因为没有做这一步导致的呢？于是早上起来后重新弄，因为无线和有线都连接不上了，所以一开始又折腾了好久，还好openwrt有安全模式，进入方式在上面的帖子里也有介绍，他说的这个方法是正确的，我一开始用的是openwrt文档里说的方法，说开机后要先等个2秒钟什么的，然后按住reset键，等指示灯快速闪烁后就ok了。可是我试了下根本不管用，还是那个帖子里说的靠谱些，正确的做法是：**开机后立即持续的按reset键，不要按住就不放了，要不停的按住松开，间隔时间要短。**    

然后就是配置U盘扩展了，这真是一个绝佳的workaround，把系统装在U盘里，这真是太妙了啊！因为路由器的内存和flash都很低，可能刚刚够装开源固件的（这算好的，一些路由器没法装就是因为flash太小，装不下！），但要想玩点别的，比如装个vpn支持，装个python啥的就别想了。但一旦把系统装在U盘里就没有空间限制了，可以随便折腾。安装过程帖子里写的很详细了，有几点不明白：   


- `mount -o bind / /tmp/root`的含义，我知道是把根目录绑定到/tmp/root下，但我想深入了解下，为什么需要先绑定再复制？为什么不直接复制呢？   
解答：bind参数的意思是将已挂载的文件系统挂载到目录树的另一个位置，所以我猜这么做的原因是若直接`cp /* /mnt/usb -a`会引起歧义，因为`/*`也包括`/mnt/usb`
- `/etc/config/fstab`中的配置如何理解？为何需要1G的交换空间？有必要分这么多吗？   
解答：[The OpenWrt Flash Layout](http://wiki.openwrt.org/doc/techref/flash.layout)，这篇wiki详细讲了openwrt对存储结构的定义，特别是讲到了`/` `/rom` `/overlay`三个文件夹的作用。   
根据这篇wiki[Fstab Configuration](http://wiki.openwrt.org/doc/uci/fstab)，swap没必要设置这么大，内存512M以下设置为内存的2倍就可以了，所以重新分了区，把原来的1G swap分区删掉合并到原来的数据分区里，建了个64M的swap文件：    

		$ dd if=/dev/zero of=/swapfile bs=1M count=64
		$ mkswap /swapfile
		$ swapon /swapfile
但是这样需要手动激活swap空间，想了下，在`/etc/rc.local`里添加`swapon /swapfile`解决！
- 我没有照做的一步是设置/etc/config/network时，没有更改lan的ip地址为192.168.20.1（默认是192.168.1.1），虽然改了后重新配置后问题就解决了，但我还是不理解为什么。

- 最后一个问题是pptp client的问题，安装很容易，按照官方文档很轻松设置完成后，ssh到路由器上ping外网已经可以ping通了，问题是在我的笔记本上ping外网却不行，总是提示network unreachable。。根据官方文档设置最后需要指定一下路由，之前都没有太关注过路由的概念，所以现在还是很迷糊。。不知道该怎么设置路由，按说路由器上能访问外网，那不用做什么设置客户端就应该能访问外网啊！折腾了好久也没搞明白，最后按照网上搜到的一个方法莫名其妙的搞定了。。不过，依然不太明白原理。具体做法如下，在`/etc/config/network`里添加：      


		config 'zone'
			option 'name'      'wan'
			option 'network'   'wan vpn'
			option 'input'     'REJECT'
			option 'forward'   'REJECT'
			option 'output'    'ACCEPT'
			option 'masq'      '1' 

不过，按上面设置后还是不起作用的，又到web界面的`network-->firewall`里设置了下才ok的，就改了下covered network选项，原来就只有wan选上了，我看到上面的配置文件里network选项里有两个（wan和vpn），所以我把vpn也勾选上了，然后重启firewall后就ok了。。。


####Crontab定时任务
`crontab -l` 显示定时任务   
`crontab -e` 编辑定时任务   
`crontab -r` 删除定时任务   
#####格式说明：    
`* * * * * /command path`  前5个字段分别表示分钟（0-59），小时（1-23），日期（1-31），月份（1-12），星期（0-6，0表示周日）   
#####特殊符号：   
`*`表示任意时刻   
`,`表示分割   
`-`表示一个时间段   
`/n`表示每n单位执行一次   
如果不想让定时任务的执行信息打印到屏幕上的话，可以这样`0 2 * * * /u01/test.sh >/dev/null 2>&1 &`，这句话的意思就是在后台执行这条命令，并将错误输出2重定向到标准输出1，然后将标准输出1全部放到/dev/null 文件，也就是清空。    
#####`2>&1`写在后面的原因   
首先是`command > file`将标准输出重定向到file中，`2>&1`是标准错误拷贝了标准输出，也就是同样被重定向到file中，最终结果就是标准输出和错误都被重定向到file中。   
如果改成： `command 2>&1 >file`，`2>&1`标准错误拷贝了标准输出的行为，但此时标准输出还是在终端。`>file`后输出才被重定向到file，但标准错误仍然保持在终端。


###参考
[在wr703n下安装配置openwrt、openvpn](http://www.imtxc.com/blog/2013/02/24/install-and-configure-openwrt-on-wr703n/)   
[使用tp-link tl-wr703n和openwrt作为bt下载机](http://dev.bjtu.edu.cn/ideal/2013/04/14/use-openwrt-as-bt-downloader/)    
[openwrt下pptp-vpn client的使用！！](http://www.openwrt.org.cn/bbs/forum.php?mod=viewthread&tid=1480)   
[Linux Crontab 定时任务 命令详解 ](http://blog.csdn.net/tianlesoftware/article/details/5315039)   
