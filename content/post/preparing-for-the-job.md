+++
title = "preparing for the job"
date = "2014-10-06"
slug = "2014/10/06/preparing-for-the-job"
Categories = []
+++

面试太渣，皆因毫无准备。

### 多进程与多线程的区别
- 进程叫process，线程叫thread，cpu在任一时刻只能运行一个进程。一个进程可以包含多个线程
- 进程开销较大，线程开销较小
- 进程是**资源分配**的基本单位，线程是cpu调度的基本单位
- 进程有自己的地址空间，系统必须分配给它独立的地址空间，建立众多的数据表来维护它的代码段、堆栈段和数据段，是一种非常昂贵的多任务工作方式。而一个进程中的线程，它们之间共享大部分数据，使用相同的地址空间，因此切换线程比切换进程快的多，也就是说线程之间的通信比较方便，但从另一方面来说，如何控制线程对共享数据的访问又是一个难点了。
- 多进程程序比多线程程序要健壮。一个进程挂了，对别的进程基本不会有影响，但一个线程挂了，很可能会导致进程内的其它线程也挂掉。

参考：
[秒杀多线程第一篇 多线程笔试面试题汇总](http://blog.csdn.net/morewindows/article/details/7392749)

### Python如何进行内存管理？
- 在一个对象的引用计数减为0时，对此对象进行内存回收

参考：
[Python深入06 Python的内存管理](http://www.cnblogs.com/vamei/p/3232088.html)

### Python里如何拷贝一个对象？
- 首先要知道为什么要复制对象，因为在Python中无论是把对象作为参数传递还是作为函数返回值，都是引用传递的。那当我们不想传递引用的时候就要复制一个全新的对象了。
- 复制分为浅复制和深复制。浅复制是指只复制对象，但对象里的元素仍然使用引用。深复制（deepcopy）就是完完全全的复制了。

参考：
[深复制和浅复制](http://blog.csdn.net/sharkw/article/details/1934090)

### RE模块中search和match的区别
- match只搜寻第一个是否满足，若满足返回它，若不满足，则返回None
- search搜索字符串，返回它遇到的第一个满足条件的，而不仅仅是查看第一个
注意这两个命令都是只返回一个满足条件的，要想返回所有满足条件的，要用findall

参考：
[search vs match](https://docs.python.org/2/library/re.html#search-vs-match)

### 用Python匹配HTML tag的时候，`<.*>`和`<.*?>`有什么区别？
贪婪模式和非贪婪模式。
The `'*'`, `'+'`, and `'?'` qualifiers are all greedy; they match as much text as possible. Sometimes this behaviour isn’t desired; if the RE `<.*>` is matched against `'<H1>title</H1>'`, it will match the entire string, and not just `'<H1>'`. Adding `'?'` after the qualifier makes it perform the match in non-greedy or minimal fashion; as few characters as possible will be matched. Using `.*?` in the previous expression will match only `'<H1>'`.
### python程序中文输出问题怎么解决？
中文输出的问题都是由于编码问题。

中文字符串使用的编码与其所在的文件使用的编码一致。所以正确处理的步骤是：先用其原有编码decode成python内部的unicode，做了该做的处理后，什么时候想输出啦，这时候再使用encode，把它encode成任何想输出的编码形式。

### 正则表达式匹配ip
`src = "security/afafsff/?ip=123.4.56.78&id=45"`，请写一段代码用正则匹配出ip
	regx = re.compile(r'([0-9]+)\.([0-9]+)\.([0-9]+)\.([0-9]+)')
	m = regx.search("security/afafsff/?ip=123.4.56.78&id=45")
	m.groups()
	('123', '4', '56', '78')
注意，使用groups可以返回找到的所有subgroups，非常方便。另外，group(0)返回整个匹配，group(1)返回第一个subgroup等等

### json
写一段代码用json数据的处理方式获取{"persons":[{"name":"yu","age":"23"},{"name":"zhang","age":"34"}]}这一段json中第一个人的名字。
	j = json.loads('{"persons":[{"name":"yu","age":"23"},{"name":"zhang","age":"34"}]}')
	j.get('persons')[0]['name']

### 平衡点
比如int[] numbers = {1,3,5,7,8,25,4,20}; 25前面的总和为24，25后面的总和也是24，25这个点就是平衡点；假如一个数组中的元素，其前面的部分等于后面的部分，那么这个点的位序就是平衡点 
我写出的版本，完全就是把题意翻译了一遍，能满足要求，效率应该不好
	def findMiddle(lst):
	    for i in xrange(1, len(lst)-1):
		if sum(lst[:i]) == sum(lst[i+1:]):
		    print i
看到别人的做法是，先算一遍总和，然后从左往右一个一个加，出现和一半的时候，说明找到平衡点了。
	def findMiddle2(lst):
	    totle = sum(lst)

	    add = 0 
	    for i in xrange(0, len(lst)-1):
		add += lst[i]
		if totle - lst[i+1] == 2 * add:
		    print i+1, lst[i+1]
### 支配点
支配数：数组中某个元素出现的次数大于数组总数的一半时就成为支配数，其所在位序成为支配点；比如int[] a = {3,3,1,2,3};3为支配数，0，1，4分别为支配点； 
	def findDomain(lst):
	    count = [ i for i,x in enumerate(lst) if lst.count(x) > len(lst)//2 ]  # elegant solution
	    if count:
		print lst[count[0]], count

### 什么是PEP 8？
Python编码规范
