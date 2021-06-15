+++
title = "error code vs exception"
date = "2021-01-26"
slug = "2021/01/26/error-code-vs-exception"
Categories = []
+++

error-code OR exception, it's a question.

然而答案是，you should use both，难点在于确定在什么场景下用什么

### 适合用 exception 的情况

- 发生的错误无法处理时

In some situations **you can't do anything with the error code**, and you just need to handle it in an upper level in the call stack, usually just log the error, display something to the user or close the program. In these cases, error codes would require you to bubble up the error codes manually level by level which is obviously much easier to do with exceptions. The point is that this is for unexpected and unhandleable situations. 与其将错误一层一层的 bubble up，不如将它一次性抛到最顶层

- In some situations you have a sequence of commands, and if any of them fail you should cleanup/dispose resources. 当有一堆调用要执行，而且每一个调用都有可能报错时，很明显用 exception 更方便，而不是每次调用后都要 handle 一次可能的error。
- In constructors you can only throw exceptions. 在初始化时若出错一律直接抛 exception，遵循速错原则

### 适合用 error-code 的情况

一般来说，除了上面适合使用 exception 的情况，其它任何时候都应该使用 error-code。

in all other situations where you're returning some information on which the caller CAN/SHOULD take some action, using return codes is probably a better alternative. This includes all expected "errors", because probably they should be handled by the immediate caller, and will hardly need to be bubbled up too many levels up in the stack.


### sum up

To sum it up:

- I like to use exceptions when I have an unexpected situation, in which there's not much to do, and usually we want to abort a large block of code or even the whole operation or program. This is like the old "on error goto".

- I like to use return codes when I have expected situations in which the caller code can/should take some action. This includes most business methods, APIs, validations, and so on.

### Unchecked Exceptions 的问题

首先，啥是 unchecked exception？

A quick recap. In an unchecked exceptions model, **you throw and catch exceptions, without it being part of the type system or a function’s signature**.

```java
// Foo throws an unhandled exception:
void Foo() {
    throw new Exception(...);
}

// Bar calls Foo, and handles that exception:
void Bar() {
    try {
        Foo();
    }
    catch (Exception e) {
        // Handle the error.
    }
}

// Baz also calls Foo, but does not handle that exception:
void Baz() {
    Foo(); // Let the error escape to our callers.
}
```

问题在哪？

In this model, **any function call – and sometimes any statement – can throw an exception, transferring control non-locally somewhere else. Where? Who knows.** There are no annotations or type system artifacts to guide your analysis. As a result, it’s difficult for anyone to reason about a program’s state at the time of the throw, the state changes that occur while that exception is propagated up the call stack – and possibly across threads in a concurrent program – and the resulting state by the time it gets caught or goes unhandled. 任何地方都有可能抛出exception，使得对于代码的分析变得非常困难。（我理解，这是因为没有限制，所以大家可以随心所欲的写，因而也非常难于理解。若大家在写的时候都遵循一定的规则，理论上也是可以写出易懂的代码，但每个人的水平不一样，对待代码的态度也不一样，在工程上没有限制的情况下，必然会造成“百花齐放”的现象）

非常经典的分析：为什么 exception 在业界如此流行？

> It’s of course possible to try. Doing so requires reading API documentation, doing manual audits of the code, leaning heavily on code reviews, and a healthy dose of luck. The language isn’t helping you out one bit here. Because failures are rare, this tends not to be as utterly disastrous as it sounds. My conclusion is that’s why many people in the industry think unchecked exceptions are “good enough.” They stay out of your way for the common success paths and, because most people don’t write robust error handling code in non-systems programs, throwing an exception usually gets you out of a pickle fast. Catching and then proceeding often works too. No harm, no foul. Statistically speaking, programs “work.”

exception 的通病（不管是checked还是unchecked）

> Another thing most exception systems get wrong is encouraging too coarse a granularity of handling errors. Proponents of return codes love that error handling is localized to a specific function call. (I do too!) In exception handling systems, it’s all too easy to slap a coarse-grained try/catch block around some huge hunk of code, without carefully reacting to individual failures. That produces brittle code that’s almost certainly wrong; if not today, then after the inevitable refactoring that will occur down the road. A lot of this has to do with having the right syntaxes. 一般人通常做法是一个`try ... catch` wrap 住一大段代码，简单是简单，但一般会埋坑

参考：

- [Conventions for exceptions or error codes](https://stackoverflow.com/questions/253314/conventions-for-exceptions-or-error-codes)
- [The Error Model](http://joeduffyblog.com/2016/02/07/the-error-model/)
