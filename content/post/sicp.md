+++
title = "Structure and Interpretation of Computer Programs"
date = "2014-07-26"
slug = "2014/07/26/sicp"
Categories = ["python", "programming"]
+++
大名鼎鼎的SICP，下面是我在学习Berkeley的CS61A时做的笔记,非常有意思的一门课！
## 高阶函数
理解这玩意儿有时候确实挺费脑筋。。。下面这个trick花了好久才理解。。关键一开始没想明白返回的到底是匿名函数还是匿名函数的执行结果    
	def cons(a, b):
		return lambda m: m(a,b) # 此处返回的是一个参数为m的匿名函数！！并且m是一个函数，m接受两个参数a和b

	def car(p):
		return p(lambda x,y:x) # 

	car(cons(1,2)) # 结果应该为1
以上代码大概类似于下面的代码：    
	def fuck(func):
		def wrapper(a,b):
			return func(a,b)
		return wrapper

	x = fuck(lambda x,y:x)
	x(1,2) # 输出1
## 递归  破钱
下面的函数实现的功能是，给定一个数值，比方说100块，和一堆小面额的纸币，找出用这些小面额纸币来表示这100块总共有几种方法。其中link函数是用来制作链表的，first函数返回链表的第一个元素，rest函数返回链表的剩余所有元素。可以看到函数是用递归实现的，依据大概是这样的：100块的表示方法可以分为两类，一类方法中使用链表中的第一个面额，另一类方法中不使用第一个面额。就这么简单。。。神奇的是它竟然是能工作的！
	def count_change(amount, denominations):
		'''
		Return the number of ways to make change for amount
		'''
		if amount < 0 or denominations == empty:
			return 0
		elif amount == 0:
			return 1
		using_coin = count_change(amount - first(denominations), denominations)
		not_using_coin = count_change(amount, rest(denominations))
		return using_coin + not_using_coin

	denominations = link(50, link(25, link(10, link(5, link(1, empty)))))
	>>> print_linked_list(denominations)
	< 50 25 10 5 1 >
	>>> count_change(100, denominations)
	292
## ADT(Abstract Data type)
ADT也属于抽象的一种，目的是控制复杂度，把复杂并且与下面要实现的功能无关的细节隐藏起来，只把几个关键的东西暴露给外部，要形成一个ADT一般需要一个constructor，一个selector，一个mutator等，有了这些，外部就可以用它们进一步实现别的功能了，像链表啊，树啊都属于ADT。其中，deep list和tree属于hierarchical data structure
## Implicit Sequences 
a sequence can be represented **without** each element being stored explicitly in the memory of the computer. That is, we can construct an object that provides access to all of the elements of some sequential dataset without computing all of those elements in advance and storing them. Instead, we compute elements on demand. 很巧妙的一种实现方式，不是把元素全部计算出来放在内存里等着去取，而是需要的时候才计算一个出来，这样可以节省大量的内存。这在计算机科学中叫lazy computation(lazy computation describes any program that delays the computation of a value until that value is needed). 但注意，实现这个强大功能的前提是sequential access(相对于random access)，也就是说只能按顺序依次获取，不能想取哪个就取哪个。嗯，iterator就是在这个背景下诞生了。。An iterator is an object that provides **sequential access** to an underlying sequential dataset. iterator就是为了方便处理那些具有内在顺序结构的数据而诞生的。The iterator abstraction has two components: a mechanism for retrieving the next element in some underlying series of elements and a mechanism for signaling that the end of the series has been reached and no further elements remain. In programming languages with built-in object systems, this abstraction typically corresponds to a particular interface that can be implemented by classes.     

- iterators与iterables    

原来这两个不是一个东西。iterators需要实现`__next__`方法，是具体干活的。     
An object is iterable if it **returns an iterator** when its `__iter__` method is invoked. Iterable values represent data collections, and they provide a fixed representation that may produce more than one iterator. iterator和iterable类似于类和实例的关系吧,iterator负责实现模型、架构，iterable负责实例化。For example, an instance of the `Letters` class below represents a sequence of consecutive letters. Each time its `__iter__` method is invoked, a new `LetterIter` instance is constructed, which allows for sequential access to the contents of the sequence.     
	>>> class LetterIter:
		"""An iterator over letters of the alphabet in ASCII order."""
		def __init__(self, start='a', end='e'):
		    self.next_letter = start
		    self.end = end
		def __next__(self):
		    if self.next_letter == self.end:
			raise StopIteration
		    letter = self.next_letter
		    self.next_letter = chr(ord(letter)+1)
		    return letter

	>>> class Letters:
		def __init__(self, start='a', end='e'):
		    self.start = start
		    self.end = end
		def __iter__(self):
		    return LetterIter(self.start, self.end)

	>>> b_to_k = Letters('b', 'k')
	>>> first_iterator = b_to_k.__iter__()
	>>> next(first_iterator)
	'b'
	>>> next(first_iterator)
	'c'
	>>> second_iterator = iter(b_to_k)
	>>> second_iterator.__next__()
	'b'
	>>> first_iterator.__next__()
	'd'
	>>> first_iterator.__next__()
	'e'
	>>> second_iterator.__next__()
	'c'
	>>> second_iterator.__next__()
	'd'

但是！在Python中`__next__`和`__iter__`方法是在同一个类中的！To use an iterator in a for loop, the iterator must also have an `__iter__` method. The [Iterator types](http://docs.python.org/3/library/stdtypes.html#iterator-types) section of the Python docs suggest that an iterator have an `__iter__` method that returns the iterator itself, so that all iterators are iterable. 只要理解了`__iter__`方法的作用，也很容易理解的啦。

- Generators and Yield Statements

Generators属于Iterator，a generator is an **iterator** returned by a special class of function called a generator function. Generator functions are distinguished from regular functions in that rather than containing return statements in their body, they use yield statement to return elements of a series.

The Letters and Positives objects above require us to introduce a new field self.current into our object to keep track of progress through the sequence. With simple sequences like those shown above, this can be done easily. With complex sequences, however, it can be quite difficult for the `__next__` method to save its place in the calculation. Generators allow us to define more complicated iterations by leveraging the features of the Python interpreter. 

Generators do not use attributes of an object to track their progress through a series. Instead, they control the execution of the generator function, which runs until the next yield statement is executed each time the generator's `__next__` method is invoked. The Letters iterator can be implemented much more compactly using a generator function. 虽然实现的功能是一样的，但Generators与iterators的实现机理是不一样的，yield的机理貌似是coroutines，到底是个神马东东，待查。

	>>> def letters_generator():
		current = 'a'
		while current <= 'd':
		    yield current
		    current = chr(ord(current)+1)

	>>> for letter in letters_generator():
		print(letter)
	a
	b
	c
	d

Even though we never explicitly defined `__iter__` or `__next__` methods, the yield statement indicates that we are defining a generator function. When called, a generator function doesn't return a particular yielded value, but instead a generator (which is a type of iterator) that itself can return the yielded values. A generator object has `__iter__` and `__next__` methods, and each call to `__next__` continues execution of the generator function from wherever it left off previously until another yield statement is executed.

The first time `__next__` is called, the program executes statements from the body of the `letters_generator` function until it encounters the yield statement. Then, it pauses and returns the value of current. yield statements do not destroy the newly created environment, they preserve it for later. When `__next__` is called again, execution resumes where it left off. The values of current and of any other bound names in the scope of `letters_generator` are preserved across subsequent calls to `__next__`.

这样，可以用yield语句来简化iterables的写法
	>>> class LettersWithYield:
		def __init__(self, start='a', end='e'):
		    self.start = start
		    self.end = end
		def __iter__(self):
		    next_letter = self.start
		    while next_letter < self.end:
			yield next_letter
			next_letter = chr(ord(next_letter)+1)

以前没有Generators的时候，还需要定义一个`__next__`方法来生成iterator，然后用`__iter__`方法来返回self，这下因为yield本身返回的就是一个Generator(也属于Iterator)，所以无需`__next__`方法来生成iterator。
## Mutation in for loops
如果在for循环中对对象做了修改（增加或删除），会出现无法预测的结果
	# Demo: Mutation in For Loops
	# lst = list(range(10, 20))
	# for item in lst:
	#     print(item)
	#     lst.remove(item)
	# lst
	# 下面是解决办法, 不过原理未搞懂，待查
	class BetterList(list):
	    def __iter__(self):
		return list.__iter__(self[:])

## nonlocal与list
看下面两段代码     
	def make_accumulator():
	    """Return an accumulator function that takes a single numeric argument and
	    accumulates that argument into total, then returns total.
	    """
	    total = []
	    def accumulator(amount):
		total.append(amount)
		return sum(total)
	    return accumulator

	def make_accumulator_nonlocal():
	    """Return an accumulator function that takes a single numeric argument and
	    accumulates that argument into total, then returns total.
	    """
	    total = 0
	    def accumulator(amount):
		nonlocal total
		total += amount
		return total
	    return accumulator
为何单个变量(total)在"全局"访问的时候需要nonlocal参数，而列表就不用了呢？
## format method of strings
	>>> 'I {0}, you {0}, we all {0} for {1}!'.format('scream', 'ice cream')
	'I scream, you scream, we all scream for ice cream!'
## Generic Functions
Generic functions are methods or functions that apply to arguments of different types. 我理解就是不管对象类型是不是一样，都可以用这个通用函数来做运算。Generic functions主要有下面三种实现方式：
- Interfaces
- Type dispatching
- Coercion
## Memoization
Keep track of all the calls we have ever made to fib and their answers. Then when we made a repeated recursive call, we immediately return the answer.
## Huffman Encoding
一种可变编码方式，可针对词频进行优化编码，出现次数多的字母用相对小的位数来编码，出现次数多的字母用相对多的位数来编码，这样相对于固定位编码方式可节省存储空间。    
使用方法：   

- 统计字母出现频率，并据此创建霍夫曼二叉树。创建的方法就是对给定的字母、频率，不停的merge掉频率最小的两个，并根据这两个生成一个新的HuffmanTree，它的频率为那两个频率最小的频率之和，以此直至只剩下两个HuffmanTree为止.
- 编码
- 解码
编码和解码部分可用递归优雅的实现
## Scheme
- 区分symbol和string. When you type things in the interpreter, Scheme will evaluate it. The rule for evaluating a symbol is to get the value bound to that symbol. This is one difference between strings and symbols--symbols don't evaluate to themselves(which strings do). However, as you saw above, when you type in 'a, you get a. This is because when you use single quote, you're telling Scheme **not to follow** the normal rules of the evaluation and just have the symbol return as itself.
- Special Forms. 类似于其它语言中的关键字，比如define if and or等等
- Lambdas. 语法： `(lambda (<PARAMETERS>) <EXPR>)`
- Functions. 语法：`(define (<FUNCTIONNAME> <PARAMETERS>) <EXPR>)`. Python will automatically transform it to `(define <FUNCTIONNAME> (lambda (<PARAMETERS>) <EXPR>)` for you.所以在Scheme中，lambda更基础.
- 关键字let. 语法：
	(let ( (<SYMBOL1> <EXPR1>)
		...
		(<SYMBOLN> <EXPRN>) )
		<BODY> )
它等价于：`( (lambda (<SYMBOL1> ... <SYMBOLN>) <BODY>) <EXPR1> ... <EXPRN>)`
- Pairs. 创建：`(define a (cons 1 2))` 获取： `(car a)` `(cdr a)`
- List. A List is a specific kind of Pair that is either nil(an empty list) or a Pair whose cdr is another List. 比如，(cons 1 (cons 2 nil))，它可以简写成'(1 2)，检查List是否为空可用null?语句

## Tail Recursion 尾递归
是对递归的一种优化，我们都知道，一般线性递归在新的调用之前需要在堆栈中保存当前调用的信息，这样随着函数调用的增多，对内存的占用会越来越大，而尾递归能够实现这一次的调用使用的堆栈空间直接覆盖上次用过的，因此虽然函数调用在不断进行，但对内存的占用却是不变的！（linuxfish注：看了一堆介绍，我肿么觉得尾递归有点过程语言中循环的感觉，是不是因为函数式编程中没有循环，才用尾递归的？）  

注意，要区分尾递归语法和尾递归优化，也就是说，你可以把递归写成尾递归的形式，但底层（解释器、编译器）是否给你优化就不一定了。比如，Python可以写成尾递归的形式，但是没用，照样会stackoverflow，因为解释器并不支持。
## Streams
这应该是Python中generator的鼻祖了吧。Streams are a clever idea that allows one to use sequence manipulations without incurring the costs of manipulating sequences as list. With streams we can achieve the best of both worlds: We can formulate programs elegantly as sequences manipulations, while attaining the efficiency of **incremental computation**. The basic idea is to arrange to construct a stream only partially, and to pass the partial construction to the program that consumes the stream. If the consumer attempts to access a part of the stream that has not yet been constructed, the stream will automatically construct just enough more of itself to produce the required part, thus preserving the illusion that the entire stream exists. In other words. In other words, although we will write programs as if we were processing complete sequences, we design our stream implemention to automatically and transparently interleave the construction of the stream with its use.感觉这玩意儿纯粹是Schemer们妄图所有问题都用list pair来解决，发现不行后又整出来的一个东东。这也是所谓的函数式语言的懒惰特性，所谓懒就是不到万不得已不动弹。。而不是傻乎乎的一下子就把整个list都计算出来。
## Declarative Programming
比如SQL语言

data values are often stored in large repositories called databases. A database consists of a data store containing structured data values and an interface for retrieving subsets of the data based on their characteristics. Each value stored in a database is called a record. Records are typically retrieved via a query, which is an expression in a query programming language. By far the most ubiquitous query language in use is called Structure Query Language or SQL.    

SQL is an example of a declarative programming language. Expressions do not **describe computations directly**, but instead state the form of the **result** of some computation. It is the role of the **query interpreter** of the database system to design and perform a computational process to produce such a result.     

This interaction differs substantially from the procedural programming paradigm of Python or Scheme. In Python, **computational processes are described directly by the programmer.** A declarative language specifies the form of the result, but abstracts away procedural details.
## Concurrency 并行
- 并行是一个现实问题，生活中经常会遇到，又是一个提升效率的好办法，并行办事比串行等待效率要高很多！
- 由于线程并行所导致的问题可以通过引入锁的机制来解决，不过要注意防止死锁现象。
- 还可以通过Message Passing来解决
