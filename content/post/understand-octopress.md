+++
title = "understand octopress"
date = "2013-10-13"
slug = "2013/10/13/understand-octopress"
Categories = ["octopress", "jekyll"]
+++

好奇害死猫。

一开始是想着给现在的博客主题添加一个sidebar，再后来就想着搞清楚Octopress的原理，仔细一研究，原来里面用到了好多好玩的新技术啊（对我而言的新技术，我对网页的认识还停留在html css php等），下面就盘点下我从Octopress/Jekyll里了解到的“新”技术。

- Octopress是Jekyll的一个wrapper程序，Jekyll transform your plain text into static websites and blogs. Jekyll是一个静态博客编译器，负责把你写的代码编译成一个完整的博客网站。这样生成的网站属于静态网站，它是想对于wordpress这样的带数据库的动态网站而言的。很显然，静态网站的一大优点就是响应速度比动态网站相对要快，因为所有页面一旦生成就不再改变了嘛，不需要跟跟数据库交互；缺点就是每次更新都需要重新编译。
- Sass && Compass。这年头写CSS是用Sass和Compass了，我已经OUT了太久了。CSS不是一种编程语言，没有变量，没有条件语句，只是一行行的描述，写起来很费事，于是程序员童鞋忍不住了，Sass诞生了！Sass是一种CSS的开发工具，提供了很多便利的写法，可大大节省设计者的时间，使得CSS的开发变得简单。Compass是Sass的工具库(toolkit)，Sass本身只是一个编译器，Compass在它的基础上，封装了一系列有用的模板，使得CSS的开发更加便利！它们之间的关系，类似Javascript和jQuery、python和Django的关系。    
**说到这里我可以自我鄙视下嘛？人家其实连CSS都不会。。** 唉，各种标签、类一会儿就被转晕了。。 

- YAML。递归的名字是YAML Ain't Markup Language，官方给出的定义：YAML is a human friendly data serialization
  standard for all programming languages.也就是说它是一种数据表示的标准，类似于xml，实际上是将来可能会取代xml的数据标准，YAML试图用一种比XML更敏捷的方式，来完成XML所完成的任务。我的感觉是，YAML确实比xml和jason更容易阅读，看起来结构比较清楚，容易理解一点。
- Liquid。Ruby library for rendering safe templates which cannot affect the security of the server they are rendered on.所谓的templating language，模板语言。可以嵌入在web页面中来形成模板页面，后期根据不同的页面参数会编译成不同的页面，甚至还支持使用if for等类编程语句，非常灵活。Jekyll在它的模板文件中大量使用了liquid语言。
- FontAwesome。The iconic font designed for Bootstrap.Font Awesome gives you scalable vector icons that can instantly be customized — size, color, drop shadow, and anything that can be done with the power of CSS.Font Awesome是一种出色的免费iconic font，别让字面意思误导了，这货其实不是字体，是网站上用的一些小图标，可能你都没有注意到，但也确实会给网站增色不少。看到一篇介绍iconic font的文章，[Iconic Fonts 簡介](http://lepture.com/work/iconic-fonts)，讲的非常不错！我目前用的主题fabric的header里用的字体就是来自google font的Slackey。
- Animate.css。animate.css is a bunch of cool, fun, and cross-browser animations for you to use in your projects. Great for emphasis, home pages, sliders, and general just-add-water-awesomeness.用纯CSS实现的动画效果集合，非常酷！   
How to use it?    
To use animate.css in your website,simply drop the stylesheet into your document's `<head>`, and add the class `animated` to an element, along with any of the animation names.That's it! You've got a CSS animated element.Super!
关于怎么用CSS3实现动画效果，看[这里](http://www.w3schools.com/css3/css3_animations.asp)    
- google web font。顾名思义，是Google提供的网上免费字体。使用方法之一是，首先在`<header>`里添加对网上字体的引用，比如`<link rel="stylesheet" type="text/css" href="http://fonts.googleapis.com/css?family=Tangerine">`，然后就可以在CSS里指定了。
- fancybox。FancyBox is a tool for displaying images, html content and multi-media in a Mac-style "lightbox" that floats overtop of web page. It was built using the jQuery library.

- 两个html5新标签。通过这番折腾知道了两个新标签`<header>`和`<article>`
- Octopress/Jekyll的layouts

其实很容易理解的东西，不知为什么搞了很久才弄明白，layouts文件夹里放的是博客的模板文件，顾名思义就是一个框架，相当于人的骨骼，把具体的博客内容填进去后就成了最终的博客。通常为了不同的用途，会有多个模板文件，为了简化代码，这几个模板文件之间通常有继承关系。比如，default.html里放的是所有页面都会有的东西，比如页面开始的引用、head、footer等，post.html在default.html的基础上再做进一步的修改，思想有点类似类的继承，不过在这里只能在原有的基础上继续添加，不能覆盖原来的“方法”。    

另外，在某个模板中还可以include进别的模板，比如在default.html中include进foot.html，这样做有利于管理代码，否则所有的代码都在一个文件里，看着混乱，也不容易管理。所有include的模板都在`source/_include`文件夹里。
