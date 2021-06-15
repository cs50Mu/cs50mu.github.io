+++
title = "Golang 中的文字编码问题"
date = "2019-05-19"
slug = "2019/05/19/a-encoding-problem-in-golang"
Categories = ["Golang"]
+++

### 过程描述

记录一个文字编码的问题

在请求支付宝支付的接口时，发现返回的Response如果有中文的话，print出来后会有乱码。第一反应肯定是编码有问题，那就先转一下编码吧。

一开始的代码大概如下：

```go
    resp, err := client.Post(aliPayBillURL, "application/x-www-form-urlencoded", bytes.NewBufferString(output))
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        panic(err)
    }
    //...
    err = json.Unmarshal(body, &queryResp)
    if err != nil {
        panic(err)
    }
    data := queryResp.Data
    // data.SubMsg里含有中文，会乱码
    if data.Code != "10000" {
        fmt.Printf("download failed: msg: %s, sub_code: %s, sub_msg: %s\n", data.Msg, data.SubCode, data.SubMsg)
        panic("download failed")
    } 
```

改为了：

```go
    resp, err := client.Post(aliPayBillURL, "application/x-www-form-urlencoded", bytes.NewBufferString(output))
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        panic(err)
    }
    //...
    err = json.Unmarshal(body, &queryResp)
    if err != nil {
        panic(err)
    }
    data := queryResp.Data
    converted, err := gbkToUtf8([]byte(data.SubMsg))
    if err != nil {
        panic(err)
    }
    if data.Code != "10000" {
        fmt.Printf("download failed: msg: %s, sub_code: %s, sub_msg: %s\n", data.Msg, data.SubCode, converted)
        panic("download failed")
    }
```

结果是依旧乱码。。难道不是编码的问题？

改`%s`为`%x`看看具体字节：

```go
    if data.Code != "10000" {
        fmt.Printf("download failed: msg: %s, sub_code: %s, sub_msg: %x\n", data.Msg, data.SubCode, converted)
        panic("download failed")
    }
```

然后看到了这个`efbfbdefbfbdd0a7efbfbdefbfbd 4170704944 efbfbdefbfbdefbfbd`，之所以分成了三段贴出来，是因为我发现中间那段对应了部分正常显示的字符“Appid”，然后尝试手动解码一下其它两段，不管是GBK还是utf8都没有找到对应的字符。

然后发现第一段和最后一段的模式有点类似啊，都是`efbfbd`啥的，有点奇怪。尝试直接搜索`efbfbd`，定位到一篇[文章](https://liudanking.com/golang/utf-8_replacement_character/)，里面主要讲到两点：

- 这个序列就是utf8编码的，但它比较特殊，它是专门用来替换那些在解码时不认识的码点的，显示出来就是「�」
- 某些语言，比如 Goang，会自动进行这个转换，不会报错


借用一下原文的例子：

```go
now := time.Now().Unix() // 一个无效的码点值
str := string(now) // golang是utf-8编码，会对无效码点进行替换
fmt.Printf("%X", []byte(str)) // EFBFBD，即字符「�」
```

这一下坚定了认为是编码的问题，那就是转码出问题了，再一看，我是在json decode之后再转码的，json decode的时候可能已经把数据按utf8解码了，而数据如果不是utf8编码的话，那就都被替换成`efbfbd`了，改为在json decode之前就转码试试：

```go
    resp, err := client.Post(aliPayBillURL, "application/x-www-form-urlencoded", bytes.NewBufferString(output))
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        panic(err)
    }
    converted, err := gbkToUtf8([]byte(body))
    if err != nil {
        panic(err)
    }
    fmt.Println(string(converted))
    err = json.Unmarshal(converted, &queryResp)
    if err != nil {
        panic(err)
    }
    data := queryResp.Data
    if data.Code != "10000" {
        fmt.Printf("download failed: msg: %s, sub_code: %s, sub_msg: %s\n", data.Msg, data.SubCode, data.SubMsg)
        panic("download failed")
    }
```

这次没问题了。

### 反思 && 教训

- 获取到数据后不要做任何操作，先做转码
- 记住`efbfbd `，它是一个信号，说明源数据一定不是utf8编码的

最后，记录一下 Golang 中如何从 GBK 转为 utf8

```go
import (
golang.org/x/text/encoding/simplifiedchinese
golang.org/x/text/transform
)

func gbkToUtf8(s []byte) ([]byte, error) {
    reader := transform.NewReader(bytes.NewReader(s), simplifiedchinese.GBK.NewDecoder())
    d, err := ioutil.ReadAll(reader)
    if err != nil {
        return nil, err
    }
    return d, nil
}
```

### 参考

- [你应该记住的一个UTF-8字符「EF BF BD」](https://liudanking.com/golang/utf-8_replacement_character/)
- [Golang 中的 UTF-8 与 GBK 编码转换](http://mengqi.info/html/2015/201507071345-using-golang-to-convert-text-between-gbk-and-utf-8.html)
