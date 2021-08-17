---
title: "记住EF BB BF"
date: 2021-08-09T11:57:13+08:00
draft: false
---

遇到一个问题，记录一下。

收到反馈说，用户上传的文件无法解析。查看日志后发现，解析第一行时就报错了，但是用普通文本编辑器查看发现文件看起来并无异常，是一个只有一列的csv文件。

通过`xxd`查看16进制格式：

```
xxd wrong_format.csv | less
```

可以看到，开头三个字节为：`ef bb bf`，感觉不正常搜索之：

- [JAVA- Integer.parseInt( str ) gives NumberFormatException, input is a str representing an integer)](https://stackoverflow.com/questions/60770180/java-integer-parseint-str-gives-numberformatexception-input-is-a-str-repres)
- [字节顺序标记](https://zh.wikipedia.org/wiki/%E4%BD%8D%E5%85%83%E7%B5%84%E9%A0%86%E5%BA%8F%E8%A8%98%E8%99%9F)

从这个 stackOverflow的 [回答](https://stackoverflow.com/a/2223926/1543462)知道，BOM 是用来标识字节序(Endianness)的，而且utf-8 是没必要使用BOM的：

> Normally, the BOM is used to signal the endianness of an encoding, but since endianness is irrelevant to UTF-8, the BOM is unnecessary. UTF-8 has the same byte order regardless of platform endianness, so a byte order mark isn't needed.

至于为啥utf-8没必要使用BOM，[据说](https://www.zhihu.com/question/20167122/answer/103632765)是由于它的编码特性，是字节序无关的，具体可能需要新开一篇帖子来学习下utf-8的原理了。

知道了问题所在，那解法就有了，在解析前先把开头三个BOM字节trim掉：

```go
bytes.Trim(fileBytes, "\xef\xbb\xbf")
```

ps, 那个惹祸的文件是由Mac OS X 系统下的 Excel 生成的。
