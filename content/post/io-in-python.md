+++
title = "理解Python中的stdin stdout stderr"
date = "2014-10-07"
slug = "2014/10/07/io-in-python"
Categories = []
+++

看到一个题目，要求用Python实现linux中的命令tee的功能，tee的功能还是很简单的，就是把接收到的标准输入再输出到重定向和一个文件中。这些用Python来做应该不是什么难事，可是在Python里面怎么接收标准输入呢？

首先，有这样一个层级关系，`print`和`raw_input`这样的高级命令调用的是sys.stdin和sys.stdout，而sys.stdin和sys.stdout是file like object（类似于linux中一切设备皆文件），这一层的实现是Python的io模块。

下面通过两个例子可以清楚的看出print和sys.stdin和sys.stdout的关系。每个例子中第一句实现的效果跟后面的一沱实现的效果是一样一样的～
	######### stdout test ########
	print 'hello world'

	import sys
	sys.stdout.write('hello world')

	######### stdin test ########
	print 'hi, %s' %raw_input('please input your name: ')

	import sys
	print 'please input your name: ',
	name = sys.stdin.readline()[:-1]
	print 'hi, %s' %name
	#############################
比较好玩的事情是可以通过更改sys.stdin sys.stdout来实现类似linux中的重定向操作。比如默认stdout是终端，你可以在代码里通过`f = open(filename, 'w')` `sys.stdout = f`来把输出重定向到一个文件。

下面说下更底层的io模块。

像我们平时通过open等命令建立file like object后所使用的所有命令都是在io模块里实现的，比如write read readline seek flush等等。

io分三种类型：text I/O、binary I/O、raw I/O，以前总是傻傻分不清楚，现在终于理解了～    

text I/O顾名思义，就是对我们人类来说能看到的、能识别的文字数据。无论什么数据都是以二进制的形式存储在计算机里的，但人类能识别的文字是以编码（assic、gbk、utf8）的形式存储在计算机里的，因为最终还要拿出来给人类读啊～～

binary I/O就是指所有非文本信息了，这类数据无需编码解码，也不用处理换行符，比如图片文件等。这个类型对应的open里的打开模式`'b'`

raw I/O文档里说不常用，目前我也没理解它是干啥用的。。

最后，简陋的tee
	import sys 

	def tee(filename):
	    with open(filename, 'w') as f:
		for line in sys.stdin.readlines():
		    sys.stdout.write(line)
		    f.write(line)

	if __name__ == '__main__':
	    filename = sys.argv[1]
	    tee(filename)


### 参考
- [Python Programming/Input and Output](http://en.wikibooks.org/wiki/Python_Programming/Input_and_Output)
- [标准输入、输出和错误](http://woodpecker.org.cn/diveintopython/scripts_and_streams/stdin_stdout_stderr.html)
- [io — Core tools for working with streams](https://docs.python.org/3.1/library/io.html#module-io)
- [Python之sys模块小探](http://5ydycm.blog.51cto.com/115934/304324)
