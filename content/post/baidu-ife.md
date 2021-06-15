+++
title = "baidu ife"
date = "2016-03-06"
slug = "2016/03/06/baidu-ife"
Categories = []
+++

###盒模型及定位
- 用两种方法来实现一个背景色为红色、宽度为960px的`<DIV>`在浏览器中居中

第一种方法：`margin:0px auto;`

第二种方法：使用绝对定位，设置左边距离body`50%`，left-margin为div宽度的一半。
```css center
#red {
    width:960px;
    height:100px;
    position:absolute;
    left:50%;
    margin-left: -480px;
/*    margin:0px auto; */
    background-color:red;
}
```

补充：     
像下面这样写css代码，使用 max-width 替代 width 可以使浏览器更好地处理小窗口的情况。这点在移动设备上显得尤为重要。      
有效的避免了如下问题：     
当浏览器窗口比元素的宽度还要窄时，浏览器会显示一个水平滚动条来容纳页面。
```css max-width
#main {
    max-width: 600px;
        margin: 0 auto; 
}
```
参考：[学习css布局](http://zh.learnlayout.com/)

- 用两种不同的方法来实现一个两列布局，其中左侧部分宽度固定、右侧部分宽度随浏览器宽度的变化而自适应变化

第一种方法： 使用绝对定位，左边的块固定好，右边的块用`margin-left`
```html absolute
<div class="container">
    <div class="nav-relative">relative method</div>
    <div class="section">的圆角矩形是复杂图案，无法直接用border-radius，请在不使用border-radius的情况下实现一个可复用的高度和宽度都自适应的圆角矩形请在不使用border-radius的情况下实现一个可复用的高度和宽>度</div>
    <div class=section-below>的圆角矩形是复杂图案，无法直接用border-radius，请在不使用border-radius的情况下实现一个可复用的高度和宽度都自适应的圆角矩形请在不使用border-radius的情况下实现一个可复用的高度>和宽度 都自适应的圆角矩形请在不使用border-radius的情况下实现一个可复用的高度和宽度</div>
</div>
```
```css absolute
.container {
    position:relative;
}
.nav-relative {
    position:absolute;
    left:0px;
    width:200px;
    height:100px;
    background-color:blue;
}
.section {
    margin-left:200px;
    height:100px;
    background-color:green;
}
.section-below {
    background-color:khaki;
    height:100px;
}
```
第二种方法：使用float，左边的使用向左浮动，右边的用`margin-left`
```html float
<div class="nav-float">haha hahah  float method</div>
<div class="section-float">圆角矩形是复杂图案，无法直接用border-radius，请在不使用border-radius的情况下实现一个可复用的高度和宽度都自适应的圆角矩形请在不使用border-radius的情况下实现一个可复用</div>
<section>的圆角矩形是复杂图案，无法直接用border-radius，请在不使用border-radius的情况下实现一个可复用的高度和宽度都自适应的圆角矩形请在不使用border-radius的情况下实现一个可复用的高度和宽度 都自适应的圆角矩形请在不使用border-radius的情况下实现一个可复用的高度和宽度</section>
```
```css float
.nav-float {
    float:left;
    width:200px;
    background-color:red;
}

.section-float {
    margin-left:200px;
}

.div3 {
    height:200px;
    background-color:grey;
}
```
第三种方法：用BFC（Block Formatting Context）来实现

```html bfc
<div class="left">left</div>
<div class="right">right</div>
<div class="main">
	flying-Swing-BFC.htmlflying-Swing-BFC.htmlflying-Swing-BFC.htmlflying-Swing-BFC.htmlflying-Swing-BFC.htmlflying-Swing-BFC.htmlflying-Swing-BFC.htmlflying-Swing-BFC.htmlflying-Swing-BFC.htmlflying-Swing-BFC.htmlflying-Swing-BFC.html
</div>
```

```css bfc
.left{
    width: 100px;
    background-color: red;
    float: left;
}
.right{
    width: 200px;
    background-color: blue;
    float: right;
}
.main{
    background-color: #eee;
    overflow: hidden;
}
```

A Block is not a BFC. 

 A block formatting context is a box that satisfies at least one of the following:

- the value of "float" is not "none",
- the used value of "overflow" is not "visible",
- the value of "display" is "table-cell", "table-caption", or "inline-block",
- the value of "position" is neither "static" nor "relative". 
参考：     
[关于Block Formatting Context](http://www.cnblogs.com/pigtail/archive/2013/01/23/2871627.html)
[CSS 101: Block Formatting Contexts](http://yuiblog.com/blog/2010/05/19/css-101-block-formatting-contexts/)

第四种方法：双飞翼布局
主要用到了float、负margin
```html double-wing
!DOCTYPE html>
<html lang="en">
    <head>
          <meta charset="utf-8" />
            <title>A tiny document</title>
            <link rel="stylesheet" href="double_wing.css">
    </head>
<body>
<div class="main">
        <div class="inner"> main </div>
</div>
<div class="aside"> aside </div>
<div class="ad"> ad </div>
</body>
</html>
```

```css double-wing
.main{
        width: 100%;
        float: left;
        background-color:green;
    }

.main > .inner{
        margin-left: 200px;
        margin-right: 150px;
        background: deeppink;
    }

.aside{
        width: 200px;
        float: left;
        margin-left: -100%;  // 把aside拉回开头
        background: pink;
    }


.ad{
        width: 150px;
        float: left;
        margin-left: -150px;   // 把ad往回拉一点,应该等于ad这一栏的宽度
        background: red;
}
```
### 实现一个浮动布局，红色容器中每一行的蓝色容器数量随着浏览器宽度的变化而变化

这个比较简单，直接全部float就可以了
```html floating box
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
<div class=box>I'm floating!</div>
```
```css floating box
.box {
    float:left;
    width:200px;
    height:100px;
    margin:1em;
    background-color:blue;
}
```
### 在做综合任务时遇到的问题
- css类名该叫什么？

这确实是一个让人头疼的问题，大概看了下规范，要求用语义化命名，比如sidebar，而不是left right等仅仅描述这个css在做什么

- 各个div之间如何定位的问题

我目前是使用absolute定位

- `bottom: 0`不起作用？

明明设置了一个div的`bottom: 0`属性了，可是里面的文字就是看着没有贴着下边界，我明明已经把所有元素的padding和margin都置为0了啊！昨晚查到好晚，偶然发现这不是`bottom: 0`配置不起作用或者padding、margin没有置为0，应该是跟字体有关系，我原来一直用的是英文，昨晚突然灵光一现换成了中文字体，文字立马就贴着下边界了！这个具体原因是什么还得继续深究。

- inline list 如何实现？

正常情况下我们看到的list都是一行一个元素这样的，要想实现所有元素都在一行上的效果，通过做这个学到两种方法：

> 一种方法是用float。所有的li元素设置`float: left`

> 另一种方法是用`inline-block`。`li`元素默认是块级元素，将它的display属性改写为`inline-block`后，顾名思义，它就是一个inlined block了，可以放在一行上。

- 高度塌陷问题（height collapse）

这个问题也是会让新手比较困惑的地方。明明我设置了这个div的某些属性（比如背景色），为啥不生效呢？     
原因我理解是这样：首先，浏览器对于高度为0的元素是不会渲染的，不管你给这个元素设置了多少属性（不行你可以试试）；其次对于使用float、absolute定位方式的元素，浏览器在渲染的时候会把他们从正常流（normal flow）中剔除，就像他们不存在一样，这样就会有一个问题，对于只包含floated或者absoluted的元素，父元素在浏览器看来就是一个空元素，所以它就不会把它渲染出来，它的属性也不会生效，但在人来看，这TMD明明有东西在里面啊，这就会引起困惑。其实这些规则在MDN或者W3的文档里都有说明的，只是太繁琐了，一般人可能都没有精力去翻吧。

前面说道的从正常流里剔除出来的元素是真的不渲染了吗？并不是。按我的理解，这些被剔除的元素会在父元素这个“独立命名空间”被渲染，不会受到其它元素的影响。

- 媒体查询

现代浏览器的高端功能，能根据屏幕尺寸的大小来动态地改变css
``` css media query
 .icon-github {
     position:absolute;
     right:10px;
     bottom:0px;
 }

@media screen and (max-width:980px) {   // 当这个条件为真的时候执行下面的css，否则就按上面正常的css
    .icon-github {
        display:none;
    }
}
```
上面css的意思是在宽度小于980px时不显示icon，当宽度大于980px时才显示

- relative和absolute的区别

一直不明白，刚看了下MDN的文档，加上代码展示，明白了。

relative是在原来的文档流布局好位置的基础上再做一次定位。比如一开始你该float就float，类似于坐座位，等大家都坐好了，你说，OK，我再做一个小小的调整~~ 而且！最关键的是我原来的位置也是不允许别人坐的！就那么空着，就是这么霸道~ 这就是relative啦！

The element's position is fixed relative to a parent element. Only a parent that is itself positioned with relative, fixed or absolute will do. You can make any parent element suitable by specifying position: relative; for it without specifying any shift.  绝对定位是指子元素相对于父元素偏移固定的位移。只有当父元素是relative、fixed或者fixed的时候才能生效。

参考： [https://developer.mozilla.org/en-US/docs/Web/CSS/position](https://developer.mozilla.org/en-US/docs/Web/CSS/position)

