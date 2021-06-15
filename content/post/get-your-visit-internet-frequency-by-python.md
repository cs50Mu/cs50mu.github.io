+++
title = "统计浏览器历史记录"
date = "2014-03-03"
slug = "2014/03/03/get-your-visit-internet-frequency-by-python"
Categories = ["python", "编程"]
+++

又是好久没更新。

知乎是个好网站。那么多大牛分享自己的经验教训～涉及范围包罗万象，无所不有。对我而言，很重要的一点是它拓展了我的眼界，让我知道，哦，原来还可以这么想，事情还可以这么做，这世界上原来还有这样一种活法。。

逛知乎的时候还是不由自主的看起编程方面的提问了，其中有一个问题是“用python可以做哪些有趣的事？”看到大家各种有趣的分享，我想这真是太酷了！一直以来，虽然对编程很感兴趣，但总是深入不进去，刚刚有点想法的时候，可总是考虑的太复杂，牵扯的知识点太多，然后要看很多东西，最后热情就这么一点点被消磨掉了，然后就是扔下好久不去想它，直到某天忽然又有热情了，如此循环往复。。看似做了很多，其实却一直在原地踏步。看了这么多知乎牛人的回答后，有了一点点启发，也有了具体的指导，那么就按各位大牛的做法来试试，从自己感兴趣的、容易实现的小玩意儿开始吧。

要统计浏览器的访问记录，首先要找到访问记录，看起来简单，以为只要随便导出一下就可以了，实际还颇费了一番周折。火狐的历史记录以前是以xml格式存储的，现在则是用数据库格式存储的，位置是在`%appdata%\Mozilla\Firefox\Profiles\xxxxx.default\places.sqlite`。东西是找到了，可是肿么读出来呢？尼玛是数据库文件啊，我还不会用肿么办呢？google了一番后发现可以用firefox的一个扩展来打开读取，这个扩展叫SQLite Manager，相当赞，各种功能齐全，关键支持csv导出！好的，长话短说，导出为csv文件后就可以请python出场了！

基本步骤是，先从csv文件中按行读出来，这数据库还是蛮大的，共13692条记录，然后从行中提取出需要的网址信息，放入一个列表中，然后再利用字典来统计各个网址的频度，程序如下：
	#!/usr/bin/python

	import csv
	histList=[]
	histDict={}
	with open('f:\moz_places.csv','rb') as f:
	    reader = csv.reader(f)
	    for row in reader:
		histList.append('/'.join((row[1].split('/'))[:3]))


	for place in histList:
	    if place not in histDict:
		histDict[place] = 1
	    else:
		histDict[place] += 1

	sortedDict = sorted(histDict.iteritems(),key=lambda d:d[1],reverse=True)

	for (place,freq) in sortedDict[:50]:
	    print '%s\t\t%s' %(freq,place)
PS，再总结下这几天学到的新东西：

- splinter。python下的一个网页应用测试库，就是可以操纵浏览器来与网站交互，玩了下还挺好上手，比底层一点的request神马的友好多了，之前还一直坚持不用这种直接操作浏览器的库，嫌太低端，现在看来，还是先从基础开始慢慢来吧。
- xpath。一种选择xml文件中节点的方法，通过它可以快速指定html文件中的元素。
- css selector。也是为了简化css而出现的，通过它也能快速指定网页中的特定元素。

以上两个玩意儿的语法并不是很难，阮一峰的博客中都有介绍。
