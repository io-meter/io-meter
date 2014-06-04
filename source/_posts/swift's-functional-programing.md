title: Swift の 函数式编程
date: 2014-06-04 22:59:16
tags: ['Functional Programing', 'Swift', 'Apple']
category: essay
---

Swift 相比原先的 Objective-C 最重要的优点之一，就是对函数式编程提供了更好的支持。
Swift 提供了更多的语法糖和一些新特性来增强函数式编程的能力，本文就在这方面进行一些讨论。

<!-- more -->

##Swift 概览

对编程语言有了一些经验的程序员，尤其是那些对多种不同类型的编程语言都有经验的开发者，
在学习新的语言的时候更加得心应手。原因在于编程语言本身也是有各种范式的，
把握住这些特点就可以比较容易的上手了。

在入手一门新的语言的时候，一般关注的内容有：

1. 原生数据结构
2. 运算符
3. 分支控制
4. 如果是面向对象的编程语言，其面向对象的实现是怎样的
5. 如果是函数式编程语言，其面向函数式编程的实现是怎样的

通过这几个点，其实只要阅读 Swift 文档的第一章，你就可以对这个语言有一个大概的印象。
比如对于数据结构，Swift 和其他的编程语言大体一样，有 Int, Float, Array, Dictionary 等，
运算符也基本与 C 语言一致等。
本文主要集中于对 Swift 函数式编程方面的特点进行一些盘点，因此在这里假设大家对 Swift 的基本语法已经有所了解。

对于一种编程范式，要掌握它也要抓住一些要点。对于支持函数式编程的语言，其一般的特点可能包含以下几种：

1. 支持递归
2. 函数本身是语言 First Class 的组成要素，且支持高阶函数和闭包
3. 函数调用尽可能没有副作用
([Side Effect](http://en.wikipedia.org/wiki/Side_effect_%28computer_science%29))的条件

接下来我们来逐个盘点这些内容。

# 递归

Swift 是支持递归的，事实上现在不支持递归的编程语言已经很难找到了。在 
Swift 里写一个递归调用和其他编程语言并没有什么区别：

```none
func fib(n: Int) -> Int {
  if n <= 1 {
    return 1
  }
  else {
    return fib(n-1) + fib(n-2)
  }
}
fib(6) // output 13
```

关于 Swift 的递归没有什么好说的。作为一个常识，我们知道递归是需要消耗栈空间的。
在函数式编程语言中，递归是一个非常常用的方法，然而使用不慎很容易导致栈溢出的问题。
如果将代码改写为非递归实现，又可能会导致代码的可读性变差，因此有一个技巧是使用“尾递归”，
然后让编译器来优化代码。

一个 Common Lisp 的尾递归的例子是

```lisp
(defun fib(n)
    (fib-iter 1 0 n))

(defun fib-iter(a b count)
    (if (= count 0)
        b
        (fib-iter (+ a b) a (- count 1))))
```

我们可以把我们上述的 Swift 代码也改写成相同形式

```none
func fibiter(a: Int, b: Int, count: Int) -> Int {
  if count==0 {
    return b
  }
  else {
    return fibiter(a + b, a, count-1)
  }
}

func fib(n: Int) -> Int {
  return fibiter(1, 1, n);
}
```

我们可以 Playground 里观察是否使用尾递归时的迭代结果变化。

![](/img/posts/recurrence-fib.png)

值得注意的是，这里出现了一个 Swift 的问题。虽然 Swift 支持嵌套函数，但是当我们将`fibiter`
作为一个高阶函数包含在`fib`函数之内的时候却发生了 EXC\_BAD\_ACCESS 报错，
并不清楚这是语言限制还是 Bug。

## Swift 的高阶函数和闭包

在 Objective-C 时代，使用 block 来实现高阶函数或者闭包已经是非常成熟的技术了。
Swift 相比 Objective-C 的提高在于为函数式编程添加了诸多语法上的方便。

首先是高阶函数的支持，可以在函数内定义函数，下面就是一个很简洁的例子。

```none
func greetingGenerator(object:String) -> (greeting:String) -> String {
  func sayGreeting(greeting:String) -> String {
    return greeting + ", " + object
  }
  return sayGreeting
}

let sayToWorld = greetingGenerator("world")
sayToWorld(greeting: "Hello") // "Hello, World"
sayToWorld(greeting: "你好") // "你好, World"
```

如果使用 block 实现上述功能，可读性就不会有这么好。而且 block 的语法本身也比较怪异，
之前没少被人吐槽。Swift 从这个角度来看比较方便。事实上，在 Swift 里可以将函数当做对象赋值，
这和很多函数式编程语言是一样的。

作为一盘大杂烩，Swift 的函数系统也很有 JavaScript 的影子在里面。比如可以向下面这样定义函数：

```none
let add = {
  (a:Int, b:Int) -> Int in
  return a+b
}

add(1, 2) // 3
```

等号之后被赋予变量`add`的是一个闭包表达式，因此更准确的说，
这是将一个闭包赋值给常量了。注意在闭包表达式中，`in`关键字之前是闭包的形式定义，之后是具体代码实现。
Swift 中的闭包跟匿名函数没有什么区别。
如果你将它赋值给对象，就跟 JavaScript 中相同的实践是一样的了。幸好 Swift 作为 C 系列的语言，
其分支语句 if 等本身是有作用域的，因此不会出现下列 JavaScript 的坑：

```javascript
if (someNum>0) {
  function a(){ alert("one") };
}
else {
  function a(){ alert("two") };
}

a() // will always alert "two" in most of browsers
```

Swift 的闭包表达式和函数都可以作为函数的参数，从下面的代码我们可以看出闭包和函数的一致性：

```none
func function() {
  println("this is a function")
}

let closure = {
  () -> () in
  println("this is a closure")
}

func run(somethingCanRun:()-> ()) {
  somethingCanRun()
}

run(function)
run(closure)
```

类似于 Ruby，Swift 作为函数参数的闭包做了一点语法糖。
在 Ruby 中使用 Block 的时候，我们可以这样写:
```ruby
(1...5).map {|x| x*2} // => [2, 4, 6, 8]
```

在 Swift 当中我们可以得到几乎一样的表达式。
```none
var a = Array(1..5).map {x in x*2}
// a = [2, 4, 6, 8]
```
也就是说， 如果一个函数的最后一个参数是闭包，那么它在语法上可以放在函数调用的外面。
闭包还可以用`$0`、`$1`等分别来表示第0、第1个参数等。 基本的运算符也可以看做函数。
下面的几种方式都可以实现逆序倒排的功能。

```none
let thingsToSort = Array(1..5)
var reversed1 = sort(thingsToSort) { a, b in a<b}
var reversed2 = sort(thingsToSort) { $0 < $1}
var reversed3 = sort(thingsToSort, <) // operator as a function
// all the above are [5, 4, 3, 2, 1]
```

总体来说，Swift 在添加方便函数操作、添加相关语法糖方面走的很远，基本上整合了目前各种语言中比较方便的特性。
实用性较好。

## Side Effects

在计算机科学中，函数副作用指当调用函数时，除了返回函数值之外，还对主调用函数产生附加的影响。例如修改全局变量
(函数外的变量)或修改参数([wiki](http://en.wikipedia.org/wiki/Side_effect_%28computer_science%29))。
函数副作用会给程序带来一些不必要的麻烦。

为了减少函数副作用，很多函数式编程语言都力求达到所谓的“纯函数”。
纯函数是指函数与外界交换数据的唯一渠道是参数和返回值， 而不会受到函数的外部变量的干扰。
乍看起来这似乎跟闭包的概念相抵触，因为闭包本身的一个重要特点就是可以访问到函数定义时的上下文环境。

事实上，为了在这种情况下支持纯函数，一些编程语言如 Clojure 等提供的数据结构都是不可变(或者说 Persist)的。
因此其实也就没有我们传统意义上的所认为的“变量”的概念。比如说，在 Python 中，字符串`str`就是一类不可变的数据结构。
你不能在原来的字符串上进行修改，每次想要进行类似的操作，其实都是生成了一个新的`str`对象。
然而 Python 中的链表结构则是可变的。且看下面的代码，在 Python 中对`a`字符串进行修改并不会影响`b`，
但是同样的操作作用于链表就会产生不一样的结果：
```python
a = "hello, "
b = a
a += "world"
print a # hello, world
print b # hello, 
```

Swift 的数据结构的 Persist 性质跟 Python 有点类似。需要注意的是，Swift 有变量和常量两种概念，
变量使用`var`声明，常量使用`let`声明，使用`var`声明的时候，Swift 中的字符串的行为跟 Python 相似，
因此修改字符串可以被理解为生成了一个新的字符串并修改了指针。同样，
使用`var`声明的数组和字典也都是可变的。

在 Swift 中使用`let`声明的对象不能被赋值，基本数据结果也会变得不可变，但是情况更复杂一点。

```none
let aDict = ["k1":"v1"]
let anArray = [1, 2, 3, 4]

aDict["k1"] = "newVal" // !! will fail !!
anArray.append(5) // !! will fail !!
anArray[0] = 5 // anArray = [5, 2, 3, 4] now !
```
从上面的代码中可以看出，使用`let`声明的字典是完全不可变的，但是数组虽然不可以改变长度，
却可以改变数组元素的值！Swift 的文档中指出这里其实是将 Array 理解为定长数组从而方便编译优化，
来获得更好的访问性能。

综上所述，对象是否可变的关系其实略有复杂的，可以总结为：

1. 使用`var`和`let`，`Int`和`String`类型都是不可变的，但是`var`时可以对变量重新赋值
2. 使用`let`声明的常量不可以被重新赋值
3. 使用`let`声明的`Dictionary`是完全不可变的
4. 使用`let`声明的`Array`长度不可变，但是可以修改元素的值
5. 使用`let`声明的类对象是可变的

综上所述，即使是使用`let`声明的对象也有可能可变，因此在多线程情况下就无法达到“无副作用”的要求了。

此外 Swift 的函数虽然没有指针，但是仍通过参数来修改变量的。只要在函数的参数定义中加入`inout`关键字即可。
这个特性很有 C 的风格。

个人觉得在支持通过元组来实现多返回值的情况下，这个特性不但显得鸡肋，也是一个导致程序产生“副作用”的特性。
Swift 支持这样的特性，恐怕更多的是为了兼容 Objective-C 以及方便在两个语言之间搭建 Bridge。

```none
func inc(inout a:Int) {
  a += 1
}
var num = 1
inc(&num) // num = 2 now!
```

综上所述，使用 Swift 自带的数据结构并不能很好的实现“无副作用”的“纯函数式”编程，
它并没有比 Python、Ruby 这类语言走的更远。幸好作为一种关注度很高的语言，
已经有开发者为其实现了一套完全满足不可变要求的数据结构和库：[Swiftz](https://github.com/maxpow4h/swiftz)。
坚持使用`let`和 Swiftz 提供的数据结构来操作，就可以实现“纯函数式”编程。

## 总结

在我看来，Swift 虽然实现了很多其他语言的亮点特性，但是总体实现来说并不是很整齐。
它在函数式编程方面添加了很多特性，但在控制副作用方面仅能达到平均水准。
有些特性看起来像是为了兼容原来的 Objective-C 才加入的。

Swift 写起来相对比 Objective-C 更方便一点，脱离 Xcode 这样的 IDE 来写也是应该是可以的。
目前 Swift 只支持集中少量的原生数据结构而没有标准库，更不具备跨平台特性，这是一个缺点。
在仔细阅读了文档之后发现 Swift 本身的语法细节还是很多的，就比如`switch`分置语句的用法就有很多内容。
入门学习的容易程度并没有原来想象的那么好。我个人并不觉得这门语言会对其他平台的开发者有很大吸引力。

Swift 是一门很强大的语言，在其稳定版本发布之后我认为我会从 Objective-C 专向 Swift 来进行编程，
它在未来很可能成为 iOS 和 Mac 开发的首选。
