---
title: A Brief Intro to Functional Programming
date: 2017-05-29 15:52:34
tags: ['Functional Programming', 'ES6', 'React', 'JavaScript', 'Y-Combinator', 'Closure', 'Lazy Evaluation']
---

近期受邀在另一家公司的一个活动里做了一个 Tech Talk 介绍 Functional Programming，
在准备 Slides 的过程中又重新审视了一下自己对函数式编程这个主题的了解深度，
感觉又有一些收获。

<!--more-->

这次趁着端午假期的空闲，将讲座的内容尤其是 slides 当中使用的一些容易理解的配图整理出来，
形成这么一篇博客，希望能对读者有所帮助。

# 概览

一开始让我去讲函数式编程，其实我是拒绝的。老实说，科技界的很多术语的内涵其实一直比较模糊。
就拿函数式编程来说，它到底是什么、涵盖了多少内容其实不是一件容易界定的事情。况且，
函数式编程尽管已经有多年的发展历史，普通的程序员们对它的认识却还是参差不齐或深有误解。

其中可能是最大的误解之一可能就是认为函数式编程主要是前端届在使用的范式，在其它领域没有用武之地。
就连邀请我进行讲座的主办方，也是以这样一种认知为基础，想要我在前端的语境下来准备主题。
这是我使用 ES6 作为讲座中使用的编程语言的主要原因。尽管如此，
我准备的内容绝大部分其实跟前端技术的联系并不紧密，主要的内容有两个：

1. 函数式编程的一些“奇技淫巧”，主要是闭包、惰性求值和 Y-Combinator
2. 根据关于函数式编程在网络上最新的一些讨论，尤其是 Haoyi Li 老师的
《[What’s functional programming all about](http://www.lihaoyi.com/post/WhatsFunctionalProgrammingAllAbout.html)》
这篇文章，浅谈了一些关于函数式编程本质的东西

在我准备的讲座当中还有一个有关 FP 在 React 当中体现的章节，因个人觉得不是特别重要，
在这篇文章当中就不赘述了。

# Closure

Closure，也就是闭包，可以说是函数式编程基础当中的基础。闭包的存在体现了函数式编程的几个重要特点：
函数是编程语言的一等成员以及高阶函数的存在。关于闭包，在圈内有两句著名的格言：

    Objects are a poor man's closures

以及

    Closures are a poor man's objects

这两句话据信都出自 MIT 早期的邮件列表当中一个著名的 Thread：
《[What's so cool about Scheme?](http://people.csail.mit.edu/gregs/ll1-discuss-archive-html/msg03277.html)》。
肤浅地理解的话，这两句加起来其实表达了闭包和对象两种模型在某种程度上是同一的。

我们来看一下闭包是如何实现对象可以完成的工作的。一个最简单的例子可能是使用闭包来实现链表：

```javascript
let getNode = (value , next) => {
    return (x) => x ? value : next;
}
let value = (node) => node(true);
let next = (node) => node(false); 
```

`getNode` 函数返回了一个闭包，我们知道，闭包是一个保存了上层函数调用时的环境上下文的函数。
因此，我们传给`getNode`函数的两个参数`value`和`next`会被保存在闭包中。如下图所示:

![Closure as a node](/img/fp/closure-as-nodes.png)

比如 C 的 `struct` 和 Java 的 `class` 中，获得一个对象的成员变量，往往是取内存地址的操作，
然而在使用闭包作为对象的时候，因为闭包本身就是个函数，我们是通过使用不同的参数调用闭包来获得不同的字段的。
譬如在这个例子里，传给闭包 `true`，函数会返回 `value`，反之则会返回 `next`。下图展示了这个过程：

![Closure: get value and next](/img/fp/closures-get-value.png) 

可以使用闭包获得值和指向下一个节点的指针之后，这个闭包就跟普通的链表节点没什么不同了。
我们可以像使用结构体那样使用它，只不过在取值和指针的时候是通过函数调用而不是直接的取字段的方法来完成的。

```javascript
let a = getNode(1, getNode(2, getNode(3, null)));
/*  1 -> 2 -> 3 -> null  */

value(a); // 1

let b = next(a); value(b); // 2

let c = next(b); value(c); // 3
next(c) // null
```

一个更复杂的例子是实现链表反转，我们首先实现了一个 `append` 函数用来将一个节点追加到一个链表当中，
之后的 `reverse` 函数其实就是通过不断对自己递归地调用而完成工作的——尽管从时间复杂度上来看这个解决方案并不怎么有效，
然而它却是在完全没有对象系统和循环的前提下而实现的。

```javascript
let append = (n, v) => {
  if (n === null) return getNode(v, null);
  else return getNode(value(n), append(next(n), v));
}

let reverse = (list) => {
  if (list === null) return null;
  else return append(reverse(next(list)), value(list));
}

reverse(a) // 3 -> 2 -> 1
```

其实，在很多 Lisp 里面的 `cons`、 `car`、 `cdr` 等函数，实际上就对应于上面例子当中的
`getNode`、 `value`、 `next` 三个函数。在一些实现里，更是直接用闭包来实现列表的。

# Lazy Evaluation

在上面链表的例子之后，我们可以来看一下惰性求值的实现方法。譬如说，
我们想得到一个表示自然数这一个无穷序列的闭包。那么可能可以有如下代码：

```javascript
let naturalNumber = (n) => {
  return (x) => x ? n : naturalNumber(n + 1);
}
```

表面上这个函数会无穷地递归自己，但是考虑到 `a ? b : c` 这样一个判断表达式当中，`b` 和 `c` 
都只有当对应的条件满足之后才会被编译器求解，因此函数并不会在调用的时候无限地递归调用自己。
要理解上述函数如何产生自然数的序列。我们先参照上一章的形式，展示我们所得到的闭包的结构。
如下图所示，可以看到只有一个字段`n`被保存在上下文环境中：

![Lazy Evaluation: Node](/img/fp/lazy-eval-node.png)

在取字段方面，使用 `true` 调用函数还是可以得到当前的值`n`，那么如果我们使用 `false`
来调用函数试图获得下一个节点会发生什么呢？

![Lazy Evaluation: get value and next](/img/fp/lazy-eval-get-value.png)

为了方便期间，我们将之前的整个闭包表示成一个小圆，把唯一的字段`n`填在小圆的中央。
下图展示了试图获取 `next` 的结果。可以看到，对包含字段`n` 的闭包取 `next` 会获得一个类似的闭包，
唯一的区别是字段的值变成了 `n + 1`。这个新的闭包实际上是对自己的递归调用，
也就是说，上述的函数通过*递归定义*的方式定义了自然数：

![Lazy Evaluation: Recusive Definition](/img/fp/lazy-eval-recursive-def.png)

这样，我们就得到了一个惰性求值的表，这个链表虽然表示了无穷多数的序列，
但是并不会直接耗尽内存——新的值只有在需要的时候才被求解出来。上面的例子同样体现了接口或者鸭子类型的思想。
对于原来链表的`value`和`next`函数，因为我们新的`naturalNumbers`会返回同样接口的闭包，
因此我们不需要对这个惰性求值的表定义新的函数来取值和后继节点。原来的函数是直接可用的。

为了体现惰性求值方式的威力，我们来用 ES6 快速实现很多函数式编程语言都会给出的一个案例：
[Sieve of Eratosthenes](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes)，
也就是素数的筛法。很多语言在给出的相关例子对于初学者来说可能很难理解，
在这篇文章的稍后部分会使用图示的方法来辅助说明到底发生了什么。现在还是先看代码：

```javascript
let nextMatch = (seq, fn) => {
  return fn(value(seq)) ? seq : nextMatch(next(seq), fn);
}

let filter = (seq, fn) => {
  let mseq = nextMatch(seq, fn);
  return (x) => x ? value(mseq) : filter(next(mseq), fn);
}

let sieve = (seq) => {
  let aPrime = value(seq);
  let nextSeq = filter(next(seq), (v) => v % aPrime != 0);

  return (x) => x ? aPrime : sieve(nextSeq);
}

```

在这里我们定义了三个方法，第一个函数找到当前序列当中第一个符合函数的匹配，
第二个函数将序列中不符合条件的元素过滤掉，最后一个函数就是我们的素数筛法实现。
可以看到，后两个方法采用了和 `naturalNumbers` 一样的惰性求值的方法。
他们会返回一个新的闭包，这个闭包服从我们之前的取值和取后继节点的接口，
因此我们可以像操作链表节点一样操作他们。

我们定义如下的`take`方法来从这样的一个惰性的表里取出若干的元素转换成普通链表，
这样我们就可以查看这些元素的内容。

```javascript
let take = (n, count) => {
  if (n == null || count <= 0) return null;
  else return getNode(value(n), take(next(n), count - 1));
}
```

像下面这样调用`sieve`函数，我们就得到了所有素数的一种表示，
使用`take`来查看前五个元素，就会得到前五个素数的值。

```
let primes = sieve(naturalNumber(2));
take(primes, 5); // 2, 3, 5, 7, 11
```

这到底是如何做到的呢？作为筛法，在这里是如何把之前筛出的素数保留下来从而再反过来用于筛选后面的素数的呢？
我们先来看函数第一次调用的时候的情况。因为我们将从 2 开始的自然数的惰性序列传给了 `sieve` 函数，
所以我们第一个得到的素数显然是 2:

![Primes: Get 2](/img/fp/lazy-eval-get-2.png)
