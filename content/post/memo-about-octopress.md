+++
title = "Octopress搭建备忘"
date = "2013-06-12"
slug = "2013/06/12/memo-about-octopress"
Categories = ["Octopress", "Markdown"]
+++

## 缘由
突然想玩玩Octopress，于是开始搜索，发现在Archlinux上使用Octopress会有两个问题   

* Ruby版本问题。Arch上的Ruby版本已经更新到2.0版本了，而Octopress用的是1.9版本的
* Octopress使用Pygments来高亮代码，而Pygments不支持Arch默认的Python 3


重新以“octopress archlinux”为参数搜索，找到一篇很好的针对Arch的教程，不过它是针对zsh写的，所以bash用户需要略做修改。   

## 安装过程
基本比较顺利，只记录下遇到的问题   

* rbenv exec gem install bundler 这一步死活进行不下去！提示找不到bundler，搜索后说是可能是`http://rubygems.org/`无法访问了，在浏览器看了下访问正常啊！但就是不行。。最终解决办法，换源（尼玛还是GFW的事）:      

	    gem sources --remove http://rubygems.ogr/
	    gem sources -a http://ruby.taobao.org/   # 淘宝的RubyGems镜像
	    gem sources -l
	    *** CURRENT SOURCES ***
	  
	    http://ruby.taobao.org

* 安装python-virtualenvwrapper的时候，有一个包一直装不上，提示不存在，纠结了很久，最后才恍然大悟，更新系统源后解决。


* 新建帖子时不能用中文，否则标题会出乱码,可以先在`rake new_post["new post name"]`中使用英文名字，然后在具体编辑帖子的时候，修改title选项使用中文。

## 网站结构
根据我粗浅的理解，安装完成后博客的结构在github上是这样的：博客代码库名称是cs50mu.github.com，下面有两个分支，master分支是根据源码生成的blog静态页面；source分支是用于生成blog的源码，所以每次提交静态页面后还需要提交更新的blog源码，这个不要忘记了。



## 发布流程

首先，需要进入blog编辑的虚拟环境（也就是为了兼容ruby和python而搭的虚拟环境）

	workon blog_env

退出这个虚拟环境需要

	deactivate

然后就可以开始啦！

1. 用`rake new_post["new post name"]`命令新建blog文章。
2. 在`souce/_post`文件夹用vim编辑新建的blog文件。
3. `rake preview`在本机预览。
4. `rake generate`和`rake deploy`提交到github。
5. 最后，不要忘记将更新后的源码提交到source分支。   

		git add .
		git commit -m 'some comment'
		git push origin source

## To-Do List  
- 换一套博客模板,目前用的是系统默认的，我发现太多懒人了。。
- 添加评论系统
- 去掉twitter等在天朝不和谐的东东

## 参考
- [Octopress on Arch Linux](http://www.wongdev.com/blog/2013/01/16/octopress-on-archlinux/)
- [Octopress 在 Arch Linux 下的一些問題](http://shadow.ma/blog/2013/04/07/octopress-on-arch-linux/)
- [使用Octopress + Github管理blog](http://ishalou.com/blog/2012/10/15/how-to-use-octopress/),这篇很不错，很适合我这样的菜鸟，看了它后基本理解了整个搭建原理。

## 不错的Markdown示例
- [Markdown语法示例](http://equation85.github.io/blog/markdown-examples/)，非常不错！上面代码，下面展示效果，非常直观。
- [Markdown 语法说明](http://wowubuntu.com/markdown/#list)

### update:
终于找到Markdown中插入代码不起作用的原因了！原来在列表中嵌套的代码需要缩进2个tab才起作用。  
> To put a code block within a list item, the code block needs to be indented twice — 8 spaces or two tabs.
[Ref](http://daringfireball.net/projects/markdown/syntax#list)

