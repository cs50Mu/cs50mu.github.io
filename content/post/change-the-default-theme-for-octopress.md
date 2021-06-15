+++
title = "A brand new blog"
date = "2013-10-08"
slug = "2013/10/08/change-the-default-theme-for-octopress"
Categories = ["octopress"]
+++

终于忍不住了。

- 添加了disqus评论系统，还是挺简单的，Octopress内建了对disqus的支持，只要去disqus官网注册，然后在Octopress里开启一下就可以用了。具体是在`_config.yml`中设置。

- 换了新主题fabric，好酷啊～～，竟然还有javascript特效！以前想着换主题应该挺麻烦，现在看来挺简单的，就是输几条命令而已：    

		$ cd octopress
		$ git clone git://github.com/panks/fabric.git .themes/fabric
		$ rake install['fabric']
		$ rake generate
- 添加了tag或category功能。其实，这个本来就有的，只是我不知道而已。。在每篇文章开始添加上category参数后它就有标签了～～一篇文章可以添加多个标签哦～

于是，现在的博客是这个样子的。。有种屌丝一秒变高帅富的感觉有木有？！
{% img center /images/newTheme.png %}
一个小tip，有时候可能会有一些帖子不想发布出去的，那要肿么办呢？    

在帖子的元信息里指定`draft = true`
