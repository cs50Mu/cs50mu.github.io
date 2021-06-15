+++
title = "Golang FAQ"
date = "2019-02-01"
slug = "2019/02/01/golang-faq"
Categories = []
+++

这几天看了下Golang FAQ，解了不少困惑，我相信有人会有同样的困惑，挑选了一些有意思的问题翻译了一下。

### Design

> Does Go have a runtime? / Go有运行时吗？

Go确实有一个runtime库(runtime library)。这个runtime库主要实现了垃圾回收、并发管理、堆栈管理等其它Go语言的关键特性。这个所谓的runtime可以类比于c语言的标准库(libc)

但很重要的一点是要理解，Go的这个所谓的runtime并没有带虚拟机（比如，像Java一样）。所以，尽管runtime这个词一般用来指某些语言带的虚拟机实现，但在Golang的语境下，它仅仅指代标准库。


> Why does Go not have generic types? / 为什么没有泛型？

泛型在将来的某个时候可能会被加上。但我们并不着急加它，虽然我们理解有些程序员可能不认同。 

Go的目的是用于服务端程序开发，它的设计的目标在于增强程序的可扩展性、并发能力以及代码的可读性。泛型对于这些设计目标并没有多大帮助，所以为了实现的简单，当时并没有支持泛型。

现在Go语言已经越来越成熟了，可以考虑加上泛型的支持，但是对于泛型我还是想多说几句。

泛型很方便，但是它大大增加了语言的类型系统(type system)和运行时的实现复杂度，我们还没有找到一种能很好平衡两者的设计方案。目前大家先用`interface{}`来顶一顶吧。

> Why does Go not have exceptions? / 为什么没有异常？

我们认为把exception耦合进正常逻辑中（比如其它语言中常见的`try-catch-finally`惯用法）会导致混乱的代码。使用exception的方式也倾向于鼓励程序员把很多普通的错误（比如无法打开文件）当成异常来处理。

对于普通的错误，需要在函数调用后立即处理。

Go也提供了一组内置函数用来处理真正的异常情况（panic和recovery）。

linuxfish注：我个人理解，关键是要区分普通错误和真正的异常。

### Types 

> Why does Go not support overloading of methods and operators? / 为何不支持方法和运算符重载？

在其它语言上的经验告诉我们，搞一堆名字一样，签名不同的方法，只是偶尔有点用处，更多的会带来理解上的混乱。相反，不支持重载大大简化了Go语言的类型系统的设计。

至于运算符重载，在我们看来它可以带来一定的便利性，但并非必须。按照我们的惯例，宁缺毋滥。

> Why doesn't Go have "implements" declarations? / 为何不用`implements`显示声明？

一个Go的类型如果实现了某个interface的方法，那它就隐式的“声明”为这个interface类型了（即可以当成这个interface类型来用了）。

好处是实现了接口和实现的真正解耦，允许在不更改现有代码的前提下定义和使用接口。

> Can I convert a []T to an []interface{}?

不能直接转，因为它们在内存中的表示并不一样，需要这样做：

```go
t := []int{1, 2, 3, 4}
s := make([]interface{}, len(t))
for i, v := range t {
    s[i] = v
}
```

一个更通用但也更慢的函数：

```go
func InterfaceSlice(slice interface{}) []interface{} {
    s := reflect.ValueOf(slice)
    if s.Kind() != reflect.Slice {
        panic("InterfaceSlice() given a non-slice type")
    }

    ret := make([]interface{}, s.Len())

    for i:=0; i<s.Len(); i++ {
        ret[i] = s.Index(i).Interface()
    }

    return ret
}
```

参考：[Type converting slices of interfaces in go](https://stackoverflow.com/questions/12753805/type-converting-slices-of-interfaces-in-go)

> Can I convert []T1 to []T2 if T1 and T2 have the same underlying type?

不可以，只能一个一个转。

```go
type T1 int
type T2 int
var t1 T1
var x = T2(t1) // OK
var st1 []T1
var sx = ([]T2)(st1) // NOT OK
```

> Why is my nil error value not equal to nil?

首先，interface类型的底层实现是一个二元组（一个类型T和一个值V），V就是具体的值，T就是这个值的类型。比如，我们存了一个int 3在接口里，那么用字符简单的表示，这个接口值现在就等于`(T=int, V=3)`。

一个接口值当且仅当V和T都未设置时（T=nil, V未设置）才为nil。特别地，一个值为nil的接口的底层T永远都是nil。那么，如果把一个值为nil类型为`*int`存入一个接口中，此时这个接口值的底层表示为`(T=*int, V=nil)`，按照定义，这个接口值不等于nil。

但这跟我们的日常认知相悖，明明是个空指针嘛！这种情况在从函数中返回error时更容易使人困惑：

```go
func returnsError() error {
	var p *MyError = nil
	if bad() {
		p = ErrBad
	}
	return p // Will always return a non-nil error.
}
```
上面的写法的错误之处在于，不管有没有错误发生，函数返回的error永远都是non-nil的，所以当无错误发生时，应当显式地返回nil：

```go
func returnsError() error {
	if bad() {
		return ErrBad
	}
	return nil
}
```

### Values

> Why does Go not provide implicit numeric conversions? / 为何不提供隐式数字类型转换？

c语言中的自动数值类型转换带来的那点便利性跟它引起的困惑相比根本不值得一提。表达式的结果什么时候是无符号的？它的值有多大？是不是溢出了？结果是不是可移植的（执行结果跟程序所在的机器无关）？考虑到移植性的问题，我们要求在进行类型转换时必须进行显式的转换，尽管有些麻烦。

一个相关的细节是，就算`int`实质上是一个`int64`（当在64位机器上时），`int`和`int64`也是两个不同的类型。如果你真的关心一个应该用多少位来表示一个整数，我们鼓励你显式地声明它（用int32或者int64，而不是int）。

> How do constants work in Go?

尽管Go要求不同的数值类型必须进行显式的转换，它对常数值的要求比较灵活。因为常数值默认是无类型的（untyped），所以不必这么写：

```go
sqrt2 := math.Sqrt(float64(2))
```

你可以直接写：

```go
sqrt2 := math.Sqrt(2)
```

想了解Go中的常数值的具体实现的可以参考[这篇Blog](https://blog.golang.org/constants)

> Should I define methods on values or pointers? / 方法定义在值上还是指针上？

```go
func (s *MyStruct) pointerMethod() { } // method on pointer
func (s MyStruct)  valueMethod()   { } // method on value
```

接收者是要定义成值还是指针本质上跟函数的参数是定义成值还是指针是一个问题。

我们在回答这个问题时通常有以下考虑：

- 方法是否需要修改这个接收者？如果需要修改，那没有什么纠结的，必须用指针（slice 和 map 本身就是引用类型，需要单独讨论）。
- 效率上的考虑。如果接收者是一个大结构体，那么很显然用指针更合适
- 一致性。如果有些方法必须用指针做为接收者，那么为了使用上的一致，其它的方法也应该用指针
- 除非必须，一般情况下使用值就可以了

> What's the difference between new and make? / new 和 make 的区别？

简而言之，new 用来分配内存，而 make 用来初始化slice，map 和 channel。

具体解释可参考 Effective Go 中的[相关章节](https://golang.org/doc/effective_go.html#allocation_new)

### Concurrency

> Why doesn't my program run faster with more CPUs?

一个程序是否会因为有更多的CPU资源而运行更快取决于这个程序所要解决的问题。Go语言提供了基本的并发原语，比如 goroutine 和 channel，但只有当要解决的问题存在并行执行的可能性时才会发挥它们的作用。本质上是串行的问题，仅仅通过增加更多的CPU资源无法获得更高的执行效率。 

### Functions and Methods

> What happens with closures running as goroutines? / 闭包的坑

在并发中使用闭包时，可能会出现一些令人困惑的现象，考虑以下代码： 

```go
func main() {
    done := make(chan bool)

    values := []string{"a", "b", "c"}
    for _, v := range values {
        go func() {
            fmt.Println(v)
            done <- true
        }()
    }

    // wait for all goroutines to complete before exiting
    for _ = range values {
        <-done
    }
}
```
我们可能错误的认为输出结果应该是`a,b,c`，但实际上你看到的结果可能是`c,c,c`。这是因为所有的闭包都共享了一个变量v，而v在循环中被不断的更新，当闭包中的程序真正在另一个goroutine中运行时，v的值已经变了。使用`go vet`可以在运行程序前提前检测这种情况。

针对这个“坑”，有两个解决方案：

要么，把v显式传入闭包中

```go
    for _, v := range values {
        go func(u string) {
            fmt.Println(u)
            done <- true
        }(v)
    }
```

或者这样写：

```
    for _, v := range values {
        v := v // create a new 'v'.
        go func() {
            fmt.Println(v)
            done <- true
        }()
    }
```

### Control flow 

> Why does Go not have the ?: operator? / 为何没有`?:`操作符？

可以使用以下代码实现类似的效果：
```go
if expr {
    n = trueVal
} else {
    n = falseVal
}
```
Go中不存在这个三元操作符的原因是Go语言的设计者认为这个三元操作符经常被用来写出晦涩难懂的表达式。`if-else`的模式，虽然写起来更长一些，但毫无疑问表达的更清楚。一门语言只需要一种条件控制结构就够了。

### Packages and Testing

> Where is my favorite helper function for testing?

使用Go标准库中的测试包很容易写单元测试，但这个测试包缺乏一些其它测试框架中常用的功能，比如断言。之前的章节中已经解释了为啥Go不支持断言，这个解释同样适用于写测试。在进行单元测试时，正确的错误处理方式是在某个测试出错后，允许其它的测试继续进行，这样进行测试的人可以从全局看到是什么地方出了问题。举个例子，现在正在测试 isPrime 这个函数，那么与其只报告了isPrime对于2这个输入的结果是错的，然后就停止测试了，直接报告 isPrime 对于输入（2，3，5，7）的结果都是错的明显更有用处。运行测试代码的人可能对被测试的代码并不熟悉，那么在写测试代码时，好好写一个错误提示信息就很有必要了，能帮助后面测试出现错误定位问题时省不少力气。

还有一点是测试框架都喜欢定义自己的一套用来表示条件、控制语句和打印等固定模式的DSL，但问题是Go本身就能实现这些功能，为什么还要重复造轮子呢？可以少学一门语言，何乐而不为呢？

如果你觉得用Go的测试包来写测试太啰嗦了，会有很多重复的代码，那么你可以考虑用驱动表的方式来写测试，这样的话，重复的代码会大大减少。Go标准库中有大量的测试代码可供你参考。

### Implementation

> Why is my trivial program such a large binary? / 为啥那么点代码就编译出这么大一个二进制文件？

gc 工具链中的链接器(linker)默认创建静态链接的二进制文件。 因此所有的 Go 二进制文件包含了Go的运行时，同时还有执行动态类型检测、反射甚至崩溃时的堆栈信息的运行时

一个简单的 hello world 版 c 程序，用 gcc 在 linux 上静态编译后，体积大约是750KB。同样的 Go 程序，体积大概有几M，但功能也更强大，因为它有 run time 支持，还有动态类型检查和 debug 信息。

Go程序在用 gc 编译时可以带上`-ldflags=-w`参数来关闭 debug 信息的生成，这不会影响程序的正常功能，但会很大程度地减少程序的体积。

> Can I stop these complaints about my unused variable/import? / 能不能别老提示我未用的变量和引入库？

代码中存在未使用的变量可能意味着潜在的 bug，而未使用的引用库只会增大编译时间，而且随着代码量的增长，这个问题会越来越严重。基于以上原因 Go 拒绝编译存在未使用变量和未使用的引用库的代码，短期来看，这个决定带来了很多麻烦，但从长远来看，它利于提升程序的编译速度和代码的整洁性。

当然了，在写代码的时候，创建一堆临时变量是常见的，每次都必须要先处理一下这些用不上的临时变量才能编译确实是挺烦的。

有人提议加一个编译器的参数可以关闭这个傻X的检查，或者至少编译能通过，比如只是发个警告信息提示用户。但我们没有加这个参数，一是因为我们认为编译器的参数不应该影响语言的语义；二是 Go 编译器没有警告，它只报告错误。

Go编译器没有警告，原因有二：一是，如果一个问题需要报警提示，那它就应该被改正（相反，如果它不值得被修正，那也根本无需发警告提示）；第二，有时候太多警告反而会掩盖了真实的问题。

使用空白标识符`_`可以一定程度的缓解这个问题。

```go
import "unused"

// This declaration marks the import as used by referencing an
// item from the package.
var _ = unused.Item  // TODO: Delete before committing!

func main() {
    debugData := debug.Profile()
    _ = debugData // Used only during debugging.
    ....
}
```

当然了，现在大部分Go程序员都知道有这个神器，goimports，它能分析你的源文件，自动清理掉未用到的变量，在你保存文件的时候自动重写源文件，大部分的编辑器都支持它。

### Changes from C

> Why is there no pointer arithmetic? / 为啥不支持指针运算？

安全。不支持指针运算能避免很多诡异的问题。编译器和硬件技术发展到现在，用数组索引和用指针来遍历一个数组，在效率上已经没有什么区别。另外，不支持指针运算使得垃圾回收也更好实现了。 

> Why are ++ and -- statements and not expressions? And why postfix, not prefix?

不支持指针运算的话，那么对`++`和`--`的需求就没那么大了。通过把`++`和`--`限制为 statement，而不是表达式(expression)，我们可以避免类似以下写法的出现：

```go
f(i++)
p[i] = q[++i])
```
