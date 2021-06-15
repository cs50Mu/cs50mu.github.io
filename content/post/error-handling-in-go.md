+++
title = "Error Handling in Go"
date = "2019-05-09"
slug = "2019/05/09/error-handling-in-go"
Categories = ["Golang"]
+++

From: [Don’t just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

### error变量

```go
var ErrNotFound = errors.New("datastore:notfound")

err := Foo()
if err == ErrNotFound {
    // ...
    }
```

优点：简单

缺点：不能带上下文，只有一个简单的固定字符描述；而且错误要耦合进调用方

### error类型

好处是可以承载更多的上下文信息

缺点还是这个错误必须耦合进调用方

```go
type PathError struct {
Op string
Path string
Err error
}

func (e *PathError) Error() string {
    return ""
}

if err, ok := err.(PathError); ok {

}
```

### opaque error handling / 黑盒错误处理

不对返回的error做任何假设，因此也不能干啥。。  直接返回error

```go
func foo() error {
    x, err := bar()
    if err != nil {
        return err
    }
    
    // use x
```

### assert errors for behavior, not type / 根据行为来处理

上面的方法太简单。。有时候确实是需要区分error种类，来按情况处理的。

```go
type temporary interface {
        Temporary() bool
}
 
// IsTemporary returns true if err is temporary.
func IsTemporary(err error) bool {
        te, ok := err.(temporary)
        return ok && te.Temporary()
}
```

优点：调用方不用依赖定义error的包了

缺点：这种方法依赖 第三方包作者的配合，需要他先让他返回的error满足一定的interface（比如temporary)，包的使用者才能这么用呢

Don’t assert errors for type, assert for behaviour.

For package authors, if your package generates errors of a temporary nature, ensure you return error types that implement the respective interface methods. If you wrap error values on the way out, ensure that your wrappers respect the interface(s) that the underlying error value implemented.

For package users, if you need to inspect an error, use interfaces to assert the behaviour you expect, not the error’s type. Don’t ask package authors for public error types; ask that they make their types conform to common interfaces by supplying Timeout() or Temporary() methods as appropriate.

参考：

- [Inspecting errors](https://dave.cheney.net/2014/12/24/inspecting-errors)


### extra tip

- annotating errors

- only handle errors once
