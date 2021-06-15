+++
title = "trivial problems"
date = "2015-01-20"
slug = "2015/01/20/trivial-problems"
Categories = []
+++

今天很高兴，因为解决了几个困扰很久的问题。

## OpenVPN
为便于办公，公司提供了VPN，不过只提供了OpenVPN的连接方案，之前在arch上一直是用pptp连接的，大概看了看wiki上OpenVPN的文档，一大堆。。。头大。

不过，今天仔细一看，原来并不麻烦，只要把管理员提供的证书文件等放到相应文件夹上就搞定了！
>将这几个文件放到/etc/openvpn目录下   xxx.ovpn ca.crt  xxx.crt  xxx.key  其中xxx.pvpn是配置文件
>然后就可以启动了！ sudo openvpn /etc/openvpn/xxx.ovpn   会提示输入密码，输入管理员提供的密码即可

## urxvt SSH远程登陆的问题
之前在Arch上用urxvt一直比较happy，不过公司里开发都是SSH远程登陆到linux主机上进行的，那么问题来了，明明在本地显示很正常的vim，SSH到了公司上的Linux主机上，显示就成了一堆渣。。明明vim的配置文件都是一样的啊。。  后来慢慢发现不是vim的问题，是终端的问题！！因为我换用lilyterm后就显示正常了！！然后，终于在今天让我google到解决方案了～
>1. Run the following command on the external host to make a terminfo directory
>   under the logged in user's home folder.
>
>       mkdir -p ~/.terminfo/r/
>
>2. Copy the appropriate terminal profile from your local machine to the newly
>   created folder on the remote host.
>
>       scp /usr/share/terminfo/r/rxvt-unicode-256color user@helloworld.com:.terminfo/r/
>
>3. Restart the SSH connection. It should work now.

## vim中使用tab自动补全
对于程序员来说，这世界上如果没有自动补全的话，简直不能想象。。之前习惯了使用TAB自动补全，现在在公司的远程Linux上却怎么也搞不出TAB自动补全的效果。。配置文件和插件都装一样的也不行。。。真是见鬼了。。 今天发现应该是之前装了supertab，然后不知道怎么又自己给删了，但是效果还在。。在远程Linux上装了supertab后，嗯，就是这个感觉～～

本事件的教训就是，无论做什么一定要做好记录。。
## 参考
- [FULL URXVT TERMINAL SUPPORT FOR REMOTE (SSH) CONNECTIONS](http://www.cs.helsinki.fi/u/andrews/misc/full_urxvt_support_on_ssh_terminals.txt')
- [Installing and setting up OpenVPN in ArchLinux]('https://stavrovski.net/blog/installing-and-setting-up-openvpn-in-archlinux')
- [Configuring OpenVPN to connect as a client to an AirVPN server]('https://wiki.archlinux.org/index.php/Airvpn')

