+++
title = "zhihu authentication analysis"
date = "2017-03-20"
slug = "2017/03/20/zhihu-authentication-analysis"
Categories = []
+++

##知乎以及得到手机App与Server端的认证机制分析

###抓包

采用Charles进行抓包

- 一个关键点是如何抓SSL包
    - [https://www.charlesproxy.com/documentation/using-charles/ssl-certificates/](https://www.charlesproxy.com/documentation/using-charles/ssl-certificates/)

    需要在手机上安装Charles自己的证书，但注意**从Android N开始，必须在手机App的配置文件里显式的指定信任Charles的根证书才行。**

在分析得到App与Server端的交互时发现，几乎每个链接都会带一个sign参数，如果不传递这个参数的话，Server端会返回错误，那么这个sign参数是从哪里来的呢？这个时候就需要反编译一下APK来分析一下了。

###逆向APK
逆向APK所用到的工具主要是Jadx，有时可能还需要Hopper Dissembler，下面会详细介绍

- Jadx

    来自GitHub首页的介绍：Command line and GUI tools for produce Java source code from Android Dex and Apk files，非常简单易用，**反编译的时候记得在配置里把反混淆勾上。**
    
    反编译完成后，可以直接在jadx-gui里浏览源代码，也可以导入Android Studio里查看，再配合一些关键字用grep在命令行定位代码不要太爽啊~~~
    
在分析反编译出来的代码过程中发现，某个关键的函数找不到定义的地方，样子如下：

        public static native String keyBaseFromJNI();
        
一番搜索后发现，这种写法是调用了C或者C++写的底层库文件（比如so或者dll文件），这样做的原因显然是：1. 效率更高；2. 增加破解难度。下面就该反汇编大神出场了。
    
- Hopper Dissembler

    Hopper is a tool that will assist you in your static analysis of executable files. 
    
    - [https://www.hopperapp.com/tutorial.html](https://www.hopperapp.com/tutorial.html)
    - [https://bestswifter.com/app-crack/](https://bestswifter.com/app-crack/)
    - [http://www.10tiao.com/html/410/201701/2656257139/1.html](http://www.10tiao.com/html/410/201701/2656257139/1.html)

    直接把从APK中提取出的so文件导入Hopper Dissembler即可，注意这个工具最牛逼的一点是可以生成C伪代码。
- TODO
    - 搞明白混淆是怎么做的
    - 为啥反编译后的代码有些是正常的，有些变量名已经被改成乱码了，基本不可读

分析知乎客户端与Server认证机制时遇到一个签名算法HMAC，顺便把各种签名算法学习一下

###签名算法

- hmac-sha1
- md5
- sha1
- 各种算法的优缺点

to be continued

