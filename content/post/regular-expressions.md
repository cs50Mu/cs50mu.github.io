+++
title = "学习正则表达式"
date = "2013-06-16"
slug = "2013/06/16/regular-expressions"
Categories = ["regular expression"]
+++

正则表达式真的很强大啊！

但我在用的过程中写的表达式总是不能达到期望的效果，今天又认真看了下，对它又有了一些新的理解。

- 贪婪模式Vs非贪婪模式。我写的表达式没效果的原因十有八九是因为这个，缺了个?。默认情况下，正则表达式的匹配都是贪婪的，也就是匹配尽可能多的字符，比如妄图用`<img.*>`来匹配所有的图片标签是不可行的，因为在贪婪模式下这会一直匹配到文末的`>`处。正确的方法是使用非贪婪模式，即最小匹配，`<img.*?>`，保险起见，最好写成`<img[^>]*?>`，表示在`<img`标签后跟任意个非`>`的字符然后再跟一个`>`。
- 组与捕获。在实际应用中，通常需要匹配两个或更多方案中的任意一种正则表达式，通常还需要捕获匹配的内容以备进一步处理，所有这些都可以通过`()`来实现，在多种方案中选取一个的情况则使用交替字符`|`。比如，`(air(craft|plane)|jet)`可表示aircraft airplane jet，这里`()`有两个作用：对表达式进行组合，捕获匹配某个表达式的文本。如果匹配的是前半部分的话（aircraft或airplane），会有两次捕获：aircraft或airplane作为第一次捕获，craft或plane作为第二次捕获；如果匹配的是后半部分的话（jet），就只有一次捕获。可以通过在左括号后面跟随一个`?:`来关闭捕获，比如，`(air(?:craft|plane)jet)`，这样的话，如果前半部分匹配，也只会有一次捕获了，即aircraft和airplane，通过关闭一些不需要的捕获，能方便后面对它的引用。   
对前面的捕获，可以通过反向引用对其进行引用，有两种方法可以实现引用：一种是使用`\i`，这里的i指的是前面捕获号。每次捕获都会有一个编号，从1开始，从左到右，每次遇到一个新的左圆括号时进行一次新的捕获编号。比如，为匹配重复单词，使用`(\w+)\s+\1`，表示匹配开始一个单词，之后至少一个空白字符，再之后是与捕获的单词相同的单词。方法二，在较长的或复杂的正则表达式中，为维护方便，通常对捕获进行命名而不是使用编号。为对捕获进行命名，须在左圆括号后跟随`?P<name>`，后面进行反向引用的语法是`(?P=name)`，比如，`(?P<word>\w+)\s+(?P=word)`
- 反义。有时候需要查找不属于某个字符类的字符，比如除了数字其他字符都可以的情况，这时需要用到反义。比如，`\S`匹配任意不是空白符的字符，`\D`匹配任意非数字的字符；还有一种描述反义的方式是使用`[]`，比如，`[^x]`表示除了x外的任意字符。
- 断言。断言不会匹配任何文本，而是对断言所在的文本施加某些规定和约束，如果不使用断言的话可能会匹配到一些我们不需要的文本，使用断言可以提高正则表达式的清晰度。比如用正则表达式`jet`来匹配文本"the jet and jetski are noisy"会出现两次匹配，即jet和jetski中的jet，这可能不是我们想要的结果，但是使用正则表达式`\bjet\b`将只匹配一次，即单词jet。

UPDATE:

今天(2014.03.08)改了下自动发天气预报的脚本，终于知道自己之前用正则各种不顺手的原因了！！尼玛不是人家电脑不听话啊！是你自己没有搞懂好不好！！之前只知道用`.+?`来代表任意多个字符，可你仔细看看`.`是能代表任意字符吗？！人家不是说了**换行符除外**嘛？！然后加上re.S的编译标签后就可以了有木有！！还有就是使用多个括号捕获时，findall函数是以元组的形式返回捕获结果的。